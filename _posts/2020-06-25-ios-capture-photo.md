---
title: iOS AVFoundation - 拍摄照片
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: ios-av
---

iOS 中的音视频处理库 AVFoundation 内容很丰富，功能也很强大，本文主要讲解使用 AVFoundation 拍摄照片和图像方向的问题。

![Camera Sea](/images/camera-sea.jpg)

## 搭建 AVCaptureSession

完整代码：

* <em class="fab fa-github"></em> [CameraViewController](https://github.com/danjiang/DTCamera/blob/master/DTCamera/iOS/Recording/CameraViewController.swift)
* <em class="fab fa-github"></em> [UIImage+Editor](https://github.com/danjiang/DTCamera/blob/master/DTCamera/Extension/UIImage%2BEditor.swift)

![iOS AVCaptureSession](/images/ios-avcapturesession.png)

上图中的第一路流程就是拍摄照片的 AVCaptureSession，iOS 10.0+ 就可以采用 AVCapturePhotoOutput 替代 AVCaptureStillImageOutput，AVCapturePhotoOutput 提供更多拍摄照片的功能，如拍 Live Photo 等。

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

第四步，配置闪光灯：

{% highlight swift %}
do {
    if videoDevice.hasFlash {
        flashModeObservation = videoDevice.observe(\.flashMode) { [weak self] device, _ in
            self?.flashModeChanged(device)
        }
        
        try videoDevice.lockForConfiguration()
        videoDevice.flashMode = .off
        videoDevice.unlockForConfiguration()
    }
} catch {
    print("Could not config video device flash mode: \(error)")
    setupResult = .configurationFailed
    session.commitConfiguration()
    return
}
{% endhighlight %}

第五步，往 AVCaptureSession 添加设备输入 AVCaptureDeviceInput，当然设备输入来源是之前的设备 AVCaptureDevice：

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

第六步，往 AVCaptureSession 添加照片输出 AVCaptureStillImageOutput：

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

## 拍摄照片

通过 AVCaptureStillImageOutput 从 AVCaptureConnection 拿到 CMSampleBuffer，再将 CMSampleBuffer 转换成 UIImage，并对 UIImage 进行方向调整：

{% highlight swift %}
private func capture() {
    guard !isCapturing,
        let captureConnection = stillImageOutput.connection(with: .video) else {
            return
    }
    guard photos.count < mode.config.limitOfPhotos else {
        DTMessageBar.info(message: "单次最多允许拍照\(mode.config.limitOfPhotos)张")
        return
    }
    isCapturing = true
    stillImageOutput.captureStillImageAsynchronously(from: captureConnection) { [weak self] sampleBuffer, error in
        if error == nil {
            guard let sampleBuffer = sampleBuffer,
                let data = AVCaptureStillImageOutput.jpegStillImageNSDataRepresentation(sampleBuffer),
                let photo = UIImage(data: data),
                let self = self else {
                    return
            }
            let cropRectHeight = photo.size.width * self.ratioMode.ratio
            let cropRectY = (photo.size.height - cropRectHeight) / 2.0
            let cropRect = CGRect(x: 0,
                                  y: cropRectY,
                                  width: photo.size.width,
                                  height: cropRectHeight)
            guard let correctImage = photo.cgImageCorrectedOrientation(),
                let croppedImage = correctImage.cropping(to: cropRect) else {
                    return
            }
            let croppedPhoto = UIImage(cgImage: croppedImage)
            self.photos.append(croppedPhoto)
            let photosCount = self.photos.count
            DispatchQueue.main.async { [weak self] in
                self?.updateCountLabel()
                self?.collectionView.reloadData()
                self?.collectionView.scrollToItem(at: IndexPath(item: photosCount - 1, section: 0),
                                                  at: .centeredHorizontally, animated: true)
                self?.isCapturing = false
            }
        } else {
            print("Could not capture still image: \(error.debugDescription)")
            self?.isCapturing = false
        }
    }
}
{% endhighlight %}

## 图像的方向

图像就是二维的像素数组，在内存中表现就是一串连续的 Bytes 数据，要知道图像数据是怎么排列的，比如第一行像素在最开始、还是最后，才能够知道图像的方向。

[UIImage.Orientation](https://developer.apple.com/documentation/uikit/uiimage/orientation) 表示从数据排列的方向到希望显示的方向，应该做的方向调整

> Orientation values are commonly found in image metadata, and specifying image orientation correctly can be important both for displaying the image and for certain kinds of image processing.
> 
> The UIImage class automatically handles the transform necessary to present an image in the correct display orientation according to its orientation metadata, so an image object's imageOrientation property simply indicates which transform was applied.
> 
> For example, an iOS device camera always encodes pixel data in the camera sensor's native landscape orientation, along with metadata indicating the camera orientation. When UIImage loads a photo shot in portrait orientation, it automatically applies a 90° rotation before displaying the image data, and the image's imageOrientation value of UIImage.Orientation.right indicates that this rotation has been applied.
> ![UIImage Orientation Camera to Display](/images/uiimage-orientation-camera-to-display.png)

再来仔细看下拍摄照片时的代码：

{% highlight swift %}
// 转换成 UIImage
guard let sampleBuffer = sampleBuffer,
    let data = AVCaptureStillImageOutput.jpegStillImageNSDataRepresentation(sampleBuffer),
    let photo = UIImage(data: data),
    let self = self else {
        return
}
// 根据比率计算出剪裁的位置和大小
let cropRectHeight = photo.size.width * self.ratioMode.ratio
let cropRectY = (photo.size.height - cropRectHeight) / 2.0
let cropRect = CGRect(x: 0,
                      y: cropRectY,
                      width: photo.size.width,
                      height: cropRectHeight)
// 将方向为 right 的 UIImage 转换成方向为 up 的 CGImage
guard let correctImage = photo.cgImageCorrectedOrientation(),
    // 这样才能正确的剪裁
    let croppedImage = correctImage.cropping(to: cropRect) else {
        return
}
// 如果没有设定 UIImage 的方向，默认为 up，前面已经调整过 CGImage 的方向，所以这里没有问题
let croppedPhoto = UIImage(cgImage: croppedImage)
{% endhighlight %}

图像方向调整和剪裁的代码：

{% highlight swift %}
import UIKit

extension UIImage {
    
    func cgImageCorrectedOrientation() -> CGImage? {
        guard imageOrientation != .up else {
            // This is default orientation, don't need to do anything
            return cgImage
        }
        
        guard let cgImage = self.cgImage else {
            // CGImage is not available
            return nil
        }
        
        guard let colorSpace = cgImage.colorSpace,
            let ctx = CGContext(data: nil,
                                width: Int(size.width),
                                height: Int(size.height),
                                bitsPerComponent: cgImage.bitsPerComponent,
                                bytesPerRow: 0,
                                space: colorSpace,
                                bitmapInfo: CGImageAlphaInfo.premultipliedLast.rawValue) else {
            return nil // Not able to create CGContext
        }
        
        var transform: CGAffineTransform = .identity
        
        switch imageOrientation {
        case .down, .downMirrored:
            transform = transform.translatedBy(x: size.width, y: size.height)
            transform = transform.rotated(by: CGFloat.pi)
        case .left, .leftMirrored:
            transform = transform.translatedBy(x: size.width, y: 0)
            transform = transform.rotated(by: CGFloat.pi / 2.0)
        case .right, .rightMirrored:
            transform = transform.translatedBy(x: 0, y: size.height)
            transform = transform.rotated(by: CGFloat.pi / -2.0)
        case .up, .upMirrored:
            break
        @unknown default:
            break
        }
        
        // Flip image one more time if needed to, this is to prevent flipped image
        switch imageOrientation {
        case .upMirrored, .downMirrored:
            transform = transform.translatedBy(x: size.width, y: 0)
            transform = transform.scaledBy(x: -1, y: 1)
        case .leftMirrored, .rightMirrored:
            transform = transform.translatedBy(x: size.height, y: 0)
            transform = transform.scaledBy(x: -1, y: 1)
        case .up, .down, .left, .right:
            break
        @unknown default:
            break
        }
        
        ctx.concatenate(transform)
        
        switch imageOrientation {
        case .left, .leftMirrored, .right, .rightMirrored:
            ctx.draw(cgImage, in: CGRect(x: 0, y: 0, width: size.height, height: size.width))
        default:
            ctx.draw(cgImage, in: CGRect(x: 0, y: 0, width: size.width, height: size.height))
        }
        
        return ctx.makeImage()
    }
    
    func correctedOrientation() -> UIImage? {
        guard let newCGImage = cgImageCorrectedOrientation() else { return nil }
        return UIImage(cgImage: newCGImage)
    }
    
}
{% endhighlight %}