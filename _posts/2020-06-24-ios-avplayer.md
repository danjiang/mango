---
title: iOS 音视频播放器 AVPlayer
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: ios-av
---

iOS 中的音视频处理库 AVFoundation 内容很丰富，功能也很强大，本文主要讲解 AVPlayer 音视频播放器，可以播放本地视频文件，也可以播放网络视频文件。

![Camera Sea](/images/camera-sea.jpg)

## 关键类

完整代码：

* <em class="fab fa-github"></em> [PlayerView](https://github.com/danjiang/DTCamera/blob/master/DTCamera/iOS/Player/PlayerView.swift)
* <em class="fab fa-github"></em> [PlayerViewController](https://github.com/danjiang/DTCamera/blob/master/DTCamera/iOS/Player/PlayerViewController.swift)
* <em class="fab fa-github"></em> [PreviewVideoViewController](https://github.com/danjiang/DTCamera/blob/master/DTCamera/iOS/Player/PreviewVideoViewController.swift)

AVAsset 对媒体资源的抽象，如视频资源和音频资源。

{% highlight swift %}
// 本地文件
if let fileURL = Bundle.main.url(forResource: "boat", withExtension: "mov") {
    let asset = AVURLAsset(url: url)
}

// 网络文件
if let fileURL = URL(string: "http://danthought.com/morning.mov") {
    let asset = AVURLAsset(url: url)
}
{% endhighlight %}

AVPlayerItem 对媒体资源的时间状态和表现状态的抽象。

{% highlight swift %}
let item = AVPlayerItem(asset: asset)
{% endhighlight %}

AVPlayer 是控制媒体资源播放的统一接口，AVPlayer 和 AVPlayerItem 是一对多的关系。

{% highlight swift %}
let player = AVPlayer()
player.actionAtItemEnd = .none
player.replaceCurrentItem(with: item) // decoding audio and video
{% endhighlight %}

AVPlayerLayer 管理 AVPlayer 输出的视频图像，在界面上显示出来。

{% highlight swift %}
class PlayerView: UIView {
    
    override class var layerClass: AnyClass {
        return AVPlayerLayer.self
    }
    
    private var playerLayer: AVPlayerLayer? {
        return layer as? AVPlayerLayer
    }
    
    private var player: AVPlayer? {
        get {
            return playerLayer?.player
        }
        set {
            playerLayer?.player = newValue
        }
    }
    
}
{% endhighlight %}

## 播放状态

获取播放器的状态都是异步的，相关状态变更时来通知你。

监控 AVPlayerItem 的 status，得到视频是否可以播放了：

{% highlight swift %}
statusObservation = item.observe(\.status) { [weak self] item, _ in
    self?.itemStatusChanged(item)
}

private func itemStatusChanged(_ item: AVPlayerItem) {
    statusObservation?.invalidate()
    if item.status == .readyToPlay {
        if !item.duration.isIndefinite {
            durationChanged(item)
        } else {
            durationObservation = item.observe(\.duration) { [weak self] item, _ in
                self?.durationChanged(item)
            }
        }
    } else {
        indicatorView.stopAnimating()
        DTMessageBar.error(message: "视频无法播放")
    }
}
{% endhighlight %}

得到可以播放的状态，还需要知道视频的时长，也是异步的通知：

{% highlight swift %}
durationObservation = item.observe(\.duration) { [weak self] item, _ in
    self?.durationChanged(item)
}

private func durationChanged(_ item: AVPlayerItem) {
    if !item.duration.isIndefinite {
        durationObservation?.invalidate()
        indicatorView.stopAnimating()
        duration = CMTimeGetSeconds(item.duration)
        let current = CMTimeGetSeconds(item.currentTime())

        let timeScale = CMTimeScale(NSEC_PER_SEC)
        var time = CMTime(seconds: 1, preferredTimescale: timeScale)
        currentTimeObservation =
            player?.addPeriodicTimeObserver(forInterval: time, queue: .main) { [weak self] time in
                self?.itemCurrentTimeChanged(time)
        }
        
        NotificationCenter.default.addObserver(self, selector: #selector(playerDidFinishPlaying(_ :)),
                                               name: .AVPlayerItemDidPlayToEndTime, object: player?.currentItem)
    }
}
{% endhighlight %}

根据视频播放进度来更新界面进度条：

{% highlight swift %}
let timeScale = CMTimeScale(NSEC_PER_SEC)
var time = CMTime(seconds: 1, preferredTimescale: timeScale)
currentTimeObservation =
    player?.addPeriodicTimeObserver(forInterval: time, queue: .main) { [weak self] time in
        self?.itemCurrentTimeChanged(time)
}


private func itemCurrentTimeChanged(_ time: CMTime) {
    let current = CMTimeGetSeconds(time)
    if style != .simple {
        updateTime(for: currentTimeLabel, time: current)
    }
    if let slider = slider {
        if !isSliding {
            slider.setValue(Float(current), animated: true)
        }
    }
    if let progressView = progressView {
        progressView.setProgress(Float(current/duration), animated: true)
    }
}
{% endhighlight %}

## Seek

iOS 中使用 UISlider 来做 seek 控制是比较自然的选择：

{% highlight swift %}
slider = UISlider()
slider.isContinuous = false
// slider开始滑动事件
slider.addTarget(self, action: #selector(sliderTouchBegin(_:)), for: .touchDown)
// slider滑动中事件
slider.addTarget(self, action: #selector(sliderValueChanged(_:)), for: .valueChanged)

@objc private func sliderTouchBegin(_ slider: UISlider) {
    isSliding = true
}

@objc private func sliderValueChanged(_ slider: UISlider) {
    let seconds = Double(slider.value)
    let timeScale = CMTimeScale(NSEC_PER_SEC)
    let time = CMTime(seconds: seconds, preferredTimescale: timeScale)
    player?.seek(to: time, completionHandler: { _ in
        self.isSliding = false
    })
}
{% endhighlight %}

上面代码中有一个 isSliding 的状态用于控制是否在滑动中，是为了避免滑动过程中，进度更新代码将其进度更新，就会产生冲突：

{% highlight swift %}
private func itemCurrentTimeChanged(_ time: CMTime) {
    let current = CMTimeGetSeconds(time)
    if let slider = slider {
        if !isSliding {
            slider.setValue(Float(current), animated: true)
        }
    }    
}
{% endhighlight %}

## 边下边播

上面使用 AVPlayer 的方式也是可以边下边播的，但是有一个场景是在一个界面的头部嵌入界面播放时，点击了全屏播放，跳到一个新界面进行全屏播放，希望两个界面的视频播放进度是同步的，之前的解决方案是传递 AVPlayer：

{% highlight swift %}
class PlayerViewController: UIViewController {
    func setPlayer(_ player: AVPlayer) {
        playerView.setPlayer(player)
    }
}
{% endhighlight %}

这种方式非常的笨拙，AVPlayer 是黑盒，也没有办法把其下载的网络视频数据拿出来，将视频数据下载和视频播放分离开就是解决方案，有如下参考：

* [iOS 分享一个边播边缓存的库(支持VOD和HLS)](https://juejin.im/post/5ee31be851882557525a8b18)
* <em class="fab fa-github"></em> [SJMediaCacheServer](https://github.com/changsanjiang/SJMediaCacheServer) 是一个 iOS 端的 HTTP 媒体数据缓存框架，播放器向本地 HTTP 代理服务器发送播放请求后，会查询本地缓存，如不存在缓存，则进行下载并返回给播放器。
* <em class="fab fa-github"></em> [KTVHTTPCache](https://github.com/ChangbaDevs/KTVHTTPCache) KTVHTTPCache is a powerful media cache framework. It can cache HTTP request, and very suitable for media resources.

## 更可控和更完整的视频播放器

总的来说，为了易用性，AVPlayer 是黑盒，很多部分开发者并不能自己调控，下面看一下一个完整播放器的整体架构：

![iOS Audio Video Consumer](/images/ios-audio-video-consumer.png)

要实现一个这样的视频播放器需要花费较多的时间，跨平台的视频播放器实现体系主要是 ffplay 和 vlc：

* <em class="fab fa-github"></em> [ijkplayer](https://github.com/bilibili/ijkplayer)
* <em class="fab fa-github"></em> [vlc](https://github.com/videolan/vlc)