---
title: Metal 示例之纹理
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: metal
---

本文主要通过自己对 [Metal By Example](https://gumroad.com/l/metalbyexample) 理解编写，这一篇文章讲解 Metal 如何画 3D。

![Metal By Example Cover](/images/mbe-cover.png)

## 纹理、纹理映射、坐标体系

如下的左图是在 3D 建模软件中制作的模型，如下的右图就是将模型导出的时候，分离出来的纹理，当然导出文件中还包含 OBJ 模型；纹理映射就是将网格模型中每一个顶点和纹理中一个点结合起来，类似于一张 2D 包装纸裹到 3D 物体表面。

![Metal By Example Texture Mapping](/images/mbe-texture-mapping.png)

Metal 中坐标体系基于左上角，UIKit 中坐标体系基于左上角，OpenGL 中坐标体系基于左下角，CG 中坐标体系基于左下角，所以通过 CG 读取的纹理图像要转换来适配 Metal 的坐标体系。

Metal 中纹理的坐标可以通过像素坐标或者规格化坐标来指定。

## 过滤

纹理就是有限数量像素组成的图像，这里像素称为纹理元素（texels），然而在绘制的时候，纹理可能被绘制在高于其大小的区域或者小于其大小的区域，前者是放大，需要处理纹理元素覆盖不到的区域，如何填充颜色值？后者是缩小，需要处理很多纹理元素挤在一个区域，如何计算颜色值？

过滤就是为了解决上面的问题，下面是两种方案：

1. Nearest，选取最近的纹理元素来填充，速度快，放大时会出现块状的现象；
2. Linear，选取最近 4 个纹理元素来计算填充值，速度足够快，效果好。

## Mipmap

Mipmap 用来可以解决缩小时，纹理闪动的现象，[下一篇文章](/programming/2018/07/06/metal-by-example-mipmap/)再详细讲解。

## Addressing

纹理映射就是将网格模型中每一个顶点和纹理中一个点结合起来，在纹理映射时，纹理坐标可能会超出 [0, 1]，超出的时候怎么处理呢，就需要 Addressing：

这是原始的纹理：

![Metal By Example Addressing Origin](/images/mbe-addressing-origin.png)

### Clamp-to-Edge Addressing

重复边界上的值

![Metal By Example Addressing Origin](/images/mbe-addressing-edge.png)

### Clamp-to-Zero Addressing

全黑色或者 Clear Color，基于纹理是否有 Alpha 色值

![Metal By Example Addressing Zero](/images/mbe-addressing-zero.png)

### Repeat Addressing

拿纹理来重复铺满

![Metal By Example Addressing Repeat](/images/mbe-addressing-repeat.png)

### Mirrored Repeat Addressing

相邻的纹理成镜像显示而不是重复显示

![Metal By Example Addressing Mirrored](/images/mbe-addressing-mirrored.png)

## Metal 中加载纹理

图像加载到内存中，需要一种格式来存储，RGBA 就是一种，Metal 中支持的像素格式在 MTLPixelFormat 中描述。

加载图像：

{% highlight objc %}
UIImage *image = [UIImage imageNamed:imageName];
{% endhighlight %}

通过 CG 绘制成位图：

{% highlight objc %}
- (uint8_t *)dataForImage:(UIImage *)image {
    CGImageRef imageRef = image.CGImage;
    
    // Create a suitable bitmap context for extracting the bits of the image
    const NSUInteger width = CGImageGetWidth(imageRef);
    const NSUInteger height = CGImageGetHeight(imageRef);
    CGColorSpaceRef colorSpace = CGColorSpaceCreateDeviceRGB();
    uint8_t *rawData = (uint8_t *)calloc(height * width * 4, sizeof(uint8_t));
    const NSUInteger bytesPerPixel = 4;
    const NSUInteger bytesPerRow = bytesPerPixel * width;
    const NSUInteger bitsPerComponent = 8;
    CGContextRef context = CGBitmapContextCreate(rawData, width, height,
                                                 bitsPerComponent, bytesPerRow, colorSpace,
                                                 kCGImageAlphaPremultipliedLast | kCGBitmapByteOrder32Big);
    CGColorSpaceRelease(colorSpace);

    CGContextTranslateCTM(context, 0, height);
    CGContextScaleCTM(context, 1, -1);
    
    CGRect imageRect = CGRectMake(0, 0, width, height);
    CGContextDrawImage(context, imageRect, imageRef);

    CGContextRelease(context);
    
    return rawData;
}
{% endhighlight %}

创建纹理：

{% highlight objc %}
MTLTextureDescriptor *textureDescriptor = [MTLTextureDescriptor texture2DDescriptorWithPixelFormat:MTLPixelFormatRGBA8Unorm
                                                                                             width:imageSize.width
                                                                                            height:imageSize.height
                                                                                         mipmapped:mipmapped];
textureDescriptor.usage = MTLTextureUsageShaderRead;
id<MTLTexture> texture = [[queue device] newTextureWithDescriptor:textureDescriptor];
{% endhighlight %}

位图数据放到 Texture 中：

{% highlight objc %}
MTLRegion region = MTLRegionMake2D(0, 0, imageSize.width, imageSize.height);
[texture replaceRegion:region mipmapLevel:0 withBytes:imageData bytesPerRow:bytesPerRow];
{% endhighlight %}

纹理传递给 Shader：

{% highlight objc %}
[commandEncoder setFragmentTexture:texture atIndex:0];
{% endhighlight %}

## Sampler

Sampler 将坐标体系、过滤、Addressing 封装起来，Sampler 可以在 Shader 中创建，也可以在应用代码中创建，这里在应用代码中创建：

{% highlight objc %}
MTLSamplerDescriptor *samplerDesc = [MTLSamplerDescriptor new];
samplerDesc.sAddressMode = MTLSamplerAddressModeClampToEdge;
samplerDesc.tAddressMode = MTLSamplerAddressModeClampToEdge;
samplerDesc.minFilter = MTLSamplerMinMagFilterNearest;
samplerDesc.magFilter = MTLSamplerMinMagFilterLinear;
samplerDesc.mipFilter = MTLSamplerMipFilterLinear;
self.samplerState = [self.device newSamplerStateWithDescriptor:samplerDesc];
{% endhighlight %}

Sampler 传递给 Shader：

{% highlight objc %}
[renderPass setFragmentSamplerState:self.samplerState atIndex:0];
{% endhighlight %}

## 代码和效果

注意，加载的 OBJ 模型新增了纹理坐标：

{% highlight objc %}
vector_float2 texCoords;
{% endhighlight %}

Shader 中根据传入值进行 sample：

{% highlight cpp %}
float3 diffuseColor = diffuseTexture.sample(samplr, vertexIn.texCoords).rgb;
{% endhighlight %}

[danjiang / MetalByExample](https://github.com/danjiang/MetalByExample/tree/texture)

{% youtube 7c-M3b6YmEE %}