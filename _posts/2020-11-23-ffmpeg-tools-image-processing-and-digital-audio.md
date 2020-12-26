---
title: FFmpeg 命令行工具 - 图像处理和数字音频
author: 但江
avatar: danjiang
location: 深圳
category: programming
tags: ffmpeg
---

这篇文章讲解 FFmpeg 命令行工具中使用图像处理和数字音频相关的功能。

![Editing Video](/images/editing-video.jpg)

## 图像处理

### 视频截图

静态图片：

{% highlight text %}
ffmpeg -i videoclip.avi -ss 01:23:45 image.jpg
{% endhighlight %}

动态图片：

{% highlight text %}
ffmpeg -i promotion.swf -pix_fmt rgb24 promotion.gif
{% endhighlight %}

### 内置视频源

![FFmpeg Build In Video Sources](/images/ffmpeg-build-in-video-sources.jpg)

使用内置视频源创建静态图片：

{% highlight text %}
ffmpeg -f lavfi -i color=c=red:s=728x90 -frames:v 1 leaderboard.png
{% endhighlight %}

### 视频和序列图片之间的转换

视频转换为序列图片：

{% highlight text %}
ffmpeg -i clip.avi frame%d.jpg
{% endhighlight %}

序列图片转换为视频：

{% highlight text %}
ffmpeg -f image2 -i img%d.jpg -r 25 video.mp4
{% endhighlight %}

### 图像编辑

可以使用 [FFmpeg 命令行工具 - 视频编辑](/programming/2020/11/21/ffmpeg-tools-video-editing/) 中讲解的视频过滤器来对图片进行编辑。

### 图片格式转换

PNG 格式转换为 JPG 格式：

{% highlight text %}
ffmpeg -i illustration.png illustration.jpg
{% endhighlight %}

## 数字音频

通过 [数字音频、PCM 音频格式和 AAC 音频格式](/programming/2020/12/18/digital-audio-pcm-aac/) 来了解这些理论知识。

常见的音频采样率：

![FFmpeg Digital Audio Sample Rate](/images/ffmpeg-digital-audio-sample-rate.jpg)

常见的音频量化格式：

![FFmpeg Digital Audio Quantization](/images/ffmpeg-digital-audio-quantization.jpg)

常见的音频格式：

![FFmpeg Digital Audio Format](/images/ffmpeg-digital-audio-format.jpg)

生产单一音调的音频：

{% highlight text %}
ffmpeg -f lavfi -i aevalsrc="sin(440*2*PI*t)" -t 10 noteA4.mp3
{% endhighlight %}

生产左右两个音调的音频：

{% highlight text %}
ffplay -f lavfi -i aevalsrc="sin(261.63*2*PI*t)|cos(523.25*2*PI*t):c=FL+FR"
{% endhighlight %}

通过 **volume** 音频过滤器调节音量：

{% highlight text %}
ffmpeg -i live.mp3 -af volume=1/2 quiet_music.mp3
ffmpeg -i sound.aac -af volume=10dB louder_sound.aac
{% endhighlight %}

通过 **pan** 音频过滤器可以进行单声道和立体声之间的转换。

通过 **-map_channel** 选项可以指定到文件的音频流的声道：

{% highlight text %}
-map_channel [in_file_id.stream_spec.channel_id|-1][:out_file_id.stream_spec]
{% endhighlight %}