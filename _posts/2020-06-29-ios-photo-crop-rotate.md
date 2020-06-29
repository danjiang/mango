---
title: iOS 图片编辑之剪裁和旋转
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: ios-av
---

本文讲解在 iOS 中如何实现图片的剪裁和旋转，重点在于对 UIScrollView、手势触控、UIView 和 CALayer 的理解。

![Camera Sea](/images/camera-sea.jpg)

完整代码：

* <em class="fab fa-github"></em> [PhotoLibraryViewController](https://github.com/danjiang/DTCamera/blob/master/DTCamera/iOS/Library/PhotoLibraryViewController.swift)
* <em class="fab fa-github"></em> [PhotoEditorViewController](https://github.com/danjiang/DTCamera/blob/master/DTCamera/iOS/PhotoEditor/PhotoEditorViewController.swift)
* <em class="fab fa-github"></em> [PhotoEditorResizeView](https://github.com/danjiang/DTCamera/blob/master/DTCamera/iOS/PhotoEditor/PhotoEditorResizeView.swift)
* <em class="fab fa-github"></em> [PhotoEditorFrameView](https://github.com/danjiang/DTCamera/blob/master/DTCamera/iOS/PhotoEditor/PhotoEditorFrameView.swift)
* <em class="fab fa-github"></em> [PhotoEditorMaskView](https://github.com/danjiang/DTCamera/blob/master/DTCamera/iOS/PhotoEditor/PhotoEditorMaskView.swift)

## 剪裁

### UIScrollView

先通过浏览如下图片的示例来深入理解下 UIScrollView：

![Bicycle](/images/bicycle.jpg)

{% highlight swift %}
class PreviewPhotoViewController: UIViewController {
    
    var page = 0
    
    private let scrollView = UIScrollView()
    private let imageView = UIImageView()
    private let photo: UIImage
    private var fitScale: CGFloat = 1

    required init?(coder aDecoder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }
    
    init(photo: UIImage) {
        self.photo = photo
        
        super.init(nibName: nil, bundle: nil)
    }
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        view.backgroundColor = .black
        
        setupScrollView()
        
        imageView.image = photo
        imageView.frame = CGRect(x: 0, y: 0, width: photo.size.width, height: photo.size.height)
        scrollView.contentSize = photo.size
        
        caculateZoomScale()
        centerContents()
    }
    
    override func viewWillDisappear(_ animated: Bool) {
        super.viewWillDisappear(animated)
        
        scrollView.zoomScale = fitScale
    }
    
    private func setupScrollView() {
        scrollView.showsHorizontalScrollIndicator = false
        scrollView.showsVerticalScrollIndicator = false
        scrollView.delegate = self
        if #available(iOS 11, *) {
            scrollView.contentInsetAdjustmentBehavior = .never
        }
        
        scrollView.addSubview(imageView)
        view.addSubview(scrollView)
        
        scrollView.snp.makeConstraints { make in
            make.edges.equalToSuperview()
        }
    }
    
    private func centerContents() {
        let maxFrame = view.bounds
        var contentsFrame = imageView.frame
        
        if contentsFrame.size.width < maxFrame.size.width {
            contentsFrame.origin.x = maxFrame.minX +
                (maxFrame.size.width - contentsFrame.size.width) / 2
        } else {
            contentsFrame.origin.x = maxFrame.minX
        }
        if contentsFrame.size.height < maxFrame.size.height {
            contentsFrame.origin.y = maxFrame.minY +
                (maxFrame.size.height - contentsFrame.size.height) / 2
        } else {
            contentsFrame.origin.y = maxFrame.minY
        }
        
        imageView.frame = contentsFrame
    }
    
    private func caculateZoomScale() {
        let scaleWidth = view.bounds.size.width / photo.size.width
        let scaleHeight = view.bounds.size.height / photo.size.height
        fitScale = min(scaleWidth, scaleHeight)
        
        if fitScale < 1 {
            scrollView.minimumZoomScale = fitScale
            scrollView.maximumZoomScale = 1
        } else {
            scrollView.minimumZoomScale = fitScale
            scrollView.maximumZoomScale = fitScale * 1.5
        }
        if scrollView.zoomScale != fitScale {
            scrollView.zoomScale = fitScale
        }
    }
    
}

extension PreviewPhotoViewController: UIScrollViewDelegate {

    func viewForZooming(in scrollView: UIScrollView) -> UIView? {
        return imageView
    }
    
    func scrollViewDidZoom(_ scrollView: UIScrollView) {
        centerContents()
    }
    
}
{% endhighlight %}

#### UIScrollView 的 contentSize

如上的代码，一个 UIImageView 作为 UIScrollView 的子 View，同样放置了一张图片到了 UIImageView，并且将图片的大小设置为 UIScrollView 的 contentSize，也就如下图所示，可显示区域就是 UIScrollView 的 bounds，图片的大小超过了 UIScrollView 的 bounds，也可以看到 contentOffset 是 UIImageView 原点到 UIScrollView 原点的偏移，目前都是正的 x 和 y。

![UIScrollView contentOffset nonzero](/images/uiscrollview-contentoffset-nonzero.jpg)

#### caculateZoomScale()

caculateZoomScale() 方法用于计算出 UIScrollView 的 zoomScale 的范围，这里限定了只计算 aspectFit 模式，后面再细讲。

#### UIScrollView 的 zoomScale

UIScrollViewDelegate 中实现的代码表明，通过 pinch 手势就可以改变 UIScrollView 的 zoomScale，示例中 UIScrollView 的内容就是 UIImageView，UIScrollView 的 zoomScale 表示缩放程度，其实就是通过改变 UIScrollView 的 contentSize 来实现，contentSize 和 UIImageView 大小一致，所以就会缩放 UIImageView，当 contentSize 都小于等于 UIScrollView 的 bounds，contentOffset 就是 0，centerContents() 方法主要作用就是在这种情况下计算 UIImageView 的 frame，使其居中，如下图所示：

![UIScrollView contentOffset zero](/images/uiscrollview-contentoffset-zero.jpg)

#### UIScrollView 的 bounds

上面说到可显示区域就是 UIScrollView 的 bounds，如下图中，1 是显示图片最左上角，2 是显示图片最右上角，3 是显示图片最左下角，4 是显示图片最右下角，和 UIScrollView 的 bounds 非常的贴合，没有任何间距。

![UIScrollView bounds](/images/uiscrollview-bounds.jpg)

### 调整剪裁区域

#### View 层级关系

{% highlight swift %}
scrollView.addSubview(imageView)
view.addSubview(scrollView)
view.addSubview(maskView)
view.addSubview(resizeView)
{% endhighlight %}

三个平级的 View：

* UIScrollView 放置图片
* PhotoEditorMaskView 遮罩视图
* PhotoEditorResizeView 剪裁区域控制视图

![Resize View Hierarchy](/images/resize-view-hierarchy.jpg)

#### 重要的 CGRect

* bounds - UIScrollView 的 bounds
* maxFrame - 剪裁区域控制视图的最大 frame
* cropRectMaxFrame - 剪裁区域的最大 frame
* cropRect - 剪裁区域

![Resize Frames](/images/resize-frames.jpg)

#### 初始化

centerContents() 初始化 UIImageView 和剪裁控制区域的位置，计算和设置 UIScrollView 的 zoomScale 和 contentOffset：

{% highlight swift %}
private func centerContents() {
    imageView.image = photo
    imageView.frame = CGRect(x: 0, y: 0, width: photo.size.width, height: photo.size.height)
    scrollView.contentSize = photo.size
    
    caculateZoomScale(cropRect: resizeView.cropRectMaxFrame)

    let maxFrame = resizeView.cropRectMaxFrame
    var contentsFrame = imageView.frame
    
    if contentsFrame.size.width < maxFrame.size.width {
        contentsFrame.origin.x = maxFrame.minX +
            (maxFrame.size.width - contentsFrame.size.width) / 2
    } else {
        contentsFrame.origin.x = maxFrame.minX
    }
    if contentsFrame.size.height < maxFrame.size.height {
        contentsFrame.origin.y = maxFrame.minY +
            (maxFrame.size.height - contentsFrame.size.height) / 2
    } else {
        contentsFrame.origin.y = maxFrame.minY
    }
    
    fitBounds = view.bounds.applying(.init(translationX: contentsFrame.origin.x,
                                           y: contentsFrame.origin.y))
    
    imageView.frame = contentsFrame
    resizeView.updateFrame(wtih: contentsFrame)
    caculateContentInset(cropRect: contentsFrame)
    hideMask()
}
{% endhighlight %}

#### 触控

![Resize Touch](/images/resize-touch.jpg)

上图中的红色区域展示了上和下触控点、左上和右下的触控点，剪裁区域控制视图，一共有 8 个触控控制区域，每个触控点改变的区域大小方式也是不同的：

{% highlight swift %}
private enum ThumbPosition {
    case unknown
    case upLeftCorner
    case upSide
    case upRightCorner
    case rightSide
    case downRightCorner
    case downSide
    case downLeftCorner
    case leftSide
}
{% endhighlight %}

iOS 上处理触控分几个阶段，第一个是开始触控阶段，第二个是移动阶段，第三个和第四个是结束触控阶段，第二个移动阶段根据移动距离来调整剪裁控制区域：

{% highlight swift %}
override func touchesBegan(_ touches: Set<UITouch>, with event: UIEvent?) {
    guard let touchPoint = touches.first?.location(in: self) else { return }
    
    thumbPosition = .unknown
    var thumbPositionViewFrame: CGRect = .zero
    for (position, area) in caculateThumbAreas() {
        if area.contains(touchPoint) {
            thumbPosition = position
            thumbPositionViewFrame = area
        }
    }
    
    if isDebug && thumbPosition != .unknown {
        thumbPositionView.isHidden = false
        thumbPositionView.frame = thumbPositionViewFrame
    }
    
    delegate?.resizeViewThumbOn(self)
}

override func touchesMoved(_ touches: Set<UITouch>, with event: UIEvent?) {
    guard let touchPoint = touches.first?.location(in: self),
        let previous = touches.first?.previousLocation(in: self) else {
            return
    }
    
    let deltaWidth =  previous.x - touchPoint.x
    let deltaHeight = previous.y - touchPoint.y

    let originX = frame.origin.x
    let originY = frame.origin.y
    let width = frame.size.width
    let height = frame.size.height
    
    let originFrame = frame
    var finalFrame = originFrame
    
    switch thumbPosition {
    case .upLeftCorner:
        let scaleX = 1.0 - (-deltaWidth / width)
        let scaleY = 1.0 - (-deltaHeight / height)
        
        finalFrame.size.width = width * scaleX
        finalFrame.size.height = height * scaleY
        finalFrame.origin.x = originX + width - finalFrame.size.width
        finalFrame.origin.y = originY + height - finalFrame.size.height
    case .upSide:
        let scaleY = 1.0 - (-deltaHeight / height)

        finalFrame.size.height = height * scaleY
        finalFrame.origin.y = originY + height - finalFrame.size.height
    case .upRightCorner:
        let scaleX = 1.0 - (deltaWidth / width)
        let scaleY = 1.0 - (-deltaHeight / height)

        finalFrame.size.width = width * scaleX
        finalFrame.size.height = height * scaleY
        finalFrame.origin.y = originY + height - finalFrame.size.height
    case .rightSide:
        let scaleX = 1.0 - (deltaWidth / width)

        finalFrame.size.width = width * scaleX
    case .downRightCorner:
        let scaleX = 1.0 - (deltaWidth / width)
        let scaleY = 1.0 - (deltaHeight / height)
        
        finalFrame.size.width = width * scaleX
        finalFrame.size.height = height * scaleY
    case .downSide:
        let scaleY = 1.0 - (deltaHeight / height)
        
        finalFrame.size.height = height * scaleY
    case .downLeftCorner:
        let scaleX = 1.0 - (-deltaWidth / width)
        let scaleY = 1.0 - (deltaHeight / height)
        
        finalFrame.size.width = width * scaleX
        finalFrame.size.height = height * scaleY
        finalFrame.origin.x = originX + width - finalFrame.size.width
    case .leftSide:
        let scaleX = 1.0 - (-deltaWidth / width)
        
        finalFrame.size.width = width * scaleX
        finalFrame.origin.x = originX + width - finalFrame.size.width
    case .unknown:
        break
    }

    if finalFrame.maxX <= maxFrame.maxX &&
        finalFrame.minX >= maxFrame.minX &&
        finalFrame.maxY <= maxFrame.maxY &&
        finalFrame.minY >= maxFrame.minY &&
        finalFrame.maxX - finalFrame.minX >= minSize &&
        finalFrame.maxY - finalFrame.minY >= minSize {
        frame = finalFrame
        delegate?.resizeView(self, didResize: cropRect)
    }
    
    if isDebug && thumbPosition != .unknown {
        thumbPositionView.isHidden = true
    }
}

override func touchesEnded(_ touches: Set<UITouch>, with event: UIEvent?) {
    delegate?.resizeViewThumbOff(self)
}

override func touchesCancelled(_ touches: Set<UITouch>, with event: UIEvent?) {
    delegate?.resizeViewThumbOff(self)
}
{% endhighlight %}

还有，非 8 个触控控制区域的地方，仍然可以响应 pinch 和 pan 的手势，通过如下方法来排除这些区域，可以参考 [iOS 基础 - 处理屏幕点击事件](/programming/2020/06/11/ios-basic-handle-touch-event/) 了解更多：

{% highlight swift %}
override func point(inside point: CGPoint, with event: UIEvent?) -> Bool {
    if bounds.contains(point) {
        for area in caculateThumbAreas().values {
            if area.contains(point) {
                return true
            }
        }
    }
    return false
}
{% endhighlight %}

#### 触控回调

根据剪裁区域的调整，改变 UIScrollView 的 zoomScale 和 contentInset：

{% highlight swift %}
extension PhotoEditorViewController: PhotoEditorResizeViewDelegate {
    
    func resizeView(_ resizeView: PhotoEditorResizeView, didResize cropRect: CGRect) {
        var isEdit = false
        if cropRect.height > scrollView.contentSize.height ||
            cropRect.width > scrollView.contentSize.width {
            isEdit = true
        }
        caculateZoomScale(cropRect: cropRect, isFit: false, isEdit: isEdit)
        caculateContentInset(cropRect: cropRect)
    }
    
}
{% endhighlight %}

调整 UIScrollView 的 zoomScale：

{% highlight swift %}
private func caculateZoomScale(cropRect: CGRect, isFit: Bool = true, isEdit: Bool = true) {
    let scaleWidth = cropRect.size.width / photo.size.width
    let scaleHeight = cropRect.size.height / photo.size.height
    let scale = isFit ? min(scaleWidth, scaleHeight) : max(scaleWidth, scaleHeight)
    
    if scale < 1 {
        scrollView.minimumZoomScale = scale
        scrollView.maximumZoomScale = 1
    } else {
        scrollView.minimumZoomScale = scale
        scrollView.maximumZoomScale = scale * 1.5
    }
    if scrollView.zoomScale != scale && isEdit {
        scrollView.zoomScale = scale
    }
}
{% endhighlight %}

isFit 用于控制如下两种等比缩放模式：aspectFit 和 aspectFill。

![UIScrollView Aspect](/images/uiscrollview-aspect.jpg)

isEdit 用于控制是否改变 zoomScale，前面代码的计算，当剪裁区域大于当前 UIScrollView 的 contentSize（也就是 UIImageView 的大小）时，才设置 isEdit 为 true。

调整 UIScrollView 的 contentInset：

{% highlight swift %}
private func caculateContentInset(cropRect: CGRect) {
    scrollView.contentInset = .init(top: cropRect.minY - fitBounds.minY,
                                    left: cropRect.minX - fitBounds.minX,
                                    bottom: fitBounds.maxY - cropRect.maxY,
                                    right: fitBounds.maxX - cropRect.maxX)
}
{% endhighlight %}

注意这里没有使用 UIScrollView 的 bounds 来计算 UIScrollView 的 contentInset，而是在 bounds 的基础上做了一定的偏移：

{% highlight swift %}
fitBounds = view.bounds.applying(.init(translationX: contentsFrame.origin.x,
                                           y: contentsFrame.origin.y))
{% endhighlight %}

### 剪裁

计算出正确的区域进行图片剪裁：

{% highlight swift %}
@objc private func done() {
    let cropRect = resizeView.convert(resizeView.cropRectInResizeView, to: imageView)
    guard let correctImage = photo.cgImageCorrectedOrientation(),
        let croppedImage = correctImage.cropping(to: cropRect) else {
            return
    }
    let croppedPhoto = UIImage(cgImage: croppedImage)
    delegate?.photoEditor(viewController: self, didEdit: croppedPhoto)
}
{% endhighlight %}

## 旋转

旋转就是调整 UIImage.Orientation 来实现的，然后再走一次初始化的流程，可以看下 [图像的方向](/programming/2020/06/25/ios-capture-photo/#图像的方向) 深入理解下：

{% highlight swift %}
@objc private func rotate() {
    if let cgImage = photo.cgImage {
        switch photoOrientation {
        case .up:
            photoOrientation = .right
        case .right:
            photoOrientation = .down
        case .down:
            photoOrientation = .left
        case .left:
            photoOrientation = .up
        default:
            break
        }
        
        photo = UIImage(cgImage: cgImage, scale: 1.0, orientation: photoOrientation)
        
        // some kinds of cache can cause frame is not right, so recreate
        imageView.removeFromSuperview()
        imageView = UIImageView()
        scrollView.addSubview(imageView)
        
        centerContents()
        
        delegate?.photoEditor(viewController: self, didRotate: photo)
    }
}
{% endhighlight %}

## Mask

在没有触控操作剪裁区域时，需要显示黑色遮罩，但是剪裁区域不需要遮罩，通过 CALayer 的 mask 属性来实现：

{% highlight swift %}
func toggleMask(rect: CGRect?) {
    if let rect = rect {
        if layer.mask == nil {
            backgroundColor = UIColor(white: 0, alpha: 0.8)
            
            let maskLayer = CAShapeLayer()
            maskLayer.fillRule = .evenOdd
            
            let path = UIBezierPath(rect: bounds)
            path.append(UIBezierPath(rect: rect))
            
            maskLayer.path = path.cgPath
            
            layer.mask = maskLayer
        }
    } else {
        if layer.mask != nil {
            backgroundColor = UIColor.clear
            
            layer.mask = nil
        }
    }
}
{% endhighlight %}
