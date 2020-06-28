---
title: iOS AVFoundation - 录制视频
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: ios-av
---

iOS 中的音视频处理库 AVFoundation 内容很丰富，功能也很强大，本文主要讲解使用 ffprobe 查看视频信息和 AVFoundation 录制视频的问题。

![Camera Sea](/images/camera-sea.jpg)

## ffprobe 查看视频信息

先介绍下 ffprobe，ffprobe 是 ffmpeg 中的音视频内容分析工具，对于后面通过 AVFoundation 录制的视频， ffprobe 可以帮助我们理解一些视频信息。

下面是视频文件中需要关注的常见信息：

* 封装格式，时长，存储大小
* 视频编码，视频码率，分辨率，帧率
* 音频编码，音频采样率，音频码率，声道

通过 ffprobe 来查看：

{% highlight text %}
$ ffprobe -show_format -pretty hd1280x720.mp4
ffprobe version N-93044-g2e2b44baba Copyright (c) 2007-2019 the FFmpeg developers
  built with Apple LLVM version 10.0.0 (clang-1000.11.45.5)
  configuration: --enable-gpl --enable-nonfree --enable-libfdk-aac --enable-libmp3lame --enable-libx264 --enable-libx265
  libavutil      56. 26.100 / 56. 26.100
  libavcodec     58. 46.100 / 58. 46.100
  libavformat    58. 26.100 / 58. 26.100
  libavdevice    58.  6.101 / 58.  6.101
  libavfilter     7. 48.100 /  7. 48.100
  libswscale      5.  4.100 /  5.  4.100
  libswresample   3.  4.100 /  3.  4.100
  libpostproc    55.  4.100 / 55.  4.100
Input #0, mov,mp4,m4a,3gp,3g2,mj2, from 'hd1280x720.mp4':
  Metadata:
    major_brand     : qt
    minor_version   : 0
    compatible_brands: qt
    creation_time   : 2019-09-09T03:39:32.000000Z
  Duration: 00:00:10.11, start: 0.000000, bitrate: 5263 kb/s
    Stream #0:0(und): Video: hevc (Main) (hvc1 / 0x31637668), yuv420p(tv, bt709), 1280x720, 5165 kb/s, 29.98 fps, 29.97 tbr, 600 tbn, 600 tbc (default)
    Metadata:
      rotate          : 90
      creation_time   : 2019-09-09T03:39:32.000000Z
      handler_name    : Core Media Video
      encoder         : HEVC
    Side data:
      displaymatrix: rotation of -90.00 degrees
    Stream #0:1(und): Audio: aac (LC) (mp4a / 0x6134706D), 44100 Hz, mono, fltp, 88 kb/s (default)
    Metadata:
      creation_time   : 2019-09-09T03:39:32.000000Z
      handler_name    : Core Media Audio
[FORMAT]
filename=hd1280x720.mp4
nb_streams=2
nb_programs=0
format_name=mov,mp4,m4a,3gp,3g2,mj2
format_long_name=QuickTime / MOV
start_time=0:00:00.000000
duration=0:00:10.108333
size=6.342447 Mibyte
bit_rate=5.263410 Mbit/s
probe_score=100
TAG:major_brand=qt
TAG:minor_version=0
TAG:compatible_brands=qt
TAG:creation_time=2019-09-09T03:39:32.000000Z
[/FORMAT]
{% endhighlight %}

可以看到此视频的情况如下：

* 封装格式：format_name=mov,mp4,m4a,3gp,3g2,mj2
* 时长：duration=0:00:10.108333
* 存储大小：size=6.342447 Mibyte
* 视频编码：hevc
* 视频码率：5165 kb/s
* 分辨率：1280x720
* 帧率：29.98 fps
* 音频编码：aac
* 音频采样率：44100 Hz
* 音频码率：88 kb/s
* 声道：mono

由此可见，在后面调节录制的参数后，将视频拷贝到 macOS 上，再通过 ffprobe 来分析将会很方便。

## 搭建 AVCaptureSession

完整代码：

* <em class="fab fa-github"></em> [CameraViewController](https://github.com/danjiang/DTCamera/blob/master/DTCamera/iOS/Recording/CameraViewController.swift)
* <em class="fab fa-github"></em> [CameraPreviewView](https://github.com/danjiang/DTCamera/blob/master/DTCamera/iOS/Recording/CameraPreviewView.swift)

![iOS AVCaptureSession](/images/ios-avcapturesession.png)

上图中的第三路流程就是视频录制的 AVCaptureSession，这里使用 AVCaptureMovieFileOutput 作为输出。

第一步，获取相机使用权限：

{% highlight swift %}
switch AVCaptureDevice.authorizationStatus(for: .video) {
case .authorized:
    break
case .notDetermined:
    sessionQueue.suspend()
    AVCaptureDevice.requestAccess(for: .video) { [weak self] granted in
        guard let self = self else { return }
        if !granted {
            self.setupResult = .notAuthorized
        }
        self.sessionQueue.resume()
    }
default:
    setupResult = .notAuthorized
}
{% endhighlight %}

第二步，选择摄像头，也就是 AVCaptureDevice：

{% highlight swift %}
var defaultVideoDevice: AVCaptureDevice?
var backCameraDevice: AVCaptureDevice?
var frontCameraDevice: AVCaptureDevice?
for cameraDevice in AVCaptureDevice.devices(for: .video) {
    if cameraDevice.position == .back {
        backCameraDevice = cameraDevice
    }
    if cameraDevice.position == .front {
        frontCameraDevice = cameraDevice
    }
}
if let backCameraDevice = backCameraDevice {
    defaultVideoDevice = backCameraDevice
} else {
    defaultVideoDevice = frontCameraDevice
}
guard let videoDevice = defaultVideoDevice else {
    print("Could not find video device")
    setupResult = .configurationFailed
    session.commitConfiguration()
    return
}
{% endhighlight %}

第三步，配置预设，根据分辨率来选择，或者，根据类型是照片或视频来选择：

{% highlight swift %}
private func configSessionPreset(for videoDevice: AVCaptureDevice) {
    guard let source = source else { return }
    var presets: [AVCaptureSession.Preset] = []
    if source == .capture {
        if #available(iOS 9, *) {
            presets.append(.hd4K3840x2160)
        }
        presets.append(.hd1920x1080)
        presets.append(.photo)
        for preset in presets {
            if videoDevice.supportsSessionPreset(preset) {
                session.sessionPreset = preset
                break
            }
        }
    } else {
        presets.append(.hd1280x720)
        presets.append(.medium)
        for preset in presets {
            if videoDevice.supportsSessionPreset(preset) {
                session.sessionPreset = preset
                break
            }
        }
    }
}
{% endhighlight %}

第四步，通过 AVCaptureDevice 来调整帧率（FPS）：

{% highlight swift %}
private func configRecordingFPS(for videoDevice: AVCaptureDevice) {
    guard let source = source, source == .recording else { return }
    let desiredFrameRate = mode.config.recordingFrameRate
    var isFPSSupported = false
    for range in videoDevice.activeFormat.videoSupportedFrameRateRanges {
        if Double(desiredFrameRate) <= range.maxFrameRate,
            Double(desiredFrameRate) >= range.minFrameRate {
            isFPSSupported = true
        }
    }
    if isFPSSupported {
        do {
            try videoDevice.lockForConfiguration()
            videoDevice.activeVideoMinFrameDuration = CMTime(value: 1, timescale: CMTimeScale(desiredFrameRate))
            videoDevice.activeVideoMaxFrameDuration = CMTime(value: 1, timescale: CMTimeScale(desiredFrameRate))
            videoDevice.unlockForConfiguration()
        } catch {
            print("Could not config video device frame duration: \(error)")
        }
    }
}
{% endhighlight %}

第五步，往 AVCaptureSession 添加视频设备输入 AVCaptureDeviceInput，当然设备输入来源是之前的视频设备 AVCaptureDevice：

{% highlight swift %}
do {
    let videoDeviceInput = try AVCaptureDeviceInput(device: videoDevice)
    
    if session.canAddInput(videoDeviceInput) {
        session.addInput(videoDeviceInput)
        self.videoDeviceInput = videoDeviceInput
    } else {
        print("Could not add video device input to the session")
        setupResult = .configurationFailed
        session.commitConfiguration()
        return
    }
} catch {
    print("Could not create video device input: \(error)")
    setupResult = .configurationFailed
    session.commitConfiguration()
    return
}
{% endhighlight %}

第六步，往 AVCaptureSession 添加音频设备输入 AVCaptureDeviceInput：

{% highlight swift %}
guard let audioDevice = AVCaptureDevice.default(for: .audio) else {
    print("Could not find audio device")
    setupResult = .configurationFailed
    session.commitConfiguration()
    return
}
do {
    let audioDeviceInput = try AVCaptureDeviceInput(device: audioDevice)
    if session.canAddInput(audioDeviceInput) {
        session.addInput(audioDeviceInput)
    } else {
        print("Could not add audio device input to the session")
        setupResult = .configurationFailed
        session.commitConfiguration()
        return
    }
} catch {
    print("Could not create audio device input: \(error)")
    setupResult = .configurationFailed
    session.commitConfiguration()
    return
}
{% endhighlight %}

第七步，往 AVCaptureSession 添加视频文件输出 AVCaptureMovieFileOutput：

{% highlight swift %}
private func configSessionOutput() {
    guard let source = source else { return }
    if source == .capture {
        session.removeOutput(movieFileOutput)
        if session.canAddOutput(stillImageOutput) {
            session.addOutput(stillImageOutput)
        } else {
            print("Could not add still image output to the session")
            setupResult = .configurationFailed
            session.commitConfiguration()
            return
        }
    } else {
        session.removeOutput(stillImageOutput)
        if session.canAddOutput(movieFileOutput) {
            session.addOutput(movieFileOutput)
        } else {
            print("Could not add movie file output to the session")
            setupResult = .configurationFailed
            session.commitConfiguration()
            return
        }
    }
}
{% endhighlight %}

第八步，配置 AVCaptureMovieFileOutput 的编码和码率：

{% highlight swift %}
private func configRecordingBitRate() {
    guard let source = source, source == .recording else { return }
    if #available(iOS 12.0, *) {
        if let recordingConnection = movieFileOutput.connection(with: .video) {
            let supportedSettingsKeys = movieFileOutput.supportedOutputSettingsKeys(for: recordingConnection)
            var outputSettings: [String: Any] = [:]
            for (settingKey, settingValue) in movieFileOutput.outputSettings(for: recordingConnection) {
                if supportedSettingsKeys.contains(settingKey) {
                    if settingKey == AVVideoCompressionPropertiesKey {
                        var compressionProperties: [String: Any] = [:]
                        if let properties = settingValue as? [String: Any] {
                            for (key, value) in properties {
                                if key == AVVideoAverageBitRateKey {
                                    compressionProperties[key] = mode.config.recordingBitRate
                                } else {
                                    compressionProperties[key] = value
                                }
                            }
                        }
                        outputSettings[settingKey] = compressionProperties
                    } else {
                        outputSettings[settingKey] = settingValue
                    }
                }
            }
            movieFileOutput.setOutputSettings(outputSettings, for: recordingConnection)
        }
    }
}
{% endhighlight %}

可能可以调节的参数查看 [Video Settings](https://developer.apple.com/documentation/avfoundation/media_assets_playback_and_editing/video_settings)，尝试了发现能够调节的参数非常有限，视频编码有 h264 和 hevc，上面的代码没有设置此项，实际的情况是在新设备编码为 hevc，在旧设备编码为 h264。

## 视频预览

通过 AVCaptureVideoPreviewLayer 来实现视频画面的预览，再将前面搭建好的 AVCaptureSession 赋值给 AVCaptureVideoPreviewLayer：

{% highlight swift %}
class CameraPreviewView: UIView {
    
    var session: AVCaptureSession? {
        set {
            videoPreviewLayer?.session = newValue
            videoPreviewLayer?.videoGravity = .resizeAspectFill
        }
        get {
            return videoPreviewLayer?.session
        }
    }
    
    override class var layerClass: AnyClass {
        return AVCaptureVideoPreviewLayer.self
    }
    
    private var videoPreviewLayer: AVCaptureVideoPreviewLayer? {
        return layer as? AVCaptureVideoPreviewLayer
    }
    
}

previewView.session = session
{% endhighlight %}

## 录制视频

通过 AVCaptureMovieFileOutput 录制视频到指定文件：

{% highlight swift %}
private func recording() {
    sessionQueue.async { [weak self] in
        guard let self = self else { return }
        if !self.movieFileOutput.isRecording {
            if let outputURL = MediaViewController.getMediaFileURL(name: "video", ext: "mp4") {
                DispatchQueue.main.async { [weak self] in
                    self?.toggleRecordingControls(isHidden: false)
                }
                if UIDevice.current.isMultitaskingSupported {
                    self.backgroundRecordingID = UIApplication.shared.beginBackgroundTask(expirationHandler: nil)
                }
                self.movieFileOutput.startRecording(to: outputURL, recordingDelegate: self)
            } else {
                print("Could not create video file")
            }
        } else {
            DispatchQueue.main.async { [weak self] in
                self?.toggleRecordingControls(isHidden: true)
            }
            self.movieFileOutput.stopRecording()
        }
    }
}
{% endhighlight %}

这里还需要实现 AVCaptureFileOutputRecordingDelegate 来响应录制视频过程中一些状态的变化：

{% highlight swift %}
extension CameraViewController: AVCaptureFileOutputRecordingDelegate {
    
    func fileOutput(_ output: AVCaptureFileOutput, didStartRecordingTo fileURL: URL, from connections: [AVCaptureConnection]) {
        startTimer()
    }
    
    func fileOutput(_ output: AVCaptureFileOutput, didFinishRecordingTo outputFileURL: URL, from connections: [AVCaptureConnection], error: Error?) {
        var success = true
        
        if let error = error {
            success = false
            print("Movie file finishing error: \(error)")
        }
        
        if success {
            DispatchQueue.main.async { [weak self] in
                self?.previewVideo(outputFileURL)
            }
        } else {
            cancelTimer()
            DispatchQueue.main.async { [weak self] in
                self?.toggleRecordingControls(isHidden: true)
                self?.updateDuration()
            }
        }
        
        cleanupRecording()
    }
    
}
{% endhighlight %}

## 获取视频帧的图像

通过 AVAssetImageGenerator 来实现：

{% highlight swift %}
func generatorVideoThumbnail(video: URL, completion: @escaping (UIImage?, TimeInterval) -> Void) {
    DispatchQueue.global().async {
        let time = CMTimeMakeWithSeconds(1, preferredTimescale: 1)
        let asset = AVURLAsset(url: video, options: nil)
        if asset.tracks(withMediaType: .video).isNotEmpty {
            let duration = CMTimeGetSeconds(asset.duration)
            let imageGenerator = AVAssetImageGenerator(asset: asset)
            imageGenerator.requestedTimeToleranceAfter = CMTime.zero
            imageGenerator.requestedTimeToleranceBefore = CMTime.zero
            imageGenerator.appliesPreferredTrackTransform = true
            imageGenerator.generateCGImagesAsynchronously(
                forTimes: [NSValue(time: time)], completionHandler: { _, image, _, _, _ in
                    DispatchQueue.main.async {
                        guard let cgimg = image else {
                            completion(nil, TimeInterval(duration))
                            return
                        }
                        let image = UIImage(cgImage: cgimg)
                        completion(image, TimeInterval(duration))
                    }

                }
            )
        } else {
            DispatchQueue.main.async {
                completion(nil, 0)
            }
        }
    }   
}
{% endhighlight %}
