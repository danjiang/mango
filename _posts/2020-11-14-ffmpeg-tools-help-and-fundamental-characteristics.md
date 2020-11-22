---
title: FFmpeg 命令行工具 - 帮助和码率、帧率和文件大小
author: 但江
avatar: danjiang
location: 深圳
category: programming
tags: ffmpeg
---

这篇文章讲解 FFmpeg 命令行工具中如何查看帮助，以及关于码率、帧率和文件大小的概念和用法。

![Editing Video](/images/editing-video.jpg)

## 帮助

基础帮助和完整帮助

{% highlight text %}
ffmpeg -h
ffmpeg -h full
{% endhighlight %}

针对特定主题的帮助，比如编码器

{% highlight text %}
ffmpeg -bsfs
ffmpeg -codecs
ffmpeg -decoders
ffmpeg -encoders
ffmpeg -filters
ffmpeg -formats
ffmpeg -layouts
ffmpeg -L
ffmpeg -pix_fmts
ffmpeg -protocols
ffmpeg -sample_fmts
ffmpeg -version
{% endhighlight %}

针对特定条目的帮助，比如 FLV 解码器

{% highlight text %}
ffmpeg -h decoder=flv
{% endhighlight %}

## 码率、帧率和文件大小

### 帧率

帧率是每秒的帧数，决定流畅度，人眼至少需要帧率为 15 才能感知到连续动画。

帧率通过 **-r** 来指定：

{% highlight text %}
ffmpeg -i input.avi -r 30 output.mp4
{% endhighlight %}

### 码率

码率是每秒的比特数，决定质量。

* Average bit rate, ABR, 平均码率：每秒处理的平均比特数量。
* Constant bit rate, CBR, 固定码率：通常用于流媒体，而不是文件。
* Variable bit rate, VBR, 变化码率：每秒处理的比特数量是变化的，运动画面需要更多的比特数量，静止画面需要较少的比特数量，同样的文件大小，VBR 比 CBR 的画面质量更好，但也需要更多运算时间和运算资源。

码率通过 **-b** 来指定， **-b:a** 针对音频，**-b:v** 针对视频：

{% highlight text %}
ffmpeg -i input.avi -r 30 output.mp4
{% endhighlight %}

CBR 设置

**-b**、**-minrate**、**-maxrate** 三项需要设置相同的值，**-maxrate** 还需要 **-bufsize** 来控制缓存大小：

{% highlight text %}
ffmpeg -i in.avi -b 0.5M -minrate 0.5M -maxrate 0.5M -bufsize 1M out.mkv
{% endhighlight %}

### 文件大小

输出文件大小通过 **-fs** 来指定：

{% highlight text %}
ffmpeg -i input.avi -fs 10MB output.mp4
{% endhighlight %}

编码视频的大小计算：

{% highlight text %}
video_size = video_bitrate * time_in_seconds / 8
{% endhighlight %}

没有编码音频的大小计算：

{% highlight text %}
audio_size = sampling_rate * bit_depth * channels * time_in_seconds / 8
{% endhighlight %}

编码音频的大小计算：

{% highlight text %}
audio_size = bitrate * time_in_seconds / 8
{% endhighlight %}

For example to calculate the final size of 10-minutes video clip with the 1500 kbits/s video bit rate and 128 kbits/s audio bitrate, we can use the equations:

{% highlight text %}
file_size = video_size + audio_size
file_size = (video_bitrate + audio_bitrate) * time_in_seconds / 8
file_size = (1500 kbit/s + 128 kbits/s) * 600 s
file_size = 1628 kbit/s * 600 s
file_size = 976800 kb = 976800000 b / 8 = 122100000 B / 1024 = 119238.28125 KB
file_size = 119238.28125 KB / 1024 = 116.443634033203125 MB ≈ 116.44 MB
{% endhighlight %}
