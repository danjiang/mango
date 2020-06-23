---
title: iOS 直播实践之视频采集、视频效果器和视频预览
author: 但江
avatar: danjiang
location: 成都
category: programming
tags: ios-av featured
---

最近在 iOS 上的音视频直播开发有些心得体会，准备写一系列的文章来总结一下，本文先会整体地谈一下音视频开发，然后谈一下 iOS 生产侧的音视频开发，最后分别说一下 iOS 视频采集、iOS 视频效果器和 iOS 视频预览。

![Camera Sea](/images/camera-sea.jpg)

## 音视频开发

从工程师的角度，下图从 4 个方面来划分了音视频开发的职责，难度系数我认为从左到右是递增的：

![Audio Video Stack](/images/audio-video-stack.png)

* 消费侧：主要就是能看视频和听音频，重点就是播放器。
* 生产侧：主要就是采集音视频，直播就需要推流，录制就需要保存到文件。
* 传输侧：要连接生产和消费两端，自然就需要网络传输，采用已有协议或者自研协议，还需要对应的流媒体服务器。
* 算法侧：音视频的原始数据是不可能直接拿到网络上传输的，太大了，所以就需要算法来进行编解码，需要考虑音视频数据自身的特性，也需要考虑网络传输的场景。

## iOS 音视频开发

iOS 平台提供的音视频工具集非常丰富，下图中标红的部分是常用到：

![iOS Media Frameworks Overview](/images/ios-media-frameworks-overview.png)

### iOS 生产侧的音视频开发

![iOS Audio Video Producer](/images/ios-audio-video-producer.png)

上图就是在 iOS 上作为生产侧比较完备的流程，每一步节点表明了当前步骤和采用的技术框架，连线中表明的是流转的数据格式和网络协议。

此流程的目的主要是为了实时推流，当然也可用于录制，如果只是录制，iOS 上有直接使用 AVFoundation 的黑盒方案。

蓝色部分的流程主要是针对视频部分，绿色部分的流程主要是针对音频部分，黄色部分的流程主要是针对编码后的数据。

涉及 OpenGL ES 和 FFmpeg 的部分都可以用 C++ 做跨平台的封装，再应用到 Android 平台上，其余部分都是依赖于 iOS 平台的实现。

这张图上涉及的内容还是比较多，后面详细讲解时，还会在这张图上回顾。

### DTLiving

出于实践的目的，开发了 <em class="fab fa-github"></em> [DTLiving](https://github.com/danjiang/DTLiving) 这个 Side Project，目前已经实现了视频采集，视频效果器和视频预览。

主要参考了 <em class="fas fa-book"></em> [音视频开发进阶指南:基于Android与iOS平台的实践](https://book.douban.com/subject/30124646/) 和 <em class="fab fa-github"></em> [GPUImage](https://github.com/BradLarson/GPUImage)

DTLiving 的整体架构：

![DTLiving Architecture](/images/dtliving-architecture.png)

整个视频的处理流程，可以看作一个管道，有源头节点，有终止节点，还有中间的处理节点，节点和节点之间传递就是一个纹理，也就是 FrameBuffer。

关键的辅助类：

* <em class="fab fa-github"></em> [VideoContext](https://github.com/danjiang/DTLiving/blob/master/DTLiving/OpenGL/VideoContext.swift) 建立了 EAGLContext 和 DispatchQueue 一对一的关系，所有关于这个 EAGLContext 的操作都发送到这个 DispatchQueue 中，这个 DispatchQueue 是线性队列，可以避免并发操作 EAGLContext 会出现的问题。
* <em class="fab fa-github"></em> [ShaderProgram](https://github.com/danjiang/DTLiving/blob/master/DTLiving/Bridge/ShaderProgramObject.h) OpenGL ES Shader Program 的封装，用 C++ 写的，通过 OC 来 Bridge 到 Swift。视频部分的处理一定是会用到 OpenGL ES，可以看下系列文章 <em class="fas fa-cube"></em> [OpenGL ES 图像处理](/#opengl) 来入个门。
* <em class="fab fa-github"></em> [FrameBuffer](https://github.com/danjiang/DTLiving/blob/master/DTLiving/OpenGL/FrameBuffer.swift) 简单来说就是就一个纹理，详细来说就是将 CVPixelBuffer 作为 CVOpenGLESTexture 的数据来源，并建立了一个 OpenGL Frame Buffer，可以将这个 OpenGL Texture 和 OpenGL Frame Buffer 绑定起来。
* <em class="fab fa-github"></em> [FrameBufferCache](https://github.com/danjiang/DTLiving/blob/master/DTLiving/OpenGL/FrameBufferCache.swift) 根据纹理大小来缓存上面的 FrameBuffer。

视频处理管道上的节点：

* <em class="fab fa-github"></em> [VideoOutput](https://github.com/danjiang/DTLiving/blob/master/DTLiving/Capture/VideoOutput.swift) 视频输出节点类，可以向后续的多个分支输出纹理。
* <em class="fab fa-github"></em> [VideoInput](https://github.com/danjiang/DTLiving/blob/master/DTLiving/Preview/VideoInput.swift) 视频输入节点类，可以接收一个纹理的输入。
* <em class="fab fa-github"></em> [VideoCamera](https://github.com/danjiang/DTLiving/blob/master/DTLiving/Capture/VideoCamera.swift) 作为视频输出节点类，采集相机画面，输出纹理。
* <em class="fab fa-github"></em> [VideoFilterProcessor](https://github.com/danjiang/DTLiving/blob/master/DTLiving/Effect/VideoFilterProcessor.swift) 同时作为视频输出节点类和视频输入节点类，接收输入纹理，经过视频效果器的处理，再输出纹理。
* <em class="fab fa-github"></em> [VideoView](https://github.com/danjiang/DTLiving/blob/master/DTLiving/Preview/VideoView.swift) 作为视频输入节点类，接收输入纹理，绘制到 CAEAGLLayer 上。

## iOS 视频采集

<em class="fab fa-github"></em> [VideoCamera](https://github.com/danjiang/DTLiving/blob/master/DTLiving/Capture/VideoCamera.swift)

![iOS AVCaptureSession](/images/ios-avcapturesession.png)

首先，通过搭建上图的 AVCaptureSession 来采集相机画面，通过实现 AVCaptureVideoDataOutputSampleBufferDelegate 获取相机采集到画面数据 CMSampleBuffer。

{% highlight swift %}
extension VideoCamera: AVCaptureVideoDataOutputSampleBufferDelegate {
    
    func captureOutput(_ output: AVCaptureOutput, didOutput sampleBuffer: CMSampleBuffer, from connection: AVCaptureConnection) {
        guard session.isRunning, !capturePaused else { return }
        
        semaphore.wait()
        
        VideoContext.sharedProcessingContext.async { [weak self] in
            self?.processVideo(with: sampleBuffer)
            self?.semaphore.signal()
        }
    }
    
}
{% endhighlight %}

然后，需要搭建 OpenGL ES Pipeline 用于将相机画面转换为纹理，也会做画面方向的调整，这里主要说下如何使 CVPixelBuffer 作为创建的纹理 CVOpenGLESTexture 的数据，CVOpenGLESTexture 就可以作为 OpenGL ES 绘制时的输入纹理：

{% highlight swift %}
guard let pixelBuffer = CMSampleBufferGetImageBuffer(sampleBuffer) else { return }

var inputTexture: CVOpenGLESTexture!
let textureCache = VideoContext.sharedProcessingContext.textureCache
let resultCode = CVOpenGLESTextureCacheCreateTextureFromImage(kCFAllocatorDefault,
                                                              textureCache,
                                                              pixelBuffer,
                                                              nil,
                                                              GLenum(GL_TEXTURE_2D),
                                                              GL_RGBA,
                                                              GLsizei(bufferWidth),
                                                              GLsizei(bufferHeight),
                                                              GLenum(GL_BGRA),
                                                              GLenum(GL_UNSIGNED_BYTE),
                                                              0,
                                                              &inputTexture)
{% endhighlight %}

最后，OpenGL ES 将结果绘制到输出纹理上，通过 [FrameBuffer](https://github.com/danjiang/DTLiving/blob/master/DTLiving/OpenGL/FrameBuffer.swift) 来承载这个输出，它的实现也是将 CVPixelBuffer 作为创建的纹理 CVOpenGLESTexture 的数据，不过这个 CVPixelBuffer 数据是空的。在 OpenGL ES 绘制前，通过 [FrameBufferCache](https://github.com/danjiang/DTLiving/blob/master/DTLiving/OpenGL/FrameBufferCache.swift) 来获取 FrameBuffer，然后激活 FrameBuffer 来绑定为当前 OpenGL ES Frame Buffer，此 OpenGL ES Frame Buffer 已经和 FrameBuffer 中的纹理建立了关联，这样就可以保证 OpenGL ES 将结果绘制到此输出纹理上。

{% highlight swift %}
class FrameBuffer {
        
    func generateFrameBuffer() {
        VideoContext.sharedProcessingContext.sync {
            VideoContext.sharedProcessingContext.useAsCurrentContext()
            
            glGenFramebuffers(1, &frameBuffer)
            glBindFramebuffer(GLenum(GL_FRAMEBUFFER), frameBuffer)
                        
            let attrs: NSDictionary = [kCVPixelBufferIOSurfacePropertiesKey:  NSDictionary()]
            var resultCode = CVPixelBufferCreate(kCFAllocatorDefault, Int(size.width), Int(size.height),
                                kCVPixelFormatType_32BGRA, attrs, &pixelBuffer)
            if resultCode != kCVReturnSuccess {
                DDLogError("\(tag) Could not create pixel buffer \(resultCode)")
                exit(1)
            }
            
            let textureCache = VideoContext.sharedProcessingContext.textureCache
            resultCode = CVOpenGLESTextureCacheCreateTextureFromImage(kCFAllocatorDefault,
                                                                      textureCache,
                                                                      pixelBuffer,
                                                                      nil,
                                                                      GLenum(GL_TEXTURE_2D),
                                                                      GL_RGBA,
                                                                      GLsizei(size.width),
                                                                      GLsizei(size.height),
                                                                      GLenum(GL_BGRA),
                                                                      GLenum(GL_UNSIGNED_BYTE),
                                                                      0,
                                                                      &texture)
            if resultCode != kCVReturnSuccess {
                DDLogError("\(tag) Could not create texture \(resultCode)")
                exit(1)
            }
            
            textureName = CVOpenGLESTextureGetName(texture)
                
            glBindTexture(GLenum(GL_TEXTURE_2D), textureName)
            glTexParameteri(GLenum(GL_TEXTURE_2D), GLenum(GL_TEXTURE_MIN_FILTER), GL_LINEAR)
            glTexParameteri(GLenum(GL_TEXTURE_2D), GLenum(GL_TEXTURE_MAG_FILTER), GL_LINEAR)
            glTexParameteri(GLenum(GL_TEXTURE_2D), GLenum(GL_TEXTURE_WRAP_S), GL_CLAMP_TO_EDGE)
            glTexParameteri(GLenum(GL_TEXTURE_2D), GLenum(GL_TEXTURE_WRAP_T), GL_CLAMP_TO_EDGE)
            
            glFramebufferTexture2D(GLenum(GL_FRAMEBUFFER), GLenum(GL_COLOR_ATTACHMENT0),
                                   GLenum(GL_TEXTURE_2D), textureName, 0)
            
            if glCheckFramebufferStatus(GLenum(GL_FRAMEBUFFER)) != GL_FRAMEBUFFER_COMPLETE {
                DDLogError("\(tag) Could not generate frame buffer")
                exit(1)
            }
            
            glBindTexture(GLenum(GL_TEXTURE_2D), 0)
        }
    }
    
    func activate() {
        glBindFramebuffer(GLenum(GL_FRAMEBUFFER), frameBuffer)
        glViewport(0, 0, GLsizei(size.width), GLsizei(size.height))
    }
    
}
{% endhighlight %}

{% highlight swift %}
outputFrameBuffer = VideoContext.sharedProcessingContext.frameBufferCache
    .fetchFrameBuffer(tag: "VideoCamera",
                      for: CGSize(width: rotatedBufferWidth,
                                  height: rotatedBufferHeight))
outputFrameBuffer?.activate()
{% endhighlight %}

## iOS 视频效果器

效果器部分较复杂，主要分为 Swift 编写的 VideoFilterProcessor，Objective-C 编写的桥接部分，C++ 编写的核心部分，下面按从下层到上层的顺序来讲：

### C++ 编写的核心部分

<em class="fab fa-github"></em> [core/effect](https://github.com/danjiang/DTLiving/tree/master/DTLiving/core/effect)

首先要明白一个视频效果器，其实就是运行一个 OpenGL ES Shader Program 来进行图像处理，Shader Program 同样会有输入和输出，这里的输出主要就是纹理，输入主要由输入纹理和其他控制 Shader Program 运行的参数组成。

![DTLiving VideoEffectProcessor](/images/dtliving-videoeffectprocessor.png)

VideoEffectProcessor 作为控制视频效果器的统一接口类，内部持有一个视频效果器链，接收输入纹理，经过视频效果器链的处理，再输出纹理，这里控制视频效果器接口基本都带有 name 参数，此参数是每个视频效果器的唯一名称，导致目前不能多次添加相同的视频效果器，name 参数的作用是在视频效果器链中定位到具体的视频效果器，这些问题，之后有时间再改进吧：

{% highlight cpp %}
class VideoEffectProcessor {
public:
    VideoEffectProcessor();
    ~VideoEffectProcessor();
    
    void Init(const char *vertex_shader_file, const char *fragment_shader_file);
    void AddEffect(const char *name, const char *vertex_shader_file, const char *fragment_shader_file);
    void ClearAllEffects();
    void SetDuration(const char *name, double duration);
    void SetClearColor(const char *name, vec4 clear_color);
    void LoadResources(const char *name, std::vector<std::string> resources);
    void SetTextures(const char *name, std::vector<VideoFrame> textures);
    void SetPositions(const char *name, GLfloat *positions);
    void SetTextureCoordinates(const char *name, GLfloat *texture_coordinates);
    void SetEffectParamInt(const char *name, const char *param, GLint *value, int size);
    void SetEffectParamFloat(const char *name, const char *param, GLfloat *value, int size);
    void Process(VideoFrame input_frame, VideoFrame output_frame, double delta);
private:
    VideoEffect *no_effect_;
    std::vector<VideoEffect *> effects_ {};
};
{% endhighlight %}

![DTLiving VideoEffect](/images/dtliving-videoeffect.png)

VideoEffect 就是视频效果器，负责加载 OpenGL ES Shader Program，以及配置 Shader Program 运行需要的其他参数，运行 Shader Program，输出处理过的纹理，根据不同视频效果器处理的特点实现了不同的子类。

{% highlight cpp %}
class VideoEffect {
public:
    static std::string VertexShader();
    static std::string FragmentShader();
    static std::string GrayScaleFragmentShader();
    static std::vector<GLfloat> CaculateOrthographicMatrix(GLfloat width, GLfloat height,
                                                           bool ignore_aspect_ratio = false);

    VideoEffect(std::string name);
    ~VideoEffect();
    
    virtual void LoadShaderFile(std::string vertex_shader_file, std::string fragment_shader_file);
    virtual void LoadShaderSource();
    virtual void LoadUniform();
    virtual void LoadResources(std::vector<std::string> resources);
    virtual void SetTextures(std::vector<VideoFrame> textures);

    void SetPositions(std::vector<GLfloat> positions);
    void SetTextureCoordinates(std::vector<GLfloat> texture_coordinates);
    void SetUniform(std::string name, VideoEffectUniform uniform);
    
    bool Update(double delta, GLsizei width, GLsizei height);
    void Render(VideoFrame input_frame, VideoFrame output_frame);
};
{% endhighlight %}

几种主要的基类：

* <em class="fab fa-github"></em> [VideoTwoPassEffect](https://github.com/danjiang/DTLiving/blob/master/DTLiving/core/effect/video_two_pass_effect.h) 运行两次 Shader Program。
* <em class="fab fa-github"></em> [VideoTwoPassTextureSamplingEffect](https://github.com/danjiang/DTLiving/blob/master/DTLiving/core/effect/video_two_pass_texture_sampling_effect.h) 继承自 VideoTwoPassEffect，运行两次 Shader Program，第一次垂直地采样，第二次水平地采样。
* <em class="fab fa-github"></em> [Video3x3TextureSamplingEffect](https://github.com/danjiang/DTLiving/blob/master/DTLiving/core/effect/video_3x3_texture_sampling_effect.h) 围绕中心点，采集周围的  8 个点，加上中心点，一共采样 9 个点。
* <em class="fab fa-github"></em> [Video3x3ConvolutionEffect](https://github.com/danjiang/DTLiving/blob/master/DTLiving/core/effect/video_3x3_convolution_effect.h) 继承自 Video3x3TextureSamplingEffect，得到 3x3 的采样点后，再经过 3x3 矩阵的处理，赋值给中心点，这就是卷积。
* <em class="fab fa-github"></em> [VideoTwoInputEffect](https://github.com/danjiang/DTLiving/blob/master/DTLiving/core/effect/video_two_input_effect.h) 输入两个纹理到同一个 Shader Program，所以是在同一个矩形上混合两个纹理。
* <em class="fab fa-github"></em> [VideoCompositionEffect](https://github.com/danjiang/DTLiving/blob/master/DTLiving/core/effect/video_composition_effect.h) 同一个 Shader Program 绘制两次，第一次绘制，输入一个视频纹理，第二次绘制，输入一个图片纹理，所以可到组合两个图像的效果，也就可以实现水印、动图贴纸和文字绘制。

5 种不同类型的实现类：

* 颜色处理
	* <em class="fab fa-github"></em> [VideoBrightnessEffect](https://github.com/danjiang/DTLiving/blob/master/DTLiving/core/effect/color_processing/video_brightness_effect.h) 调整明亮度。
	* <em class="fab fa-github"></em> [VideoExposureEffect](https://github.com/danjiang/DTLiving/blob/master/DTLiving/core/effect/color_processing/video_exposure_effect.h) 调整曝光度。
	* <em class="fab fa-github"></em> [VideoContrastEffect](https://github.com/danjiang/DTLiving/blob/master/DTLiving/core/effect/color_processing/video_contrast_effect.h) 调整对比度。
	* <em class="fab fa-github"></em> [VideoSaturationEffect](https://github.com/danjiang/DTLiving/blob/master/DTLiving/core/effect/color_processing/video_saturation_effect.h) 调整饱和度。
	* <em class="fab fa-github"></em> [VideoGammaEffect](https://github.com/danjiang/DTLiving/blob/master/DTLiving/core/effect/color_processing/video_gamma_effect.h) 调整伽马值。
	* <em class="fab fa-github"></em> [VideoLevelsEffect](https://github.com/danjiang/DTLiving/blob/master/DTLiving/core/effect/color_processing/video_levels_effect.h) 类似 Photoshop 的色阶调整。
	* <em class="fab fa-github"></em> [VideoColorMatrixEffect](https://github.com/danjiang/DTLiving/blob/master/DTLiving/core/effect/color_processing/video_color_matrix_effect.h) 图像的颜色值是 1x4 的列向量，和 4x4 矩阵相乘得到新的颜色值。
	* <em class="fab fa-github"></em> [VideoSepiaFilter](https://github.com/danjiang/DTLiving/blob/master/DTLiving/Bridge/ColorProcessing/VideoSepiaFilter.h) 利用上面的 VideoColorMatrixEffect 实现怀旧效果。
	* <em class="fab fa-github"></em> [VideoRGBEffect](https://github.com/danjiang/DTLiving/blob/master/DTLiving/core/effect/color_processing/video_rgb_effect.h) 单独调整 R、G 和 B。
	* <em class="fab fa-github"></em> [VideoHueEffect](https://github.com/danjiang/DTLiving/blob/master/DTLiving/core/effect/color_processing/video_hue_effect.h) 调整色相。
	* <em class="fab fa-github"></em> [Gray Scale](https://github.com/danjiang/DTLiving/blob/master/DTLiving/core/effect/color_processing/effect_gray_scale_fragment.glsl) 灰度。
* 图像处理
	* <em class="fab fa-github"></em> [VideoTransformEffect](https://github.com/danjiang/DTLiving/blob/master/DTLiving/core/effect/image_processing/video_transform_effect.h) 对图像进行缩放、旋转、平移的 2D 或 3D 的变换。
	* <em class="fab fa-github"></em> [VideoCropFilter](https://github.com/danjiang/DTLiving/blob/master/DTLiving/Bridge/ImageProcessing/VideoCropFilter.h) 剪裁，不是真正的剪裁，图像的大小并没有改变，只是改变了可显示的内容。
	* <em class="fab fa-github"></em> [VideoGaussianBlurEffect](https://github.com/danjiang/DTLiving/blob/master/DTLiving/core/effect/image_processing/video_gaussian_blur_effect.h) 高斯模糊。
	* <em class="fab fa-github"></em> [VideoBoxBlurEffect](https://github.com/danjiang/DTLiving/blob/master/DTLiving/core/effect/image_processing/video_box_blur_effect.h) 均值模糊。
	* <em class="fab fa-github"></em> [VideoSobelEdgeDetectionEffect](https://github.com/danjiang/DTLiving/blob/master/DTLiving/core/effect/image_processing/video_sobel_edge_detection_effect.h) Sobel 边缘检测。
	* <em class="fab fa-github"></em> [VideoSharpenEffect](https://github.com/danjiang/DTLiving/blob/master/DTLiving/core/effect/image_processing/video_sharpen_effect.h) 锐化。
	* <em class="fab fa-github"></em> [VideoBilateralEffect](https://github.com/danjiang/DTLiving/blob/master/DTLiving/core/effect/image_processing/video_bilateral_effect.h) 双边滤波。
* 混合
	* <em class="fab fa-github"></em> [Add Blend](https://github.com/danjiang/DTLiving/blob/master/DTLiving/core/effect/blend/effect_add_blend_fragment.glsl) 添加混合。
	* <em class="fab fa-github"></em> [Alpha Blend](https://github.com/danjiang/DTLiving/blob/master/DTLiving/core/effect/blend/video_alpha_blend_effect.h) 透明度混合。
	* <em class="fab fa-github"></em> [VideoMaskEffect](https://github.com/danjiang/DTLiving/blob/master/DTLiving/core/effect/blend/video_mask_effect.h) 提供一张图片纹理作为视频纹理的 Mask，白色通过，黑色去掉。
	* <em class="fab fa-github"></em> [Multiply Blend](https://github.com/danjiang/DTLiving/blob/master/DTLiving/core/effect/blend/effect_multiply_blend_fragment.glsl) 正片叠底混合。
	* <em class="fab fa-github"></em> [Screen Blend](https://github.com/danjiang/DTLiving/blob/master/DTLiving/core/effect/blend/effect_screen_blend_fragment.glsl) 滤色混合。
	* <em class="fab fa-github"></em> [Overlay Blend](https://github.com/danjiang/DTLiving/blob/master/DTLiving/core/effect/blend/effect_overlay_blend_fragment.glsl) 叠加混合。
	* <em class="fab fa-github"></em> [Soft Light Blend](https://github.com/danjiang/DTLiving/blob/master/DTLiving/core/effect/blend/effect_soft_light_blend_fragment.glsl) 柔光混合。
	* <em class="fab fa-github"></em> [Hard Light Blend](https://github.com/danjiang/DTLiving/blob/master/DTLiving/core/effect/blend/effect_hard_light_blend_fragment.glsl) 强光混合。
* 组合
	* <em class="fab fa-github"></em> [VideoWaterMaskEffect](https://github.com/danjiang/DTLiving/blob/master/DTLiving/core/effect/composition/video_water_mask_effect.h) 加水印，其实就是加一张图片。
	* <em class="fab fa-github"></em> [VideoAnimatedStickerEffect](https://github.com/danjiang/DTLiving/blob/master/DTLiving/core/effect/composition/video_animated_sticker_effect.h) 动图贴纸，通过一组图片序列来完成动画。
	* <em class="fab fa-github"></em> [VideoTextEffect](https://github.com/danjiang/DTLiving/blob/master/DTLiving/core/effect/composition/video_text_effect.h) 加文字，文字由 iOS 系统绘制成图片传入。
* 效果
	* <em class="fab fa-github"></em> [VideoEmbossFilter](https://github.com/danjiang/DTLiving/blob/master/DTLiving/Bridge/Effect/VideoEmbossFilter.h) 复古效果。
	* <em class="fab fa-github"></em> [VideoToonEffect](https://github.com/danjiang/DTLiving/blob/master/DTLiving/core/effect/effect/video_toon_effect.h) 卡通效果。
	* <em class="fab fa-github"></em> [VideoSketchEffect](https://github.com/danjiang/DTLiving/blob/master/DTLiving/core/effect/effect/video_sketch_effect.h) 素描效果。
	* <em class="fab fa-github"></em> [VideoMosaicEffect](https://github.com/danjiang/DTLiving/blob/master/DTLiving/core/effect/effect/video_mosaic_effect.h) 马赛克效果。

这部分的内容挺多，有时间再细讲。

### Objective-C 编写的桥接部分

<em class="fab fa-github"></em> [Bridge](https://github.com/danjiang/DTLiving/tree/master/DTLiving/Bridge)

一个 VideoFilter 对应一个 VideoEffect，前面讲了 VideoEffect 才是运行 Shader Program 实现效果，这里的 VideoFilter 只是作为数据模型，VideoEffectProcessorObject 内部持有 VideoEffectProcessor，通过 VideoEffectProcessorObject 向 C++ 编写的核心部分传递 VideoFilter 数据，这样就达到了控制效果器的目的，不过目前需要写很多冗余代码，其本质就是 Objective-C 层和 C++ 层之间的数据交换，只要定义好数据交换的协议，后期可以考虑通过 Protocol Buffers 来自动生成：

{% highlight objc %}
@interface VideoFilter : NSObject

@property (nonatomic, copy) NSString *name;
@property (nonatomic, assign) double duration;
@property (nonatomic, copy, readonly) NSString *vertexShaderFile;
@property (nonatomic, copy, readonly) NSString *fragmentShaderFile;
@property (nonatomic, copy, readonly) NSArray<NSNumber*> *positions;
@property (nonatomic, copy, readonly) NSArray<NSNumber*> *textureCoordinates;
@property (nonatomic, copy, readonly) NSDictionary<NSString*, NSArray<NSNumber*>*> *intParams;
@property (nonatomic, copy, readonly) NSDictionary<NSString*, NSArray<NSNumber*>*> *floatParams;
@property (nonatomic, copy, readonly) NSArray<NSString*> *resources;
@property (nonatomic, copy, readonly) NSArray<GLKTextureInfo*> *textures;
@property (nonatomic, assign) VideoVec4 backgroundColor;
@property (nonatomic, assign, readonly) BOOL isRotationAware;
@property (nonatomic, assign) VideoRotation rotation;
@property (nonatomic, assign, readonly) BOOL isSizeAware;
@property (nonatomic, assign) CGSize size;

- (instancetype)initWithName:(const char *)name;

- (NSArray<NSNumber*> *)sizeToArray:(CGSize)size;
- (NSArray<NSNumber*> *)boolToArray:(BOOL)isYES;
- (NSArray<NSNumber*> *)vec2ToArray:(VideoVec2)vec;
- (NSArray<NSNumber*> *)vec3ToArray:(VideoVec3)vec;
- (NSArray<NSNumber*> *)vec4ToArray:(VideoVec4)vec;
- (NSArray<NSNumber*> *)mat3ToArray:(VideoMat3)mat;
- (NSArray<NSNumber*> *)mat4ToArray:(VideoMat4)mat;

@end
{% endhighlight %}

{% highlight objc %}
@interface VideoEffectProcessorObject : NSObject

- (instancetype)init;
- (void)addFilter:(VideoFilter *)filter;
- (void)updateFilter:(VideoFilter *)filter;
- (void)processs:(GLuint)inputTexture outputTexture:(GLuint)outputTexture size:(CGSize)size delta:(double)delta;
- (void)clearAllFilters;

@end
{% endhighlight %}

### VideoFilterProcessor

<em class="fab fa-github"></em> [VideoFilterProcessor.swift](https://github.com/danjiang/DTLiving/blob/master/DTLiving/Effect/VideoFilterProcessor.swift)

VideoFilterProcessor 也是视频处理管道上的节点，同时作为视频输出节点类和视频输入节点类，接收输入纹理，经过视频效果器的处理，再输出纹理。同样 VideoFilterProcessor 内部持有 VideoEffectProcessorObject 来控制效果器：

{% highlight swift %}
func clearAllFilters() {
    filters.removeAll()
    VideoContext.sharedProcessingContext.sync {
        processor.clearAllFilters()
    }
}

var numberOfFilters: Int {
    return filters.count
}

func fetchFilter(at index: Int) -> VideoFilter {
    return filters[index]
}

func addFilter(_ filter: VideoFilter) {
    if filter.isRotationAware {
        filter.rotation = inputRotation
    }
    if filter.isSizeAware {
        filter.size = inputSize
    }
    filters.append(filter)
    VideoContext.sharedProcessingContext.sync {
        processor.add(filter)
    }
}
    
func updateFilter(_ filter: VideoFilter, at index: Int) {
    filters[index] = filter
    updateFilter(filter)
}
{% endhighlight %}

可以添加多个视频效果器组成一个视频效果器的处理链：

{% highlight swift %}
filterProcessor = VideoFilterProcessor()
filterProcessor.addFilter(VideoGaussianBlurFilter())
filterProcessor.addFilter(VideoToonFilter())
{% endhighlight %}

## iOS 视频预览

<em class="fab fa-github"></em> [VideoView](https://github.com/danjiang/DTLiving/blob/master/DTLiving/Preview/VideoView.swift)

接收输入纹理，通过 OpenGL ES 绘制到 CAEAGLLayer 上，前面说过将 OpenGL ES Frame Buffer 关联到一个纹理上，就可以绘制到纹理上，这里需要将 OpenGL ES Frame Buffer 关联到 Render Buffer 上，此 Render Buffer 和 CAEAGLLayer 有所关联，这样就可以保证 OpenGL ES 将结果绘制到 CAEAGLLayer上。

{% highlight swift %}
private func createDisplayFrameBuffer() {
    VideoContext.sharedProcessingContext.useAsCurrentContext()

    glGenFramebuffers(1, &displayFrameBuffer)
    glBindFramebuffer(GLenum(GL_FRAMEBUFFER), displayFrameBuffer)
    
    glGenRenderbuffers(1, &displayRenderBuffer)
    glBindRenderbuffer(GLenum(GL_RENDERBUFFER), displayRenderBuffer)
    
    if !VideoContext.sharedProcessingContext.context
        .renderbufferStorage(Int(GL_RENDERBUFFER), from: eaglLayer) {
        DDLogError("Could not bind a drawable object’s storage to a render buffer object")
        exit(1)
    }
    
    var backingWidth: GLint = 0
    var backingHeight: GLint = 0
    
    glGetRenderbufferParameteriv(GLenum(GL_RENDERBUFFER), GLenum(GL_RENDERBUFFER_WIDTH), &backingWidth)
    glGetRenderbufferParameteriv(GLenum(GL_RENDERBUFFER), GLenum(GL_RENDERBUFFER_HEIGHT), &backingHeight)
    
    if backingWidth == 0 || backingHeight == 0 {
        destroyDisplayFrameBuffer()
        return
    }
    
    outputSize.width = CGFloat(backingWidth)
    outputSize.height = CGFloat(backingHeight)
    
    glFramebufferRenderbuffer(GLenum(GL_FRAMEBUFFER),
                              GLenum(GL_COLOR_ATTACHMENT0),
                              GLenum(GL_RENDERBUFFER),
                              displayRenderBuffer)
    
    if glCheckFramebufferStatus(GLenum(GL_FRAMEBUFFER)) != GL_FRAMEBUFFER_COMPLETE {
        DDLogError("[VideoView] Could not generate frame buffer")
        exit(1)
    }
            
    recalculateViewGeometry()
}
{% endhighlight %}
