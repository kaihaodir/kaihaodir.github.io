---
layout: post
title: "NSTimer 使用注意事项"
date: 2016-03-21 23:13:50 +0800
comments: true
categories: 
---

NSTimer是ios上比较常用的定时器组件，在使用了一段时间后，发现有些地方是需要注意一下的。

1. NSTimer 是需要配合NSRunLoop 才可以正常工作的。

	``` 
	+ (NSTimer *)scheduledTimerWithTimeInterval:(NSTimeInterval)seconds
                                 invocation:(NSInvocation *)invocation
                                    repeats:(BOOL)repeats
                                    
    + (NSTimer *)scheduledTimerWithTimeInterval:(NSTimeInterval)ti 
    									  target:(id)aTarget 
    									selector:(SEL)aSelector 
    									userInfo:(nullable id)userInfo 
    									repeats:(BOOL)yesOrNo;
    ```
    使用这个类方法，会自动添加到当前的RunLoop里面。关于RunLoop的介绍网上有很多资料，推荐看看 [深入理解RunLoop](http://blog.ibireme.com/2015/05/18/runloop/)。
    <!--MORE-->
    
2. 	当RunLoop处于UITrackingRunLoopMode模式的时候（滑动UIScrollView的时候），使用 
	
	```
	scheduledTimerWithTimeInterval:(NSTimeInterval)seconds
                        invocation:(NSInvocation *)invocation
                           repeats:(BOOL)repeats
                           
    ```
    
    的类方法创建的Timer，是不会收到响应事件。只有RunLoop切换到Default模式时才可以正常响应。如果希望滑动时也可以响应Timer时间，需要把Timer加到RunLoop并指定模式为**NSRunLoopCommonModes**
   
    ```
    NSTimer *timer = [NSTimer timerWithTimeInterval:0.5 target:self selector:@selector(test) userInfo:nil repeats:YES];
    [[NSRunLoop mainRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
    
    ```
    
    
3. NSTimer 会强引用 target 对象，很容易造成内存泄露或者其它因生命周期和预期不一至导致的问题。
	
	我们先看一段常见的事例代码
	
	```
	@implementation TViewController
	{
	    NSTimer *_timer;
	}
	
	- (void)dealloc
	{
	    NSLog(@"%s", __func__);
	}
	
	- (void)viewDidLoad
	{
	    [super viewDidLoad];
	    _timer = [NSTimer scheduledTimerWithTimeInterval:1
	                                              target:self 
	                                            selector:@selector(onTimeout) 
	                                            userInfo:nil 
	                                             repeats:YES];
	}
	
	- (void)onTimeout
	{
	    NSLog(@"%s", __func__);
	}
	
	@end
	```
	大家可能会觉得，当这个ViewController被 pop 掉后会正常释放，timer 也会停掉。但实际的情况不是你想的那样。以下log是Push这个ViewController后，然后点击返回的过程。
	
	
	2016-03-24 00:42:19.663 NSTimerDemo[14916:3982566] -[TViewController onTimeout]
	2016-03-24 00:42:20.663 NSTimerDemo[14916:3982566] -[TViewController onTimeout]
	2016-03-24 00:42:21.663 NSTimerDemo[14916:3982566] -[TViewController onTimeout]
	2016-03-24 00:42:22.663 NSTimerDemo[14916:3982566] -[TViewController onTimeout]
	**2016-03-24 00:42:23.369 NSTimerDemo[14916:3982566] -[TViewController viewDidDisappear:]**
	2016-03-24 00:42:23.663 NSTimerDemo[14916:3982566] -[TViewController onTimeout]
	2016-03-24 00:42:24.663 NSTimerDemo[14916:3982566] -[TViewController onTimeout]
	2016-03-24 00:42:25.663 NSTimerDemo[14916:3982566] -[TViewController onTimeout]

	从日志上来看，dealloc方法确实没有执行，而且timer事件还一直在触发。
	
	OK,既然Timer强引用了ViewController，那把ViewController改成**__weak**不就是可以解决问题了？
	于是我们把创建Timer的代码改成
	
	```
	__weak typeof(self) weak_self = self;
    _timer = [NSTimer scheduledTimerWithTimeInterval:1
                                              target:weak_self
                                            selector:@selector(onTimeout)
                                            userInfo:nil
                                             repeats:YES];
     ```
	发现输出的log和之前的一样，难道weak对象根本没起作用？
	用Instrement查看了一下内存情况，发现真的是Timer强引用Target对象
	
	![Timer Retain Target](http://7xryar.com1.z0.glb.clouddn.com/NSTimer-Retain-Target.png)
	
	目前主要是处于一个闭环（环形引用）的状态，我们要想办法打破这种状态，而且**__weak**设置给Timer也不会破坏Timer强引用Target。
	
	于是，我们引用一个包装对象，让Timer强引用这个包装对象，包装对象弱引用Target(ViewController)
	ViewController ---> Timer --->Wrapper ...>ViewController 这样就可以破坏环形引用。
	
	```
	@Interface Wrapper
	@property (weak, nonatom) id target;
	@end
	```
	那么创建Timer的类方法的Target对象不是传```self```, 而是传 wrapper 对象。
	另外，wrapper对象还要把Timer的事件传递到真正的target上。
	
	详细的 Timer Wrapper 可以看完代码 [BSTimer](https://github.com/kaihaodir/BSUiKit/blob/master/BSTimer.h)
	
	
	最后其实可以用dispatch_time解决强引用问题，但是dispatch_time在暂停功能上处理起来比较麻烦。
	
	