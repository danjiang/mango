---
title: FFmpeg 接口使用 - 基础和转封装
author: 但江
avatar: danjiang
location: 成都 
category: programming
tag: ffmpeg
---

作为开发者，使用 FFmpeg 主要分两部分：命令行工具和接口使用，本文讲解如何在 macOS 上交叉编译 FFmpeg，再将其集成到 Xcode 中，再初步介绍 FFmpeg 接口使用时会用到的常用结构，最后实际编写音视频文件转封装的代码。

![Editing Video](/images/editing-video.jpg)

## 交叉编译和集成

利用如下脚本交叉编译 ffmpeg 2.3 + libfdk_aac 0.1.5 + x264-snapshot-20160814-2245-stable：

* <em class="fab fa-github"></em> [FFmpeg-iOS-build-script](https://github.com/danjiang/FFmpeg-iOS-build-script)
* <em class="fab fa-github"></em> [fdk-aac-build-script-for-iOS](https://github.com/danjiang/fdk-aac-build-script-for-iOS)
* <em class="fab fa-github"></em> [x264-ios](https://github.com/danjiang/x264-ios)

iOS 支持的架构配置信息如下，上面的交叉编译就需要编译支持的架构：

![Xcode Architectures](/images/xcode-architectures.jpg)

Build Settings 里面的 Architectures 选项。Architectures 指的是该 App 支持的指令集，一般情况下，在 Xcode 中新建一个项目，其默认的 Architectures 选项值是 Standard architectures（armv7、arm64），表示该 App 仅支持 armv7 和 arm64 的指令集；Valid Architectures 选项指即将编译的指令集，一般设置为 armv7、armv7s、arm64，表示一般会编译这三个指令集；Build Active Architecture Only 选项表示是否只编译当前适用的指令集，一般情况下在 Debug 的时候设置为 YES，以便可以更加快速、高效地调试程序，而在 Release 的情况下设置为 NO，以便 App 在各个机器上都能够以最高效率运行，因为 Valid Architectures 选择的对应指令集是 armv7、armv7s 和 arm64，在 Release 下会为各个指令集编译对应的代码，因此最后的 ipa 体积基本上翻了 3 倍。

把交叉编译好的头文件和库文件拷贝到如下目录：

![DTCamera ThirdParty](/images/DTCamera-ThirdParty.jpg)

头文件搜索配置：

![Xcode Header Search Paths](/images/xcode-header-search-paths.jpg)

库文件搜索配置：

![Xcode Library Search Paths](/images/xcode-library-search-paths.jpg)

交叉编译 FFmpeg 挺麻烦的，有人已经整理了一个项目 <em class="fab fa-github"></em> [mobile-ffmpeg](https://github.com/tanersener/mobile-ffmpeg) 来解决这个问题，我还没有使用过。

## 接口使用

### 官方文档

* <em class="fas fa-file-alt"></em> [FFmpeg Documentaion](https://ffmpeg.org/documentation.html)
* <em class="fas fa-file-alt"></em> [FFmpeg WIKI](https://trac.ffmpeg.org)
* <em class="fas fa-file-alt"></em> [FFmpeg API Documentation](https://ffmpeg.org/documentation.html)

### 代码和资料

* <em class="fab fa-github"></em> [FFmpeg Examples](https://github.com/FFmpeg/FFmpeg/tree/master/doc/examples)
* <em class="fas fa-code"></em> [An FFmpeg and SDL Tutorial](http://dranger.com/ffmpeg/)
* <em class="fab fa-github"></em> [ffmpeg-libav-tutorial](https://github.com/leandromoreira/ffmpeg-libav-tutorial)
* <em class="fas fa-book"></em> [FFmpeg 从入门到精通](https://book.douban.com/subject/30178432/)

### 库的介绍

下图中，实线是强制依赖，虚线是选择依赖。

![FFmpeg Libraries Dependencies](/images/ffmpeg-libraries-dependencies.jpg)

* libavformat 封装模块
* libavcodec 编解码模块
* libavfilter 滤镜模块
* libswscale 视频图像缩放、颜色转换模块
* libswresample 音频采样率转换模块
* libavutil 多媒体编程工具模块
* libavdevice 多媒体设备输入输出模块

### 常用结构

![FFmpeg Common Structs](/images/ffmpeg-common-structs.png)

**AVFormatContext** 是 API 层直接接触到的结构体，它会进行格式的封装与解封装，它的数据部分由底层提供，底层使用了 **AVIOContext**，这个 AVIOContext 实际上就是为普通的 I/O 增加了一层 Buffer 缓冲区，再往底层就是 **URLContext**，也就是到达了协议层，协议层的具体实现有很多，包括 rtmp、http、hls、file 等。

AVFormatContext 中的 **AVInputFormat** 对应于解封装时的输入容器格式，**AVOutputFormat** 对应于封装时的输出容器格式，同一时间，两者只能有一个存在。

AVFormatContext 就是对容器或者说媒体文件层次的一个抽象，该文件中（或者说在这个容器里面）包含了多路流（音频流、视频流、字幕流等），对流的抽象就是 **AVStream**；在每一路流中都会描述这路流的编码格式，对编解码格式以及编解码器的抽象就是 **AVCodecContext** 与 **AVCodec**；对于编码器或者解码器的输入输出部分，也就是压缩数据以及原始数据的抽象就是 **AVPacket** 与 **AVFrame**。

## 音视频文件转封装

### 示例代码

* <em class="fab fa-github"></em> [video_remuxer.cpp](https://github.com/danjiang/DTCamera/blob/master/DTCamera/CrossPlatform/FFmpeg/video_remuxer.cpp)，可在 iOS 中对 mp4 和 flv 互相转换。
* <em class="fab fa-github"></em> [remuxing.c](https://github.com/FFmpeg/FFmpeg/blob/release/2.3/doc/examples/remuxing.c)，FFmpeg 2.3 转封装的示例代码。

### extern \"C\" 的解释

{% highlight cpp %}
extern "C" {
    #include "libavformat/avformat.h"
    #include "libavcodec/avcodec.h"
    #include "libswresample/swresample.h"
    #include "libavutil/avutil.h"
}
{% endhighlight %}

作为一种面向对象的语言，C++ 支持函数的重载，而面向过程的 C 语言是不支持函数重载的。同一个函数在 C++ 中编译后与其在 C 中编译后，在符号表中的签名是不同的，假如对于同一个函数：

{% highlight cpp %}
void decode(float position, float duration)
{% endhighlight %}

在 C 语言中编译出来的签名是 _decoder，而在 C++ 语言中，一般编译器的生成则类似于 _decode_float_float。虽然在编译阶段是没有问题的，但是在链接阶段，如果不加 extern \"C\" 关键字的话，那么将会链接 _decoder_float_float 这个方法签名；而如果加了 extern \"C\" 关键字的话，那么寻找的方法签名就是 _decoder。而 FFmpeg 就是 C 语言书写的，编译 FFmpeg 的时候所产生的方法签名都是 C 语言类型的签名，所以在 C 中引用 FFmpeg 必须要加 extern \"C\" 关键字。

### 关键步骤

**第一步，API 注册**

首先在使用 FFmpeg 接口之前，需要进行 FFmpeg 使用接口的注册操作：

{% highlight cpp %}
av_register_all();
{% endhighlight %}

**第二步，构建输入 AVFormatContext**

注册之后，打开输入文件并与 AVFormatContext 建立关联：

{% highlight cpp %}
AVFormatContext *ifmt_ctx = NULL;
if ((ret = avformat_open_input(&ifmt_ctx, in_file.c_str(), 0, 0)) < 0) {
    std::cerr << "Could not open input file " << in_file.c_str() << std::endl;
    exit(1);
}
{% endhighlight %}

**第三步，查找流信息**

建立关联之后，与解封装操作类似，可以通过接口 avformat_find_stream_info 获得流信息：

{% highlight cpp %}
if ((ret = avformat_find_stream_info(ifmt_ctx, 0)) < 0) {
    std::cerr << "Failed to retrieve input stream information" << std::endl;
    exit(1);
}
{% endhighlight %}

**第四步，构建输出 AVFormatContext**

输入文件打开完成之后，可以打开输出文件并与 AVFormatContext 建立关联：

{% highlight cpp %}
AVFormatContext *ofmt_ctx = NULL;
avformat_alloc_output_context2(&ofmt_ctx, NULL, NULL, out_file.c_str());
if (!ofmt_ctx) {
    std::cerr << "Could not create output context " << out_file.c_str() << std::endl;
    exit(1);
}
{% endhighlight %}

**第五步，申请 AVStream 和 stream 信息复制**

建立关联之后，需要申请输入的 stream 信息与输岀的 stream 信息，输入的 stream 信息可以从 ifmt_ctx 中获得，但是存储至 ofmt_ctx 中的 stream 信息需要申请独立内存空间，输出的 stream 信息建立完成之后，需要从输入的 stream 中将信息复制到输出的 stream 中，由于只是转封装，所以 stream 的信息不变，仅仅是改变了封装格式：

{% highlight cpp %}
for (int i = 0; i < ifmt_ctx->nb_streams; i++) {
    AVStream *in_stream = ifmt_ctx->streams[i];
    AVStream *out_stream = avformat_new_stream(ofmt_ctx, in_stream->codec->codec);
    if (!out_stream) {
        std::cerr << "Failed allocating output stream" << std::endl;
        exit(1);
    }
    
    ret = avcodec_copy_context(out_stream->codec, in_stream->codec);
    if (ret < 0) {
        std::cerr << "Failed to copy context from input to output stream codec context" << std::endl;
        exit(1);
    }
    
    out_stream->codec->codec_tag = 0;
    if (ofmt_ctx->oformat->flags & AVFMT_GLOBALHEADER)
        out_stream->codec->flags |= CODEC_FLAG_GLOBAL_HEADER;
    
    out_stream->time_base = out_stream->codec->time_base;
}
{% endhighlight %}

**第六步，打开输出 AVFormatContext 的缓冲区**

{% highlight cpp %}
if (!(ofmt_ctx->flags & AVFMT_NOFILE)) {
    ret = avio_open(&ofmt_ctx->pb, out_file.c_str(), AVIO_FLAG_WRITE);
    if (ret < 0) {
        std::cerr << "Could not open output file " << out_file.c_str() << std::endl;
        exit(1);
    }
}
{% endhighlight %}

**第七步，写文件头信息**

输出文件打开之后，接下来可以进行写文件头的操作：

{% highlight cpp %}
ret = avformat_write_header(ofmt_ctx, NULL);
if (ret < 0) {
    std::cerr << "Error occurred when opening output file" << std::endl;
    exit(1);
}
{% endhighlight %}

**第八步，数据包读取和写入**

输入与输岀均已经打开，并与对应的 AVFormatContext 建立了关联，接下来可以从输入格式中读取数据包，然后将数据包写入至输出文件中，当然，随着输入的封装格式与输出的封装格式的差别化，时间戳也需要进行对应的计算改变：

{% highlight cpp %}
AVPacket pkt;
while (1) {
    AVStream *in_stream, *out_stream;
    
    ret = av_read_frame(ifmt_ctx, &pkt);
    if (ret < 0) {
        break;
    }
    
    in_stream = ifmt_ctx->streams[pkt.stream_index];
    out_stream = ofmt_ctx->streams[pkt.stream_index];
    
    pkt.pts = av_rescale_q_rnd(pkt.pts, in_stream->time_base, out_stream->time_base, AV_ROUND_NEAR_INF);
    pkt.dts = av_rescale_q_rnd(pkt.dts, in_stream->time_base, out_stream->time_base, AV_ROUND_NEAR_INF);
    pkt.duration = av_rescale_q(pkt.duration, in_stream->time_base, out_stream->time_base);
    pkt.pos = -1;
    
    ret = av_interleaved_write_frame(ofmt_ctx, &pkt);
    if (ret < 0) {
        std::cerr << "Error muxing packet" << std::endl;
        exit(1);
    }
    av_free_packet(&pkt);
}
{% endhighlight %}

**第九步，写文件尾信息**

解封装读取数据并将数据写入新的封装格式的操作已经完成，接下来即可进行写文件尾至输出格式的操作：

{% highlight cpp %}
av_write_trailer(ofmt_ctx);
{% endhighlight %}

**第十步，收尾**

关闭输入格式，释放输出格式：

{% highlight cpp %}
avformat_close_input(&ifmt_ctx);
avformat_free_context(ofmt_ctx);
{% endhighlight %}

**第十一步，测试验证**

测试了 mp4 和 flv 之间的转换，mp4 的视频流要是 h.264 格式，如果是 hevc 格式就会失败。