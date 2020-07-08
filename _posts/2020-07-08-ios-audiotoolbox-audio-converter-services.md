---
title: iOS 利用 AudioToolbox 将 PCM 编码为 AAC
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: ios-av
---

本文讲解了数字音频、PCM 音频格式和AAC 音频格式这些理论知识，然后讲解在 iOS 上通过 AudioToolbox 中的 Audio Converter Services 来将 PCM 编码为 AAC。

![Camera Sea](/images/camera-sea.jpg)

完整代码：

* <em class="fab fa-github"></em> [AudioEncoder](https://github.com/danjiang/DTCamera/blob/master/DTCamera/iOS/Encode/AudioEncoder.swift)
* <em class="fab fa-github"></em> [iPhoneACFileConvertTest-Swift](https://github.com/danjiang/iPhoneACFileConvertTest-Swift)

## 数字音频

要理解音频编码，首先要理解音频数据在计算机中是怎么表示的，也就是数字音频，前面说过如下的视频文件中需要关注的常见信息，这里主要关注最后一行关于音频的部分：

* 封装格式，时长，存储大小
* 视频编码，视频码率，分辨率，帧率
* 音频编码，音频采样率，音频码率，声道

数字音频就是将模拟信号数字化，如下图所示：

![Analog Signal To Digital Signal](/images/analog-signal-to-digital-signal.jpg)

首先，要对模拟信号进行**采样**，由此引出音频的 **采样率 sampleRate，1 秒种会采样的次数**，根据奈奎斯特定理（也称为采样定理），按比声音最高频率高 2 倍以上的频率对声音进行采样（也称为 AD 转换），对于高质量的音频信号，其频率范围（人耳能够听到的频率范围）是 20 Hz～20 kHz，所以采样频率一般为 44.1kHz，这样就可以保证采样声音达到 20 kHz 也能被数字化，从而使得经过数字化处理之后，人耳听到的声音质量不会被降低。

其次，**量化**是指在幅度轴上对信号进行数字化，由此引出音频的**量化格式**，比如用 **16 比特**的二进制信号来表示声音的一个采样，而 16 比特（一个 short）所表示的范围是 [-32768，32767]，共有 65536 个可能取值，因此最终模拟的音频信号在幅度上也分为了 65536 层。

最后，**编码**就是按照一定的格式记录采样和量化后的数字数据，比如顺序存储或压缩存储。

**声道**分为单声道 mono 和立体声 stereo，也就是不分左右和分左右。

由此可以得出，音频**码率** = 采样率 * 量化格式 * 声道数

## PCM

脉冲编码调制 Pulse Code Modulation，简写 PCM，是非压缩数据的音频编码，原始音频数据，查看 [视音频数据处理入门：PCM 音频采样数据处理](https://blog.csdn.net/leixiaohua1020/article/details/50534316) 来了解如何分离声道、调节音量、调节速度、量化格式转换、音频截取等，这些基本概念很有用。

## AAC

AAC 是压缩数据的音频编码，并且多用于视频中的音频编码，参考 [视音频数据处理入门：AAC音频码流解析](https://blog.csdn.net/leixiaohua1020/article/details/50535042) 的代码和 [音视频封装格式：AAC音频基础和ADTS打包方案详解](https://mp.weixin.qq.com/s?__biz=MzI0NTMxMjA1MQ==&mid=2247483717&idx=1&sn=b3f11c98f5cdf99753a07fb461d5d2a5&chksm=e9513e19de26b70f437397e6430be75b09cb8d93d82623be0943cd9b9b6c4e07fc3686b9b151&scene=21#wechat_redirect) 对 AAC 格式的详细解读，我编写了 [AACParser](https://github.com/danjiang/MediaParser/blob/master/AACParser.cpp) 来解析 AAC 音频文件。

## 将 PCM 编码为 AAC

### Audio Converter Services

通过 AudioToolbox 中的 [Audio Converter Services](https://kapeli.com/dash_share?docset_file=Apple_API_Reference&docset_name=Apple%20API%20Reference&path=dash-apple-api://load?request_key=ts1653485&platform=apple&repo=Main&source=https://developer.apple.com/documentation/audiotoolbox/audio_converter_services) 来完成编码。

> Audio converter objects convert between various linear PCM audio formats.（LPCM 格式之间转换）They can also convert between linear PCM and compressed formats.（LPCM 和压缩格式之间转换） Supported transformations include the following:
> 
> * PCM bit depth - 转换量化格式的比特数
> * PCM sample rate - 转换采样率
> * PCM floating point to and from PCM integer - 转换量化格式的表示格式
> * PCM interleaved to and from PCM deinterleaved - 转换存储格式（交错存储和非交错存储）
> * PCM to and from compressed formats - PCM 和压缩格式之间转换
> 
> A single audio converter may perform more than one of the listed transformations.（一次可以进行上面的多个转换）

### AudioStreamBasicDescription

后面多个地方会用 AudioStreamBasicDescription 来指定音频的格式，其定义如下：

{% highlight swift %}
struct AudioStreamBasicDescription {
    Float64 mSampleRate;        // sample frames per second
    UInt32  mFormatID;          // a four-char code indicating stream type
    UInt32  mFormatFlags;       // flags specific to the stream type
    UInt32  mBytesPerPacket;    // bytes per packet of audio data
    UInt32  mFramesPerPacket;   // frames per packet of audio data
    UInt32  mBytesPerFrame;     // bytes per frame of audio data
    UInt32  mChannelsPerFrame;  // number of channels per frame
    UInt32  mBitsPerChannel;    // bit depth
    UInt32  mReserved;          // padding
};
typedef struct AudioStreamBasicDescription AudioStreamBasicDescription;
{% endhighlight %}

### 存储方式

存储格式是指交错存储或非交错存储，输出或者输入数据都存储于 AudioBufferList 中的属性 ioData 中，假设声道是双声道的，那么对于交错存储（isPacked）来讲，对应的数据格式如下：

{% highlight text %}
ioData->mBuffers[0]: LRLRLRLRLRLR...
{% endhighlight %}

而对于非交错的存储（Non Interleaved）来讲，对应的数据格式如下：

{% highlight text %}
ioData->mBuffers[0]: LLLLLLLLLLLL...
ioData->mBuffers[1]: RRRRRRRRRRRR...
{% endhighlight %}

### 步骤

#### 准备工作

**第一步，打开 PCM 音频输入文件：**

{% highlight swift %}
statusCode = AudioFileOpenURL(inputFileURL as CFURL, .readPermission, 0, &inputFileId)
guard statusCode == noErr, let inputFileId = inputFileId else {
    DDLogError("AudioFileOpenURL failed for input file with URL: \(inputFileURL)")
    exit(1)
}
{% endhighlight %}

**第二步，获取 PCM 音频输入文件的 AudioStreamBasicDescription：**

{% highlight swift %}
size = UInt32(MemoryLayout.stride(ofValue: inputAudioStreamFormat))
statusCode = AudioFileGetProperty(inputFileId, kAudioFilePropertyDataFormat, &size, &inputAudioStreamFormat)
if statusCode != noErr {
    DDLogError("AudioFileGetProperty couldn't get the inputAudioStreamFormat data format")
    exit(1)
}
{% endhighlight %}

打印输出 PCM 音频输入文件的 AudioStreamBasicDescription：

{% highlight text %}
Input Audio Stream Format:
Sample Rate: 44100.0 // 采样率
Format ID: lpcm // 格式
Format Flags: kAudioFormatFlagIsFloat kAudioFormatFlagIsPacked kLinearPCMFormatFlagIsFloat kLinearPCMFormatFlagIsPacked kLinearPCMFormatFlagsSampleFractionShift kAppleLosslessFormatFlag_16BitSourceData kAppleLosslessFormatFlag_24BitSourceData // kAudioFormatFlagIsPacked 是存储格式为交错存储，kAudioFormatFlagIsFloat 是量化格式的表示格式为 Float
Bits per Channel: 32 // 量化格式的比特数为 32 位，也就是 4 字节
Channels per Frame: 2 // 双声道
Bytes per Frame: 8 // 上面两个相乘 4 * 2
Frames per Packet: 1 // 1 个包中 1 个帧
Bytes per Packet: 8 // 也是 8 字节
{% endhighlight %}

**第三步，配置 AAC 音频输出文件的 AudioStreamBasicDescription：**

{% highlight swift %}
bzero(&outputAudioStreamFormat, MemoryLayout.size(ofValue: outputAudioStreamFormat))
outputAudioStreamFormat.mSampleRate = inputAudioStreamFormat.mSampleRate // 同输入音频的采样率
outputAudioStreamFormat.mFormatID = kAudioFormatMPEG4AAC // AAC 格式
outputAudioStreamFormat.mChannelsPerFrame = inputAudioStreamFormat.mChannelsPerFrame // 同输入音频的声道数

size = UInt32(MemoryLayout.stride(ofValue: outputAudioStreamFormat))
statusCode = AudioFormatGetProperty(kAudioFormatProperty_FormatInfo, 0, nil, &size, &outputAudioStreamFormat) // 根据上面的信息填充其他信息
if statusCode != noErr {
    DDLogError("AudioFormatGetProperty couldn't fill out the outputAudioStreamFormat data format")
    exit(1)
}
{% endhighlight %}

打印输出 AAC 音频输出文件的 AudioStreamBasicDescription：

{% highlight text %}
Output Audio Stream Format:
Sample Rate: 44100.0
Format ID: aac 
Format Flags: 
Bits per Channel: 0
Channels per Frame: 2
Bytes per Frame: 0
Frames per Packet: 1024 // 1 个包中 1024 个帧
Bytes per Packet: 0
{% endhighlight %}

**第四步，获取音频编码器描述：**

{% highlight swift %}
// 根据编码格式 AAC，软编码或硬编码来查找可用的音频编码器描述，却找不到硬编码相关的音频编码器描述
guard var description = getAudioClassDescription(with: kAudioFormatMPEG4AAC, from: kAppleSoftwareAudioCodecManufacturer) else {
    DDLogError("Could not getAudioClassDescription")
    exit(1)
}

// 但是可以直接创建编码格式 AAC 和硬编码的音频编码器描述
var description = AudioClassDescription(mType: kAudioEncoderComponentType, mSubType: kAudioFormatMPEG4AAC, mManufacturer: kAppleHardwareAudioCodecManufacturer)
{% endhighlight %}

**第五步，创建音频转换器：**

根据前面的音频输入格式描述，音频输出格式描述和音频编码器描述来创建音频转换器：

{% highlight swift %}
statusCode = AudioConverterNewSpecific(&inputAudioStreamFormat, &outputAudioStreamFormat, 1, &description, &audioConverter)
if statusCode != noErr || audioConverter == nil {
    DDLogError("AudioConverterNew failed \(statusCode)")
    exit(1)
}
{% endhighlight %}

**第六步，创建 AAC 音频输出文件：**

{% highlight swift %}
statusCode = AudioFileCreateWithURL(outputFileURL as CFURL, kAudioFileAAC_ADTSType, &outputAudioStreamFormat, .eraseFile, &outputFileId)
if statusCode != noErr {
    DDLogError("AudioFileCreateWithURL failed \(statusCode)")
    exit(1)
}
{% endhighlight %}

#### 编码转换

编码的整体流程如下图，其数据流转的核心是包 packet，从 PCM 文件读取 PCM 包到输入缓冲，发送输入缓冲给音频转换器，音频转换器进行编码转换，输出 AAC 包到输出缓冲，再把输出缓冲写到 AAC 文件：

![iOS Audio Converter Services](/images/ios-audio-converter-services.png)

**第七步，请求 AAC 包来填充输出缓冲，再把输出缓冲写到 AAC 文件：**

输出缓冲的大小由自己指定，输出的每个包的大小可以是固定的、也可以是变化的，如果是变化的，需要得到每个包的大小的最大值以及初始化每个包的描述信息 AudioStreamPacketDescription 的存储空间：

{% highlight swift %}
outputBuffer = UnsafeMutablePointer<UInt8>.allocate(capacity: Int(outputBufferSize))
outputSizePerPacket = outputAudioStreamFormat.mBytesPerPacket
if outputSizePerPacket == 0 {
    // if the destination format is VBR, we need to get max size per packet from the converter
    var size = UInt32(MemoryLayout.size(ofValue: outputSizePerPacket))
    
    statusCode = AudioConverterGetProperty(audioConverter, kAudioConverterPropertyMaximumOutputPacketSize, &size, &outputSizePerPacket)
    if statusCode != noErr {
        DDLogError("AudioConverterGetProperty kAudioConverterPropertyMaximumOutputPacketSize failed \(statusCode)")
        exit(1)
    }
    
    outputPacketDescriptions = .allocate(capacity: Int(outputBufferSize / outputSizePerPacket))
}
{% endhighlight %}

计算输出缓冲中最多能存多少个包，根据前面的计算得出是否需要每个包的描述信息：

{% highlight swift %}
var outAudioBufferList = AudioBufferList()
outAudioBufferList.mNumberBuffers = 1
outAudioBufferList.mBuffers.mNumberChannels = UInt32(channels)
outAudioBufferList.mBuffers.mDataByteSize = outputBufferSize
outAudioBufferList.mBuffers.mData = UnsafeMutableRawPointer(outputBuffer)

var ioOutputDataPackets = outputBufferSize / outputSizePerPacket
statusCode = AudioConverterFillComplexBuffer(audioConverter,
                                             inputDataProc,
                                             UnsafeMutableRawPointer(Unmanaged.passUnretained(self).toOpaque()),
                                             &ioOutputDataPackets,
                                             &outAudioBufferList,
                                             outputPacketDescriptions)
{% endhighlight %}

把输出缓冲写到 AAC 文件：

{% highlight swift %}
let inNumBytes = outAudioBufferList.mBuffers.mDataByteSize
statusCode = AudioFileWritePackets(outputFileId, false, inNumBytes, outputPacketDescriptions, outputFilePosition, &ioOutputDataPackets, outputBuffer)
if statusCode != noErr {
    DDLogError("AudioFileWritePackets failed \(statusCode)")
    exit(1)
}
{% endhighlight %}

**第八步，从 PCM 文件读取 PCM 包到输入缓冲，发送输入缓冲给音频转换器：**

输入缓冲的大小由自己指定，输入的每个包的大小可以是固定的、也可以是变化的，如果是变化的，需要得到每个包的大小的最大值以及初始化每个包的描述信息 AudioStreamPacketDescription 的存储空间：

{% highlight swift %}
inputBuffer = UnsafeMutablePointer<UInt8>.allocate(capacity: Int(inputBufferSize))
if inputAudioStreamFormat.mBytesPerPacket == 0 {
    /*
     if the source format is VBR, we need to get the maximum packet size
     use kAudioFilePropertyPacketSizeUpperBound which returns the theoretical maximum packet size
     in the file (without actually scanning the whole file to find the largest packet,
     as may happen with kAudioFilePropertyMaximumPacketSize)
     */
    size = UInt32(MemoryLayout.size(ofValue: inputSizePerPacket))
    statusCode = AudioFileGetProperty(inputFileId, kAudioFilePropertyPacketSizeUpperBound, &size, &inputSizePerPacket)
    if statusCode != noErr {
        DDLogError("AudioFileGetProperty kAudioFilePropertyPacketSizeUpperBound failed \(statusCode)")
        exit(1)
    }
    
    // How many packets can we read for our buffer size?
    let numPacketsPerRead = inputBufferSize / inputSizePerPacket
    
    // Allocate memory for the PacketDescription structs describing the layout of each packet.
    inputPacketDescriptions = .allocate(capacity: Int(numPacketsPerRead))
} else {
    // CBR source format
    inputSizePerPacket = inputAudioStreamFormat.mBytesPerPacket
    inputPacketDescriptions = nil
}
{% endhighlight %}

计算输入缓冲中最多能存多少个包，和回调请求的包数量进行比较，计算出从 PCM 文件读取包的数量：

{% highlight swift %}
let maxPackets = encoder.inputBufferSize / encoder.inputSizePerPacket
if ioNumberDataPackets.pointee > maxPackets {
    ioNumberDataPackets.pointee = maxPackets
}

var outNumBytes = maxPackets * encoder.inputSizePerPacket

statusCode = AudioFileReadPacketData(encoder.inputFileId, false, &outNumBytes, encoder.inputPacketDescriptions, encoder.inputFilePosition, ioNumberDataPackets, encoder.inputBuffer)
if statusCode != noErr {
    DDLogError("AudioFileReadPacketData failed \(statusCode)")
    exit(1)
}
{% endhighlight %}

发送输入缓冲给给音频转换器，如果输入的每个包的大小不固定，还需要提供每个包描述信息：

{% highlight swift %}
// put the data pointer into the buffer list
ioData.pointee.mBuffers.mData = UnsafeMutableRawPointer(encoder.inputBuffer)
ioData.pointee.mBuffers.mDataByteSize = outNumBytes
ioData.pointee.mBuffers.mNumberChannels = UInt32(encoder.channels)
    
// don't forget the packet descriptions if required
if let outDataPacketDescription = outDataPacketDescription {
    if let inputPacketDescriptions = encoder.inputPacketDescriptions {
        outDataPacketDescription.pointee = inputPacketDescriptions
    } else {
        outDataPacketDescription.pointee = nil
    }
}
{% endhighlight %}

## 测试验证

可以利用 [iOS Audio Unit 录制音频文件和播放音频文件](/programming/2020/07/02/ios-audio-unit-record-play/) 或 [iOS AVAudioEngine 录制音频文件和播放音频文件](/programming/2020/07/03/ios-avaudioengine-record-play/) 录制 PCM 音频文件，然后利用上面代码转换为 AAC 音频文件。

### 日志输出

前面打印过 PCM 格式的信息：

{% highlight text %}
Frames per Packet: 1 // 1 个包中 1 个帧
{% endhighlight %}

前面打印过 AAC 格式的信息：

{% highlight text %}
Frames per Packet: 1024 // 1 个包中 1024 个帧
{% endhighlight %}

这样算下来，1 个 AAC 包需要 1024 个 PCM 包，和如下转换过程输出信息一致：

{% highlight text %}
read 4096 packets with 32768 btyes at position 0
read 4096 packets with 32768 btyes at position 4096
read 4096 packets with 32768 btyes at position 8192
read 4096 packets with 32768 btyes at position 12288
read 4096 packets with 32768 btyes at position 16384
read 1024 packets with 8192 btyes at position 20480
write 21 packets with 6937 btyes at position 0
{% endhighlight %}

### ffprobe

通过 ffprobe 来查看，最直观的可以看到 AAC 文件大小比 PCM 文件小了很多：

{% highlight text %}
$ ffprobe -show_format -pretty audio.caf
Input #0, caf, from 'audio.caf':
  Duration: 00:00:04.90, start: 0.000000, bitrate: 2829 kb/s
    Stream #0:0: Audio: pcm_f32le (lpcm / 0x6D63706C), 44100 Hz, 2 channels, flt, 2822 kb/s
[FORMAT]
filename=audio.caf
nb_streams=1
nb_programs=0
format_name=caf
format_long_name=Apple CAF (Core Audio Format)
start_time=0:00:00.000000
duration=0:00:04.900000
size=1.652542 Mibyte
bit_rate=2.829087 Mbit/s
probe_score=100
[/FORMAT]
{% endhighlight %}

{% highlight text %}
$ ffprobe -show_format -pretty audio.aac
Input #0, aac, from 'audio.aac':
  Duration: 00:00:04.78, bitrate: 167 kb/s
    Stream #0:0: Audio: aac (LC), 44100 Hz, stereo, fltp, 167 kb/s
[FORMAT]
filename=audio.aac
nb_streams=1
nb_programs=0
format_name=aac
format_long_name=raw ADTS AAC (Advanced Audio Coding)
start_time=N/A
duration=0:00:04.775559
size=97.811523 Kibyte
bit_rate=167.786000 Kbit/s
probe_score=51
[/FORMAT]{% endhighlight %}

### [AACParser](https://github.com/danjiang/MediaParser/blob/master/AACParser.cpp)

AACParser 对每一个 ADTS Frame 的输出（截取了部分）：

{% highlight text %}
  Num|     Profile| Frequency|  Channel| Size|
    0|      AAC LC|   44100Hz|        2|   13|
    1|      AAC LC|   44100Hz|        2|  209|
    2|      AAC LC|   44100Hz|        2|  370|
    3|      AAC LC|   44100Hz|        2|  332|
    4|      AAC LC|   44100Hz|        2|  314|
    5|      AAC LC|   44100Hz|        2|  325|
    6|      AAC LC|   44100Hz|        2|  397|
    7|      AAC LC|   44100Hz|        2|  437|
    8|      AAC LC|   44100Hz|        2|  364|
    9|      AAC LC|   44100Hz|        2|  364|
   10|      AAC LC|   44100Hz|        2|  354|
{% endhighlight %}

