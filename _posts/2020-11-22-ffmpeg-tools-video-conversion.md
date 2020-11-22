---
title: FFmpeg 命令行工具 - 格式转换、时间操作、数学函数、元数据和字幕
author: 但江
avatar: danjiang
location: 深圳
category: programming
tags: ffmpeg
---

这篇文章讲解 FFmpeg 命令行工具中使用格式转换、时间操作、数学函数、元数据和字幕相关的功能。

![Editing Video](/images/editing-video.jpg)

## 格式转换

媒体容器是存储多媒体流和相关元数据的特定文件格式，因为音频和视频可以由多种方式进行编码和解码，容器提供了更方便方式来在一个文件中存储多种媒体流，一些容器只能存储音频，还有一些容器只能存储图像，但大多数容器可以存储音频、视频、字幕和元数据等。

如果只是容器转化，编解码器没有变化，可以使用 **-c copy** 或 **-c:a copy** 或 **-c:v copy**：

{% highlight text %}
ffmpeg -i car.mov -q 1 -c copy car.avi
{% endhighlight %}

转码的工作流程：读取文件、解封装、解码、转换参数或经过过滤器、编码、封装、写入文件。

![FFmpeg Transcoding](/images/ffmpeg-transcoding.png)

## 时间操作

使用 **-t** 时长和 **-ss** (seek from start) 来抽取特定时间段的视频：

{% highlight text %}
ffmpeg -i live.mp4 -ss 5 -t 10 clip.mp4
{% endhighlight %}

时间格式有如下两种：

{% highlight text %}
[-]HH:MM:SS[.m...]
[-]S+[.m...]
{% endhighlight %}

改变视频的播放速度使用 **setpts** (set presentation timestamp) 视频滤镜：

{% highlight text %}
ffplay -i car.mov -vf setpts=PTS/3
{% endhighlight %}

改变音频的播放速度使用 **atempo** 音频滤镜：

{% highlight text %}
ffplay -i live.mp3 -af atempo=2
{% endhighlight %}

## 数学函数

在一些参数里面可以使用数学函数。

## 元数据和字幕

### 元数据

{% highlight text %}
ffplay -i live.mp3
{% endhighlight %}

可以看到如下内容：

{% highlight text %}
Input #0, mp3, from 'live.mp3':
  Metadata:
    minor_version   : 512
    major_brand     : isom
    compatible_brands: isomiso2avc1mp41
    information     : {"com.bytedance.info": "{}"}
    comment         : vid:v0300fdb0000btbmkrhucgba10erkqn0
    encoder         : Lavf58.26.100
{% endhighlight %}

### 字幕

字幕可以是外部独立的文件，也可以做为一支流包含在媒体容器中，字幕像音频或者视频一样，有特定的格式，同样需要编解码器，如下从 SRT 格式转换为 ASS 格式：

{% highlight text %}
ffmpeg -i subtitles.srt subtitles.ass
{% endhighlight %}

字幕编码进媒体容器：

{% highlight text %}
ffmpeg -f lavfi -i rgbtestsrc -vf subtitles=rgb.srt rgb.mp4
{% endhighlight %}
