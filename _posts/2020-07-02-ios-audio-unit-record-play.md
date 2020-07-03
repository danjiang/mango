---
title: iOS Audio Unit 录制音频文件和播放音频文件
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: ios-av
---

Audio Unit 是 iOS 比较底层的音频处理库，提供了一些效果器的功能，都是 C API，本文讲解如何实现录制音频文件（可以混合一路背景音乐文件）和播放音频文件。

![Camera Sea](/images/camera-sea.jpg)

完整代码和参考：

* <em class="fab fa-github"></em> [AudioUnitRecorder](https://github.com/danjiang/DTCamera/blob/master/DTCamera/iOS/Recorder/AudioUnitRecorder.swift)
* <em class="fab fa-github"></em> [AUGraphPlayer](https://github.com/danjiang/DTCamera/blob/master/DTCamera/iOS/Player/AUGraphPlayer.swift)
* <em class="fab fa-github"></em> [syedhali / AudioStreamer](https://github.com/syedhali/AudioStreamer)
* [Audio Unit Programming Guide](https://developer.apple.com/library/archive/documentation/MusicAudio/Conceptual/AudioUnitProgrammingGuide/TheAudioUnit/TheAudioUnit.html#//apple_ref/doc/uid/TP40003278-CH12-SW1)

## Audio Unit Processing Graph Services

### 基本概念

Audio Unit Processing Graph Services 简称 AUGraph，AUGraph 提供了基于图的接口来创建互相连接的节点，从而播放非压缩 LPCM 音频数据。

![Audio Band](/images/audio-band.png)

上图的结构，建模到 AUGraph 如下：

![Audio Band AUGraph](/images/audio-band-augraph.png)

### AUGraph 拉的模式

如下图所示，音频图采用拉的模式，音频图中的最后一个节点从连接的前一个节点拉取数据，前一个节点从连接的前前一个节点拉取数据，直到第一个节点：

![AUGraph Pull Mode](/images/augraph-pull-mode.png)

## Audio Unit

### Audio Unit 类型

通过 AudioComponentDescription 的 componentType 和 componentSubType 来描述 Audio Unit 的类型，查看 [AudioUnit > Audio Unit Data Types](https://developer.apple.com/documentation/audiounit/audio_unit_data_types) 和 AudioUnit.framework > AUComponent.h 文件了解更多：

{% highlight swift %}
var ioDescription = AudioComponentDescription()
bzero(&ioDescription, MemoryLayout.size(ofValue: ioDescription))
ioDescription.componentManufacturer = kAudioUnitManufacturer_Apple
ioDescription.componentType = kAudioUnitType_Output
ioDescription.componentSubType = kAudioUnitSubType_RemoteIO
var statusCode = AUGraphAddNode(auGraph, &ioDescription, &ioNode)
if statusCode != noErr {
    DDLogError("Could not add I/O node to AUGraph \(statusCode)")
    exit(1)
}
{% endhighlight %}

### Audio Unit 结构

如下图所示，Audio Unit 有 3 个不同的 scope，每个 scope 下面有多个不同 elements（类似于 bus）：

![iOS Audio Unit](/images/ios-audio-unit.png)

如下图所示，Audio Unit 的中 I/O Unit 的结构，有两个 elements，element 0 针对音响输出，element 1 针对麦克风输入：

![iOS Audio Unit I/O Unit](/images/ios-audio-unit-io-unit.png)

### Audio Unit 属性

可以给不同 scope 下的 element 设置对应的属性，查看 [AudioUnit > Audio Unit Properties](https://developer.apple.com/documentation/audiounit/audio_unit_properties) 和 AudioToolbox.framework > AudioUnitProperties.h 文件了解更多：

{% highlight swift %}
var enableIO: UInt32 = 1
var statusCode = AudioUnitSetProperty(ioUnit, // audio unit
                                      kAudioOutputUnitProperty_EnableIO, // property
                                      kAudioUnitScope_Input, // scope
                                      inputBus, // element
                                      &enableIO, // value
                                      UInt32(MemoryLayout.size(ofValue: enableIO))) // value size
if statusCode != noErr {
    DDLogError("Could not enable I/O for I/O unit input element 1 \(statusCode)")
    exit(1)
}
{% endhighlight %}

### Audio Unit 参数

可以给不同 scope 下的 element 设置对应的参数，查看 [AudioUnit > Audio Unit Parameters](https://developer.apple.com/documentation/audiounit/audio_unit_parameters) 和 AudioToolbox.framework > AudioUnitParameters.h 文件了解更多：

{% highlight swift %}
statusCode = AudioUnitSetParameter(mixerUnit, // audio unit
                                   kMultiChannelMixerParam_Volume, // parameter
                                   kAudioUnitScope_Output, // scope
                                   0, // element
                                   3.0, // value
                                   0)
if statusCode != noErr {
    DDLogError("Could not set volume for mixer unit output element 0 \(statusCode)")
    exit(1)
}
{% endhighlight %}

## Audio Unit 录制音频文件

从前面内容可以看出，Audio Unit 的流程就是创建节点，设置属性和参数，连接这些节点来组成图。

**第一步，创建 AUGraph：**

{% highlight swift %}
var statusCode = NewAUGraph(&auGraph)
{% endhighlight %}

**第二步，添加 AUNode：**

{% highlight swift %}
var mixerDescription = AudioComponentDescription()
bzero(&mixerDescription, MemoryLayout.size(ofValue: mixerDescription))
mixerDescription.componentManufacturer = kAudioUnitManufacturer_Apple
mixerDescription.componentType = kAudioUnitType_Mixer
mixerDescription.componentSubType = kAudioUnitSubType_MultiChannelMixer
statusCode = AUGraphAddNode(auGraph, &mixerDescription, &mixerNode)
if statusCode != noErr {
    DDLogError("Could not add mixer node to AUGraph \(statusCode)")
    exit(1)
}
{% endhighlight %}

**第三步，打开 AUGraph：**

{% highlight swift %}
statusCode = AUGraphOpen(auGraph)
{% endhighlight %}

**第四步，从 AUNode 拿到 AudioUnit：**

{% highlight swift %}
var statusCode = AUGraphNodeInfo(auGraph, ioNode, nil, &ioUnit)
{% endhighlight %}

**第五步，打开 AUGraph：**

{% highlight swift %}
statusCode = AUGraphOpen(auGraph)
{% endhighlight %}

**第六步，设置 AudioUnit 的属性：**

{% highlight swift %}
var enableIO: UInt32 = 1
var statusCode = AudioUnitSetProperty(ioUnit, // audio unit
                                      kAudioOutputUnitProperty_EnableIO, // property
                                      kAudioUnitScope_Input, // scope
                                      inputBus, // element
                                      &enableIO, // value
                                      UInt32(MemoryLayout.size(ofValue: enableIO))) // value size
if statusCode != noErr {
    DDLogError("Could not enable I/O for I/O unit input element 1 \(statusCode)")
    exit(1)
}
{% endhighlight %}

**第七步，设置 AudioUnit 的参数：**

{% highlight swift %}
statusCode = AudioUnitSetParameter(mixerUnit, // audio unit
                                   kMultiChannelMixerParam_Volume, // parameter
                                   kAudioUnitScope_Output, // scope
                                   0, // element
                                   3.0, // value
                                   0)
if statusCode != noErr {
    DDLogError("Could not set volume for mixer unit output element 0 \(statusCode)")
    exit(1)
}
{% endhighlight %}

**第八步，连接 AUNode 和 构造 Render Callback：**

连接 AUNode

{% highlight swift %}
var statusCode = AUGraphConnectNodeInput(auGraph, ioNode, inputBus, convertNode, 0)
{% endhighlight %}

构造一个 AURenderCallbackStruct 的结构体，并给结构体指定一个回调函数，将结构体设置给 AUNode 的输入端，当该 AUNode 需要数据的时候就会回调前面指定的回调函数：

{% highlight swift %}
var inputCallback = AURenderCallbackStruct()
inputCallback.inputProc = renderCallback
inputCallback.inputProcRefCon = UnsafeMutableRawPointer(Unmanaged.passUnretained(self).toOpaque())
statusCode = AUGraphSetNodeInputCallback(auGraph, ioNode, outputBus, &inputCallback)
if statusCode != noErr {
    DDLogError("Could not set input callback for I/O node \(statusCode)")
    exit(1)
}
{% endhighlight %}

**第九步，实现 Render Callback 函数：**

通过 ExtAudioFileWriteAsync 函数写到文件或者传递音频数据给外部回调：

{% highlight swift %}
func renderCallback(inRefCon: UnsafeMutableRawPointer,
                    ioActionFlags: UnsafeMutablePointer<AudioUnitRenderActionFlags>,
                    inTimeStamp: UnsafePointer<AudioTimeStamp>,
                    inBusNumber: UInt32,
                    inNumberFrames: UInt32,
                    ioData: UnsafeMutablePointer<AudioBufferList>?) -> OSStatus {
    let recorder: AudioUnitRecorder = Unmanaged.fromOpaque(inRefCon).takeUnretainedValue()
    var statusCode = AudioUnitRender(recorder.mixerUnit, ioActionFlags, inTimeStamp, inBusNumber, inNumberFrames, ioData!)
//    DDLogDebug("audio recorder receive \(inNumberFrames) frames with \(ioData?.pointee.mBuffers.mDataByteSize ?? 0) btyes")
    if let audioFile = recorder.audioFile {
        statusCode = ExtAudioFileWriteAsync(audioFile, inNumberFrames, ioData)
        if statusCode != noErr {
            DDLogError("ExtAudioFileWriteAsync failed \(statusCode)")
            exit(1)
        }
    } else if let audioBuffer = ioData?.pointee.mBuffers {
        recorder.delegate?.audioRecorder(recorder, receive: audioBuffer)
    }
    return statusCode
}
{% endhighlight %}

**第十步，启动 AUGraph：**

{% highlight swift %}
let statusCode = AUGraphStart(auGraph)
{% endhighlight %}

**第十一步，停止 AUGraph：**

{% highlight swift %}
let statusCode = AUGraphStop(auGraph)
{% endhighlight %}

## Audio Unit 播放音频文件

通过 Audio Unit 来播放音频文件和录制音频文件的步骤并没有什么太大不同，唯一值得关注的是如何设置子类型为 kAudioUnitSubType_AudioFilePlayer 的 AUNode 需要的音频文件，前面的录制音频文件也用到此功能来混合一路背景音乐文件：

{% highlight swift %}
private func setupFilePlayer() {
    // 打开音频文件
    var fileId: AudioFileID!
    var statusCode = AudioFileOpenURL(fileURL as CFURL, .readPermission, 0, &fileId)
    if statusCode != noErr {
        DDLogError("Could not open audio file \(statusCode)")
        exit(1)
    }
    
    // 给 AudioUnit 设置音频文件 ID
    statusCode = AudioUnitSetProperty(filePlayerUnit,
                                      kAudioUnitProperty_ScheduledFileIDs,
                                      kAudioUnitScope_Global,
                                      0,
                                      &fileId,
                                      UInt32(MemoryLayout.size(ofValue: fileId)))
    if statusCode != noErr {
        DDLogError("Could not tell file player unit load which file \(statusCode)")
        exit(1)
    }
    
    // 获取音频文件的格式信息
    var fileAudioStreamFormat = AudioStreamBasicDescription()
    var size = UInt32(MemoryLayout.size(ofValue: fileAudioStreamFormat))
    statusCode = AudioFileGetProperty(fileId,
                                      kAudioFilePropertyDataFormat,
                                      &size,
                                      &fileAudioStreamFormat)
    if statusCode != noErr {
        DDLogError("Could not get the audio data format from the file \(statusCode)")
        exit(1)
    }
    
    // 获取音频文件的包数量
    var numberOfPackets: UInt64 = 0
    size = UInt32(MemoryLayout.size(ofValue: numberOfPackets))
    statusCode = AudioFileGetProperty(fileId,
                                      kAudioFilePropertyAudioDataPacketCount,
                                      &size,
                                      &numberOfPackets)
    if statusCode != noErr {
        DDLogError("Could not get number of packets from the file \(statusCode)")
        exit(1)
    }
    
    // 设置音频文件播放的范围：是否循环，起始帧，播放多少帧
    var rgn = ScheduledAudioFileRegion(mTimeStamp: .init(),
                                       mCompletionProc: nil,
                                       mCompletionProcUserData: nil,
                                       mAudioFile: fileId,
                                       mLoopCount: 0,
                                       mStartFrame: 0,
                                       mFramesToPlay: UInt32(numberOfPackets) * fileAudioStreamFormat.mFramesPerPacket)
    memset(&rgn.mTimeStamp, 0, MemoryLayout.size(ofValue: rgn.mTimeStamp))
    rgn.mTimeStamp.mFlags = .sampleTimeValid
    rgn.mTimeStamp.mSampleTime = 0
    statusCode = AudioUnitSetProperty(filePlayerUnit,
                                      kAudioUnitProperty_ScheduledFileRegion,
                                      kAudioUnitScope_Global,
                                      0,
                                      &rgn,
                                      UInt32(MemoryLayout.size(ofValue: rgn)))
    if statusCode != noErr {
        DDLogError("Could not set file player unit`s region \(statusCode)")
        exit(1)
    }
    
    // 设置 prime，I don`t know why
    var defaultValue: UInt32 = 0
    statusCode = AudioUnitSetProperty(filePlayerUnit,
                                      kAudioUnitProperty_ScheduledFilePrime,
                                      kAudioUnitScope_Global,
                                      0,
                                      &defaultValue,
                                      UInt32(MemoryLayout.size(ofValue: defaultValue)))
    if statusCode != noErr {
        DDLogError("Could not set file player unit`s prime \(statusCode)")
        exit(1)
    }
    
    // 设置 start time，I don`t know why
    var startTime = AudioTimeStamp()
    memset(&startTime, 0, MemoryLayout.size(ofValue: startTime))
    startTime.mFlags = .sampleTimeValid
    startTime.mSampleTime = -1
    statusCode = AudioUnitSetProperty(filePlayerUnit,
                                      kAudioUnitProperty_ScheduleStartTimeStamp,
                                      kAudioUnitScope_Global,
                                      0,
                                      &startTime,
                                      UInt32(MemoryLayout.size(ofValue: startTime)))
    if statusCode != noErr {
        DDLogError("Could not set file player unit`s start time \(statusCode)")
        exit(1)
    }
}
{% endhighlight %}