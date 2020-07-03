---
title: iOS AVAudioEngine 录制音频文件和播放音频文件
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: ios-av
---

AVAudioEngine 是 AVFoundation 的其中一部分，AVAudioEngine 封装了底层的 Audio Unit，并提供了 Swift / Objective-C API，本文讲解如何实现录制音频文件（可以混合一路背景音乐文件）和播放音频文件。

![Camera Sea](/images/camera-sea.jpg)

完整代码和参考：

* <em class="fab fa-github"></em> [AudioEngineRecorder](https://github.com/danjiang/DTCamera/blob/master/DTCamera/iOS/Recorder/AudioEngineRecorder.swift)
* <em class="fab fa-github"></em> [AudioEnginePlayer](https://github.com/danjiang/DTCamera/blob/master/DTCamera/iOS/Player/AudioEnginePlayer.swift)
* <em class="fab fa-github"></em> [syedhali / AudioStreamer](https://github.com/syedhali/AudioStreamer)
* [AVAudioEngine Tutorial for iOS: Getting Started](https://www.raywenderlich.com/5154-avaudioengine-tutorial-for-ios-getting-started)

## AVAudioEngine

iOS 中的音频处理框架如下，Audio Unit 位于 AudioToolbox 中，提供 C API，iOS 8 后已经弃用，AVAudioEngine 位于 AVFoundation 中，提供 Swift / Objective-C API：

![iOS Audio Frameworks](/images/ios-audio-frameworks.png)

AVAudioEngine 封装了底层的 [Audio Unit](/programming/2020/07/02/ios-audio-unit-record-play/)，所以很多概念是相同的，这里主要说一下，首先，Audio Unit 是拉的模式，而 AVAudioEngine 是推的模式。

![AVAudioEngine Push Mode](/images/avaudioengine-push-mode.png)

其次，AVAudioEngine 中已经有 inputNode、outputNode 和 mainMixerNode 用于输入、输出和混合，所以不需要再单独创建节点，只需要将其和相应节点连接起来。

{% highlight swift %}
var inputNode: AVAudioInputNode { get }
var outputNode: AVAudioOutputNode { get }
var mainMixerNode: AVAudioMixerNode { get }
{% endhighlight %}

## AVAudioEngine 录制音频文件

**第一步，创建 AVAudioEngine：**

{% highlight swift %}
private let engine = AVAudioEngine()
{% endhighlight %}

**第二步，添加 AVAudioNode：**

{% highlight swift %}
private let filePlayer = AVAudioPlayerNode()
private let rateEffect = AVAudioUnitTimePitch()

engine.attach(filePlayer)
engine.attach(rateEffect)
{% endhighlight %}

**第三步，连接 AVAudioNode：**

{% highlight swift %}
engine.connect(engine.inputNode, to: engine.mainMixerNode, fromBus: 0,
               toBus: 0, format: format)
if let bgmFile = bgmFile {
    engine.connect(filePlayer, to: rateEffect, format: format)
    engine.connect(rateEffect, to: engine.mainMixerNode, fromBus: 0,
                   toBus: 1, format: bgmFile.processingFormat)
}
{% endhighlight %}

**第四步，准备 AVAudioEngine：**

{% highlight swift %}
engine.prepare()
{% endhighlight %}

**第五步，安装 Tap：**

Tap 就是水龙头，比喻从数据流中安装一个水龙头来获取音频数据，从而可以写到文件或者传递给外部回调，类似于 Audio Unit 中的 Render Callback，注意拉的模式和推的模式导致使用方法是不同的：

{% highlight swift %}
private func installTap() {
    let outputFormat = engine.mainMixerNode.outputFormat(forBus: 0)
    engine.mainMixerNode.installTap(onBus: 0, bufferSize: 4096, format: outputFormat) { [weak self] buffer, when in
        guard let self = self else { return }
        if let outputFile = self.outputFile {
            do {
              try outputFile.write(from: buffer)
            } catch let error {
                DDLogError("Could not write buffer data to file \(error.localizedDescription)")
            }
        } else if let delegate = self.delegate, let converter = self.converter {
            let inputCallback: AVAudioConverterInputBlock = { inNumPackets, outStatus in
                outStatus.pointee = AVAudioConverterInputStatus.haveData
                return buffer
            }
            let convertedBuffer = AVAudioPCMBuffer(pcmFormat: self.converterFormat,
                                                   frameCapacity: AVAudioFrameCount(self.converterFormat.sampleRate) * buffer.frameLength / AVAudioFrameCount(buffer.format.sampleRate))!
            var error: NSError? = nil
            let statusCode = converter.convert(to: convertedBuffer, error: &error, withInputFrom: inputCallback)
            if statusCode == .error {
                DDLogError("AVAudioConverter failed \(error?.localizedDescription ?? "")")
                exit(1)
            }
            delegate.audioRecorder(self, receive: convertedBuffer.audioBufferList.pointee.mBuffers)
        }
    }
}
{% endhighlight %}

**第六步，启动 AVAudioEngine：**

{% highlight swift %}
try engine.start()
{% endhighlight %}

**第七步，停止 AVAudioEngine：**

{% highlight swift %}
engine.stop()
{% endhighlight %}

## AVAudioEngine 播放音频文件

通过 AVAudioEngine 来播放音频文件和录制音频文件的步骤并没有什么太大不同，唯一值得关注的是如何设置 AVAudioPlayerNode 需要的音频文件，前面的录制音频文件也用到此功能来混合一路背景音乐文件：

{% highlight swift %}
class AudioEnginePlayer {
    
    private let fileURL: URL
    private var file: AVAudioFile?

    private let filePlayer = AVAudioPlayerNode()

    private func setupFilePlayer() {
        file = try? AVAudioFile(forReading: fileURL)
    }
    
    private func scheduleAudioFile() {
        guard let file = file else { return }

        filePlayer.scheduleFile(file, at: nil) { [weak self] in
            self?.scheduleAudioFile()
        }
    }

}
{% endhighlight %}