---
layout: post
title: "Dealloc 时取 weak self 时崩溃 "
date: 2016-05-10 15:15:20 +0800
comments: true
categories: ios crash
---

今天无意这中遇到一个奇怪的崩溃，先上引起崩溃的代码：

```
- (void)dealloc
{
    __weak __typeof(self)weak_self = self;
    NSLog(@"%@", weak_self);
}

```

当执行到dealloc的时候，程序就crash 掉了。
<!--MORE--> 崩溃信息如下：

```
objc[4572]: Cannot form weak reference to instance (0x160f6f890) of class MFChatRoomBoardController. It is possible that this object was over-released, or is in the process of deallocation.
(lldb) 
error: empty command
(lldb) bt
* thread #1: tid = 0x35914d, 0x0000000182307aac libobjc.A.dylib`_objc_trap(), queue = 'com.apple.main-thread', stop reason = EXC_BREAKPOINT (code=1, subcode=0x182307aac)
  * frame #0: 0x0000000182307aac libobjc.A.dylib`_objc_trap()
    frame #1: 0x0000000182307b24 libobjc.A.dylib`_objc_fatal(char const*, ...) + 88
    frame #2: 0x0000000182319890 libobjc.A.dylib`weak_register_no_lock + 316
    frame #3: 0x0000000182320688 libobjc.A.dylib`objc_initWeak + 224
    frame #4: 0x000000010022bf8c MakeFriends`-[MFChatRoomBoardController dealloc](self=0x0000000160f6f890, _cmd="dealloc") + 36 at MFChatRoomBoardController.m:31


```

其中，可以在控制台明确看到这样一段描述:
> objc[4572]: Cannot form weak reference to instance (0x160f6f890) of class MFChatRoomBoardController. It is possible that this object was over-released, or is in the process of deallocation.

说明不允许在 dealloc 的时候取 weak self.

查看了一下 ``` weak_register_no_lock ``` 的函数代码，找到问题所在。

```
id 
weak_register_no_lock(weak_table_t *weak_table, id referent_id, id *referrer_id)
{
    objc_object *referent = (objc_object *)referent_id;
    objc_object **referrer = (objc_object **)referrer_id;

    if (!referent  ||  referent->isTaggedPointer()) return referent_id;

    // ensure that the referenced object is viable
    bool deallocating;
    if (!referent->ISA()->hasCustomRR()) {
        deallocating = referent->rootIsDeallocating();
    }
    else {
        BOOL (*allowsWeakReference)(objc_object *, SEL) = 
            (BOOL(*)(objc_object *, SEL))
            object_getMethodImplementation((id)referent, 
                                           SEL_allowsWeakReference);
        if ((IMP)allowsWeakReference == _objc_msgForward) {
            return nil;
        }
        deallocating =
            ! (*allowsWeakReference)(referent, SEL_allowsWeakReference);
    }

    if (deallocating) {
        _objc_fatal("Cannot form weak reference to instance (%p) of "
                    "class %s. It is possible that this object was "
                    "over-released, or is in the process of deallocation.",
                    (void*)referent, object_getClassName((id)referent));
    }

    // now remember it and where it is being stored
    weak_entry_t *entry;
    if ((entry = weak_entry_for_referent(weak_table, referent))) {
        append_referrer(entry, referrer);
    } 
    else {
        weak_entry_t new_entry;
        new_entry.referent = referent;
        new_entry.out_of_line = 0;
        new_entry.inline_referrers[0] = referrer;
        for (size_t i = 1; i < WEAK_INLINE_COUNT; i++) {
            new_entry.inline_referrers[i] = nil;
        }
        
        weak_grow_maybe(weak_table);
        weak_entry_insert(weak_table, &new_entry);
    }

    // Do not set *referrer. objc_storeWeak() requires that the 
    // value not change.

    return referent_id;
}

```

可以看出，runtime 是通过检查引用计数的个数来判断对象是否在 deallocting， 然后通过

```
    if (deallocating) {
        _objc_fatal("Cannot form weak reference to instance (%p) of "
                    "class %s. It is possible that this object was "
                    "over-released, or is in the process of deallocation.",
                    (void*)referent, object_getClassName((id)referent));
    }
```
这段代码让程序crash。


再看一下 _objc_fatal 这个函数

```
void _objc_fatal(const char *fmt, ...)
{
    va_list ap; 
    char *buf1;
    char *buf2;

    va_start(ap,fmt); 
    vasprintf(&buf1, fmt, ap);
    va_end (ap);

    asprintf(&buf2, "objc[%d]: %s\n", getpid(), buf1);
    _objc_syslog(buf2);
    _objc_crashlog(buf2);

    _objc_trap();
}
```

可以看到这个函数实际会在控制台输出一段信息，然后调用 _bojc_trap() 引起 crash. 而最后一个函数调用刚好也对上我们之前的崩溃堆栈。


