---
title: Metal 示例之基础准备
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: metal
---

本文主要通过自己对 [Metal By Example](https://gumroad.com/l/metalbyexample) 理解编写，这一篇文章讲解 Metal 基础准备。

![Metal By Example Cover](/images/mbe-cover.png)

## Device

device 是对 GPU 的抽象

{% highlight objc %}
id<MTLDevice> device
{% endhighlight %}

调用 C 函数 MTLCreateSystemDefaultDevice 返回一个 device，再将 device 配置到 metal layer 上，还需要设置 metal layer 的 pixelFormat 为 MTLPixelFormatBGRA8Unorm，每 1 个像素由蓝、绿、红和透明度 4 个单元构成，每 1 个单元是 8-bit 整数：

{% highlight objc %}
_device = MTLCreateSystemDefaultDevice();
self.metalLayer.device = _device;
self.metalLayer.pixelFormat = MTLPixelFormatBGRA8Unorm;

- (CAMetalLayer *)metalLayer {
  return (CAMetalLayer *)self.layer;
}
{% endhighlight %}

## Texture & Drawable

Texture 是图像的容器，可以只放一张图像，也可以放多张图像，这样的每一张图像叫切片。Drawable 是 metal layer 提供的，可以从中拿到 texture：

{% highlight objc %}
id<CAMetalDrawable> drawble = [self.metalLayer nextDrawable];
id<MTLTexture> texture = drawble.texture;
{% endhighlight %}

## Render Pass

render pass descriptor 告诉 Metal 在一个图像被渲染的过程中需要做什么动作，loadAction 决定前一次 texture 的内容需要清除、还是保留，storeAction 决定这次渲染的内容需要存储、还是丢弃，clearColor 是画任何内容前用来清除屏幕的颜色：

{% highlight objc %}
MTLRenderPassDescriptor *passDescriptor = [MTLRenderPassDescriptor renderPassDescriptor];
passDescriptor.colorAttachments[0].texture = texture;
passDescriptor.colorAttachments[0].loadAction = MTLLoadActionClear;
passDescriptor.colorAttachments[0].storeAction = MTLStoreActionStore;
passDescriptor.colorAttachments[0].clearColor = MTLClearColorMake(1, 0, 0, 1);
{% endhighlight %}

## Queue, Buffer & Encoder

command queue 持有一串需要执行的 command buffer，command buffer 代表一串需要执行并且作为一个整体的 render command，command encoder 将 render pass descriptor 中描述的高级指令转换为可以写入 command buffer 的低级指令 render command：

{% highlight objc %}
id<MTLCommandQueue> commandQueue = [self.device newCommandQueue];  
id<MTLCommandBuffer> commandBuffer = [commandQueue commandBuffer];
id<MTLRenderCommandEncoder> commandEncoder = [commandBuffer renderCommandEncoderWithDescriptor:passDescriptor];
[commandEncoder endEncoding];
{% endhighlight %}

最后，command buffer 将会通知 drawable 准备在屏幕上显示，只要上面的 render command 执行完毕；然后就可以 commit，表明 command buffer 已经完成，可以在放置到 command queue 发到 GPU 上执行：

{% highlight objc %}
[commandBuffer presentDrawable:drawble];
[commandBuffer commit];
{% endhighlight %}

## 代码和效果

[danjiang / MetalByExample / Basic](https://github.com/danjiang/MetalByExample/tree/basic)

![Metal By Example Basic Result](/images/mbe-basic-result.png)
