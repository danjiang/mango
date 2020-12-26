---
title: iOS 利用 VideoToolbox 和 FFmpeg 推 RTMP 流
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: ios-av
---

本文讲解了在 iOS 上通过 VideoToolbox 进行视频硬编码，再通过 FFmpeg 封装 FLV，并通过 FFmpeg 推 RTMP 流或者录制到本地文件。

![Camera Sea](/images/camera-sea.jpg)

## VideoToolbox

完整代码

* <em class="fab fa-github"></em> [VideoEncoder.swift](https://github.com/danjiang/DTCamera/blob/master/DTCamera/iOS/Encode/VideoEncoder.swift)

iOS 上通过 VideoToolbox 来进行视频硬件编码和硬件解码，前面说过如下的视频文件中需要关注的常见信息，这里主要关注第二行关于视频的部分：

* 封装格式，时长，存储大小
* 视频编码，视频码率，分辨率，帧率
* 音频编码，音频采样率，音频码率，声道

**第一步，创建和配置 VTCompressionSession**

VTCompressionSessionCreate 中的 outputCallback 指定了编码后输出回调，剩下的就是配置前面提到的视频信息：

{% highlight swift %}
private var session: VTCompressionSession!

init(width: Int, height: Int, fps: Int, maxBitRate: Int, avgBitRate: Int) {
    self.width = width
    self.height = height
    self.fps = fps
    self.maxBitRate = maxBitRate
    self.avgBitRate = avgBitRate
    
    encodeQueue.async { [weak self] in
        guard let self = self else { return }
        let statusCode = VTCompressionSessionCreate(allocator: kCFAllocatorDefault,
                                                    width: Int32(width),
                                                    height: Int32(height),
                                                    codecType: kCMVideoCodecType_H264,
                                                    encoderSpecification: nil,
                                                    imageBufferAttributes: nil,
                                                    compressedDataAllocator: nil,
                                                    outputCallback: didCompressH264,
                                                    refcon: UnsafeMutableRawPointer(Unmanaged.passUnretained(self).toOpaque()),
                                                    compressionSessionOut: &self.session)
        if statusCode != noErr {
            DDLogError("H264: Unable to create a H264 session status is \(statusCode)")
            self.delegate?.videoEncoderInitFailed(self)
            return
        }
        
        VTSessionSetProperty(self.session, key: kVTCompressionPropertyKey_RealTime, value: kCFBooleanTrue)
        VTSessionSetProperty(self.session, key: kVTCompressionPropertyKey_ProfileLevel, value: kVTProfileLevel_H264_High_AutoLevel)
        VTSessionSetProperty(self.session, key: kVTCompressionPropertyKey_AllowFrameReordering, value: kCFBooleanFalse) // 不产生 B 帧
        self.setMaxBitRate(maxBitRate, avgBitRate: avgBitRate, fps: fps)
        
        VTCompressionSessionPrepareToEncodeFrames(self.session)
        
        self.isReady = true
        encodingSessionValid = true
    }
}

func setMaxBitRate(_ maxBitRate: Int, avgBitRate: Int, fps: Int) {
    VTSessionSetProperty(session, key: kVTCompressionPropertyKey_MaxKeyFrameInterval, value: fps as CFTypeRef) // 关键帧间隔, gop size
    VTSessionSetProperty(session, key: kVTCompressionPropertyKey_ExpectedFrameRate, value: fps as CFTypeRef) // 帧率
    VTSessionSetProperty(session, key: kVTCompressionPropertyKey_DataRateLimits, value: [maxBitRate / 8, 1] as CFArray) // 控制码率
    VTSessionSetProperty(session, key: kVTCompressionPropertyKey_AverageBitRate, value: avgBitRate as CFTypeRef) // 控制码率
}
{% endhighlight %}

**第二步，编码 CVPixelBuffer**

将传入的视频图像 CVPixelBuffer 进行编码：

{% highlight swift %}
func encode(pixelBuffer: CVPixelBuffer) {
    if continuousEncodeFailureTimes > continuousEncodeFailureTimesTreshold {
        delegate?.videoEncoderEncodedFailed(self)
    }
    encodeQueue.async { [weak self] in
        guard let self = self, self.isReady else {
                return
        }
        let currentTimeMills = Int64(CFAbsoluteTimeGetCurrent() * 1000)
        if self.encodingTimeMills == -1 {
            self.encodingTimeMills = currentTimeMills
        }
        let encodingDuration = currentTimeMills - self.encodingTimeMills
        
        let pts = CMTimeMake(value: encodingDuration, timescale: 1000) // 当前编码视频帧的时间戳，单位为毫秒
        let duration = CMTimeMake(value: 1, timescale: Int32(self.fps)) // 当前编码视频帧的时长
        
        let statusCode = VTCompressionSessionEncodeFrame(self.session,
                                                         imageBuffer: pixelBuffer,
                                                         presentationTimeStamp: pts,
                                                         duration: duration,
                                                         frameProperties: nil,
                                                         sourceFrameRefcon: nil,
                                                         infoFlagsOut: nil)
        
        if statusCode != noErr {
            DDLogError("H264: VTCompressionSessionEncodeFrame failed \(statusCode)")
            return
        }
    }
}
{% endhighlight %}

**第三步，处理编码后的输出回调**

编码数据是放在 CMSampleBuffer 中的，下面的代码首先从 CMSampleBuffer 拆分出 sps 和 pps，再从中分离出视频帧数据：

{% highlight swift %}
func didCompressH264(outputCallbackRefCon: UnsafeMutableRawPointer?,
                     sourceFrameRefCon: UnsafeMutableRawPointer?,
                     status: OSStatus,
                     infoFlags: VTEncodeInfoFlags,
                     sampleBuffer: CMSampleBuffer?) -> Void {
    if status != noErr {
        continuousEncodeFailureTimes += 1
        return
    }
    continuousEncodeFailureTimes = 0
    
    guard let sampleBuffer = sampleBuffer,
        CMSampleBufferDataIsReady(sampleBuffer),
        encodingSessionValid else {
            return
    }
    
    let encoder: VideoEncoder = Unmanaged.fromOpaque(outputCallbackRefCon!).takeUnretainedValue()
    
    if let attachments = CMSampleBufferGetSampleAttachmentsArray(sampleBuffer, createIfNecessary: true) {
        let rawDictionary: UnsafeRawPointer = CFArrayGetValueAtIndex(attachments, 0)
        let dictionary: CFDictionary = Unmanaged.fromOpaque(rawDictionary).takeUnretainedValue()
        let isKeyframe = !CFDictionaryContainsKey(dictionary, Unmanaged.passUnretained(kCMSampleAttachmentKey_NotSync).toOpaque())
        if isKeyframe && !encoder.isKeyframeFound { // 每一个关键帧前面都会输出 SPS 和 PPS 信息
            let format = CMSampleBufferGetFormatDescription(sampleBuffer)
            // sps
            var spsSize: Int = 0
            var spsCount: Int = 0
            var nalHeaderLength: Int32 = 0
            var sps: UnsafePointer<UInt8>!
            var statusCode = CMVideoFormatDescriptionGetH264ParameterSetAtIndex(format!,
                                                                                parameterSetIndex: 0,
                                                                                parameterSetPointerOut: &sps,
                                                                                parameterSetSizeOut: &spsSize,
                                                                                parameterSetCountOut: &spsCount,
                                                                                nalUnitHeaderLengthOut: &nalHeaderLength)
            if statusCode == noErr {
                // pps
                var ppsSize: Int = 0
                var ppsCount: Int = 0
                var pps: UnsafePointer<UInt8>!
                statusCode = CMVideoFormatDescriptionGetH264ParameterSetAtIndex(format!,
                                                                                parameterSetIndex: 1,
                                                                                parameterSetPointerOut: &pps,
                                                                                parameterSetSizeOut: &ppsSize,
                                                                                parameterSetCountOut: &ppsCount,
                                                                                nalUnitHeaderLengthOut: &nalHeaderLength)
                if statusCode == noErr {
                    let spsData = Data(bytes: sps, count: spsSize)
                    let ppsData = Data(bytes: pps, count: ppsSize)
                    let timeMills = CMTimeGetSeconds(CMSampleBufferGetPresentationTimeStamp(sampleBuffer)) * 1000
                    DDLogDebug("videoEncoder spsSize: \(spsSize) nalHeaderLength: \(nalHeaderLength) ppsSize: \(ppsSize) nalHeaderLength: \(nalHeaderLength)")
                    encoder.isKeyframeFound = true
                    encoder.delegate?.videoEncoder(encoder, encoded: spsData, pps: ppsData, timestamp: timeMills)
                }
            }
        }
        
        guard let dataBuffer = CMSampleBufferGetDataBuffer(sampleBuffer) else {
            return
        }
        var lengthAtOffset: Int = 0
        var totalLength: Int = 0
        var dataPointer: UnsafeMutablePointer<Int8>!
        let statusCode = CMBlockBufferGetDataPointer(dataBuffer,
                                                     atOffset: 0,
                                                     lengthAtOffsetOut: &lengthAtOffset,
                                                     totalLengthOut: &totalLength,
                                                     dataPointerOut: &dataPointer)
        if statusCode == noErr {
            var bufferOffset: Int = 0
            let AVCCHeaderLength = 4
            while bufferOffset < totalLength - AVCCHeaderLength {
                var NALUnitLength: UInt32 = 0
                // first four character is NAL Unit length
                memcpy(&NALUnitLength, dataPointer.advanced(by: bufferOffset), AVCCHeaderLength)
                // big endian to host endian. in iOS it's little endian
                NALUnitLength = CFSwapInt32BigToHost(NALUnitLength)
                
                let data: Data = Data(bytes: dataPointer.advanced(by: bufferOffset + AVCCHeaderLength), count: Int(NALUnitLength))
                let timeMills = CMTimeGetSeconds(CMSampleBufferGetPresentationTimeStamp(sampleBuffer)) * 1000
//                DDLogDebug("videoEncoder encodedData: \(Int(NALUnitLength))")
                encoder.delegate?.videoEncoder(encoder, encoded: data, isKeyframe: isKeyframe, timestamp: timeMills)
                
                // move forward to the next NAL Unit
                bufferOffset += Int(AVCCHeaderLength)
                bufferOffset += Int(NALUnitLength)
            }
        }
    }
    

}
{% endhighlight %}

## FFmpeg 推流

### 回顾整体流程

根据下图来回顾下整体流程，查看 [iOS 利用 OpenGL ES 和 AVAssetWriter 录制视频](/programming/2020/07/10/ios-record-video-with-opengl-avassetwriter/) 和前面对 VideoToolbox 的讲解，蓝色节点部分都已经涉及到了；查看 [iOS Audio Unit 录制音频文件和播放音频文件](/programming/2020/07/02/ios-audio-unit-record-play/)、[iOS AVAudioEngine 录制音频文件和播放音频文件](/programming/2020/07/03/ios-avaudioengine-record-play/) 和 [FFmpeg 接口使用 - 音频编码和音频解码](/programming/2020/07/14/ffmpeg-api-audio-encode-decode/)，绿色节点部分都已经涉及到了，下面要重点讲解的就是黄色节点部分。

![iOS Audio Video Producer](/images/ios-audio-video-producer.png)

### FFmpeg 编码和封装的流程

![iOS Audio Video Producer Publish](/images/ios-audio-video-producer-publish.png)

### 生产者-消费者模型的队列实现

完整代码

* <em class="fab fa-github"></em> [live_video_packet_queue.cpp](https://github.com/danjiang/DTCamera/blob/master/DTCamera/CrossPlatform/Live/live_video_packet_queue.cpp)
* <em class="fab fa-github"></em> [live_audio_packet_queue.cpp](https://github.com/danjiang/DTCamera/blob/master/DTCamera/CrossPlatform/Live/live_audio_packet_queue.cpp)
* <em class="fab fa-github"></em> [live_packet_pool.cpp](https://github.com/danjiang/DTCamera/blob/master/DTCamera/CrossPlatform/Live/live_packet_pool.cpp)
* <em class="fab fa-github"></em> [live_audio_packet_pool.cpp](https://github.com/danjiang/DTCamera/blob/master/DTCamera/CrossPlatform/Live/live_audio_packet_pool.cpp)

![Producer Consumer Queue](/images/producer-consumer-queue.png)

生产者-消费者模型主要是针对多线程场景，有一个生产者线程往队列中添加数据，另外有一个消费者线程从队列中消费数据，针对这里的场景就是，VideoToolbox 做为生产者往 H.264 Video Queue 中添加 H.264 数据，后面会讲到的 VideoConsumerThread 会从 H.264 Video Queue 中消费 H.264 数据，这里的线程同步机制采用了互斥量和条件变量，查看 [iOS 基础 - Posix Thread](/programming/2020/06/17/pthread/) 了解更多：

{% highlight cpp %}
typedef struct LiveVideoPacketList {
    LiveVideoPacket *pkt;
    struct LiveVideoPacketList *next;
    LiveVideoPacketList() {
        pkt = NULL;
        next = NULL;
    }
} LiveVideoPacketList;

void LiveVideoPacketQueue::init() {
    pthread_mutex_init(&mLock, NULL);
    pthread_cond_init(&mCondition, NULL);
    mNbPackets = 0;
    mFrist = NULL;
    mLast = NULL;
    mAbortRequest = false;
    currentTimeMills = NON_DROP_FRAME_FLAG;
}

int LiveVideoPacketQueue::put(LiveVideoPacket *pkt) {
    if (mAbortRequest) {
        delete pkt;
        return -1;
    }
    LiveVideoPacketList *pkt1 = new LiveVideoPacketList();
    if (!pkt1) {
        return -1;
    }
    pkt1->pkt = pkt;
    pkt1->next = NULL;
    pthread_mutex_lock(&mLock);
    if (mLast == NULL) {
        mFrist = pkt1;
    } else {
        mLast->next = pkt1;
    }
    mLast = pkt1;
    mNbPackets++;
    pthread_cond_signal(&mCondition);
    pthread_mutex_unlock(&mLock);
    return 0;
}

int LiveVideoPacketQueue::get(LiveVideoPacket **pkt, bool block) {
    LiveVideoPacketList *pkt1;
    int ret;
    pthread_mutex_lock(&mLock);
    for (;;) {
        if (mAbortRequest) {
            ret = -1;
            break;
        }
        pkt1 = mFrist;
        if (pkt1) {
            mFrist = pkt1->next;
            if (!mFrist) {
                mLast = NULL;
            }
            mNbPackets--;
            *pkt = pkt1->pkt;
            if (NON_DROP_FRAME_FLAG != currentTimeMills) {
                (*pkt)->timeMills = currentTimeMills;
                currentTimeMills += (*pkt)->duration;
            }
            delete pkt1;
            pkt1 = NULL;
            ret = 1;
            break;
        } else if (!block) {
            ret = 0;
            break;
        } else {
            pthread_cond_wait(&mCondition, &mLock);
        }
    }
    pthread_mutex_unlock(&mLock);
    return ret;
}
{% endhighlight %}

### 消费者的关键步骤

#### 完整代码

* <em class="fab fa-github"></em> [recording_publisher.cpp](https://github.com/danjiang/DTCamera/blob/master/DTCamera/CrossPlatform/Live/recording_publisher.cpp)
* <em class="fab fa-github"></em> [recording_h264_publisher.cpp](https://github.com/danjiang/DTCamera/blob/master/DTCamera/CrossPlatform/Live/recording_h264_publisher.cpp)
* <em class="fab fa-github"></em> [live_thread.cpp](https://github.com/danjiang/DTCamera/blob/master/DTCamera/CrossPlatform/Live/live_thread.cpp)
* <em class="fab fa-github"></em> [video_consumer_thread.cpp](https://github.com/danjiang/DTCamera/blob/master/DTCamera/CrossPlatform/Live/video_consumer_thread.cpp)

#### FFmpeg 封装 FLV 的准备

创建 AVFormatContext，指定封装格式为 FLV，在其上添加一路 H.264 视频流和一路 AAC 音频流，这部分的代码同 [FFmpeg 接口使用 - 基础和转封装](/programming/2020/07/12/ffmpeg-api-fundamentals/) 和 [FFmpeg 接口使用 - 音频编码和音频解码](/programming/2020/07/14/ffmpeg-api-audio-encode-decode/) 讲的道理是一样的。

#### FFmpeg 写 H.264 数据

从 H.264 Video Queue 中拿到 H.264 数据，再将其装到 AVPacket 中，再通过 av_interleaved_write_frame 来写：

{% highlight cpp %}
int RecordingH264Publisher::write_video_frame(AVFormatContext *oc, AVStream *st) {
    int ret = 0;
    AVCodecContext *c = st->codec;
    
    // 调用注册的回调方法来拿到我们的 h264 的 EncodedData
    LiveVideoPacket *h264Packet = NULL;
    fillH264PacketCallback(&h264Packet, fillH264PacketContext);
    if (h264Packet == NULL) {
        printf("fillH264PacketCallback get null packet\n");
        return VIDEO_QUEUE_ABORT_ERR_CODE;
    }
    int bufferSize = (h264Packet)->size;
    uint8_t *outputData = (uint8_t *)(h264Packet->buffer);
            
    lastPresentationTimeMs = h264Packet->timeMills;
    // 填充起来我们的AVPacket
    AVPacket pkt = { 0 };
    av_init_packet(&pkt);
    pkt.stream_index = st->index;
    int64_t cal_pts = lastPresentationTimeMs / 1000.0f / av_q2d(video_st->time_base);
    int64_t pts = h264Packet->pts == PTS_PARAM_UN_SETTIED_FLAG ? cal_pts : h264Packet->pts;
    int64_t dts = h264Packet->dts == DTS_PARAM_UN_SETTIED_FLAG ? pts : h264Packet->dts == DTS_PARAM_NOT_A_NUM_FLAG ? AV_NOPTS_VALUE : h264Packet->dts;
    int nalu_type = (outputData[4] & 0x1F);
    if (nalu_type == H264_NALU_TYPE_SEQUENCE_PARAMETER_SET) {
        // 我们这里要求 sps 和 pps 一块拼接起来构造成 AVPacket 传过来
        headerSize = bufferSize;
        headerData = new uint8_t[headerSize];
        memcpy(headerData, outputData, bufferSize);
        
        uint8_t *spsFrame = 0;
        uint8_t *ppsFrame = 0;
        
        int spsFrameLen = 0;
        int ppsFrameLen = 0;
        
        parseH264SequenceHeader(headerData, headerSize, &spsFrame, spsFrameLen, &ppsFrame, ppsFrameLen);
        
        // 将 SPS 和 PPS 封装到视频编码器上下文的 extradata 中，参考 FFmpeg 源码中 avc.c
        int extradata_len = 8 + spsFrameLen - 4 + 1 + 2 + ppsFrameLen - 4;
        c->extradata = (uint8_t *)av_mallocz(extradata_len);
        c->extradata_size = extradata_len;
        c->extradata[0] = 0x01; // version
        c->extradata[1] = spsFrame[4 + 1];  // profile
        c->extradata[2] = spsFrame[4 + 2];  // profile compat
        c->extradata[3] = spsFrame[4 + 3];  // level
        c->extradata[4] = 0xFC | 3; // 保留位
        c->extradata[5] = 0xE0 | 1; // 保留位
        int tmp = spsFrameLen - 4; // 开始写 SPS
        c->extradata[6] = (tmp >> 8) & 0x00ff;
        c->extradata[7] = tmp & 0x00ff;
        int i = 0;
        for (i = 0; i < tmp; i++) {
            c->extradata[8 + i] = spsFrame[4 + i];
        }
        c->extradata[8 + tmp] = 0x01; // 结束写 SPS
        int tmp2 = ppsFrameLen - 4; // 开始写 PPS
        c->extradata[8 + tmp + 1] = (tmp2 >> 8) & 0x00ff;
        c->extradata[8 + tmp + 2] = tmp2 & 0x00ff;
        for (i = 0; i < tmp2; i++) {
            c->extradata[8 + tmp + 3 + i] = ppsFrame[4 + i];
        }
        // 结束写 PPS
        
        int ret = avformat_write_header(oc, NULL);
        if (ret < 0) {
            printf("Error occurred when opening output file: %s\n", av_err2str(ret));
        } else {
            isWriteHeaderSuccess = true;
        }
    } else {
        if (nalu_type == H264_NALU_TYPE_IDR_PICTURE || nalu_type == H264_NALU_TYPE_SEI) {
            pkt.size = bufferSize;
            pkt.data = outputData;
            
            if (pkt.data[0] == 0x00 && pkt.data[1] == 0x00 &&
                pkt.data[2] == 0x00 && pkt.data[3] == 0x01) {
                bufferSize -= 4;
                pkt.data[0] = ((bufferSize) >> 24) & 0x00ff;
                pkt.data[1] = ((bufferSize) >> 16) & 0x00ff;
                pkt.data[2] = ((bufferSize) >> 8) & 0x00ff;
                pkt.data[3] = ((bufferSize)) & 0x00ff;
                
//                printf("write_video_frame %d %d %x %x %x %x\n", nalu_type, bufferSize,
//                       pkt.data[0], pkt.data[1], pkt.data[2], pkt.data[3]);

                pkt.pts = pts;
                pkt.dts = dts;
                pkt.flags = AV_PKT_FLAG_KEY; // 标识为关键帧
                c->frame_number++;
            }
        } else {
            pkt.size = bufferSize;
            pkt.data = outputData;
            
            if (pkt.data[0] == 0x00 && pkt.data[1] == 0x00 &&
                pkt.data[2] == 0x00 && pkt.data[3] == 0x01) {
                bufferSize -= 4;
                pkt.data[0] = ((bufferSize) >> 24) & 0x00ff;
                pkt.data[1] = ((bufferSize) >> 16) & 0x00ff;
                pkt.data[2] = ((bufferSize) >> 8) & 0x00ff;
                pkt.data[3] = ((bufferSize)) & 0x00ff;
                
//                printf("write_video_frame %d %d %x %x %x %x\n", nalu_type, bufferSize,
//                       pkt.data[0], pkt.data[1], pkt.data[2], pkt.data[3]);

                pkt.pts = pts;
                pkt.dts = dts;
                pkt.flags = 0; // 标识为不是关键帧
                c->frame_number++;
            }
        }
        // 写出数据
        if (pkt.size) {
            ret = RecordingPublisher::interleavedWriteFrame(oc, &pkt);
            if (ret != 0) {
                printf("Error while writing Video frame: %s\n", av_err2str(ret));
            }
        } else {
            ret = 0;
        }
    }
    delete h264Packet;
    return ret;
}
{% endhighlight %}

#### FFmpeg 写 AAC 数据

从 AAC Audio Queue 中拿到 AAC 数据，再将其装到 AVPacket 中，再通过 av_interleaved_write_frame 来写：

{% highlight cpp %}
int RecordingPublisher::write_audio_frame(AVFormatContext *oc, AVStream *st) {
    int ret = AUDIO_QUEUE_ABORT_ERR_CODE;
    LiveAudioPacket *audioPacket = NULL;
    if ((ret = fillAACPacketCallback(&audioPacket, fillAACPacketContext)) > 0) {
        AVPacket pkt = {0};
        av_init_packet(&pkt);
        lastAudioPacketPresentationTimeMills = audioPacket->position;
        pkt.data = audioPacket->data;
        pkt.size = audioPacket->size;
        pkt.dts = pkt.pts = lastAudioPacketPresentationTimeMills / 1000.0f / av_q2d(st->time_base);
        pkt.duration = 1024;
        pkt.stream_index = st->index;
        AVPacket newPacket;
        av_init_packet(&newPacket);
        ret = av_bitstream_filter_filter(bsfc, st->codec, NULL, &newPacket.data, &newPacket.size, pkt.data, pkt.size, pkt.flags & AV_PKT_FLAG_KEY);
        if (ret >= 0) {
            newPacket.pts = pkt.pts;
            newPacket.dts = pkt.dts;
            newPacket.duration = pkt.duration;
            newPacket.stream_index = pkt.stream_index;
//            printf("write_audio_frame %d\n", newPacket.size);
            ret = this->interleavedWriteFrame(oc, &newPacket);
            if (ret != 0) {
                printf("Error while writing audio frame: %s\n", av_err2str(ret));
            }
        } else {
            printf("Error av_bitstream_filter_filter: %s\n", av_err2str(ret));
        }
        av_free_packet(&newPacket);
        av_free_packet(&pkt);
        delete audioPacket;
    } else {
        ret = AUDIO_QUEUE_ABORT_ERR_CODE;
    }
    return ret;
}
{% endhighlight %}

#### VideoConsumerThread 消费者线程的实现

VideoConsumerThread 消费者线程通过 pthread 实现，线程运行时会执行下面的代码：

{% highlight cpp %}
void VideoConsumerThread::handleRun(void *ptr) {
    while (mRunning) {
        int ret = videoPublisher->encode();
        if (ret < 0) {
            printf("videoPublisher->encode result is invalid, so we will stop encode...\n");
            break;
        }
    }
}
{% endhighlight %}

通过比较两路流上当前的时间戳信息，决定写视频流，还是写音频流：

{% highlight cpp %}
int RecordingPublisher::encode() {
    int ret = 0;
    double video_time = getVideoStreamTimeInSecs();
    double audio_time = getAudioStreamTimeInSecs();
    printf("video_time is %lf, audio_time is %f\n", video_time, audio_time);
    // 通过比较两路流上当前的时间戳信息，将时间戳比较小的那一路流进行封装和输出，音视频是交错存储的，即存储完一帧视频帧之后，再存储一段时间的音频，不一定是一帧音频，要看视频的 FPS 是多少
    if (!video_st || (video_st && audio_st && audio_time < video_time)) {
        ret = write_audio_frame(oc, audio_st);
    } else if (video_st) {
        ret = write_video_frame(oc, video_st);
    }
    sendLatestFrameTimemills = platform_4_live::getCurrentTimeMills();
    duration = MIN(audio_time, video_time);
    if (ret < 0 && VIDEO_QUEUE_ABORT_ERR_CODE != ret && AUDIO_QUEUE_ABORT_ERR_CODE != ret && !isInterrupted()) {
        if (NULL != onPublishTimeoutCallback) {
            onPublishTimeoutCallback(timeoutContext);
        }
        this->isConnected = false;
    }
    return ret;
}
{% endhighlight %}

### 生产者的关键步骤

#### 完整代码

* <em class="fab fa-github"></em> [LivePublisher.mm](https://github.com/danjiang/DTCamera/blob/master/DTCamera/CrossPlatform/Live/LivePublisher.mm)
* <em class="fab fa-github"></em> [live_audio_encoder_adapter.cpp](https://github.com/danjiang/DTCamera/blob/master/DTCamera/CrossPlatform/Live/live_audio_encoder_adapter.cpp)

#### 接收 VideoToolbox 编码的 H.264 数据

给 sps 和 pps，以及视频帧数据，再加上 startcode 0x00000001 就添加到 H.264 Video Queue：

{% highlight objc %}
- (void)gotSpsPps:(NSData*)sps pps:(NSData*)pps timestramp:(Float64)miliseconds {
    const char bytesHeader[] = "\x00\x00\x00\x01";
    size_t headerLength = 4;
    
    LiveVideoPacket *videoPacket = new LiveVideoPacket();
    
    size_t length = 2 * headerLength + sps.length + pps.length;
    videoPacket->buffer = new unsigned char[length];
    videoPacket->size = int(length);
    memcpy(videoPacket->buffer, bytesHeader, headerLength);
    memcpy(videoPacket->buffer + headerLength, (unsigned char*)[sps bytes], sps.length);
    memcpy(videoPacket->buffer + headerLength + sps.length, bytesHeader, headerLength);
    memcpy(videoPacket->buffer + headerLength * 2 + sps.length, (unsigned char*)[pps bytes], pps.length);
    videoPacket->timeMills = 0;
    
    LivePacketPool::GetInstance()->pushRecordingVideoPacketToQueue(videoPacket);
}

- (void)gotEncodedData:(NSData*)data isKeyFrame:(BOOL)isKeyFrame timestramp:(Float64)miliseconds {
    const char bytesHeader[] = "\x00\x00\x00\x01";
    size_t headerLength = 4;

    LiveVideoPacket *videoPacket = new LiveVideoPacket();
    
    videoPacket->buffer = new unsigned char[headerLength + data.length];
    videoPacket->size = int(headerLength + data.length);
    memcpy(videoPacket->buffer, bytesHeader, headerLength);
    memcpy(videoPacket->buffer + headerLength, (unsigned char*)[data bytes], data.length);
    videoPacket->timeMills = miliseconds;
    
    LivePacketPool::GetInstance()->pushRecordingVideoPacketToQueue(videoPacket);
}
{% endhighlight %}

#### 接收 AVAudioEngine 采集的 PCM 数据

注意是接收的非压缩音频数据 PCM，发送到 PCM Audio Queue：

{% highlight objc %}
- (void)receiveAudioBuffer:(AudioBuffer)buffer sampleRate:(int)sampleRate startRecordTimeMills:(Float64)startRecordTimeMills {
    double maxDiffTimeMills = 25;
    double minDiffTimeMills = 10;
    double audioSamplesTimeMills = CFAbsoluteTimeGetCurrent() * 1000 - startRecordTimeMills;
    int audioSampleRate = sampleRate;
    int audioChannels = 2;
    double dataAccumulateTimeMills = self.totalSampleCount * 1000 / audioSampleRate / audioChannels;
    if (dataAccumulateTimeMills <= audioSamplesTimeMills - maxDiffTimeMills) {
        double correctTimeMills = audioSamplesTimeMills - dataAccumulateTimeMills - minDiffTimeMills;
        int correctBufferSize = (int)(correctTimeMills / 1000.0 * audioSampleRate * audioChannels);
        LiveAudioPacket *audioPacket = new LiveAudioPacket();
        audioPacket->buffer = new short[correctBufferSize];
        memset(audioPacket->buffer, 0, correctBufferSize * sizeof(short));
        audioPacket->size = correctBufferSize;
        LivePacketPool::GetInstance()->pushAudioPacketToQueue(audioPacket);
        self.totalSampleCount += correctBufferSize;
        NSLog(@"Correct Time Mills is %lf\n", correctTimeMills);
        NSLog(@"audioSamplesTimeMills is %lf, dataAccumulateTimeMills is %lf\n", audioSamplesTimeMills, dataAccumulateTimeMills);
    }
    int sampleCount = buffer.mDataByteSize / 2;
    self.totalSampleCount += sampleCount;
    short *packetBuffer = new short[sampleCount];
    memcpy(packetBuffer, buffer.mData, buffer.mDataByteSize);
    LiveAudioPacket *audioPacket = new LiveAudioPacket();
    audioPacket->buffer = packetBuffer;
    audioPacket->size = sampleCount;
    LivePacketPool::GetInstance()->pushAudioPacketToQueue(audioPacket);
}
{% endhighlight %}

#### FFmpeg 将 PCM 编码为 AAC

从 PCM Audio Queue 中拿到 PCM 数据，FFmpeg 编码为 AAC 数据，再发送到 AAC Audio Queue，这部分的代码同 [FFmpeg 接口使用 - 音频编码和音频解码](/programming/2020/07/14/ffmpeg-api-audio-encode-decode/) 讲的道理是一样的。

## 测试验证

### 安装 RTMP 流媒体服务器

在 macOS 参考 [Mac上搭建直播服务器 Nginx+rtmp](https://www.jianshu.com/p/cf74a34af15d) 进行安装：

{% highlight text %}
$ brew tap denji/nginx
$ brew install nginx-full --with-rtmp-module
{% endhighlight %}

修改配置 /usr/local/etc/nginx/nginx.conf：

{% highlight text %}
http {
...
}

rtmp {
	server {
		listen 1935;
		chunk_size 4000;
		# TV mode: one publisher, many subscribers
		application mytv {
			live on;
			record off;
		}
	}
}
{% endhighlight %}

在 Linux 参考 <em class="fab fa-github"></em> [SRS](https://github.com/ossrs/srs) 进行安装。

### VLC 播放器

下载 [VLC](https://www.videolan.org) 播放器，开始直播后，通过下面的步骤就可以观看直播：

![VLC Media Player](/images/vlc_media_player.jpg)

![VLC Media Player RTMP](/images/vlc_media_player_rtmp.jpg)

### ffprobe

前面的代码也可以将 flv 写到本地文件，写 flv 同时也将 h264 单独写到了本地文件，通过 ffprobe 来查看：

{% highlight text %}
$ ffprobe -show_format -pretty video.h264
Input #0, h264, from 'video.h264':
  Duration: N/A, bitrate: N/A
    Stream #0:0: Video: h264 (High), yuv420p(progressive), 720x720, 25 fps, 25 tbr, 1200k tbn, 50 tbc
[FORMAT]
filename=video.h264
nb_streams=1
nb_programs=0
format_name=h264
format_long_name=raw H.264 video
start_time=N/A
duration=N/A
size=800.383789 Kibyte
bit_rate=N/A
probe_score=51
[/FORMAT]
{% endhighlight %}

{% highlight text %}
$ ffprobe -show_format -pretty video.flv
Input #0, flv, from 'video.flv':
  Metadata:
    encoder         : Lavf55.48.100
  Duration: 00:00:05.31, start: 0.000000, bitrate: 1299 kb/s
    Stream #0:0: Video: h264 (High), yuv420p(progressive), 720x720, 1500 kb/s, 24 fps, 23.92 tbr, 1k tbn
    Stream #0:1: Audio: aac (LC), 44100 Hz, stereo, fltp, 64 kb/s
[FORMAT]
filename=video.flv
nb_streams=2
nb_programs=0
format_name=flv
format_long_name=FLV (Flash Video)
start_time=0:00:00.000000
duration=0:00:05.306000
size=841.477539 Kibyte
bit_rate=1.299167 Mbit/s
probe_score=100
TAG:encoder=Lavf55.48.100
[/FORMAT]
{% endhighlight %}

### ffplay

也可以通过 ffplay 来播放 flv 和 h264：

{% highlight text %}
$ ffplay video.h264
$ ffplay video.flv
{% endhighlight %}

### [H264Parser](https://github.com/danjiang/MediaParser/blob/master/H264Parser.cpp)

H264Parser 对每一个 NALU 的输出（截取了部分）：

{% highlight text %}
-----+-------- NALU Table ------+---------+
 NUM |    POS  |    IDC |  TYPE |   LEN   |
-----+---------+--------+-------+---------+
    0|        0|     LOW|    SPS|       10|
    1|       14|     LOW|    PPS|        4|
    2|       22|  DISPOS|    SEI|       30|
    3|       56|     LOW|    IDR|     7425|
    4|     7485|     LOW|  SLICE|     1942|
    5|     9431|     LOW|  SLICE|     2136|
    6|    11571|     LOW|  SLICE|     2536|
    7|    14111|     LOW|  SLICE|     3504|
    8|    17619|     LOW|  SLICE|     3193|
    9|    20816|     LOW|  SLICE|     4105|
   10|    24925|     LOW|  SLICE|     5774|
{% endhighlight %}

### [FLVParser](https://github.com/danjiang/MediaParser/blob/master/FLVParser.cpp) 

FLVParser 对 FLV Header 以及每个 TAG 的输出（截取了部分）：

{% highlight text %}
============== FLV Header ==============
Signature:  0x F L V
Version:    0x 1
Flags  :    0x 5
HeaderSize: 0x 9
========================================
PreviousTagSize: 0 Position: 13
[SCRIPT]    293      0 |
PreviousTagSize: 304 Position: 321
[ VIDEO]     30      0 || key frame | AVC | SPS PPS
PreviousTagSize: 41 Position: 366
[ AUDIO]      4      0 |
PreviousTagSize: 15 Position: 385
[ VIDEO]     39      0 || key frame | AVC | NALU | 30
PreviousTagSize: 50 Position: 439
[ VIDEO]   7434      0 || key frame | AVC | NALU | 7425
PreviousTagSize: 7445 Position: 7888
[ AUDIO]    187      0 |
PreviousTagSize: 198 Position: 8090
[ AUDIO]    188     23 |
PreviousTagSize: 199 Position: 8293
[ VIDEO]   1951     46 || inter frame | AVC | NALU | 1942
PreviousTagSize: 1962 Position: 10259
[ AUDIO]    188     46 |
PreviousTagSize: 199 Position: 10462
[ AUDIO]    188     69 |
{% endhighlight %}