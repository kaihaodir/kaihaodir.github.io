---
layout: post
title: "_delegateScrollViewAnimationEnded Crash处理"
date: 2016-03-30 00:57:31 +0800
comments: true
categories: ios crash
---

在项目开发过程中，多次遇到UIScrollView滑动时引起的崩溃，在这里分析一下，mark 下处理过程。

先看一下崩溃堆栈：

```
(lldb) bt all
* thread #1: tid = 0x4bb501, 0x00000001058451a8 CoreFoundation`___forwarding___ + 776, queue = 'com.apple.main-thread', stop reason = EXC_BAD_ACCESS  (code=EXC_I386_GPFLT)
    frame #0: 0x00000001058451a8 CoreFoundation`___forwarding___ + 776
    frame #1: 0x0000000105844e18 CoreFoundation`__forwarding_prep_0___ + 120
  * frame #2: 0x0000000105d6884d UIKit`-[UIScrollView(UIScrollViewInternal) _delegateScrollViewAnimationEnded] + 46
    frame #3: 0x0000000105d6894f UIKit`-[UIScrollView(UIScrollViewInternal) _scrollViewAnimationEnded:finished:] + 181
    frame #4: 0x0000000105dddab7 UIKit`-[UIAnimator stopAnimation:] + 395
    frame #5: 0x0000000105dde0bf UIKit`-[UIAnimator(Static) _advanceAnimationsOfType:withTimestamp:] + 234
    frame #6: 0x00000001095a5747 QuartzCore`CA::Display::DisplayLinkItem::dispatch() + 37
    frame #7: 0x00000001095a560f QuartzCore`CA::Display::DisplayLink::dispatch_items(unsigned long long, unsigned long long, unsigned long long) + 315
    frame #8: 0x000000010584df64 CoreFoundation`__CFRUNLOOP_IS_CALLING_OUT_TO_A_TIMER_CALLBACK_FUNCTION__ + 20
    frame #9: 0x000000010584db25 CoreFoundation`__CFRunLoopDoTimer + 1045
    frame #10: 0x0000000105810e5d CoreFoundation`__CFRunLoopRun + 1901
    frame #11: 0x0000000105810486 CoreFoundation`CFRunLoopRunSpecific + 470
    frame #12: 0x0000000108e799f0 GraphicsServices`GSEventRunModal + 161
    frame #13: 0x0000000105cd1420 UIKit`UIApplicationMain + 1282
    frame #14: 0x000000010504f32f UIScrollViewDelegate`main(argc=1, argv=0x00007fff5abb0360) + 111 at main.m:14
    frame #15: 0x0000000107e5e145 libdyld.dylib`start + 1
    
```
<!--MORE-->
从崩溃信息里面，我们可以看到 ** stop reason = EXC_BAD_ACCESS **， 非法地址访问（野指针）。从堆栈 **[UIAnimator stopAnimation:] -> [UIScrollView(UIScrollViewInternal) _scrollViewAnimationEnded:finished:] ** 可以看出是动画完成后，调用scrollview的方法引起的非法访问。经过实验验证，是scrollview.delegate 引起的。 **也就是说，scrollview在做动画的过程中，scrollview.delegate 被释放了。***

简单写了一些测试代码重现了这个崩溃

```
@implementation ScrollViewController

- (IBAction)onBackClick:(id)sender
{
    [self.scrollview setContentOffset:CGPointMake(0, self.scrollview.bounds.size.height * 3) animated:YES];
    [self.navigationController popViewControllerAnimated:NO];
}

```
* 在iOS7 和 iOS8 的系统下，一点返回按钮，pop 出当前页面，就会马上崩溃。
* 在iOS9下没有问题（由于属性修饰符改成weak）

```
//iOS9 以前
@property(nonatomic,unsafe_unretain) id<UIScrollViewDelegate> delegate; 

//iOS9
@property(nullable,nonatomic,weak) id<UIScrollViewDelegate> delegate; 

```

###解决方法：
崩溃的原因已经很明确，只要可以保证UIScrollView 的 delegate对象在释放的时候，把```scrollview.dlegate = nil;``` 就可以解决问题。

```
@implementation ScrollViewController
- (void)dealloc {
	self.scrollview.delegate = nil;
}
``` 

主动设置 scrollview.delegate = nil; 可以很好的解决崩溃。但是稍不注意，项目组的其他开发同事又很容易忘记，或者一不小心，又引起崩溃了。有没有办法可以一劳永逸呢？


###更好的解决方法：

主要思路：通过Runtime，修改 dealloc 方法，让代理对象在释放时自动把scrollview.delegate置空。

1. 首先，通过 **method swizzling** 给NSObject添加 deallocBlock [查看完成代码](https://github.com/kaihaodir/BSUiKit/blob/master/NSObject%2BDealloc.m)
	
	```
	typedef void (^DeallocCallback)();

	@interface NSObject(Deallocing)

	+ (void)hookNSObjectDealloc;

	- (void)setDeallocCallback:(DeallocCallback)callback;

	- (DeallocCallback)deallocCallback;

	@end
	
	
	
	@implementation NSObject(Deallocing)
	- (void)myselfDealloc {
    	DeallocCallback callback = [self deallocCallback];
    	if (callback) {
        	callback(); //对象释放前的主要操作
    	}
    
    [self originalDealloc];
	}
	@end

	```
	
2. 通过**method swizzling** 修改UIScrollView 的 **setDelegate** 方法

	```
	- (void)myselfSetDelegate:(NSObject *)delegate
	{
	    if (delegate) {
	        UIScrollView * __weak weak_self = (UIScrollView *)self;
	        [delegate setDeallocCallback:^{
	            weak_self.delegate = nil;
	        }];
	    }
	    //调用原来方法
	    [self originalSetDelegate:delegate];
	}	
	```
	
3. 在App初始化的时候，调用一下 ScrollView 的 swizzling 方法，就可以解决这类型的崩溃（包括：UITableView, UIWebView, UICollectionView 动画时delegate被释放引起的崩溃）
 

