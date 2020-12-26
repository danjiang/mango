---
title: FFmpeg 接口使用 - 播放器教程和示例代码
author: 但江
avatar: danjiang
location: 深圳
category: programming
tag: ffmpeg
---

本文介绍学习 FFmpeg 接口使用的相关教程和示例代码，工程师掌握一门技术比较好的方式就是阅读代码和造个小轮子。

![Editing Video](/images/editing-video.jpg)

## 播放器教程

> [An FFmpeg and SDL Tutorial](http://dranger.com/ffmpeg/) by Stephen Dranger, explains how to write a video player based on FFmpeg.

参照上面的教程编写了一个 <em class="fab fa-github"></em> [视频播放器](https://github.com/danjiang/FFmpegAndSDLTutorial)，是在 CLion 下建立的 C 工程，在 Mac 上已经编译和安装了 FFmpeg 和 SDL，在配置 CMake 的时候费了不少时间，工程中的音频播放有问题，需要进行从 fltp 到 s16 的采样格式转换，教程中的 [Tutorial 06: Synching Audio](http://dranger.com/ffmpeg/tutorial06.html) 和 [Tutorial 07: Seeking](http://dranger.com/ffmpeg/tutorial07.html) 也没有完成，不过还是有不少收获。

## 示例代码

[FFmpeg 从入门到精通](https://book.douban.com/subject/30178432/) 中介绍的示例代码总结如下：

### libavformat

* <em class="fab fa-github"></em> [音视频流封装](https://github.com/FFmpeg/FFmpeg/tree/master/doc/examples/muxing.c)
* <em class="fab fa-github"></em> [音视频文件解封装](https://github.com/FFmpeg/FFmpeg/tree/master/doc/examples/demuxing_decoding.c)
* <em class="fab fa-github"></em> [音视频文件转封装](https://github.com/FFmpeg/FFmpeg/tree/master/doc/examples/remuxing.c)
* <em class="fab fa-github"></em> [视频截取 av_seek_frame](https://github.com/FFmpeg/FFmpeg/tree/master/doc/examples/remuxing.c)	
* <em class="fab fa-github"></em> [avio 内存数据操作](https://github.com/FFmpeg/FFmpeg/tree/master/doc/examples/avio_reading.c)

### libavcodec

#### 旧接口 < 3.1.3

* <em class="fab fa-github"></em> [视频解码](https://github.com/FFmpeg/FFmpeg/blob/release/2.3/doc/examples/demuxing_decoding.c)
* <em class="fab fa-github"></em> [视频编码](https://github.com/FFmpeg/FFmpeg/blob/release/2.3/doc/examples/decoding_encoding.c)
* <em class="fab fa-github"></em> [音频解码](https://github.com/FFmpeg/FFmpeg/blob/release/2.3/doc/examples/demuxing_decoding.c)
* <em class="fab fa-github"></em> [音频编码](https://github.com/FFmpeg/FFmpeg/blob/release/2.3/doc/examples/muxing.c)

#### 新接口 > 3.1.3

* <em class="fab fa-github"></em> [视频解码](https://github.com/FFmpeg/FFmpeg/tree/master/doc/examples/decode_video.c)
* <em class="fab fa-github"></em> [视频编码](https://github.com/FFmpeg/FFmpeg/tree/master/doc/examples/encode_video.c)
* <em class="fab fa-github"></em> [音频解码](https://github.com/FFmpeg/FFmpeg/tree/master/doc/examples/decode_audio.c)
* <em class="fab fa-github"></em> [音频编码](https://github.com/FFmpeg/FFmpeg/tree/master/doc/examples/encode_audio.c)

### libavfilter

#### Filter 的类型

* Source Filter 没有输入端
* Sink Filter 没有输出端
* Filter 既有输入端又有输出端
		
#### 过滤器使用的示例代码
	
* <em class="fab fa-github"></em> [视频过滤器](https://github.com/FFmpeg/FFmpeg/tree/master/doc/examples/filtering_video.c)
