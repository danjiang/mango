---
title: 数字音频、PCM 音频格式和 AAC 音频格式
author: 但江
avatar: danjiang
location: 深圳
category: programming
tag: ios-av
---

本文讲解了数字音频、PCM 音频格式和 AAC 音频格式这些理论知识。

![Camera Sea](/images/camera-sea.jpg)

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

![AAC ADTS Sequence](/images/aac-adts-sequence.png)

AAC 是压缩数据的音频编码，并且多用于视频中的音频编码，参考 [视音频数据处理入门：AAC音频码流解析](https://blog.csdn.net/leixiaohua1020/article/details/50535042) 的代码和 [音视频封装格式：AAC音频基础和ADTS打包方案详解](https://mp.weixin.qq.com/s?__biz=MzI0NTMxMjA1MQ==&mid=2247483717&idx=1&sn=b3f11c98f5cdf99753a07fb461d5d2a5&chksm=e9513e19de26b70f437397e6430be75b09cb8d93d82623be0943cd9b9b6c4e07fc3686b9b151&scene=21#wechat_redirect) 对 AAC 格式的详细解读，我编写了 [AACParser](https://github.com/danjiang/MediaParser/blob/master/AACParser.cpp) 来解析 AAC 音频文件。