---
title: FFmpeg 命令行工具 - 视频编辑
author: 但江
avatar: danjiang
location: 深圳
category: programming
tags: ffmpeg
---

这篇文章讲解 FFmpeg 命令行工具中使用视频编辑相关的功能：调整视频大小和视频缩放、剪裁视频、填充视频、翻转视频和旋转视频、模糊视频、锐化视频和其他降噪处理、视频叠加（画中画）、视频上添加文字。

![Editing Video](/images/editing-video.jpg)

## 调整视频大小和视频缩放

### 调整视频大小

调整视频大小并不是剪裁，画幅比有可能会变化。

{% highlight text %}
ffmpeg -i input.avi -s 640x480 output.avi
ffmpeg -i input.avi -s vga output.avi
{% endhighlight %}

> **Nyquist sampling theorem**
>
> Its general form is related to any signals and informs that for the complete reconstruction of a sampled signal, we must use at least 2 times higher frequency than is the frequency of the source.

### 视频缩放

通常都是保持画幅比进行缩放，也就是使用 **scale** 视频滤镜：

{% highlight text %}
ffmpeg -i input.mpg -vf scale=iw/2:ih/2 output.mp4
ffmpeg -i input.mpg -vf scale=iw*0.9:ih*0.9 output.mp4
ffmpeg -i input.mpg -vf scale=iw/PHI:ih/PHI output.mp4
{% endhighlight %}

保持长或者宽固定，另一边根据比例缩放：

{% highlight text %}
ffmpeg -i input.avi -vf scale=400:400/a
ffmpeg -i input.avi -vf scale=300*a:300
{% endhighlight %}

## 剪裁视频

使用 **crop** 视频滤镜：

{% highlight text %}
crop=ow[:oh[:x[:y[:keep_aspect]]]]
{% endhighlight %}

{% highlight text %}
ffmpeg -i input -vf crop=iw/3:ih:iw/3:0 output
{% endhighlight %}

从中心点剪裁：

{% highlight text %}
ffmpeg -i input.avi -vf crop=iw/2:ih/2 output.avi
{% endhighlight %}

自动检测剪裁边缘：

{% highlight text %}
ffmpeg -i input.mpg -vf cropdetect=limit=0 output.mp4
{% endhighlight %}

剪裁计时器：

{% highlight text %}
ffmpeg -f lavfi -i testsrc -vf crop=29:52:256:94 -t 10 timer1.mpg
{% endhighlight %}

## 填充视频

使用 **pad** 视频滤镜：

{% highlight text %}
pad=width[:height[:x[:y[:color]]]]
{% endhighlight %}

{% highlight text %}
ffmpeg -i film.mpg -vf pad=ih*16/9:ih:(ow-iw)/2:0 film_wide.avi
{% endhighlight %}

水平填充：

{% highlight text %}
ffmpeg -i input -vf pad=ih*ar:ih:(ow-iw)/2:0:color output
{% endhighlight %}

垂直填充：

{% highlight text %}
ffmpeg -i input -vf pad=iw:iw*ar:0:(oh-ih)/2:color output
{% endhighlight %}

## 翻转视频和旋转视频

### 翻转视频

水平翻转视频，使用 **hflip** 视频滤镜：

{% highlight text %}
ffplay -f lavfi -i testsrc -vf hflip
{% endhighlight %}

垂直翻转视频，使用 **vflip** 视频滤镜：

{% highlight text %}
ffplay -f lavfi -i rgbtestsrc -vf vflip
{% endhighlight %}

### 旋转视频

旋转视频，使用 **transpose** 视频滤镜：

* 0 - input is rotated by 90° counterclockwise and flipped vertically* 1 - input is rotated by 90° clockwise* 2 - input is rotated by 90° counterclockwise* 3 - input is rotated by 90° clockwise and flipped vertically

{% highlight text %}
ffplay -f lavfi -i smptebars -vf transpose=0
ffplay -f lavfi -i smptebars -vf transpose=2,vflip
{% endhighlight %}

## 模糊视频、锐化视频和其他降噪处理

### 模糊视频

box blur，使用 **boxblur** 视频滤镜：

{% highlight text %}
ffmpeg -i input.mpg -vf boxblur=1.5:1 output.mp4
{% endhighlight %}

smart blur，使用 **smartblur** 视频滤镜：

{% highlight text %}
ffmpeg -i halftone.jpg -vf smartblur=5:0.8:0 blurred_halftone.png
{% endhighlight %}

### 锐化视频

锐化视频使用 **unsharp** 视频滤镜：

{% highlight text %}
ffmpeg -i input -vf unsharp output.mp4
{% endhighlight %}

### 其他降噪处理

{% highlight text %}
ffmpeg -i input.mpg -vf mp=denoise3d output.webm
ffmpeg -i input.avi -vf hqdn3d output.mp4
ffplay -i input.avi -nr 500
{% endhighlight %}

## 视频叠加（画中画）

画中画使用 **overlay** 视频滤镜：

{% highlight text %}
ffmpeg -i pair.mp4 -i logo.png -filter_complex overlay=W-w:H-h pair3.mp4
ffmpeg -i input1 -vf movie=input2[logo];[in][logo]overlay=x:y output
{% endhighlight %}

通过 **-itsoffset** 指定前景出现的时间：

{% highlight text %}
ffmpeg -i video_with_timer.mp4 -itsoffset 5 -i logo.png -filter_complex overlay timer_with_logo.mp4
{% endhighlight %}

## 视频上添加文字

在编译 FFmpeg 前，要配置 **--enable-libfreetype**

{% highlight text %}
./configure --enable-gpl --enable-nonfree --enable-libfdk-aac --enable-libmp3lame --enable-libx264 --enable-libx265 --enable-libfreetype
make
make install
{% endhighlight %}

添加文字使用 **drawtext** 视频滤镜，添加静态文字如下：

{% highlight text %}
ffmpeg -i live.mp4 -vf drawtext=fontfile=SourceCodePro-Bold.ttf:text=Welcome:fontsize=48:fontcolor=red live_text.mp4
{% endhighlight %}

添加动态文字会使用到 **t** 时间参数，代表当前秒数：

{% highlight text %}
ffmpeg -i car.mov -vf drawtext="fontfile=SourceCodePro-Bold.ttf:textfile=info.txt:x=w-t*50:y=h-th:fontcolor=blue:fontsize=30" text.mp4
{% endhighlight %}