---
title: iOS 基础 - RunLoop 和 定时器
author: 但江
avatar: danjiang
location: 成都 
category: programming
tag: ios-basic
---

一些 iOS 基础知识，业务开发中经常用到，面试时也常会被问到，这里总结一下，此篇文章讲解 RunLoop 和 定时器。

![Apple Develop Xcode](/images/apple-develop-xcode.jpg)

## RunLoop

### 基本概念

RunLoop 的知识点非常的多，详细解读请看下面这篇文章，这里主要是对下面这篇文章中，我认为的关键内容进行摘录：

* [深入理解RunLoop · Garan no dou](https://blog.ibireme.com/2015/05/18/runloop/)
* [CFRunLoop.c 中入口函数 __CFRunLoopRun](https://opensource.apple.com/source/CF/CF-1153.18/CFRunLoop.c.auto.html)

线程和 RunLoop 之间是一一对应的。

一个 RunLoop 包含若干个 Mode，每个 Mode 又包含若干个 Source、Timer 和 Observer。Source 是事件产生的地方，Timer 是基于时间的触发器，Observer 是观察者，当 RunLoop 的状态发生变化时，观察者就能通过回调接受到这个变化。

每次调用 RunLoop 的主函数时，只能指定其中一个 Mode。如果需要切换 Mode，只能退出 Loop，再重新指定一个 Mode 进入。这样做主要是为了分隔开不同组的 Source、Timer 和 Observer，让其互不影响。

### 整个 RunLoop 的过程

* 通知 Observers：**即将进入 Loop**
* do while
	* 通知 Observers：**即将触发 Timer 回调**
	* 通知 Observers：**即将触发 Source0 回调**
	* **触发 Source0 回调**
	* 如果有 Source1 是 ready 状态的话，就会跳转到 handle_msg 去处理消息
	* 通知 Observers：**即将进入休眠**
	* 等待 mach_port 的消息，且进入休眠，唤醒方式：
		* 基于 port 的 Source 事件
		* Timer 时间到
		* RunLoop 超时
		* 被调用者唤醒
	* 通知 Observers：**刚刚被唤醒**
	* handle_msg 处理消息：
		* **如果 Timer 时间到，就触发 Timer 回调**
		* **如果 dispatch 就执行 block**
		* **Source1 事件的话，就处理这个事件**
	* 判断是否需要走下一个 Loop：
		* 事件已处理完
		* RunLoop 超时
		* 外部调用者强制停止
		* mode 为空，RunLoop 结束
* 通知 Observers: **即将退出**

### 一些应用场景

#### 监控卡顿

App 启动后，在主线程 RunLoop 里注册了下面的 Observer：

* 通知 Observers：即将进入 Loop
* do while
	* ...
	* 通知 Observers：**即将触发 Source0 回调** => runLoopObserverCallBack
	* ...	
	* 通知 Observers：**刚刚被唤醒** => runLoopObserverCallBack
	* ...
* 通知 Observers: 即将退出

runLoopObserverCallBack 主要是更新 runLoopActivity 和触发信号量通知：

{% highlight objc %}
static void runLoopObserverCallBack(CFRunLoopObserverRef observer, CFRunLoopActivity activity, void *info){
    SMLagMonitor *lagMonitor = (__bridge SMLagMonitor*)info;
    lagMonitor->runLoopActivity = activity;
    
    dispatch_semaphore_t semaphore = lagMonitor->dispatchSemaphore;
    dispatch_semaphore_signal(semaphore);
}
{% endhighlight %}

开启一个子线程进行监控，当信号量在等待：第一种情况超时几秒返回，检测 runLoopActivity 没有发生变化，说明 **Source0 回调** 和 **handle_msg 处理消息** 有耗时情况发生；第二种情况被 runLoopObserverCallBack 通过信号量唤醒，检测 runLoopActivity 发生变化，没有耗时情况发生，继续监控。

{% highlight objc %}
// 创建子线程监控
dispatch_async(dispatch_get_global_queue(0, 0), ^{
    // 子线程开启一个持续的 loop 用来进行监控
    while (YES) {
        long semaphoreWait = dispatch_semaphore_wait(dispatchSemaphore, dispatch_time(DISPATCH_TIME_NOW, 3 * NSEC_PER_SEC));
        if (semaphoreWait != 0) {
            if (!runLoopObserver) {
                timeoutCount = 0;
                dispatchSemaphore = 0;
                runLoopActivity = 0;
                return;
            }
            // BeforeSources 和 AfterWaiting 这两个状态能够检测到是否卡顿
            if (runLoopActivity == kCFRunLoopBeforeSources || runLoopActivity == kCFRunLoopAfterWaiting) {
                // 将堆栈信息上报服务器的代码放到这里
            } // end activity
        }// end semaphore wait
        timeoutCount = 0;
    }// end while
});
{% endhighlight %}

参考 [如何利用 RunLoop 原理去监控卡顿？](https://time.geekbang.org/column/article/89494) 和 <em class="fab fa-github"></em> [SMLagMonitor.m](https://github.com/ming1016/DecoupleDemo/blob/master/DecoupleDemo/SMLagMonitor.m)

#### AutoreleasePool

App 启动后，苹果在主线程 RunLoop 里注册了下面的 Observer：

* 通知 Observers：**即将进入 Loop** => 调用 _objc_autoreleasePoolPush() 创建自动释放池
* do while
	* ...
	* 通知 Observers：**即将进入休眠** => 调用_objc_autoreleasePoolPop() 和 _objc_autoreleasePoolPush() 释放旧的池并创建新池
	* ...
* 通知 Observers: **即将退出** => 调用 _objc_autoreleasePoolPop() 来释放自动释放池

#### 事件响应

苹果注册了一个 **Source1** 来接收触摸、加速、传感器等系统事件，随后苹果注册的那个 Source1 就会触发回调，并调用 _UIApplicationHandleEventQueue() 进行应用内部的分发。

#### 手势识别

当上面的 _UIApplicationHandleEventQueue() 识别了一个手势时，其首先会调用 Cancel 将当前的 touchesBegin/Move/End 系列回调打断，随后系统将对应的 UIGestureRecognizer 标记为待处理，苹果注册了一个 Observer 监测 **即将进入休眠**，其回调函数会获取所有刚被标记为待处理的 UIGestureRecognizer，并执行UIGestureRecognizer 的回调，当有 UIGestureRecognizer 的状态变化时，这个回调都会进行相应处理。

#### 界面更新

当在操作 UI 时，这个 UIView 或 CALayer 就被标记为待处理，并被提交到一个全局的容器去，苹果注册了一个 Observer 监测 **即将进入休眠** 和 **即将退出**，回调去执行一个很长的函数，这个函数里会遍历所有待处理的 UIView 或 CALayer 以执行实际的绘制和调整，并更新 UI 界面。

## 定时器

### Timer

创建和取消 Timer 的代码如下，注意创建 Timer 调用 scheduledTimer(timeInterval:target:selector:userInfo:repeats:) 会一直强引用 target，直到调用 invalidate()，如果忘记调用 invalidate() 就会造成内存泄露，最好使用 block 版本 scheduledTimer(withTimeInterval:repeats:block:) 来创建 Timer：

{% highlight swift %}
func createTimer() {
  if timer == nil {
    timer = Timer.scheduledTimer(timeInterval: 1.0,
                                 target: self,
                                 selector: #selector(updateTimer),
                                 userInfo: nil,
                                 repeats: true)
  }
}

func cancelTimer() {
  timer?.invalidate()
  timer = nil
}
{% endhighlight %}

一个 Timer 注册到 RunLoop 后，RunLoop 会为其重复的时间点注册好事件。例如 10:00, 10:10, 10:20 这几个时间点。RunLoop为了节省资源，并不会在非常准确的时间点回调这个 Timer。Timer 有个属性叫做 Tolerance，标示了当时间点到后，容许有多少最大误差。

主线程的 RunLoop 里有两个预置的 mode：default 和 tracking。default 是 App 平时所处的状态，tracking 是追踪 ScrollView 滑动时的状态。通过 Timer.scheduledTimer(timeInterval:target:selector:userInfo:repeats:) 创建的定时器是加入 default mode，所以在滑动的时候，RunLoop 切换到 tracking mode，定时器就会暂定。

解决方案 1 - timer 同时加入 default mode 和 tracking mode：

{% highlight swift %}
RunLoop.main.add(timer, forMode: .default)
RunLoop.main.add(timer, forMode: .tracking)
{% endhighlight %}

解决方案 2 - timer 加入 common mode，加入 common mode 的 timer，每次 RunLoop 运行时都会执行：

{% highlight swift %}
RunLoop.main.add(timer, forMode: .common)
{% endhighlight %}

当 App 退到后台，iOS 会暂停运行的 timer，当 App 又回到前台，iOS 会重启 timer。

### CADisplayLink

也是一个定时器，但是可以同步屏幕刷新的频率：

{% highlight swift %}
displayLink = CADisplayLink(target: self,
  selector: #selector(updateAnimation))
displayLink?.add(to: RunLoop.main, forMode: .common)

@objc func updateAnimation() {
}
{% endhighlight %}

### DispatchSourceTimer

此定时器的精度更高且不受 RunLoop 的影响：

{% highlight swift %}
var timer: DispatchSourceTimer?

func cancelTimer() {
    timer?.cancel()
    timer = nil
}

func startTimer() {
    if timer == nil {
        let queue = DispatchQueue.global()
        timer = DispatchSource.makeTimerSource(queue: queue)
        timer?.schedule(deadline: .now(), repeating: .seconds(1))
    }
    
    timer?.setEventHandler(handler: { [weak self] in
        guard let self = self else { return }
        self.countDown()
    })
    timer?.resume()
}
{% endhighlight %}