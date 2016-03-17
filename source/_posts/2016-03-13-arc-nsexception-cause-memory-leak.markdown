---
layout: post
title: "Arc下NSException有可能会内存泄露"
date: 2016-03-13 17:00:52 +0800
comments: true
categories: ios
---


在使用Instrumnents对程序进行内存测试的时候，发现有几处异常处理的地方提示内存泄露。经过分析得知，**因为异常处理，改变代码执行路径，导致编译生成的 release 代码没有执行**。

<!--more-->

简单写了些测试代码验证：

```
- (void)viewDidLoad {
    
    [super viewDidLoad];
    
    @try {
        NSMutableArray *stringList = [[NSMutableArray alloc] init];
        for (int i = 0; i < 100; i++) {
            [stringList addObject: @"string instance."];
        }
        [self throwException];
    }
    @catch (NSException *exception) {
        NSLog(@"catch Exception : %@", exception);
    }
    @finally {
        
    }
}

- (void)throwException
{
    [NSException raise:@"An Exception." format:@"Exception Description."];
}

```

反复进出这个页面几次后，在Instruments上果然看到提示内存泄露，而且泄露的地方就指明是 ```@try{ }``` 里面创建的对象。

![img](http://7xryar.com1.z0.glb.clouddn.com/exception-memory-leak.png)  

####解决思路
1, 尽量不要在@try{}里面有操作，导致对象的引用引用计数增加。```@try{ }``` 语句块里面只有有可能抛出异常的语句。

**修改后的代码:**

```
- (void)viewDidLoad {
    
    [super viewDidLoad];
    
    NSMutableArray *stringList = [[NSMutableArray alloc] init];
    for (int i = 0; i < 100; i++) {
        [stringList addObject: @"string instance."];
    }
    
    @try {
        [self throwException];
    }
    @catch (NSException *exception) {
        NSLog(@"catch Exception : %@", exception);
    }
    @finally {
        
    }
}

```
最后用Instruments检查一下，一切正常了。


2, 查看了苹果的开发文档，发现有这么一段描述:

>
	By default in Objective C, ARC is not exception-safe for normal releases:
	It does not end the lifetime of __strong variables when their scopes are abnormally terminated by an exception.
	It does not perform releases which would occur at the end of a full-expression if that full-expression throws an exception.
	A program may be compiled with the option -fobjc-arc-exceptions in order to enable these, or with the option -fno-objc-arc-exceptions to explicitly disable them, with the last such argument “winning”.


