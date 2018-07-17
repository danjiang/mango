---
title: Metal 示例之画 3D
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: metal
---

本文主要通过自己对 [Metal By Example](https://gumroad.com/l/metalbyexample) 理解编写，这一篇文章讲解 Metal 如何画 3D。

![Metal By Example Cover](/images/mbe-cover.png)

## 定义立方体

立方体需要 8 个顶点：

{% highlight objc %}
static const MBEVertex vertices[] =
{
  { .position = { -1,  1,  1, 1 }, .color = { 0, 1, 1, 1 } },
  { .position = { -1, -1,  1, 1 }, .color = { 0, 0, 1, 1 } },
  { .position = {  1, -1,  1, 1 }, .color = { 1, 0, 1, 1 } },
  { .position = {  1,  1,  1, 1 }, .color = { 1, 1, 1, 1 } },
  { .position = { -1,  1, -1, 1 }, .color = { 0, 1, 0, 1 } },
  { .position = { -1, -1, -1, 1 }, .color = { 0, 0, 0, 1 } },
  { .position = {  1, -1, -1, 1 }, .color = { 1, 0, 0, 1 } },
  { .position = {  1,  1, -1, 1 }, .color = { 1, 1, 0, 1 } }
};

self.vertexBuffer = [self.device newBufferWithBytes:vertices
                                             length:sizeof(vertices)
                                            options:MTLResourceCPUCacheModeDefaultCache];
{% endhighlight %}

立方体的每一个平面都是由三角形绘制而成，需要一个 Index Buffer 包含每个三角形对应顶点的下标：

{% highlight objc %}
typedef uint16_t MBEIndex;
const MTLIndexType MBEIndexType = MTLIndexTypeUInt16;

static const MBEIndex indices[] =
{
  3, 2, 6, 6, 7, 3,
  4, 5, 1, 1, 0, 4,
  4, 0, 3, 3, 7, 4,
  1, 5, 6, 6, 2, 1,
  0, 1, 2, 2, 3, 0,
  7, 6, 5, 5, 4, 7
};

self.indexBuffer = [self.device newBufferWithBytes:indices
                                            length:sizeof(indices)
                                           options:MTLResourceOptionCPUCacheModeDefault];
self.indexBuffer.label = @"Indices";
{% endhighlight %}

## 坐标转换

**物体空间（object space）转换为世界空间（world space）**

物体空间就是立方体各顶点坐标相对立方体中心的本地坐标体系，世界空间就是真实世界坐标体系：

{% highlight objc %}
@property (nonatomic, assign) float rotationX, rotationY, time;

self.time += duration;
self.rotationX += duration * (M_PI / 2);
self.rotationY += duration * (M_PI / 3);
float scaleFactor = sinf(5 * self.time) * 0.25 + 1;
const vector_float3 xAxis = { 1, 0, 0 };
const vector_float3 yAxis = { 0, 1, 0 };
const matrix_float4x4 xRot = matrix_float4x4_rotation(xAxis, self.rotationX);
const matrix_float4x4 yRot = matrix_float4x4_rotation(yAxis, self.rotationY);
const matrix_float4x4 scale = matrix_float4x4_uniform_scale(scaleFactor);
const matrix_float4x4 modelMatrix = matrix_multiply(matrix_multiply(xRot, yRot), scale);
{% endhighlight %}

**世界空间（world space）转换为相机空间（camera space、view space、eye space），再转换为裁剪空间（clip space）**

相机空间就是从相机或者观察者的眼睛角度看物体时的坐标体系，物体在下面这个截头锥体中，就是相机或眼睛中能看见物体：

![Metal By Example View Frustum](/images/mbe-view-frustum.png)

相机空间这一步的转换可以简单的理解为调整相机的位置，这里调整方式是世界空间的右手系，Y 轴向上，Z 轴垂直于屏幕向外：

{% highlight objc %}
const vector_float3 cameraTranslation = { 0, 0, -5 };
const matrix_float4x4 viewMatrix = matrix_float4x4_translation(cameraTranslation);
{% endhighlight %}

前面说到物体在截头锥体中才能被看见，物体不在截头锥体的部分怎么剪裁掉呢？通过裁剪空间，裁剪空间被 GPU 用来决定三角形的可见性，如果三角形的三个顶点都在裁剪空间之外，三角形就不会被渲染（culled），三角形的至少一个顶点在裁剪空间之内，三角形会根据边界被剪裁掉，相机空间转换为裁剪空间通过 perspective projection 矩阵的运算，可以向 matrix_float4x4_perspective 运算方法提供 field of view 改变视锥体竖直方向的张开角度、near 和 far 控制视锥体的近裁剪平面和远裁剪平面距离摄像机的远近：

{% highlight objc %}
const CGSize drawableSize = view.metalLayer.drawableSize;
const float aspect = drawableSize.width / drawableSize.height;
const float fov = (2 * M_PI) / 5;
const float near = 1;
const float far = 100;
const matrix_float4x4 projectionMatrix = matrix_float4x4_perspective(aspect, fov, near, far);
{% endhighlight %}

我们可以计算出 model-view-projection (MVP) 矩阵：

{% highlight objc %}
MBEUniforms uniforms;
uniforms.modelViewProjectionMatrix = matrix_multiply(projectionMatrix, matrix_multiply(viewMatrix, modelMatrix));
{% endhighlight %}

**裁剪空间（clip space）转换为 NDC 空间，再转换为窗口坐标**

这里不需要自己写代码转换，MVP 坐标除以 w 转换为规格化设备坐标（normalized device coordinates）， Metal 的 NDC 空间就是这样一个立方体 [−1, 1]×[−1, 1]×[0, 1]，最后 NDC 再转换为窗口坐标。

## 渲染

vertex shader 中每一个顶点坐标和 MVP 矩阵相乘得到转化后的坐标，fragment shader 只是单纯地返回颜色，注意 device 前缀表明地址空间是 per-vertex 或 per-fragment，constant 前缀表明地址空间在每次调用都不会改变：

{% highlight cpp %}
#include <metal_stdlib>

using namespace metal;

struct Vertex
{
  float4 position [[position]];
  float4 color;
};

struct Uniforms
{
  float4x4 modelViewProjectionMatrix;
};

vertex Vertex vertex_project(device Vertex *vertices [[buffer(0)]],
                             constant Uniforms *uniforms [[buffer(1)]],
                             uint vid [[vertex_id]])
{
  Vertex vertexOut;
  vertexOut.position = uniforms->modelViewProjectionMatrix * vertices[vid].position;
  vertexOut.color = vertices[vid].color;
  
  return vertexOut;
}

fragment half4 fragment_flatcolor(Vertex vertexIn [[stage_in]])
{
  return half4(vertexIn.color);
}
{% endhighlight %}

配置 Render Pass，向 Render Pass 中传递对应的 Buffer，根据 Index 获取 Buffer 中的三角形顶点来绘制三角形：

{% highlight objc %}
id<MTLRenderCommandEncoder> renderPass = [commandBuffer renderCommandEncoderWithDescriptor:passDescriptor];
[renderPass setRenderPipelineState:self.pipeline];
[renderPass setDepthStencilState:self.depthStencilState];
[renderPass setFrontFacingWinding:MTLWindingCounterClockwise];
[renderPass setCullMode:MTLCullModeBack];

const NSUInteger uniformBufferOffset = sizeof(MBEUniforms) * self.bufferIndex;

[renderPass setVertexBuffer:self.vertexBuffer offset:0 atIndex:0];
[renderPass setVertexBuffer:self.uniformBuffer offset:uniformBufferOffset atIndex:1];

[renderPass drawIndexedPrimitives:MTLPrimitiveTypeTriangle
                       indexCount:[self.indexBuffer length] / sizeof(MBEIndex)
                        indexType:MBEIndexType
                      indexBuffer:self.indexBuffer
                indexBufferOffset:0];

[renderPass endEncoding];
{% endhighlight %}

## 代码和效果

[danjiang / MetalByExample / 3D](https://github.com/danjiang/MetalByExample/tree/3d)

{% youtube 83lBOrW3fZQ %}