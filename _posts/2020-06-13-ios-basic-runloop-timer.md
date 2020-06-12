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

RunLoop 的知识点非常的多，详细解读请看下面这篇文章，这里主要是对下面这篇文章中，我认为的关键内容进行摘录：

* [深入理解RunLoop · Garan no dou](https://blog.ibireme.com/2015/05/18/runloop/)
* [CFRunLoop.c 中入口函数 __CFRunLoopRun](https://opensource.apple.com/source/CF/CF-1153.18/CFRunLoop.c.auto.html)

线程和 RunLoop 之间是一一对应的。

一个 RunLoop 包含若干个 Mode，每个 Mode 又包含若干个 Source、Timer 和 Observer。Source 是事件产生的地方，Timer 是基于时间的触发器，Observer 是观察者，当 RunLoop 的状态发生变化时，观察者就能通过回调接受到这个变化。

每次调用 RunLoop 的主函数时，只能指定其中一个 Mode。如果需要切换 Mode，只能退出 Loop，再重新指定一个 Mode 进入。这样做主要是为了分隔开不同组的 Source、Timer 和 Observer，让其互不影响。

整个 RunLoop 的过程：
* 通知 Observers：即将进入 Loop
* do while
	* 通知 Observers：即将触发 Timer 回调
	* 通知 Observers：即将触发 Source0 回调
	* 触发 Source0 回调
	* 如果有 Source1 是 ready 状态的话，就会跳转到 handle_msg 去处理消息
	* 通知 Observers：即将进入休眠
	* 等待 mach_port 的消息，且进入休眠，唤醒方式：
		* 基于 port 的 Source 事件
		* Timer 时间到
		* RunLoop 超时
		* 被调用者唤醒
	* 通知 Observers：刚刚被唤醒
	* handle_msg 处理消息：
		* 如果 Timer 时间到，就触发 Timer 回调
		* 如果 dispatch 就执行 block
		* Source1 事件的话，就处理这个事件
	* 判断是否需要走下一个 Loop：
		* 事件已处理完
		* RunLoop 超时
		* 外部调用者强制停止
		* mode 为空，RunLoop 结束
* 通知 Observers: 即将退出

一些应用场景：
* AutoreleasePool：通过 Observer 监听事件来创建和释放池，这样事件回调和Timer 回调内的代码被 AutoreleasePool 环绕着。
* 事件响应：注册了一个 Source1 来接收触摸、加速、传感器等事件。
* 界面更新：当在操作 UI 时，这个 UIView 或 CALayer 就被标记为待处理，通过 Observer 监听事件来更新 UI 界面。
* 定时器：后面讲。

## 定时器

### Timer

创建和取消 Timer 的代码如下：

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