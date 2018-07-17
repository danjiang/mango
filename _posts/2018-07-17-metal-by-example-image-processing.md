---
title: Metal 示例之图像处理
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: metal
---

本文主要通过自己对 [Metal By Example](https://gumroad.com/l/metalbyexample) 理解编写，这一篇文章讲解 Metal 中图像处理。

![Metal By Example Cover](/images/mbe-cover.png)

## 并行运算基础准备

GPU 设备，加载 Shader 函数的库，还有 Command Queue 和之前渲染 3D 图像一样也是需要的：

{% highlight objc %}
@interface MBEContext : NSObject

@property (strong) id<MTLDevice> device;
@property (strong) id<MTLLibrary> library;
@property (strong) id<MTLCommandQueue> commandQueue;

+ (instancetype)newContext;

@end
{% endhighlight %}

加载 Kernel Shader 函数，专门用于并行计算，再通过 newComputePipelineStateWithFunction 创建好 Pipeline：

{% highlight objc %}
_kernelFunction = [_context.library newFunctionWithName:functionName];
_pipeline = [_context.device newComputePipelineStateWithFunction:_kernelFunction error:&error];
{% endhighlight %}

编码 Render Command，设置上面创建好的 Pipeline，传入纹理图像，再传入 Threadgroups：

{% highlight objc %}
id<MTLComputeCommandEncoder> commandEncoder = [commandBuffer computeCommandEncoder];
[commandEncoder setComputePipelineState:self.pipeline];
[commandEncoder setTexture:inputTexture atIndex:0];
[commandEncoder setTexture:self.internalTexture atIndex:1];
[self configureArgumentTableWithCommandEncoder:commandEncoder];
[commandEncoder dispatchThreadgroups:threadgroups threadsPerThreadgroup:threadgroupCounts];
[commandEncoder endEncoding];

[commandBuffer commit];
[commandBuffer waitUntilCompleted];
{% endhighlight %}

要并行计算，就需要将任务分解成几个 Threadgroups，Threadgroup 可以被 GPU 进一步分解然后交给线程并行处理，如下图像，根据长宽除以 8 得到有多少个 Threadgroups，每一个 Threadgroup 又由 8 乘 8 个线程组成：

![Metal By Example Threadgroup](/images/mbe-threadgroups.png)

{% highlight objc %}
MTLSize threadgroupCounts = MTLSizeMake(8, 8, 1);
MTLSize threadgroups = MTLSizeMake([texture width] / threadgroupCounts.width,
[texture height] / threadgroupCounts.height, 1);

[commandEncoder dispatchThreadgroups:threadgroups threadsPerThreadgroup:threadgroupCounts];
{% endhighlight %}

## 图像处理

### 基础结构

通过 Metal 实现图像的模糊和饱和度调整的滤镜：

![Metal By Example Filter](/images/mbe-filter.png)

MBETextureConsumer 表示纹理图像输入协议 和 MBETextureProvider 表示纹理图像输出协议：

{% highlight objc %}
@protocol MBETextureProvider;

@protocol MBETextureConsumer <NSObject>

@property (nonatomic, strong) id<MBETextureProvider> provider;

@end
{% endhighlight %}

{% highlight objc %}
@protocol MTLTexture;

@protocol MBETextureProvider <NSObject>

@property (nonatomic, readonly) id<MTLTexture> texture;

@end
{% endhighlight %}

MBEImageFilter 实现了 MBETextureConsumer 和 MBETextureProvider，滤镜就是输入图像、变换、输出图像，每一个滤镜使用的 Kernel Shader 函数不同，所以要通过 initWithFunctionName 来传入：

{% highlight objc %}
@interface MBEImageFilter : NSObject <MBETextureProvider, MBETextureConsumer>

@property (nonatomic, strong) MBEContext *context;
@property (nonatomic, strong) id<MTLBuffer> uniformBuffer;
@property (nonatomic, strong) id<MTLComputePipelineState> pipeline;
@property (nonatomic, strong) id<MTLTexture> internalTexture;
@property (nonatomic, assign, getter=isDirty) BOOL dirty;

- (instancetype)initWithFunctionName:(NSString *)functionName context:(MBEContext *)context;

- (void)configureArgumentTableWithCommandEncoder:(id<MTLComputeCommandEncoder>)commandEncoder;

@end
{% endhighlight %}

实现滤镜效果的代码： 

{% highlight objc %}
- (void)applyFilter
{
  id<MTLTexture> inputTexture = self.provider.texture;
  
  if (!self.internalTexture ||
      [self.internalTexture width] != [inputTexture width] ||
      [self.internalTexture height] != [inputTexture height])
  {
    MTLTextureDescriptor *textureDescriptor = [MTLTextureDescriptor texture2DDescriptorWithPixelFormat:[inputTexture pixelFormat]
                                                                                                 width:[inputTexture width]
                                                                                                height:[inputTexture height]
                                                                                             mipmapped:NO];
    textureDescriptor.usage = MTLTextureUsageShaderWrite | MTLTextureUsageShaderRead;
    self.internalTexture = [self.context.device newTextureWithDescriptor:textureDescriptor];
  }
  
  MTLSize threadgroupCounts = MTLSizeMake(8, 8, 1);
  MTLSize threadgroups = MTLSizeMake([inputTexture width] / threadgroupCounts.width,
                                     [inputTexture height] / threadgroupCounts.height,
                                     1);
  
  id<MTLCommandBuffer> commandBuffer = [self.context.commandQueue commandBuffer];
  
  id<MTLComputeCommandEncoder> commandEncoder = [commandBuffer computeCommandEncoder];
  [commandEncoder setComputePipelineState:self.pipeline];
  [commandEncoder setTexture:inputTexture atIndex:0];
  [commandEncoder setTexture:self.internalTexture atIndex:1];
  [self configureArgumentTableWithCommandEncoder:commandEncoder];
  [commandEncoder dispatchThreadgroups:threadgroups threadsPerThreadgroup:threadgroupCounts];
  [commandEncoder endEncoding];
  
  [commandBuffer commit];
  [commandBuffer waitUntilCompleted];
}
{% endhighlight %}

如上所示的滤镜代码，针对不同的滤镜需要传入不同的参数给 Kernel Shader 函数，通过在子类实现下面的方法来实现：

{% highlight objc %}
- (void)configureArgumentTableWithCommandEncoder:(id<MTLComputeCommandEncoder>)commandEncoder {
}
{% endhighlight %}

### 饱和度调整

通过下面的公式拿到颜色值中的明亮度：

![Metal By Example Brightness](/images/mbe-brightness.png)

再通过 mix 函数根据 saturationFactor 取值，0 取第一个值，1 取第二个值，在 0 和 1 之间，就通过线性插值计算第一个值和第二个值得到结果：

{% highlight cpp %}
struct AdjustSaturationUniforms
{
    float saturationFactor;
};

kernel void adjust_saturation(texture2d<float, access::read> inTexture [[texture(0)]],
                              texture2d<float, access::write> outTexture [[texture(1)]],
                              constant AdjustSaturationUniforms &uniforms [[buffer(0)]],
                              uint2 gid [[thread_position_in_grid]])
{
    float4 inColor = inTexture.read(gid);
    float value = dot(inColor.rgb, float3(0.299, 0.587, 0.114));
    float4 grayColor(value, value, value, 1.0);
    float4 outColor = mix(grayColor, inColor, uniforms.saturationFactor);
    outTexture.write(outColor, gid);
}
{% endhighlight %}

实现 configureArgumentTableWithCommandEncoder 传入饱和度：

{% highlight objc %}
- (void)configureArgumentTableWithCommandEncoder:(id<MTLComputeCommandEncoder>)commandEncoder {
  struct AdjustSaturationUniforms uniforms;
  uniforms.saturationFactor = self.saturationFactor;
  
  if (!self.uniformBuffer)
  {
    self.uniformBuffer = [self.context.device newBufferWithLength:sizeof(uniforms)
                                                          options:MTLResourceOptionCPUCacheModeDefault];
  }
  
  memcpy([self.uniformBuffer contents], &uniforms, sizeof(uniforms));
  
  [commandEncoder setBuffer:self.uniformBuffer offset:0 atIndex:0];
}
{% endhighlight %}

### 模糊

#### 盒状模糊

根据半径选择周围的像素平均取值，如果半价是 1，就取周围的 9 个点平均后得到最终值。

#### 高斯模糊

高斯也是根据半径选择周围的像素，但是每个像素比重不同，越近的比重越大，越远的比重越小，公式如下：

![Metal By Example Gaussian Blur](/images/mbe-gaussian-blur.png)

{% highlight cpp %}
kernel void gaussian_blur_2d(texture2d<float, access::read> inTexture [[texture(0)]],
                             texture2d<float, access::write> outTexture [[texture(1)]],
                             texture2d<float, access::read> weights [[texture(2)]],
                             uint2 gid [[thread_position_in_grid]])
{
    int size = weights.get_width();
    int radius = size / 2;
    
    float4 accumColor(0, 0, 0, 0);
    for (int j = 0; j < size; ++j)
    {
        for (int i = 0; i < size; ++i)
        {
            uint2 kernelIndex(i, j);
            uint2 textureIndex(gid.x + (i - radius), gid.y + (j - radius));
            float4 color = inTexture.read(textureIndex).rgba;
            float4 weight = weights.read(kernelIndex).rrrr;
            accumColor += weight * color;
        }
    }

    outTexture.write(float4(accumColor.rgb, 1), gid);
}
{% endhighlight %}

实现 configureArgumentTableWithCommandEncoder 传入比重：

{% highlight objc %}
- (void)configureArgumentTableWithCommandEncoder:(id<MTLComputeCommandEncoder>)commandEncoder {
  if (!self.blurWeightTexture)
  {
    [self generateBlurWeightTexture];
  }
  
  [commandEncoder setTexture:self.blurWeightTexture atIndex:2];
}
{% endhighlight %}

## 代码

[danjiang / mbe-sample-code / ImageProcessing](https://github.com/danjiang/mbe-sample-code/tree/master/objc/14-ImageProcessing)