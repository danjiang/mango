---
title: iOS OpenGL ES 编程入门之基础
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: opengl
---

之前学习的 iOS OpenGL ES 编程的资料和知识比较零散，现在希望通过观看 [Ray Wenderlich - Beginning OpenGL for iOS](https://www.youtube.com/playlist?list=PL23Revp-82LL_XoQEiTT6zsgHHrpjr1D9) 的同时，把目前掌握的 OpenGL ES 知识点整理一下，第一篇文章会讲一些基本概念和基础工程搭建。

![OpenGL](/images/opengl.jpg)

## OpenGL ES 2.0

OpenGL 目的就是为了使用 GPU，OpenGL ES 1.0 没有 Shader 编程，OpenGL ES 3.0 是向前兼容 OpenGL ES 2.0 的，所以主要是针对 OpenGL ES 2.0 来讲解。

## GLKit

GLKit 是 Apple 公司提供对 OpenGL 使用的简易封装：

* GLKView and GLKViewController 渲染视图，绘制和刷新的周期方法
* GLKMath 向量和矩阵的计算
* GLKTextureLoader 纹理数据文件有不同的格式，帮助我们加载使用
* GLKBaseEffect 对 Shader 的封装，不用自己编写 Shader，所以没什么用处，毕竟我们的目的就是要自己写 Shader

> ### GLKit Features
> 
> GLKit provides functionality in four key areas:
> 
> * Texture loading allows your app to easily load textures from a variety of sources. Textures can even be loaded asynchronously in the background with just a few lines of code. For more information, see GLKTextureLoader.
> 
> * Math libraries provide commonly used vector, quaternion and matrix operations. These implementations are optimized to provide great performance.
> 
> * Effects provide standard implementations of common shader effects. You configure the effect and the associated vertex data; the effect creates and loads an appropriate shader. GLKit includes three effects: The GLKBaseEffect class implements a critical subset of the OpenGL ES 1.1 shading and lighting model, the GLKReflectionMapEffect class extends the base effect to include reflection mapping support, and the GLKSkyboxEffect class provides an implementation of a skybox effect.
> 
> * Views and View Controllers provide a standard implementation of an OpenGL ES view and a corresponding view controller. This reduces the amount of code needed to create an iOS app that use OpenGL ES. For more information, see GLKView and GLKViewController.
> 
> On iOS, GLKit requires an OpenGL ES 2.0 context. In macOS, GLKit requires an OpenGL context that supports the OpenGL 3.2 Core Profile.
>
> ---- Quote from [GLKit - Apple Developer Documentation](https://developer.apple.com/documentation/glkit#overview)

## 基础工程搭建

这里会采用两种方式来搭建基础工程：

* GLKView 和 GLKViewController，这种方式要简单些，视图是全屏的，对于学习 OpenGL ES 更方便，本系列文章都是采用此方式。
* CAEAGLLayer 和 CADisplayLink，这种方式灵活性更强一些，视图的位置和大小自己控制，渲染时机也是自己控制。

### GLKView 和 GLKViewController

<em class="fab fa-github"></em> [示例代码](https://github.com/danjiang/LearningOpenGLES2/tree/master/01.HelloOpenGL "danjiang / LearningOpenGLES2 / 01.HelloOpenGL")

{% highlight swift %}
import UIKit
import GLKit

class ViewController: GLKViewController {
    
    var glkView: GLKView!

    override func viewDidLoad() {
        super.viewDidLoad()
        
        glkView = self.view as? GLKView
        // 指定 OpenGL ES 版本来初始化 OpenGL Context
        glkView.context = EAGLContext(api: .openGLES2)!
    }
    
    override func glkView(_ view: GLKView, drawIn rect: CGRect) {
        // 设置 clear 的颜色
        glClearColor(1.0, 0.0, 0.0, 1.0)
        // clear color buffer，当然还有其他 buffer
        glClear(GLbitfield(GL_COLOR_BUFFER_BIT))
    }

}
{% endhighlight %}

### CAEAGLLayer

<em class="fab fa-github"></em> [示例代码](https://github.com/danjiang/LearningOpenGLES2/tree/master/01.HelloOpenGL2 "danjiang / LearningOpenGLES2 / 01.HelloOpenGL2")

这种方式可以说是对 GLKView 和 GLKViewController 所做事情的拆解。

首先创建一个 UIView，backend by CAEAGLLayer：

{% highlight swift %}
class OpenGLView: UIView {
        
    private var context: EAGLContext?
    private var frameBuffer = GLuint()
    private var colorRenderBuffer = GLuint()

    override class var layerClass: AnyClass {
        return CAEAGLLayer.self
    }

    private var eaglLayer: CAEAGLLayer? {
        return layer as? CAEAGLLayer
    }
    
}
{% endhighlight %}

配置 CAEAGLLayer，和创建 OpenGL Context：

{% highlight swift %}
class OpenGLView: UIView {
        
    private var context: EAGLContext?
    private var frameBuffer = GLuint()
    private var colorRenderBuffer = GLuint()

    override class var layerClass: AnyClass {
        return CAEAGLLayer.self
    }

    private var eaglLayer: CAEAGLLayer? {
        return layer as? CAEAGLLayer
    }
    
}
{% endhighlight %}

下面代码内容可以这样分步理解：

1. OpenGL Context 只会将内容绘制到 frame buffer 中
2. 设置 render buffer 和 CAEAGLLayer 共享内存的存储内容，关联 frame buffer 和 render buffer
3. opengl context draw -> frame buffer -> render buffer -> CAEAGLLayer

frame buffer 还可以和 texture 关联，将内容最终绘制到 texture 中。

> Framebuffer objects are the destination for rendering commands.
> 
> When you create a framebuffer object, you have precise control over its storage for color, depth, and stencil data. You provide this storage by attaching images to the framebuffer. The most common image attachment is a renderbuffer object.
> 
> You can also attach an OpenGL ES texture to the color attachment point of a framebuffer, which means that any drawing commands are rendered into the texture.
>
> ---- Qutoe from [OpenGL ES Programming Guide - Drawing to Other Rendering Destinations](https://developer.apple.com/library/archive/documentation/3DDrawing/Conceptual/OpenGLES_ProgrammingGuide/WorkingwithEAGLContexts/WorkingwithEAGLContexts.html#//apple_ref/doc/uid/TP40008793-CH103-SW1)

{% highlight swift %}
private func setupFrameBuffer() {
    guard let context = context, let eaglLayer = eaglLayer else { return }
    
    // 创建 frame buffer
    glGenFramebuffers(1, &frameBuffer)
    glBindFramebuffer(GLenum(GL_FRAMEBUFFER), frameBuffer)

    // 创建 render buffer
    glGenRenderbuffers(1, &colorRenderBuffer)
    glBindRenderbuffer(GLenum(GL_RENDERBUFFER), colorRenderBuffer)
    
    // eaglLayer 和 render buffer 共享数据，才能在 layer 上显示
    if !context.renderbufferStorage(Int(GL_RENDERBUFFER), from: eaglLayer) {
        print("Could not bind a drawable object’s storage to a render buffer object")
        exit(1)
    }

    // 关联 frame buffer 和 render buffer，frame buffer 内容渲染到 render buffer
    glFramebufferRenderbuffer(GLenum(GL_FRAMEBUFFER),
                              GLenum(GL_COLOR_ATTACHMENT0),
                              GLenum(GL_RENDERBUFFER),
                              colorRenderBuffer)
    
    // 检查创建 frame buffer 有没有错误
    if glCheckFramebufferStatus(GLenum(GL_FRAMEBUFFER)) != GL_FRAMEBUFFER_COMPLETE {
        print("Could not generate frame buffer")
        exit(1)
    }
}
{% endhighlight %}

下面的代码就和 **func glkView(_ view: GLKView, drawIn rect: CGRect)** 中差不多了，不过多了两步，一是设置 OpenGL Context 绘制区域的大小，还有就是将 render buffer 的内容显示到 layer 上：

{% highlight swift %}
func display() {
    let oldContext = EAGLContext.current()
    if context != oldContext {
        if !EAGLContext.setCurrent(context) {
            print("Could not set current OpenGL context with new context")
            exit(1)
        }
    }
    
    if frameBuffer == 0 {
        setupFrameBuffer()
    }
    
    // OpenGL Context 绘制的区域大小
    glViewport(0, 0, GLint(bounds.size.width), GLint(bounds.size.height))
    // 设置 clear 的颜色
    glClearColor(0.0, 0.0, 1.0, 1.0)
    // clear color buffer，当然还有其他 buffer
    glClear(GLbitfield(GL_COLOR_BUFFER_BIT))
    // 将 render buffer 的内容显示到 layer 上
    context?.presentRenderbuffer(Int(GL_RENDERBUFFER))

    if oldContext != context {
        if !EAGLContext.setCurrent(oldContext) {
            print("Could not set current OpenGL context with old context")
            exit(1)
        }
    }
}
{% endhighlight %}

最后还要创建一个定时器来告诉 OpenGL ES 重新绘制，替代 GLKViewController 帮我们做的事：

{% highlight swift %}
class ViewController: UIViewController {
    
    private let openGLView = OpenGLView()
    private var displayLink: CADisplayLink?

    override func viewDidLoad() {
        super.viewDidLoad()
                
        openGLView.frame = view.bounds
        
        view.addSubview(openGLView)
        
        displayLink = CADisplayLink(target: self, selector: #selector(drawFrame))
        displayLink?.add(to: .main, forMode: .default)
    }

    @objc private func drawFrame() {
        openGLView.display()
    }

}
{% endhighlight %}