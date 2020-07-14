---
title: FFmpeg 接口使用 - 音频编码和音频解码
author: 但江
avatar: danjiang
location: 成都 
category: programming
tag: ffmpeg
---

作为开发者，使用 FFmpeg 主要分两部分：命令行工具和接口使用，本文讲解如何在 macOS 上交叉编译 FFmpeg，再将其集成到 Xcode 中，再初步介绍 FFmpeg 接口使用时会用到的常用结构，最后实际编写音视频文件转封装的代码。

![Editing Video](/images/editing-video.jpg)

## 预备知识

通过 [iOS 利用 AudioToolbox 将 PCM 编码为 AAC](/programming/2020/07/08/ios-audiotoolbox-audio-converter-services/) 来了解数字音频、PCM 音频格式和 AAC 音频格式这些理论知识，再通过 [FFmpeg 接口使用 - 基础和转封装](/programming/2020/07/12/ffmpeg-api-fundamentals/) 来了解如何在 macOS 上交叉编译 FFmpeg，再将其集成到 Xcode 中，以及使用 FFmpeg 接口时会用到的常用结构。

## 音频解码

### 示例代码

* <em class="fab fa-github"></em> [audio_decoder.cpp](https://github.com/danjiang/DTCamera/blob/master/DTCamera/CrossPlatform/FFmpeg/audio_decoder.cpp)，将 AAC 解码成 PCM。
* <em class="fab fa-github"></em> [demuxing_decoding.c](https://github.com/FFmpeg/FFmpeg/blob/release/2.3/doc/examples/demuxing_decoding.c)，FFmpeg 2.3 解封装解码的示例代码。

### 关键步骤

**第一步，API 注册**

首先在使用 FFmpeg 接口之前，需要进行 FFmpeg 使用接口的注册操作：

{% highlight cpp %}
avcodec_register_all(); // 注册所有可用的 encoder 和 decoder
av_register_all(); // 注册所有可用的 muxer, demuxer 和 protocol
{% endhighlight %}

**第二步，构建输入 AVFormatContext**

注册之后，打开输入文件并与 AVFormatContext 建立关联：

{% highlight cpp %}
int result = avformat_open_input(&avFormatContext, audioFile, NULL, NULL);
if (result != 0) {
    printf("can't open file %s result is %d\n", audioFile, result);
    return -1;
} else {
    printf("open file %s success and result is %d\n", audioFile, result);
}
{% endhighlight %}

**第三步，查找流信息**

建立关联之后，与解封装操作类似，可以通过接口 avformat_find_stream_info 获得流信息：

{% highlight cpp %}
result = avformat_find_stream_info(avFormatContext, NULL); // 检查在文件中的流的信息
if (result < 0) {
    printf("fail avformat_find_stream_info result is %d\n", result);
    return -1;
} else {
    printf("sucess avformat_find_stream_info result is %d\n", result);
}
{% endhighlight %}

**第四步，得到输入音频流**

从输入的 AVFormatContext 中得到 AVStream：

{% highlight cpp %}
stream_index = av_find_best_stream(avFormatContext, AVMEDIA_TYPE_AUDIO, -1, -1, NULL, 0); // 寻找音频流下标
AVStream *audioStream = avFormatContext->streams[stream_index]; // 得到音频流
{% endhighlight %}

**第五步，查找解码器**

从 AVStream 获得解码器上下文，再根据 codec_id 找到解码器：

{% highlight cpp %}
avCodecContext = audioStream->codec; // 获得音频流的解码器上下文
AVCodec *avCodec = avcodec_find_decoder(avCodecContext->codec_id); // 根据解码器上下文找到解码器
{% endhighlight %}

**第六步，打开解码器**

{% highlight cpp %}
result = avcodec_open2(avCodecContext, avCodec, NULL);
if (result < 0) {
    printf("fail avcodec_open2 result is %d\n", result);
    return -1;
} else {
    printf("sucess avcodec_open2 result is %d\n", result);
}
{% endhighlight %}

**第七步，根据情况初始化音频转换上下文**

这里判断输入音频的量化格式的表示格式是否 AV_SAMPLE_FMT_S16 来决定是否初始化音频转换上下文，这里情况是将 AV_SAMPLE_FMT_FLTP 转换为 AV_SAMPLE_FMT_S16：

{% highlight cpp %}
bool AudioDecoder::audioCodecIsSupported() {
    if (avCodecContext->sample_fmt == AV_SAMPLE_FMT_S16) {
        return true;
    }
    return false;
}

if (!audioCodecIsSupported()) {
    printf("because of audio Codec Is Not Supported so we will init swresampler...\n");
    swrContext = swr_alloc_set_opts(NULL, av_get_default_channel_layout(OUT_PUT_CHANNELS), AV_SAMPLE_FMT_S16, avCodecContext->sample_rate, av_get_default_channel_layout(avCodecContext->channels), avCodecContext->sample_fmt, avCodecContext->sample_rate, 0, NULL);
    if (!swrContext || swr_init(swrContext)) {
        if (swrContext) {
            swr_free(&swrContext);
        }
        avcodec_close(avCodecContext); // 调用对应第三方库的 API 来关闭掉对应的编码库
        printf("init resampler failed...\n");
        return -1;
    }
}
{% endhighlight %}

**第八步，解码音频数据**

使用 av_read_frame 读取 AVPacket，通过 avcodec_decode_audio4 将 AVPacket 解码为 AVFrame，如果需要音频转换，通过 swr_convert 进行转换，解码后的音频数据放在 audioBuffer 中，其大小是 audioBufferSize：

{% highlight cpp %}
int AudioDecoder::readFrame() {
    int ret = 1;
    av_init_packet(&packet);
    int gotframe = 0;
    int readFrameCode = -1;
    while (true) {
        /*
         * 内部处理了数据不能被解码器完全处理完的情况，
         * 该函数的实现首先会委托到 Demuxer 的 read_packet 方法中去，
         * 对于音频流，一个 AVPacket 可能包含多个 AVFrame，
         * 但是对于视频流，一个 AVPacket 只包含一个 AVFrame，
         * 该函数最终只会返回一个 AVPacket 结构体。
         */
        readFrameCode = av_read_frame(avFormatContext, &packet);
        if (readFrameCode >= 0) {
            if (packet.stream_index == stream_index) {
                int len = avcodec_decode_audio4(avCodecContext, pAudioFrame, &gotframe, &packet); // 音频解码
                if (len < 0) {
                    printf("decode audio error, skip packet\n");
                }
                if (gotframe) {
                    int numChannels = OUT_PUT_CHANNELS;
                    int numFrames = 0;
                    void *audioData;
                    if (swrContext) {
                        const int bufSize = av_samples_get_buffer_size(NULL, numChannels, pAudioFrame->nb_samples, AV_SAMPLE_FMT_S16, 1);
                        if (!swrBuffer || swrBufferSize < bufSize) {
                            swrBufferSize = bufSize;
                            swrBuffer = realloc(swrBuffer, swrBufferSize);
                        }
                        uint8_t *outbuf[2] = { (uint8_t*)swrBuffer, NULL };
                        numFrames = swr_convert(swrContext, outbuf, pAudioFrame->nb_samples, (const uint8_t **)pAudioFrame->data, pAudioFrame->nb_samples);
                        if (numFrames < 0) {
                            printf("fail resample audio\n");
                            ret = -1;
                            break;
                        }
                        audioData = swrBuffer;
                    } else {
                        if (avCodecContext->sample_fmt != AV_SAMPLE_FMT_S16) {
                            printf("bucheck, audio format is invalid\n");
                            ret = -1;
                            break;
                        }
                        audioData = pAudioFrame->data[0];
                        numFrames = pAudioFrame->nb_samples;
                    }
                    if (isNeedFirstFrameCorrectFlag && position >= 0) {
                        float expectedPosition = position + duration;
                        float actualPosition = av_frame_get_best_effort_timestamp(pAudioFrame) * timeBase;
                        firstFrameCorrectionInSecs = actualPosition - expectedPosition;
                        isNeedFirstFrameCorrectFlag = false;
                    }
                    duration = av_frame_get_pkt_duration(pAudioFrame) * timeBase;
                    position = av_frame_get_best_effort_timestamp(pAudioFrame) * timeBase - firstFrameCorrectionInSecs;
                    audioBufferSize = numFrames * numChannels;
                    audioBuffer = (short *)audioData;
                    audioBufferCursor = 0;
                    break;
                }
            }
        } else {
            ret = -1;
            break;
        }
    }
    av_free_packet(&packet);
    return ret;
}
{% endhighlight %}

**第九步，把解码后的音频数据写到文件**

先把上面解码后的音频数据装到一个大缓冲里：

{% highlight cpp %}
int AudioDecoder::readSamples(short *samples, int size) {
    int sampleSize = size;
    while (size > 0) {
        if (audioBufferCursor < audioBufferSize) {
            int audioBufferDataSize = audioBufferSize - audioBufferCursor;
            int copySize = MIN(size, audioBufferDataSize);
            memcpy(samples + (sampleSize - size), audioBuffer + audioBufferCursor, copySize * 2); // the last param is number of bytes to copy
            size -= copySize;
            audioBufferCursor += copySize;
        } else {
            if (readFrame() < 0) {
                break;
            }
        }
    }
    int fillSize = sampleSize - size;
    if (fillSize == 0) {
        return -1;
    }
    return fillSize;
}
{% endhighlight %}

再把这个大缓冲写到文件：

{% highlight objc %}
while (true) {
    AudioPacket *packet = decoder->decodePacket();
    if (packet->size == -1) {
        break;
    }
    fwrite(packet->buffer, sizeof(short), packet->size, outputFile);
}
{% endhighlight %}

**第十步，收尾**

释放各种资源：

{% highlight cpp %}
void AudioDecoder::destroy() {
    printf("AudioDecoder start destroy!!!\n");
    if (NULL != swrBuffer) {
        free(swrBuffer);
        swrBuffer = NULL;
        swrBufferSize = 0;
    }
    if (NULL != swrContext) {
        swr_free(&swrContext);
        swrContext = NULL;
    }
    if (NULL != pAudioFrame) {
        av_free(pAudioFrame);
        pAudioFrame = NULL;
    }
    if (NULL != avCodecContext) {
        avcodec_close(avCodecContext);
        avCodecContext = NULL;
    }
    if (NULL != avFormatContext) {
        /*
         * 该函数负责释放对应的资源，
         * 首先会调用对应的 Demuxer 中的生命周期 read_close 方法，
         * 然后释放掉 AVFormatContext，
         * 最后关闭文件或者远程网络连接。
         */
        avformat_close_input(&avFormatContext);
        avFormatContext = NULL;
    }
    printf("AudioDecoder end destroy!!!\n");
}
{% endhighlight %}

## 音频编码

### 示例代码

* <em class="fab fa-github"></em> [audio_encoder.cpp](https://github.com/danjiang/DTCamera/blob/master/DTCamera/CrossPlatform/FFmpeg/audio_encoder.cpp)，将 PCM 编码成 AAC。
* <em class="fab fa-github"></em> [decoding_encoding.c](https://github.com/FFmpeg/FFmpeg/blob/release/2.3/doc/examples/decoding_encoding.c)，FFmpeg 2.3 解码和编码的示例代码。

### 关键步骤

**第一步，API 注册**

首先在使用 FFmpeg 接口之前，需要进行 FFmpeg 使用接口的注册操作：

{% highlight cpp %}
avcodec_register_all(); // 注册所有可用的 encoder 和 decoder
av_register_all(); // 注册所有可用的 muxer, demuxer 和 protocol
{% endhighlight %}

**第二步，构建输出 AVFormatContext**

可以打开输出文件并与 AVFormatContext 建立关联：

{% highlight cpp %}
/*
 * 最关键的还是根据上一步注册的 Muxer 和 Demuxer 部分（也就是封装格式部分）去找到对应的格式，
 * 有可能是 flv 格式、MP4 格式、mov 格式，甚至是 MP3 格式等，
 * 如果找不到对应的格式（即在 configure 选项中没有打开这个格式的开关），那么这里会返回找不到对应的格式的错误提示。
 */
if ((ret = avformat_alloc_output_context2(&avFormatContext, NULL, NULL, aacFilePath)) != 0 ) {
    printf("avFormatContext   alloc   failed : %s\n", av_err2str(ret));
    return -1;
}
{% endhighlight %}

**第三步，打开输出 AVFormatContext 的缓冲区**

{% highlight cpp %}
/*
 * 首先调用函数 ffurl_open，构造出 URLContext 结构体，这个结构体中包含了 URLProtocol（需要去第一步 register_protocol 中已经注册的协议链表中寻找）,
 * 接着会调用 avio_alloc_context 方法，分配出 AVIOContext 结构体，并将上一步构造出来的 URLProtocol 传递进来,
 * 然后把上一步分配出来 AVIOContext 结构体赋值给 AVFormatContext 的属性。
 */
if (ret = avio_open2(&avFormatContext->pb, aacFilePath, AVIO_FLAG_WRITE, NULL, NULL)) {
    printf("Could not avio open fail %s\n", av_err2str(ret));
    return -1;
}
{% endhighlight %}

**第四步，申请 AVStream**

{% highlight cpp %}
audioStream = avformat_new_stream(avFormatContext, NULL);
audioStream->id = 1;
{% endhighlight %}

**第五步，获取 AVCodecContext**

从 AVStream 获得编码器上下文，设置编码参数：

{% highlight cpp %}
avCodecContext = audioStream->codec;
avCodecContext->codec_type = AVMEDIA_TYPE_AUDIO; // 基本属性 - 音频类型
avCodecContext->sample_rate = audioSampleRate; // 基本属性 - 采样率
if (publishBitRate > 0) { // 基本属性 - 码率
    avCodecContext->bit_rate = publishBitRate;
} else {
    avCodecContext->bit_rate = PUBLISH_BITE_RATE;
}
avCodecContext->sample_fmt = preferedSampleFMT; // 基本属性 - 量化格式
printf("audioChannels is %d\n", audioChannels);
avCodecContext->channel_layout = preferedChannels == 1 ? AV_CH_LAYOUT_MONO : AV_CH_LAYOUT_STEREO; // 基本属性 - 声道
avCodecContext->channels = av_get_channel_layout_nb_channels(avCodecContext->channel_layout); // 基本属性 - 声道
avCodecContext->profile = FF_PROFILE_AAC_LOW;
printf("avCodecContext->channels is %d\n", avCodecContext->channels);
avCodecContext->flags |= CODEC_FLAG_GLOBAL_HEADER;
{% endhighlight %}

**第六步，查找编码器**

根据编码器名称查找编码器：

{% highlight cpp %}
codec = avcodec_find_encoder_by_name(codec_name); // 寻找 encoder
if (!codec) {
    printf("Couldn't find a valid audio codec\n");
    return -1;
}
avCodecContext->codec_id = codec->id;
{% endhighlight %}

**第七步，根据情况初始化音频转换上下文**

如果期望输入的 PCM 格式和实际输入的 PCM 格式有差异，就需要初始化音频转换上下文：

{% highlight cpp %}
// 有些编码器只允许特定格式的 PCM 作为输入源，所以有时需要构造一个重采样器来将 PCM 数据转换为可适配编码器输入的 PCM 数据
if (preferedChannels != avCodecContext->channels
    || preferedSampleRate != avCodecContext->sample_rate
    || preferedSampleFMT != avCodecContext->sample_fmt) {
    swrContext = swr_alloc_set_opts(NULL,
                                    av_get_default_channel_layout(avCodecContext->channels),
                                    avCodecContext->sample_fmt,
                                    avCodecContext->sample_rate,
                                    av_get_default_channel_layout(preferedChannels),
                                    preferedSampleFMT,
                                    preferedSampleRate,
                                    0,
                                    NULL);
    if (!swrContext || swr_init(swrContext)) {
        if (swrContext) {
            swr_free(&swrContext);
        }
        return -1;
    }
}
{% endhighlight %}

**第八步，打开编码器**

{% highlight cpp %}
if (avcodec_open2(avCodecContext, codec, NULL) < 0) {
    printf("Couldn't open codec\n");
    return -2;
}
{% endhighlight %}

**第九步，申请帧结构 AVFrame**

申请音频帧存储空间，配置音频参数，根据音频参数计算出缓冲大小，再创建缓冲，把缓冲和帧结构关联起来：

{% highlight cpp %}
int ret = 0;
AVSampleFormat preferedSampleFMT = AV_SAMPLE_FMT_S16;
int preferedChannels = audioChannels;
int preferedSampleRate = audioSampleRate;
input_frame = av_frame_alloc();
if (!input_frame) {
    printf("Could not allocate audio frame\n");
    return -1;
}
input_frame->nb_samples = avCodecContext->frame_size;
input_frame->format = preferedSampleFMT;
input_frame->channel_layout = preferedChannels == 1 ? AV_CH_LAYOUT_MONO : AV_CH_LAYOUT_STEREO;
input_frame->sample_rate = preferedSampleRate;
buffer_size = av_samples_get_buffer_size(NULL, av_get_channel_layout_nb_channels(input_frame->channel_layout), input_frame->nb_samples, preferedSampleFMT, 0); // 计算公式 frame_size * sizeof(SInt16) * channels
samples = (uint8_t*)av_malloc(buffer_size);
samplesCursor = 0;
if (!samples) {
    printf("Could not allocate %d bytes for samples buffer\n", buffer_size);
    return -2;
}
printf("allocate %d bytes for samples buffer\n", buffer_size);
ret = avcodec_fill_audio_frame(input_frame, av_get_channel_layout_nb_channels(input_frame->channel_layout), preferedSampleFMT, samples, buffer_size, 0);
if (ret < 0) {
    printf("Could not setup audio frame\n");
}
{% endhighlight %}

**第十步，申请用于转换的帧结构 AVFrame**

如果需要音频转换，还需要额外申请用于转换的帧结构 AVFrame：

{% highlight cpp %}
if (swrContext) {
    if (av_sample_fmt_is_planar(avCodecContext->sample_fmt)) {
        printf("Codec Context SampleFormat is Planar...\n");
    }
    convert_data = (uint8_t**)calloc(avCodecContext->channels, sizeof(*convert_data));
    av_samples_alloc(convert_data, NULL, avCodecContext->channels, avCodecContext->frame_size, avCodecContext->sample_fmt, 0);
    swrBufferSize = av_samples_get_buffer_size(NULL, avCodecContext->channels, avCodecContext->frame_size, avCodecContext->sample_fmt, 0);
    swrBuffer = (uint8_t*)av_malloc(swrBufferSize);
    printf("After av_malloc swrBuffer\n");
    swrFrame = av_frame_alloc();
    if (!swrFrame) {
        printf("Could not allocate swrFrame frame\n");
        return -1;
    }
    swrFrame->nb_samples = avCodecContext->frame_size;
    swrFrame->format = avCodecContext->sample_fmt;
    swrFrame->channel_layout = avCodecContext->channels == 1 ? AV_CH_LAYOUT_MONO : AV_CH_LAYOUT_STEREO;
    swrFrame->sample_rate = avCodecContext->sample_rate;
    ret = avcodec_fill_audio_frame(swrFrame, avCodecContext->channels, avCodecContext->sample_fmt, (uint8_t*)swrBuffer, swrBufferSize, 0);
    printf("After avcodec_fill_audio_frame\n");
    if (ret < 0) {
        printf("avcodec_fill_audio_frame error\n");
        return -1;
    }
}
{% endhighlight %}

**第十一步，编码音频数据**

把读入的缓冲数据根据帧结构的缓冲大小，分批拷贝到帧结构的缓冲：

{% highlight cpp %}
void AudioEncoder::encode(uint8_t *buffer, int size) {
    int bufferCursor = 0;
    int bufferSize = size;
    while (bufferSize >= (buffer_size - samplesCursor)) {
        int cpySize = buffer_size - samplesCursor;
        memcpy(samples + samplesCursor, buffer + bufferCursor, cpySize);
        bufferCursor += cpySize;
        bufferSize -= cpySize;
        this->encodePacket();
        samplesCursor = 0;
    }
    if (bufferSize > 0) {
        memcpy(samples + samplesCursor, buffer + bufferCursor, bufferSize);
        samplesCursor += bufferSize;
    }
}
{% endhighlight %}

如果需要音频转换，先进行音频转换：

{% highlight cpp %}
int ret, got_output;
AVPacket pkt;
av_init_packet(&pkt);
AVFrame* encode_frame;
if (swrContext) {
    swr_convert(swrContext, convert_data, avCodecContext->frame_size, (const uint8_t**)input_frame->data, avCodecContext->frame_size);
    int length = avCodecContext->frame_size * av_get_bytes_per_sample(avCodecContext->sample_fmt);
    for (int k = 0; k < 2; ++k) {
        for (int j = 0; j < length; ++j) {
            swrFrame->data[k][j] = convert_data[k][j];
        }
    }
    encode_frame = swrFrame;
} else {
    encode_frame = input_frame;
}
pkt.stream_index = 0;
pkt.duration = (int)AV_NOPTS_VALUE;
pkt.pts = pkt.dts = 0;
pkt.data = samples;
pkt.size = buffer_size;
{% endhighlight %}

编码 AVFrame 为 AVPacket，再将 AVPacket 写入文件：

{% highlight cpp %}
ret = avcodec_encode_audio2(avCodecContext, &pkt, encode_frame, &got_output); // 编码音频
if (ret < 0) {
    printf("Error encoding audio frame\n");
    return;
}
if (got_output) {
    if (avCodecContext->coded_frame && avCodecContext->coded_frame->pts != AV_NOPTS_VALUE) {
        pkt.pts = av_rescale_q(avCodecContext->coded_frame->pts, avCodecContext->time_base, audioStream->time_base);
    }
    pkt.flags |= AV_PKT_FLAG_KEY;
    this->duration = pkt.pts * av_q2d(audioStream->time_base);
    /*
     * 会将编码后的 AVPacket 结构体作为 Muxer 中的 write_packet 生命周期方法的输入，
     * write_packet函数会加上自己封装格式的头信息，然后调用协议层写到本地文件或者网络服务器上。
     */
    int writeCode = av_interleaved_write_frame(avFormatContext, &pkt); // 写入文件
}
av_free_packet(&pkt);
{% endhighlight %}

## 测试验证

可以利用 [iOS Audio Unit 录制音频文件和播放音频文件](/programming/2020/07/02/ios-audio-unit-record-play/) 或 [iOS AVAudioEngine 录制音频文件和播放音频文件](/programming/2020/07/03/ios-avaudioengine-record-play/) 录制 PCM 音频文件，然后参考 [iOS 利用 AudioToolbox 将 PCM 编码为 AAC](/programming/2020/07/08/ios-audiotoolbox-audio-converter-services/) 得到 AAC 音频文件，接着就可以用上面代码将 AAC 解码成 PCM，再将 PCM 编码成 AAC。

### ffplay

ffplay 播放 pcm 音频数据，要指定量化格式的表示格式、声道数和采样率，s16le 的含义是 signed 16 bits little endian：

{% highlight text %}
$ ffplay audio.pcm -f s16le -channels 2 -ar 44100
{% endhighlight %}

### ffprobe

{% highlight text %}
$ ffprobe -show_format -pretty audio.aac
Input #0, aac, from 'audio.aac':
  Duration: 00:00:03.50, bitrate: 134 kb/s
    Stream #0:0: Audio: aac (LC), 44100 Hz, stereo, fltp, 134 kb/s
[FORMAT]
filename=audio.aac
nb_streams=1
nb_programs=0
format_name=aac
format_long_name=raw ADTS AAC (Advanced Audio Coding)
start_time=N/A
duration=0:00:03.496978
size=57.210938 Kibyte
bit_rate=134.022000 Kbit/s
probe_score=51
[/FORMAT]
{% endhighlight %}

### [AACParser](https://github.com/danjiang/MediaParser/blob/master/AACParser.cpp)

AACParser 对每一个 ADTS Frame 的输出（截取了部分）：

{% highlight text %}
  Num|     Profile| Frequency|  Channel| Size|
    0|      AAC LC|   44100Hz|        2|  387|
    1|      AAC LC|   44100Hz|        2|  387|
    2|      AAC LC|   44100Hz|        2|  388|
    3|      AAC LC|   44100Hz|        2|  489|
    4|      AAC LC|   44100Hz|        2|  488|
    5|      AAC LC|   44100Hz|        2|  459|
    6|      AAC LC|   44100Hz|        2|  438|
    7|      AAC LC|   44100Hz|        2|  413|
    8|      AAC LC|   44100Hz|        2|  402|
    9|      AAC LC|   44100Hz|        2|  403|
   10|      AAC LC|   44100Hz|        2|  405|
{% endhighlight %}



