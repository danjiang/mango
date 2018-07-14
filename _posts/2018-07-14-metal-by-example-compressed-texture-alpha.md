---
title: Metal 示例之压缩纹理和透明
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: metal
---

本文主要通过自己对 [Metal By Example](https://gumroad.com/l/metalbyexample) 理解编写，这一篇文章讲解 Metal 中压缩纹理和透明。

![Metal By Example Cover](/images/mbe-cover.png)

## 压缩纹理

前文提到的 MTLPixelFormatRGBA8Unorm 格式的纹理是没有压缩过的，这里介绍的格式是通过有损的方式换取存储和性能上的开销，这些格式的纹理不需要解压缩，直接交给 GPU，GPU 会按需使用的时候再解压缩。

压缩纹理格式：

* S3TC，iOS 设备不支持
* PVRTC
* ETC
* ASTC

容器格式，纹理存储在文件上的时候，需要在文件头部写入关于纹理的长宽、像素格式等信息，这个文件头部就是容器格式，一个容器格式可以支持多种压缩纹理格式：

* PVR
* KTX

示例代码：

[danjiang / mbe-sample-code](https://github.com/danjiang/mbe-sample-code/tree/master/objc/09-CompressedTextures)

## 透明

透明检查（Alpha Testing），决定 Fragment Shader 返回的颜色是否需要放到 Render Buffer 中显示出来，这里利用纹理中的 Alpha 值和设置阀值进行比较来决定是否要 **discard_fragment**：

{% highlight cpp %}
float4 textureColor = texture.sample(texSampler, vert.texCoords);

if (textureColor.a < kAlphaTestReferenceValue)
	discard_fragment();
{% endhighlight %}

注意在 Fragment Shader 执行之前，已经通过 Early Depth Test 计算了物体之间的遮挡关系，所以在 Fragment Shader 中再进行透明检查可能只是浪费，造成不必要的性能浪费。

透明混合（Alpha Blending），在前后有遮挡关系的物体，前面的物体有透明值，需要计算两物体的颜色和透明值来得到最终值：

![Metal By Example Alpha Blending](/images/mbe-alpha-blending.png)

Cs 是前面物体的 RGB 值，Cd 是后面物体的 RGB 颜色值，As 是前面物体的透明值。

Metal 中通过在 Render Pipeline Descriptor 上设置来实现 Alpha Blending：

{% highlight objc %}
MTLRenderPipelineColorAttachmentDescriptor *renderbufferAttachment = pipelineDescriptor.colorAttachments[0];

renderbufferAttachment.pixelFormat = MTLPixelFormatBGRA8Unorm;

if (blendingEnabled)
{
  renderbufferAttachment.blendingEnabled = YES;
  renderbufferAttachment.rgbBlendOperation = MTLBlendOperationAdd;
  renderbufferAttachment.alphaBlendOperation = MTLBlendOperationAdd;
  
  renderbufferAttachment.sourceRGBBlendFactor = MTLBlendFactorSourceAlpha;
  renderbufferAttachment.destinationRGBBlendFactor = MTLBlendFactorOneMinusSourceAlpha;
  
  renderbufferAttachment.sourceAlphaBlendFactor = MTLBlendFactorSourceAlpha;
  renderbufferAttachment.destinationAlphaBlendFactor = MTLBlendFactorOneMinusSourceAlpha;
}
{% endhighlight %}

示例代码：

[danjiang / mbe-sample-code](https://github.com/danjiang/mbe-sample-code/tree/master/objc/10-AlphaBlending)