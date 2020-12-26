---
title: iOS 利用 OpenGL ES 和 AVAssetWriter 录制视频
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: ios-av
---

本文讲解了利用 OpenGL ES 和 AVAssetWriter 录制视频的解决方案，可以使用 OpenGL ES 做特效。

![Camera Sea](/images/camera-sea.jpg)

完整代码：

* <em class="fab fa-github"></em> [RecordingViewController](https://github.com/danjiang/DTCamera/blob/master/DTCamera/CrossPlatform/Capture/RecordingViewController.swift)
* <em class="fab fa-github"></em> [RecordingPipeline](https://github.com/danjiang/DTCamera/blob/master/DTCamera/CrossPlatform/Capture/RecordingPipeline.swift)
* <em class="fab fa-github"></em> [EffectFilter](https://github.com/danjiang/DTCamera/blob/master/DTCamera/CrossPlatform/Effect/EffectFilter.swift)
* <em class="fab fa-github"></em> [EffectCPUFilter](https://github.com/danjiang/DTCamera/blob/master/DTCamera/CrossPlatform/Effect/EffectCPUFilter.swift)
* <em class="fab fa-github"></em> [EffectOpenCVFilter](https://github.com/danjiang/DTCamera/blob/master/DTCamera/CrossPlatform/Effect/EffectOpenCVFilter.swift)
* <em class="fab fa-github"></em> [EffectCIFilter](https://github.com/danjiang/DTCamera/blob/master/DTCamera/CrossPlatform/Effect/EffectCIFilter.swift)
* <em class="fab fa-github"></em> [EffectOpenGLFilter](https://github.com/danjiang/DTCamera/blob/master/DTCamera/CrossPlatform/Effect/EffectOpenGLFilter.swift)
* <em class="fab fa-github"></em> [OpenGLPreviewView](https://github.com/danjiang/DTCamera/blob/master/DTCamera/CrossPlatform/Preview/OpenGLPreviewView.swift)
* <em class="fab fa-github"></em> [AssetRecorder](https://github.com/danjiang/DTCamera/blob/master/DTCamera/iOS/Recorder/AssetRecorder.swift)
* <em class="fab fa-github"></em> [RosyWriter](https://github.com/danjiang/RosyWriter2.1-Swift)

## 一段说明

<em class="fas fa-video"></em> [iOS AVFoundation - 录制视频](/programming/2020/06/27/ios-record-video/) 已经讲过如何录制视频，但其缺点是录制原视频到文件，并没有办法做一些视频效果器，所以才有了利用 OpenGL ES 和 AVAssetWriter 录制视频的解决方案，之前我主要参考了 Apple 的 Sample Code - <em class="fab fa-github"></em> [RosyWriter](https://github.com/danjiang/RosyWriter2.1-Swift) 来实现了这部分代码，后来我阅读了 <em class="fab fa-github"></em> [GPUImage](https://github.com/danjiang/GPUImage) 的源代码，发现其架构很有参考价值，尤其是效果器的实现和组件之间的衔接，才有了 <em class="fas fa-video"></em> [iOS 直播实践之视频采集、视频效果器和视频预览](/programming/2020/06/10/ios-living-video-capture-effect-preview/)，当然其代码就更复杂了，这篇文章讲解的实现方式不是最优的，但对于理解 OpenGL ES 在视频处理上的基本概念还是很有价值。

## 音视频采集

![iOS AVCaptureSession](/images/ios-avcapturesession.png)

上图中的第二路流程就是视频录制的 AVCaptureSession，这里使用 AVCaptureVideoDataOutput 作为输出，
通过实现 AVCaptureVideoDataOutputSampleBufferDelegate 和 AVCaptureAudioDataOutputSampleBufferDelegate 获取相机采集到的画面数据 CMSampleBuffer 或麦克风采集到的音频数据 CMSampleBuffer：

{% highlight swift %}
extension RecordingPipeline: AVCaptureVideoDataOutputSampleBufferDelegate, AVCaptureAudioDataOutputSampleBufferDelegate {
    
    func captureOutput(_ output: AVCaptureOutput, didOutput sampleBuffer: CMSampleBuffer, from connection: AVCaptureConnection) {
        let formatDescription = CMSampleBufferGetFormatDescription(sampleBuffer)
        if connection === videoConnection {
            let timestamp = CMSampleBufferGetPresentationTimeStamp(sampleBuffer)
            calculateFrameRate(at: timestamp)
            if videoFormatDescription == nil {
                if let formatDescription = formatDescription {
                    videoDimensions = CMVideoFormatDescriptionGetDimensions(formatDescription)
                    effectFilter.prepare(with: ratioMode, positionMode: positionMode,
                                         formatDescription: formatDescription, retainedBufferCountHint: retainedBufferCountHint)
                    if let outputFormatDescription = effectFilter.outputFormatDescription {
                        videoFormatDescription = outputFormatDescription
                    } else {
                        videoFormatDescription = formatDescription
                    }
                    if let videoFormatDescription = videoFormatDescription {
                        effectFilterVideoDimensions = CMVideoFormatDescriptionGetDimensions(videoFormatDescription)
                    }
                }
            } else if isRenderingEnabled, let inputPixelBuffer = CMSampleBufferGetImageBuffer(sampleBuffer) {
                let outputPixelBuffer = effectFilter.filter(pixelBuffer: inputPixelBuffer)
                previewPixelBuffer = outputPixelBuffer
                DispatchQueue.main.async { [weak self] in
                    guard let self = self, let previewPixelBuffer = self.previewPixelBuffer else { return }
                    self.delegate?.recordingPipeline(self, display: previewPixelBuffer)
                }
                if recordingStatus == .recording {
                    recorder?.appendVideoPixelBuffer(outputPixelBuffer, withPresentationTime: timestamp)
                }
            }
        } else if connection === audioConnection {
            audioFormatDescription = formatDescription
            if recordingStatus == .recording {
                recorder?.appendAudioSampleBuffer(sampleBuffer)
            }
        }
    }
    
}
{% endhighlight %}

## 视频效果器

### EffectFilter

EffectFilter 是视频效果器的基类：

- **prepare** - 根据画幅比（1:1、3:4、9:16）、摄像头位置（前置、后置）、输入视频图像的 CMFormatDescription 来进行设置。
- **filter** - 实现效果器，注意输入是 CVPixelBuffer，输出也是 CVPixelBuffer，CVPixelBuffer 就是一帧视频图像在内存中的表示，视频效果器就是改变视频图像在内存中的表示。
- **outputFormatDescription** - 调用 prepare 后，得到的输出视频图像的 CMFormatDescription。

{% highlight swift %}
protocol EffectFilter {
    var outputFormatDescription: CMFormatDescription? { get }
    
    func prepare(with ratioMode: CameraRatioMode, positionMode: CameraPositionMode,
                 formatDescription: CMFormatDescription, retainedBufferCountHint: Int)
    func filter(pixelBuffer: CVPixelBuffer) -> CVPixelBuffer
    func addEmitter(x: CGFloat, y: CGFloat)
}
{% endhighlight %}

### EffectCPUFilter

EffectCPUFilter 是在 CPU 上直接修改内存中的 CVPixelBuffer 数据，实现去掉 RGB 中的绿色：

{% highlight swift %}
func filter(pixelBuffer: CVPixelBuffer) -> CVPixelBuffer {
    let bytesPerPixel = 4
    
    CVPixelBufferLockBaseAddress(pixelBuffer, [])
    
    let bufferWidth = CVPixelBufferGetWidth(pixelBuffer)
    let bufferHeight = CVPixelBufferGetHeight(pixelBuffer)
    let bytesPerRow = CVPixelBufferGetBytesPerRow(pixelBuffer)
    let baseAddress = CVPixelBufferGetBaseAddress(pixelBuffer)!.assumingMemoryBound(to: UInt8.self)
    
    for row in 0..<bufferHeight {
        var pixel = baseAddress.advanced(by: Int(row * bytesPerRow))
        for _ in 0..<bufferWidth {
            pixel[1] = 0 // De-green (second pixel in BGRA is green)
            pixel += bytesPerPixel
        }
    }
    
    CVPixelBufferUnlockBaseAddress(pixelBuffer, [])
    
    return pixelBuffer
}
{% endhighlight %}

Xcode 中查看运行时的 CPU 和 FPS，CPU 占满了，FPS 只有 4 帧，上面代码中循环遍历方式来修改视频图像数据完全不可行:

![EffectCPUFilter](/images/EffectCPUFilter.jpg)

### EffectOpenCVFilter

EffectOpenCVFilter 通过 OpenCV 来实现去掉 RGB 中的蓝色：

{% highlight swift %}
func filter(pixelBuffer: CVPixelBuffer) -> CVPixelBuffer {
    CVPixelBufferLockBaseAddress(pixelBuffer, [])
    
    let bufferHeight = CVPixelBufferGetHeight(pixelBuffer)
    let bytesPerRow = CVPixelBufferGetBytesPerRow(pixelBuffer)
    let extendedWidth = bytesPerRow / MemoryLayout<UInt32>.size  // each pixel is 4 bytes or 32 bits
    let baseAddress = CVPixelBufferGetBaseAddress(pixelBuffer)!.assumingMemoryBound(to: UInt8.self)

    openCVWrapper?.filterImage(baseAddress, width: Int32(extendedWidth), height: Int32(bufferHeight))

    CVPixelBufferUnlockBaseAddress(pixelBuffer, [])
    
    return pixelBuffer
}
{% endhighlight %}

{% highlight objc %}
@implementation OpenCVWrapper

- (void)filterImage:(unsigned char *)image width:(int)width height:(int)height {
    cv::Mat bgraImage = cv::Mat(height, width, CV_8UC4, image);
    for (uint32_t y = 0; y < height; y++) {
        for (uint32_t x = 0; x < width; x++) {
            bgraImage.at<cv::Vec<uint8_t, 4>>(y, x)[0] = 0; // De-blue
        }
    }
}

@end
{% endhighlight %}

Xcode 中查看运行时的 CPU 和 FPS，CPU 占了 39%，FPS 为期望的 24 帧，OpenCV 主要也是在 CPU 中运行代码:

![EffectOpenCVFilter](/images/EffectOpenCVFilter.jpg)

### EffectCIFilter

EffectCIFilter 通过 Core Image 来实现去掉 RGB 中的红色：

{% highlight swift %}
let eaglContext = EAGLContext(api: .openGLES2)
ciContext = CIContext(eaglContext: eaglContext!, options: [.workingColorSpace : NSNull()])

filter = CIFilter(name: "CIColorMatrix")
let redCoefficients: [CGFloat] = [0, 0, 0, 0]
filter.setValue(CIVector(values: redCoefficients, count: 4), forKey: "inputRVector")
{% endhighlight %}

{% highlight swift %}
func filter(pixelBuffer: CVPixelBuffer) -> CVPixelBuffer {
    let inputImage = CIImage(cvPixelBuffer: pixelBuffer, options: nil)
    
    filter.setValue(inputImage, forKey: kCIInputImageKey)
    let outputImage = filter.value(forKey: kCIOutputImageKey) as! CIImage

    let outputPixelBuffer = createPixelBuffer()

    ciContext.render(outputImage,
                     to: outputPixelBuffer,
                     bounds: outputImage.extent,
                     colorSpace: CGColorSpaceCreateDeviceRGB())
    
    return outputPixelBuffer
}
{% endhighlight %}

Xcode 中查看运行时的 CPU 和 FPS，CPU 占了 31%，FPS 为期望的 24 帧，Core Image 实现的效果器会根据情况选择是在 GPU、还是 CPU 中运行:

![EffectCIFilter](/images/EffectCIFilter.jpg)

### EffectOpenGLFilter

EffectOpenGLFilter 通过 OpenGL ES 来实现去掉 RGB 中的绿色、加水印、加动图贴纸、加文字和点击屏幕会出现的粒子效果，实现的代码杂糅在一起没有清晰的结构，更好的架构方式可以参考 <em class="fas fa-video"></em> [iOS 直播实践之视频采集、视频效果器和视频预览](/programming/2020/06/10/ios-living-video-capture-effect-preview/)。

代码较多，就不贴了，查看 <em class="fab fa-github"></em> [EffectOpenGLFilter](https://github.com/danjiang/DTCamera/blob/master/DTCamera/CrossPlatform/Effect/EffectOpenGLFilter.swift)，这里说一下其实现的原理，将传入的视频图像 CVPixelBuffer Input 通过 CVOpenGLESTextureCacheCreateTextureFromImage 转换为 CVOpenGLESTexture 作为输入纹理，CVOpenGLESTexture 就是 OpenGL ES 纹理，再创建一个 CVPixelBuffer Output，也通过 CVOpenGLESTextureCacheCreateTextureFromImage 转换为 CVOpenGLESTexture 作为输出纹理，再将 OpenGL ES Frame Buffer 和输出纹理关联，OpenGL ES 运行 Shader Program 实现效果，CVPixelBuffer Output 就有添加了效果的视频图像，如果理解了 OpenGL ES，还是很好理解的，可以看下系列文章 <em class="fas fa-cube"></em> [OpenGL ES 图像处理](/#opengl) 来入个门。

Xcode 中查看运行时的 CPU 和 FPS，CPU 占了 22%，FPS 为期望的 24 帧，比较高效:

![EffectOpenGLFilter](/images/EffectOpenGLFilter.jpg)

## 视频预览

视频预览也是将传入的视频图像 CVPixelBuffer 通过 CVOpenGLESTextureCacheCreateTextureFromImage 转换为 CVOpenGLESTexture 作为输入纹理，再将 OpenGL ES Frame Buffer 关联到 Render Buffer 上，此 Render Buffer 和 CAEAGLLayer 有所关联，OpenGL ES 运行 Shader Program 实现直接绘制，CAEAGLLayer 就有了视频图像。

## 方向和画幅比的转换

### 方向转换

OpenGL 顶点坐标系的原点在中间：

![OpenGL 2D Coordinate](/images/opengl-2d-coordinate.jpg)

OpenGL 的纹理坐标原点在左下脚：

![OpenGL Texture Coordinates](/images/opengl-texture-coordinates.png)

iOS 的 UIKit 坐标和 Core Graphics 坐标分别如下：

![iOS Coordinates](/images/ios-coordinates.png)

下图展示了，从采集到效果器，再到预览，视频图像方向的转换：

![iOS OpenGL Texture Coordinate](/images/ios-opengl-texture-coordinate.png)

采集部分是通过直接设置 AVCaptureConnection 的方向来实现的，参考 <em class="fab fa-apple"></em> [Setting the orientation of video with AV Foundation](https://developer.apple.com/library/archive/qa/qa1744/_index.html)：

{% highlight swift %}
private func configRecordingVideoOrientation() {
    videoConnection?.videoOrientation = .portrait
}
{% endhighlight %}

预览部分方向转换通过调整纹理坐标来实现：

{% highlight swift %}
private var textureVertices: [Float] = [ // no rotation
    0, 0, // bottom left
    1, 0, // bottom right
    0, 1, // top left
    1, 1, // top right
]

private var textureVertices: [Float] = [ // vertical flip
    0, 1, // top left
    1, 1, // top right
    0, 0, // bottom left
    1, 0, // bottom right
]
{% endhighlight %}

### 画幅比转换

采集到的图像大小为 720 x 1280 px
期望的画幅比为 3:4，保持宽度不变，计算出图像大小为 720 x 960 px，需要对图像进行如下的剪裁：

![iOS Clipping Texture](/images/ios-clipping-texture.jpg)

效果器中的相关代码：

{% highlight swift %}
private var textureVertices: [Float] = [
    0, 0, // bottom left
    1, 0, // bottom right
    0, 1, // top left
    1, 1, // top right
]

let dimensions = CMVideoFormatDescriptionGetDimensions(formatDescription)
let sourceWidth = Float(dimensions.width)
let sourceHeight = Float(dimensions.height)
let targetWidth = sourceWidth
let targetHeight = targetWidth * Float(ratioMode.ratio)
let fromY = ((sourceHeight - targetHeight) / 2) / sourceHeight
let toY = 1.0 - fromY
textureVertices[1] = fromY
textureVertices[3] = fromY
textureVertices[5] = toY
textureVertices[7] = toY
let fromX: Float = 0.0
let toX: Float = 1.0
if positionMode == .front {
    textureVertices[0] = toX
    textureVertices[2] = fromX
    textureVertices[4] = toX
    textureVertices[6] = fromX
} else {
    textureVertices[0] = fromX
    textureVertices[2] = toX
    textureVertices[4] = fromX
    textureVertices[6] = toX
}
{% endhighlight %}

预览中的 CAEAGLLayer 会根据期望的画幅比调整大小，期望的画幅比为 3:4，iPhone XR 的屏幕大小为 414 x 896 points，得出 CAEAGLLayer 的大小为 414 x 552 points，这样就可以保持视频图像画幅比不变，只是进行缩放：

{% highlight swift %}
private func updatePreview() {
    previewView.snp.remakeConstraints { make in
        make.left.right.equalToSuperview()
        let offset: CGFloat = pipeline.ratioMode == .r1to1 ? 58 : 0
        if #available(iOS 11, *) {
            make.top.equalTo(self.view.safeAreaLayoutGuide.snp.top).offset(offset)
        } else {
            make.top.equalTo(self.topLayoutGuide.snp.bottom).offset(offset)
        }
        make.height.equalTo(previewView.snp.width).multipliedBy(pipeline.ratioMode.ratio)
    }
}
{% endhighlight %}

## 音视频录制

利用 AVAssetWriter 来实现音视频录制到文件：

![iOS AVAssetWriter](/images/ios-avassetwrriter.jpg)

**第一步，创建 AVAssetWriter**

{% highlight swift %}
self.assetWriter = try AVAssetWriter(outputURL: self.url, fileType: .mp4)
{% endhighlight %}

**第二步，设置视频参数**

videoTrackSourceFormatDescription 来源于前面讲的视频效果器的 outputFormatDescription，videoTrackSettings 来源于 AVCaptureVideoDataOutput 推荐的视频参数：

{% highlight swift %}
if let settings = videoDataOutput.recommendedVideoSettingsForAssetWriter(writingTo: .mp4) {
    videoCompressionSettings = settings
}
{% endhighlight %}

需设置的视频参数主要就是：视频编码、视频码率、分辨率、帧率，如果有推荐的视频参数，就只设置分辨率，如果没有，就需要全部设置：

{% highlight swift %}
guard let formatDescription = videoTrackSourceFormatDescription,
    let assetWriter = assetWriter else {
        DDLogError("Cannot setup asset writer`s video input")
        exit(1)
}

let dimensions = CMVideoFormatDescriptionGetDimensions(formatDescription)
var videoSettings = videoTrackSettings
if videoSettings.isEmpty {
    DDLogWarn("No video settings provided, using default settings")

    var bitsPerPixel: Float
    let numPixels = dimensions.width * dimensions.height
    var bitsPerSecond: Int
    
    // Assume that lower-than-SD resolutions are intended for streaming, and use a lower bitrate
    if numPixels < 640 * 480 {
        bitsPerPixel = 4.05 // This bitrate approximately matches the quality produced by AVCaptureSessionPresetMedium or Low.
    } else {
        bitsPerPixel = 10.1 // This bitrate approximately matches the quality produced by AVCaptureSessionPresetHigh.
    }
    
    bitsPerSecond = Int(Float(numPixels) * bitsPerPixel)
    
    let compressionProperties: NSDictionary = [AVVideoAverageBitRateKey : bitsPerSecond,
                                               AVVideoExpectedSourceFrameRateKey : 30,
                                               AVVideoMaxKeyFrameIntervalKey : 30]
    
    videoSettings = [AVVideoCodecKey : AVVideoCodecH264,
                     AVVideoWidthKey : dimensions.width,
                     AVVideoHeightKey : dimensions.height,
                     AVVideoCompressionPropertiesKey : compressionProperties]
} else {
    videoSettings[AVVideoWidthKey] = dimensions.width
    videoSettings[AVVideoHeightKey] = dimensions.height
}
{% endhighlight %}

**第三步，创建视频 AVAssetWriterInput**

给 AVAssetWriter 添加视频 AVAssetWriterInput：

{% highlight swift %}
if assetWriter.canApply(outputSettings: videoSettings, forMediaType: .video) {
    videoInput = AVAssetWriterInput(mediaType: .video, outputSettings: videoSettings, sourceFormatHint: formatDescription)
    videoInput.expectsMediaDataInRealTime = true
    
    if assetWriter.canAdd(videoInput) {
        assetWriter.add(videoInput)
    } else {
        DDLogError("Cannot add video input to asset writer")
        exit(1)
    }
} else {
    DDLogError("Cannot apply video settings to asset writer")
    exit(1)
}
{% endhighlight %}

**第四步，设置音频参数**

audioTrackSourceFormatDescription 来源于麦克风采集到的音频数据 CMSampleBuffer 第一帧，audioTrackSettings 来源于 AVCaptureAudioDataOutput 推荐的音频参数：

{% highlight swift %}
if let settings =  audioDataOutput.recommendedAudioSettingsForAssetWriter(writingTo: .mp4) as? [String: Any] {
    audioCompressionSettings = settings
}
{% endhighlight %}

需设置的音频参数主要就是：音频编码，音频采样率，音频码率，声道，如果有推荐的音频参数，就直接使用，如果没有，就只设置音频编码，其他参数还没有尝试过：

{% highlight swift %}
guard let formatDescription = audioTrackSourceFormatDescription,
    let assetWriter = assetWriter else {
        DDLogError("Cannot setup asset writer`s audio input")
        exit(1)
}

var audioSettings = audioTrackSettings
if audioSettings.isEmpty {
    DDLogWarn("No audio settings provided, using default settings")
    audioSettings = [AVFormatIDKey : kAudioFormatMPEG4AAC]
}
{% endhighlight %}

**第五步，创建音频 AVAssetWriterInput**

给 AVAssetWriter 添加音频 AVAssetWriterInput：

{% highlight swift %}
if assetWriter.canApply(outputSettings: audioSettings, forMediaType: .audio) {
    audioInput = AVAssetWriterInput(mediaType: .audio, outputSettings: audioSettings, sourceFormatHint: formatDescription)
    audioInput.expectsMediaDataInRealTime = true
    
    if assetWriter.canAdd(audioInput) {
        assetWriter.add(audioInput)
    } else {
        DDLogError("Cannot add audio input to asset writer")
        exit(1)
    }
} else {
    DDLogError("Cannot apply audio settings to asset writer")
    exit(1)
}
{% endhighlight %}

**第六步，AVAssetWriter 开始写入**

{% highlight swift %}
self.assetWriter?.startWriting()
{% endhighlight %}

**第七步，写入视频数据**

根据传入类型写视频或写音频：

{% highlight swift %}
private func appendSampleBuffer(_ sampleBuffer: CMSampleBuffer, ofMediaType mediaType: AVMediaType) {
    if status.rawValue < RecorderStatus.recording.rawValue {
        DDLogError("Not ready to record yet")
        exit(1)
    }

    writingQueue.async { [weak self] in
        guard let self = self else { return }
        // From the client's perspective the movie recorder can asynchronously transition to an error state as the result of an append.
        // Because of this we are lenient when samples are appended and we are no longer recording.
        // Instead of throwing an exception we just release the sample buffers and return.
        if self.status.rawValue > RecorderStatus.finishingRecordingPart1.rawValue {
            return
        }

        if !self.isSessionStarted {
            self.assetWriter?.startSession(atSourceTime: CMSampleBufferGetPresentationTimeStamp(sampleBuffer))
            self.isSessionStarted = true
        }
        
        let input = (mediaType == .video) ? self.videoInput : self.audioInput
        
        if let input = input {
            if input.isReadyForMoreMediaData {
                let success = input.append(sampleBuffer)
                if !success {
                    let error = self.assetWriter?.error
                    self.transitionToStatus(.failed, error: error)
                }
            } else {
                DDLogWarn("\(mediaType) input not ready for more media data, dropping buffer")
            }
        }
    }
}
{% endhighlight %}

外部传入 CMSampleBuffer 就是直接写，如果是 CVPixelBuffer，则还需要时间戳信息来创建 CMSampleBuffer：

{% highlight swift %}
func appendVideoSampleBuffer(_ sampleBuffer: CMSampleBuffer) {
    appendSampleBuffer(sampleBuffer, ofMediaType: .video)
}

func appendVideoPixelBuffer(_ pixelBuffer: CVPixelBuffer, withPresentationTime presentationTime: CMTime) {
    guard let formatDescription = videoTrackSourceFormatDescription else {
        DDLogError("Cannot append video pixel buffer")
        exit(1)
    }

    var sampleBuffer: CMSampleBuffer?
    
    var timingInfo = CMSampleTimingInfo()
    timingInfo.duration = .invalid
    timingInfo.decodeTimeStamp = .invalid
    timingInfo.presentationTimeStamp = presentationTime
    
    let statusCode = CMSampleBufferCreateForImageBuffer(allocator: kCFAllocatorDefault,
                                                        imageBuffer: pixelBuffer,
                                                        dataReady: true,
                                                        makeDataReadyCallback: nil,
                                                        refcon: nil,
                                                        formatDescription: formatDescription,
                                                        sampleTiming: &timingInfo,
                                                        sampleBufferOut: &sampleBuffer)
    
    if let sampleBuffer = sampleBuffer {
        self.appendSampleBuffer(sampleBuffer, ofMediaType: .video)
    } else {
        DDLogError("sample buffer create failed (\(statusCode))")
        exit(1)
    }
}
{% endhighlight %}

**第八步，写入音频数据**

{% highlight swift %}
func appendAudioSampleBuffer(_ sampleBuffer: CMSampleBuffer) {
    appendSampleBuffer(sampleBuffer, ofMediaType: .audio)
}
{% endhighlight %}

**第九步，AVAssetWriter 结束写入**

{% highlight swift %}
self.assetWriter?.finishWriting { [weak self] in
    guard let self = self else { return }
    if let error = self.assetWriter?.error {
        self.transitionToStatus(.failed, error: error)
    } else {
        self.transitionToStatus(.finished, error: nil)
    }
}
{% endhighlight %}

## ffprobe

{% highlight text %}
$ ffprobe -show_format -pretty video.mp4
Input #0, mov,mp4,m4a,3gp,3g2,mj2, from 'video.mp4':
  Metadata:
    major_brand     : mp42
    minor_version   : 1
    compatible_brands: isommp41mp42
    creation_time   : 2020-07-10T06:42:07.000000Z
  Duration: 00:00:07.25, start: 0.000000, bitrate: 2986 kb/s
    Stream #0:0(und): Video: hevc (Main) (hvc1 / 0x31637668), yuv420p(tv), 720x960, 2905 kb/s, 24 fps, 24 tbr, 600 tbn, 600 tbc (default)
    Metadata:
      creation_time   : 2020-07-10T06:42:07.000000Z
      handler_name    : Core Media Video
    Stream #0:1(und): Audio: aac (LC) (mp4a / 0x6134706D), 44100 Hz, mono, fltp, 89 kb/s (default)
    Metadata:
      creation_time   : 2020-07-10T06:42:07.000000Z
      handler_name    : Core Media Audio
[FORMAT]
filename=video.mp4
nb_streams=2
nb_programs=0
format_name=mov,mp4,m4a,3gp,3g2,mj2
format_long_name=QuickTime / MOV
start_time=0:00:00.000000
duration=0:00:07.245034
size=2.579659 Mibyte
bit_rate=2.986839 Mbit/s
probe_score=100
TAG:major_brand=mp42
TAG:minor_version=1
TAG:compatible_brands=isommp41mp42
TAG:creation_time=2020-07-10T06:42:07.000000Z
[/FORMAT]
{% endhighlight %}