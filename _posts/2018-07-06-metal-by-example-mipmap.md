---
title: Metal 示例之 Mipmap
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: metal
---

本文主要通过自己对 [Metal By Example](https://gumroad.com/l/metalbyexample) 理解编写，这一篇文章讲解 Metal 在纹理映射中使用 Mipmap。

![Metal By Example Cover](/images/mbe-cover.png)

## Mipmap 理论

当纹理元素与屏幕像素不一致的时候，就需要过滤方式来处理两者之间的映射，纹理图像就可能会被拉升或缩小，Mipmap 先将纹理图像，按照每次比之前缩小 4/1 的方式处理，直到只有 1 个像素大小，每一次处理的图像称为 1 个等级，处理多少次就有多少个等级。

当 Mipmaped 的纹理图像在被 Sample 的时候，根据区域片段的大小来决定采用什么等级的纹理图像。

通过设置不同的 minFilter 和 mipFilter 可以产生 4 种组合：

选择最近的一个等级的纹理图像，Sample 的时候选择一个纹理元素：

{% highlight objc %}
MTLSamplerDescriptor *samplerDesc = [MTLSamplerDescriptor new];
samplerDesc.minFilter = MTLSamplerMinMagFilterNearest;
samplerDesc.mipFilter = MTLSamplerMipFilterNearest;
{% endhighlight %}

选择最近的两个等级的纹理图像，Sample 的时候从中各选一个纹理元素，再取平均得到最终结果：

{% highlight objc %}
MTLSamplerDescriptor *samplerDesc = [MTLSamplerDescriptor new];
samplerDesc.minFilter = MTLSamplerMinMagFilterNearest;
samplerDesc.mipFilter = MTLSamplerMipFilterLinear;
{% endhighlight %}

选择最近的一个等级的纹理图像，Sample 的时候从中各选四个纹理元素，再取平均得到最终结果：

{% highlight objc %}
MTLSamplerDescriptor *samplerDesc = [MTLSamplerDescriptor new];
samplerDesc.minFilter = MTLSamplerMinMagFilterLinear;
samplerDesc.mipFilter = MTLSamplerMipFilterNearest;
{% endhighlight %}

选择最近的两个等级的纹理图像，Sample 的时候从中各选四个纹理元素，再取平均得到最终结果：

{% highlight objc %}
MTLSamplerDescriptor *samplerDesc = [MTLSamplerDescriptor new];
samplerDesc.minFilter = MTLSamplerMinMagFilterLinear;
samplerDesc.mipFilter = MTLSamplerMipFilterLinear;
{% endhighlight %}

## 创建 Mipmap 各等级的纹理图像

### 创建纹理

mipmapped 为 YES

{% highlight objc %}
MTLTextureDescriptor *descriptor = [MTLTextureDescriptor texture2DDescriptorWithPixelFormat:MTLPixelFormatRGBA8Unorm
                                                                                      width:size.width
                                                                                     height:size.height
                                                                                  mipmapped:YES];
id<MTLTexture> texture = [device newTextureWithDescriptor:descriptor];
{% endhighlight %}

### 通过 CG 创建每一等级的纹理图像

{% highlight objc %}
MTLRegion region = MTLRegionMake2D(0, 0, mipWidth, mipHeight);
[texture replaceRegion:region mipmapLevel:level withBytes:[mipData bytes] bytesPerRow:mipBytesPerRow];
{% endhighlight %}

### 通过 Blit Command Encoder 创建每一等级的纹理图像

{% highlight objc %}
id<MTLBlitCommandEncoder> commandEncoder = [commandBuffer blitCommandEncoder];
[commandEncoder generateMipmapsForTexture:texture];
[commandEncoder endEncoding];
[commandBuffer addCompletedHandler:^(id<MTLCommandBuffer> buffer) {
    completionBlock(texture);
}];
[commandBuffer commit];
{% endhighlight %}

## 代码和效果

[danjiang / mbe-sample-code](https://github.com/danjiang/mbe-sample-code/tree/master/objc/07-Mipmapping)

{% youtube Px2RhCIjwBc %}