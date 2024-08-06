---
layout:     post
title:      NSTimer 循环引用解决方案
subtitle: 
date:       2017-08-10
author:     潘世民
header-img:  img/2016-07-10/2016-08-10-bg.webp
catalog: true
tags:
    - NSTimer
    - 循环引用
    - 内存泄露
---

#### 复现 NSTimer 循环引用

考虑这种场景：进入一个页面后，启动 timer；退出此页面后，timer 也销毁。
``` swift
class NSTimerTestViewController: UIViewController {
    
    private var timer: Timer?
    
    deinit {
        print("NSTimerTestViewController deinit ...")
        timer?.invalidate()
        timer = nil
    }

    override func viewDidLoad() {
        super.viewDidLoad()
        self.view.backgroundColor = .white
        
        /// init
        timer = Timer(timeInterval: 0.3, target: self, selector: #selector(handleTimerCallback), userInfo: nil, repeats: true)
        if let timer = timer {
            RunLoop.current.add(timer, forMode: .common)
        }
    }
    
    override func viewDidAppear(_ animated: Bool) {
        super.viewDidAppear(animated)
        timer?.fire()
    }
    
    @objc
    func handleTimerCallback() {
        print("timer called ...")
    }
}
```
但是当我们从 NSTimerTestViewController 退出时，控制台还是不断地输出 log `"timer called ..."`；而且 `deinit` 函数也没有调用；基本上可以断定 NSTimerTestViewController 页面发生了内存泄露。

#### 原因分析
在上面的代码中 NSTimerTestViewController 强引用了 timer，同时 timer 通过 target 也强引用了
NSTimerTestViewController 这样就形成了循环引用；
由于 timer 强引用了 NSTimerTestViewController，所以即使从 NSTimerTestViewController 页面退出后，其引用计数也大于 0，导致 NSTimerTestViewController 的 deinit 函数不会执行，因此 deinit 里的 timer?.invalidate() 也就无法执行了。

#### 方法一
使用 iOS 新添加的接口


