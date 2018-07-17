---
title: Metal 示例之实例渲染和渲染文本
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: metal
---

本文主要通过自己对 [Metal By Example](https://gumroad.com/l/metalbyexample) 理解编写，这一篇文章讲解 Metal 中实例渲染和渲染文本。

![Metal By Example Cover](/images/mbe-cover.png)

## 实例渲染

{% youtube gYpmVTldLOk %}

绘制地形

通过中点位移（Midpoint Displacement）算法来绘制地形，它是一种近似的分形布朗运动生成算法，它是利用细分过程中在两点和或多个点之间进行插值的方法来进行地形建模的。

[Terrain generation with the diamond square algorithm](http://www.paulboxley.com/blog/2011/03/terrain-generation-mark-one)

绘制奶牛

每一个奶牛的朝向和位置是不同的，所以这里需要针对每一个奶牛传入不同的 Buffer，也就是下面的 self.cowUniformBuffer：

{% highlight objc %}
[commandEncoder setVertexBuffer:self.cowMesh.vertexBuffer offset:0 atIndex:0];
[commandEncoder setVertexBuffer:self.sharedUniformBuffer offset:0 atIndex:1];
[commandEncoder setVertexBuffer:self.cowUniformBuffer offset:0 atIndex:2];
{% endhighlight %}

将 modelMatrix 从 modelViewProjectionMatrix 中分离出来，才可以针对每一个奶牛进行设置：

{% highlight objc %}
typedef struct
{
    matrix_float4x4 viewProjectionMatrix;
} Uniforms;

typedef struct
{
    matrix_float4x4 modelMatrix;
    matrix_float3x3 normalMatrix;
} PerInstanceUniforms;
{% endhighlight %}

需要将每一个奶牛的 PerInstanceUniforms 存储起来：

{% highlight objc %}
_cowUniformBuffer = [_device newBufferWithLength:sizeof(PerInstanceUniforms) * MBECowCount
                                         options:MTLResourceOptionCPUCacheModeDefault];
{% endhighlight %}

计算每一个奶牛的 PerInstanceUniforms：

{% highlight objc %}
PerInstanceUniforms uniforms;
uniforms.modelMatrix = matrix_multiply(translation, rotation);
uniforms.normalMatrix = matrix_upper_left3x3(uniforms.modelMatrix);
memcpy([self.cowUniformBuffer contents] + sizeof(PerInstanceUniforms) * i, &uniforms, sizeof(PerInstanceUniforms));
{% endhighlight %}

一次性绘制所有奶牛，注意这里通过传入 instanceCount 表明有多少个实例：

{% highlight objc %}
[commandEncoder drawIndexedPrimitives:MTLPrimitiveTypeTriangle
                           indexCount:[self.cowMesh.indexBuffer length] / sizeof(MBEIndex)
                            indexType:MTLIndexTypeUInt16
                          indexBuffer:self.cowMesh.indexBuffer
                    indexBufferOffset:0
                        instanceCount:MBECowCount];
{% endhighlight %}

Vertex Shader 函数中 ushort iid [[instance_id]] 表明当前被绘制的实例的下标：

{% highlight cpp %}
vertex ProjectedVertex vertex_project(InVertex vertexIn [[stage_in]],
                                      constant Uniforms &uniforms [[buffer(1)]],
                                      constant PerInstanceUniforms *perInstanceUniforms [[buffer(2)]],
                                      ushort vid [[vertex_id]],
                                      ushort iid [[instance_id]])
{
    float4x4 instanceModelMatrix = perInstanceUniforms[iid].modelMatrix;
    float3x3 instanceNormalMatrix = perInstanceUniforms[iid].normalMatrix;
{% endhighlight %}

示例代码：

[danjiang / mbe-sample-code / InstancedDrawing](https://github.com/danjiang/mbe-sample-code/tree/master/objc/11-InstancedDrawing)

## 渲染文本

{% youtube Q92YpyM2yR4 %}

动态栅格化（Dynamic Rasterization）

将字符串在 CPU 上绘制成位图，再将位图传给 GPU 绘制，字符串有任何变化就需要重新绘制位图，CPU 的压力很大，放大已绘制在屏幕上的字符串就会看起来模糊。	

字体地图册（Font Atlases）

将所有可能出现的字符先绘制成一张图，通过坐标可以找到确定的字符，在 GPU 绘制字符时，一个字符可由两个组合三角形（也就是长方形）设定其网格，再将字体地图册做为纹理图像贴到长方形上。

Signed Distance Fields

这个算法可以优化采用字体地图册绘制字符，其原理可参考文章：[Drawing Text with Signed Distance Fields in Mapbox GL](https://blog.mapbox.com/drawing-text-with-signed-distance-fields-in-mapbox-gl-b0933af6f817)。

正交投射（The Orthographic Projection）

绘制文字在屏幕上，采用正交投射的方式，可以理解会从正面俯视一本书的视角：

{% highlight objc %}
matrix_float4x4 projectionMatrix = matrix_orthographic_projection(0, drawableSize.width, 0, drawableSize.height);
uniforms.viewProjectionMatrix = projectionMatrix;
{% endhighlight %}

示例代码：

[danjiang / mbe-sample-code / TextRendering](https://github.com/danjiang/mbe-sample-code/tree/master/objc/12-TextRendering)