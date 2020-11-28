---
title: FFmpeg 命令行工具 - 预设、交错式视频、采集设备和颜色调整
author: 但江
avatar: danjiang
location: 深圳
category: programming
tags: ffmpeg
---

这篇文章讲解 FFmpeg 命令行工具中预设、交错式视频、采集设备和颜色调整相关的功能。

![Editing Video](/images/editing-video.jpg)

## 预设

将一些配置信息做为键值对放在 **.ffpreset** 文件中：

{% highlight text %}
vcodec=flv # video codec
b:v=300k # video bitrate
g=160 # group of picture size
mbd=2 # macroblock decision algorithm
flags=+aic+mv0+mv4 # aic - h263 advanced intra coding; always try a mb with mv=<0,0>; mv4 - use 4 motion vector by macroblock
trellis=1 # rate-distortion optimal quantization
ac=1 # number of audio channels
ar=22050 # audio sampling rate
b:a=56k # audio bitrate
{% endhighlight %}

使用预设文件：

{% highlight text %}
ffmpeg -i input.avi -f flv -r 29.97 -vf scale=320:240 -aspect 4:3 -cmp dct -subcmp dct -fpre flv.ffpreset output.flv
{% endhighlight %}

## 交错式视频 Interlaced Video

An interlacing is a technology invented during development of monochrome analog TV to eliminate flicker of old CRT monitors. The video frame is divided horizontally to regular lines and then to 2 fields, where the first field contains odd lines and the second field contains even lines.

通过视频过滤器来进行转换。

{% highlight text %}
ffmpeg -i input.vob -vf setfield=tff output.mov
{% endhighlight %}

## 采集设备

### 查看设备列表

macOS 下查看设备列表：

{% highlight text %}
ffmpeg -devices
{% endhighlight %}

输出结果：

{% highlight text %}
Devices:
 D. = Demuxing supported
 .E = Muxing supported
 --
 D  avfoundation    AVFoundation input device
 D  lavfi           Libavfilter virtual input device
  E sdl,sdl2        SDL2 output device
{% endhighlight %}

macOS 下查看 avfoundation 支持的设备列表：

{% highlight text %}
ffmpeg -f avfoundation -list_devices true -i ""
{% endhighlight %}

输出结果：

{% highlight text %}
[AVFoundation input device @ 0x7fc6acf04cc0] AVFoundation video devices:
[AVFoundation input device @ 0x7fc6acf04cc0] [0] FaceTime高清摄像头（内建）
[AVFoundation input device @ 0x7fc6acf04cc0] [1] Capture screen 0
[AVFoundation input device @ 0x7fc6acf04cc0] AVFoundation audio devices:
[AVFoundation input device @ 0x7fc6acf04cc0] [0] Built-in Microphone
{% endhighlight %}

### 采集内置摄像头

macOS 下采集内置摄像头到视频文件：

{% highlight text %}
ffmpeg -f avfoundation -framerate 30 -pixel_format yuyv422 -i "FaceTime高清摄像头（内建）" out.mp4
{% endhighlight %}

macOS 下预览内置摄像头：

{% highlight text %}
ffplay -f avfoundation -framerate 30 -pixel_format yuyv422 -i "FaceTime高清摄像头（内建）"
{% endhighlight %}

### 采集桌面

macOS 下采集桌面到视频文件：

{% highlight text %}
ffmpeg -f avfoundation -pixel_format uyvy422 -i "Capture screen 0" -r:v 30 out.mp4
{% endhighlight %}

带上鼠标的图像：

{% highlight text %}
ffmpeg -f avfoundation -pixel_format uyvy422 -capture_cursor 1 -i "Capture screen 0" -r:v 30 out.mp4
{% endhighlight %}

### 采集麦克风

macOS 下采集麦克风到音频文件：

{% highlight text %}
ffmpeg -f avfoundation -framerate 30 -pixel_format yuyv422 -i "0:0" -t 10 mic.aac
{% endhighlight %}

### 其他操作系统

参考 [FFmpeg 从入门到精通 - 第 7 章 FFmpeg 采集设备](https://book.douban.com/subject/30178432/)

## 颜色调整

### LUT (Lookup Table)

输入的像素格式可以是 YUV 或者 RGB：

{% highlight text %}
ffplay -f lavfi -i smptebars -vf lut=c1=128:c2=128
{% endhighlight %}

输入的像素格式是 YUV：

{% highlight text %}
ffplay -f lavfi -i smptebars -vf lutyuv=u=128:v=128
{% endhighlight %}

输入的像素格式是 RGB：

{% highlight text %}
ffplay -f lavfi -i rgbtestsrc -vf lutrgb=r=0:g=0
{% endhighlight %}

### 调整强度 Intensity

{% highlight text %}
ffplay -f lavfi -i rgbtestsrc -vf lutrgb=b=val*2
{% endhighlight %}

### 调整明亮度 Brightness

{% highlight text %}
ffplay -f lavfi -i rgbtestsrc -vf lutyuv=y=val*0.9
{% endhighlight %}

### 调整色相 Hue

{% highlight text %}
ffplay -i coconut.jpg -vf hue=60
{% endhighlight %}

### 调整饱和度 Saturation

{% highlight text %}
ffplay -i strawberry.jpg -vf hue=s=5
{% endhighlight %}

> 可以通过过滤器 **overlay** 组成过滤器图来同时显示多个效果进行比较。