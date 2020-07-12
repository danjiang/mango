---
title: FFmpeg 命令行工具 - 基础
author: 但江
avatar: danjiang
location: 成都 
category: programming
tags: ffmpeg featured
---

作为开发者，使用 FFmpeg 主要分两部分：命令行工具和接口使用，本文讲解如何在 macOS 上编译 FFmpeg，以及 FFmpeg 命令行工具使用的一些基本概念。

![Editing Video](/images/editing-video.jpg)

## 编译

针对 macOS 平台来编译。

首先需要安装 [Xcode](https://developer.apple.com/xcode/)，这样才有编译工具链 LLVM。

然后需要安装汇编编译工具，[Yasm](https://yasm.tortall.net) 或者 [NASM](https://www.nasm.us)：

{% highlight text %}
wget http://www.tortall.net/projects/yasm/releases/yasm-1.3.0.tar.gz
tar xvzf yasm-1.3.0.tar.gz
cd yasm-1.3.0
./configure
make
make install
{% endhighlight %}

{% highlight text %}
brew install nasm
{% endhighlight %}

如果希望通过 ffplay 来播放音视频文件，还需要安装 [SDL](https://www.libsdl.org)：

{% highlight text %}
brew install sdl2
{% endhighlight %}

在编译 FFmpeg 前，可以通过如下命令来配置要支持和禁用的一些功能模块：

{% highlight text %}
./configure --disable-xxx --enable-xxx
{% endhighlight %}

查看有哪些编码格式：

{% highlight text %}
./configure --list-encoders
./configure --list-decoders
{% endhighlight %}

查看有哪些文件封装格式：

{% highlight text %}
./configure --list-muxers
./configure --list-demuxers
{% endhighlight %}

查看有哪些流媒体传输协议：

{% highlight text %}
./configure --list-protocols
{% endhighlight %}

最后编译 FFmpeg：

{% highlight text %}
make
make install
{% endhighlight %}

## 命令行工具

### 官方文档

* <em class="fas fa-file-alt"></em> [FFmpeg Documentaion](https://ffmpeg.org/documentation.html)
* <em class="fas fa-file-alt"></em> [FFmpeg WIKI](https://trac.ffmpeg.org)
* <em class="fas fa-file-alt"></em> [FFmpeg Command Line Tools Documentation](https://ffmpeg.org/documentation.html)

### 其他资料

* <em class="fas fa-book"></em> [FFmpeg Basics](http://ffmpeg.tv)
* <em class="fas fa-book"></em> [FFmpeg 从入门到精通](https://book.douban.com/subject/30178432/)

### 命令行工具的介绍

* ffmpeg 音视频编解码。
* ffplay 音视频播放和可视化分析，对于视频播放器，不得不提的一个问题就是音画同步，在 ffplay 中音画同步的实现方式其实有三种，分别是：
	* 以音频为主时间轴作为同步源；
	* 以视频为主时间轴作为同步源；
	* 以外部时钟为主时间轴作为同步源。
* ffprobe 音视频内容分析。

## 命令基本格式

{% highlight text %}
ffmpeg [global options] [input file options] -i input_file [output file options] output_file
{% endhighlight %}

{% highlight text %}
ffmpeg -y -i video.avi -s vga video.mp4
{% endhighlight %}

## 显示输出预览

需要先安装 [Simple DirectMedia Layer](https://www.libsdl.org)，简称 SDL。这样就可以不用先将输入文件转换为输出文件后，再播放输出文件来看效果。

{% highlight text %}
ffmpeg -f lavfi -i rgbtestsrc -pix_fmt yuv420p -f sdl Example
{% endhighlight %}

## 转码的工作流程

工作流程：读取文件、解封装、解码、转换参数或经过过滤器、编码、封装、写入文件。

![FFmpeg Transcoding](/images/ffmpeg-transcoding.png)

## 过滤器、过滤器链、过滤器图

简单过滤器

![FFmpeg Simple Filter](/images/ffmpeg-simple-filter.png)

{% highlight text %}
ffplay -f lavfi -i testsrc -vf transpose=1
{% endhighlight %}

复杂过滤器

![FFmpeg Complex Filter](/images/ffmpeg-complex-filter.png)

{% highlight text %}
ffmpeg -i liveshow.mp4 -i lighthouse.png -filter_complex overlay=w output.mp4
{% endhighlight %}

一个过滤器图由多个过滤器链组成，通过 **;** 分割，一个过滤器链由多个过滤器组成，通过 **,** 分割。

下面的示例，第 1 个过滤器链将输入分割成 [a] 和 [b]，第 2 个过滤器链将 [a] 通过 pad 处理变成 [1]，第 3 个过滤器链将 [b] 通过 vflip 处理变成 [2]，第 4 个过滤器链将 [1] 和 [2] 合并为一个画面：

{% highlight text %}
ffplay -f lavfi -i rgbtestsrc -vf "split[a][b];[a]pad=2*iw[1];[b]vflip[2];[1][2]overlay=w"
{% endhighlight %}

## 流选择

一个视频文件可能包含多路流，通过 -map 去选择流，文件序号:流类型:流序号，序号从 0 开始，流类型有 a 视频、d 数据、s 字幕、t 附件、v 视频。

{% highlight text %}
ffplay -i A.mov -i B.mov -i C.mov -map 0:v:0 -map 1:a:0 -map 2:s:0 clip.mov
{% endhighlight %}

## 指定数据大小

比特大小

{% highlight text %}
ffmpeg -i input.avi -b:v 1500000 output.mp4
ffmpeg -i input.avi -b:v 1500K output.mp4
ffmpeg -i input.avi -b:v 1.5M output.mp4
ffmpeg -i input.avi -b:v 0.0015G output.mp4
{% endhighlight %}

字节大小

{% highlight text %}
ffmpeg -i input.mpg -fs 10MB output.mp4
{% endhighlight %}

## 指定颜色

格式为 0xRRGGBB[@AA] 和 #RRGGBB[@AA]，@ 后面的代表透明度。还有内建的如 red, black 等这样的颜色名称。

## Lavfi 虚拟设备

通过 Lavfi 虚拟设备产生输入视频。

{% highlight text %}
ffplay -f lavfi -i smptebars

ffplay -f lavfi -i color=c=blue
{% endhighlight %}

