---
layout:     post
title:      NSTimer 循环引用解决方案
subtitle: 
date:       2017-08-10
author:     潘世民
header-img:  img/2017-08-10/2017-08-10-background.jpeg
catalog: true
tags:
    - NSTimer
    - 循环引用
    - 内存泄露
---

#### 复现 NSTimer 循环引用

考虑这种场景：进入一个页面后，启动 timer；退出此页面后，timer 也销毁。
``` Objective-c
@implementation NSTimerViewController

- (void)dealloc {
    NSLog(@"NSTimerViewController delloc");
    [self.timer invalidate];
    self.timer = nil;
}

- (void)viewDidLoad {
    [super viewDidLoad];
    self.view.backgroundColor = UIColor.whiteColor;
    self.timer = [NSTimer timerWithTimeInterval:0.3 target:self selector:@selector(handleTimerCallback) userInfo:nil repeats:true];
    
    [[NSRunLoop currentRunLoop] addTimer:self.timer forMode:NSRunLoopCommonModes];
}

- (void)viewDidAppear:(BOOL)animated {
    [super viewDidAppear:animated];
    [self.timer fire];
}

- (void)handleTimerCallback {
    NSLog(@"timer called ...");
}

```
但是当我们从 NSTimerTestViewController 退出时，控制台还是不断地输出 log `"timer called ..."`；而且 `dealloc` 函数也没有调用；基本上可以断定 NSTimerTestViewController 页面发生了内存泄露。

#### 原因分析
在上面的代码中 NSTimerTestViewController 强引用了 timer，同时 timer 通过 target 也强引用了
![target_img](/img/2017-08-10/nstimer_target_ee.jpg)
NSTimerTestViewController 这样就形成了循环引用；
由于 timer 强引用了 NSTimerTestViewController，所以即使从 NSTimerTestViewController 页面退出后，其引用计数也大于 0，导致 NSTimerTestViewController 的 deinit 函数不会执行，因此 deinit 里的 timer?.invalidate() 也就无法执行了。

#### 方法一 使用 iOS 10 新添加的接口
在 iOS 10 及以上的项目中，可使用 NSTimer 新增的 block 范式的方法，只要确保 block 内没有循环引用即可
``` swift
timer = Timer(timeInterval: 0.3, repeats: true) { [weak self] timer in
    guard let self = self else { return }
    print("timer called ...")
}
```

#### 方法二 使用 NSProxy 做中间件
使用 NSProxy 当做中间件，并利用 NSProxy 的 methodSignatureForSelector:方法、forwardInvocation: 方法进行消息转发
1、把 NSTimer 的 target 设置成 NSProxy；
2、NSProxy 弱引用 NSTimerViewController，同时把 NSTimer 的定时任务转发给 NSTimerViewController；

``` Objective-c
@interface TTTimerProxy ()

@property (nonatomic, weak, nullable) id target;

@end

@implementation TTTimerProxy

+ (instancetype)proxyWithTarget:(id)target {
    TTTimerProxy *proxy = [TTTimerProxy alloc];
    proxy.target = target;
    return proxy;
}

- (NSMethodSignature *)methodSignatureForSelector:(SEL)sel {
    if ([self.target respondsToSelector:sel]) {
        NSMethodSignature *signature = [self.target methodSignatureForSelector:sel];
        return signature;
    } else {
        return [super methodSignatureForSelector: sel];
    }
}

- (void)forwardInvocation:(NSInvocation *)invocation {
    SEL sel = invocation.selector;
    if([self.target respondsToSelector:sel]) {
        invocation.target = self.target;
        [invocation invoke];
    } else {
        [super forwardInvocation: invocation];
    }
}

@end
```

在使用时把 NSTimer 的 target 替换成 TTTimerProxy 实例

``` Objective-c
    self.timerProxy = [TTTimerProxy proxyWithTarget:self];
    self.timer = [NSTimer timerWithTimeInterval:0.3
                                         target:self.timerProxy
                                       selector:@selector(handleTimerCallback)
                                       userInfo:nil
                                        repeats:true];
    [[NSRunLoop currentRunLoop] addTimer:self.timer forMode:NSRunLoopCommonModes];
```
这个改造之后，在退出 NSTimerViewController 页面时，dealloc 函数会被调用到；
这样改造之后 NSTimer 、NSTimerViewController、TTTimerProxy 它们之间的引用关系变成这样了(虚线代表弱引用)：
![proxy_timer_vc](/img/2017-08-10/nstimer_proxy_reference.png)

#### 方案三 利用 UIViewController 的生命周期，在页面消失的时候调用 invalidate 函数

在 viewWillDisappear 或 viewDidDisappear 的时候调用 invalidate 也能避免循环引用；
![invalidate](/img/2017-08-10/invivalite.jpg)
从文档上可以看出，调用 invalidate 方法可令 timer 失效；从而打破循环引用；

我们在开发中，如果使用到了 NSTimer，一定要检查是否存在内存泄露；



