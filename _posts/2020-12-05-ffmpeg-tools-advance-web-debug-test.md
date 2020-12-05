---
title: FFmpeg 命令行工具 - 高级特性、Web 音视频、调试和测试
author: 但江
avatar: danjiang
location: 深圳
category: programming
tags: ffmpeg
---

这篇文章讲解 FFmpeg 命令行工具中使用高级特性、Web 音视频、调试和测试相关的功能。

![Editing Video](/images/editing-video.jpg)

## 高级特性

### 拼接视频文件

多个视频文件按顺序拼接在一起：

{% highlight text %}
ffmpeg -i city.mp4 -i car.mp4 -filter_complex concat video.mp4
{% endhighlight %}

### 移除水印

通过 **delogo** 过滤器来处理：

{% highlight text %}
ffmpeg -i eagles.mpg -vf delogo=x=700:y=0:w=100:h=50:t=3:show=1 nologo.mpg
{% endhighlight %}

### 修复抖动

通过 **deshake** 过滤器来处理：

{% highlight text %}
ffmpeg -i travel.avi -vf deshake fixed_travel.avi
{% endhighlight %}

### 添加颜色框

通过 **drawbox** 过滤器来处理：

{% highlight text %}
ffmpeg -i ship.avi -vf drawbox=x=150:w=600:h=400:c=yellow ship1.avi
{% endhighlight %}

### 帧数检测

{% highlight text %}
ffmpeg -i city.mp4 -f null /dev/null
{% endhighlight %}

输出结果中 **frame** 就是帧数：

{% highlight text %}
frame=  990 fps=833 q=-0.0 Lsize=N/A time=00:00:33.00 bitrate=N/A speed=27.8x
video:518kB audio:5684kB subtitle:0kB other streams:0kB global headers:0kB muxing overhead: unknown
{% endhighlight %}

### 检测 dark frames

通过 **blackdetect** 过滤器来处理：

{% highlight text %}
ffmpeg -f lavfi -i mptestsrc -vf blackdetect -f sdl 'test'
{% endhighlight %}

通过 **blackframe** 过滤器来处理：

{% highlight text %}
ffmpeg -f lavfi -i mptestsrc -vf blackframe -f sdl 'test'
{% endhighlight %}

### 选择特定的帧来输出

通过 **select** 过滤器来处理：

{% highlight text %}
ffmpeg -i input.avi -vf select="gte(t\,20)*lte(t\,25)" output.avi
{% endhighlight %}

### 缩放视频

通过 **setdar** 或者 **setsar** 过滤器来设置屏幕高宽比进行缩放：

{% highlight text %}
ffplay -i input.avi -vf setdar=16/9
ffplay -i input.avi -vf setsar=1.234
{% endhighlight %}

### 显示视频帧的详细信息

通过 **showinfo** 过滤器：

{% highlight text %}
ffmpeg -report -f lavfi -i testsrc -vf showinfo -t 10 showinfo.mpg
{% endhighlight %}

### 音频频谱

通过 **showspectrum** 过滤器：

{% highlight text %}
ffmpeg -i audio.mp3 -vf showspectrum audio_spectrum.mp4
{% endhighlight %}

### 音频波形显示

通过 **showwaves** 过滤器：

{% highlight text %}
ffmpeg -i music.mp3 -vf showwaves waves.mp4
{% endhighlight %}

### 一次生成多个格式

{% highlight text %}
ffmpeg -i clip.avi clip.flv clip.mov clip.mp3 clip.mp4 clip.webm
{% endhighlight %}

### 过滤器图使用额外媒体输入

{% highlight text %}
ffmpeg -i video.mpg -vf movie=logo.png:sp=5[a];[in][a]overlay video1.mp4
{% endhighlight %}

## Web 音视频

### 浏览器

主流浏览器对 H5 音视频格式的支持：

![Web Browser Video](/images/web-browser-video.jpg)

### HTML5 Audio

{% highlight html %}
<audio controls='controls' loop='loop'>
  <source src='music.mp3' type='audio/mpeg' />
  <source src='music.ogg' type='audio/ogg' />
  Audio element is not supported in your browser, please update.
</audio>
{% endhighlight %}

### HTML5 Video

{% highlight html %}
<video controls='controls' loop='loop' width='640' height='480'>
  <source src='videoclip.mp4' type='video/mp4' />
  <source src='videoclip.webm' type='video/webm' />
  video element is not supported in your browser, please update.
</video>
{% endhighlight %}

## 调试和测试

### 生成测试报告

带上 **-report** 参数就可以生成测试报告 ffmpeg-yyyymmdd-hhmmss.log：

{% highlight text %}
ffmpeg -report -i live.mp4 -vf showinfo -t 10 showinfo.mpg
{% endhighlight %}

### 调试

带上 **-debug** 参数，后面再跟上具体类型：

{% highlight text %}
ffmpeg -debug mmco -f lavfi -i mptestsrc -t 0.5 output.mp4
{% endhighlight %}

带上 **-debug_ts** 参数，在处理过程中会打印出时间戳信息：

{% highlight text %}
ffmpeg -debug_ts -f lavfi -i mptestsrc -t 0.1 output.mp4
{% endhighlight %}

带上 **-fdebug ts** 参数，在处理过程中会打印出更多时间戳信息：

{% highlight text %}
ffmpeg -fdebug ts -f lavfi -i mptestsrc -t 0.1 output.mp4
{% endhighlight %}