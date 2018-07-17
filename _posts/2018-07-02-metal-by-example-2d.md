---
title: Metal 示例之画 2D
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: metal
---

本文主要通过自己对 [Metal By Example](https://gumroad.com/l/metalbyexample) 理解编写，这一篇文章讲解 Metal 如何画 2D。

![Metal By Example Cover](/images/mbe-cover.png)

## Buffer 存储数据

MTLBuffer 代表没有类型化的 Buffer 数据、且有固定的长度，可以将坐标和颜色混合在一个 Buffer 中连续存储：

{% highlight objc %}
typedef struct {
  vector_float4 position;
  vector_float4 color;
} MBEVertex;

static const MBEVertex vertices[] = {
  { .position = {  0.0,  0.5, 0, 1 }, .color = { 1, 0, 0, 1 } },
  { .position = { -0.5, -0.5, 0, 1 }, .color = { 0, 1, 0, 1 } },
  { .position = {  0.5, -0.5, 0, 1 }, .color = { 0, 0, 1, 1 } }
};

self.vertexBuffer = [self.device newBufferWithBytes:vertices
                                             length:sizeof(vertices)
                                            options:MTLResourceCPUCacheModeDefaultCache];
{% endhighlight %}

上面坐标表示方式是 4D homogeneous coordinates，每一个点由 x、y、z 和 w 坐标组成，w 固定为 1。 

## Function 和 Library

通常可以跑在 GPU 上，处理每个顶点的小程序叫 Shader，在 Metal 中叫 Function，Shader 文件的语法由 C++ 编写：

{% highlight cpp %}
#include <metal_stdlib>
using namespace metal;

struct Vertex
{
  float4 position [[position]];
  float4 color;
};

vertex Vertex vertex_main(device Vertex *vertices [[buffer(0)]],
                          uint vid [[vertex_id]])
{
  return vertices[vid];
}

fragment float4 fragment_main(Vertex inVertex [[stage_in]])
{
  return inVertex.color;
}
{% endhighlight %}

[[position]] 做为属性，向 Metal 表明 position 而不是 color 做为顶点坐标，如果 vertex_main 返回的是 float4，就不需要特别表明。

Metal 的 shader 函数必须由 vertex、fragment 或 kernel 三者之一做为前缀，vertex shader 函数针对几何图形中的每一个顶点只运行一次，如 projecting、每个顶点的打光。

在界面上画的几何图形应该会包含很多像素，所以必须有一个 pipeline 拿到从 vertex shader 函数返回的值转换为针对每个像素（也称为 fragment）的内插值，这个过程被称为栅格化，被转换后内插值交给 fragment shader 函数做针对每个 fragment 的操作，如加纹理、每个像素打光。

shader 函数会被编译为 library，加载相应 library 就可以使用其中的 shader 函数，可以事先编译 shader 函数为 library 放到 bundle 中，也可以在运行时编译 shader 函数为 library 再使用：

{% highlight objc %}
id<MTLLibrary> library = [self.device newDefaultLibrary];
id<MTLFunction> vertexFunc = [library newFunctionWithName:@"vertex_main"];
id<MTLFunction> fragmentFunc = [library newFunctionWithName:@"fragment_main"];
{% endhighlight %}

## Pipeline

Metal 提供的虚拟 Pipeline，一头输入顶点数据，一头输出栅格化图像，创建 render pipeline descriptor 来配置 pipeline：

{% highlight objc %}
id<MTLLibrary> library = [self.device newDefaultLibrary];
id<MTLFunction> vertexFunc = [library newFunctionWithName:@"vertex_main"];
id<MTLFunction> fragmentFunc = [library newFunctionWithName:@"fragment_main"];

MTLRenderPipelineDescriptor *pipelineDescriptor = [MTLRenderPipelineDescriptor new];
pipelineDescriptor.vertexFunction = vertexFunc;
pipelineDescriptor.fragmentFunction = fragmentFunc;
pipelineDescriptor.colorAttachments[0].pixelFormat = self.metalLayer.pixelFormat;

NSError *error = nil;
self.pipeline = [self.device newRenderPipelineStateWithDescriptor:pipelineDescriptor
                                                            error:&error];

if (!self.pipeline) {
  NSLog(@"Error occurred when creating render pipeline state: %@", error);
}
{% endhighlight %}

attachment 描述绘制出 texture，colorAttachments[0] 代表在界面上显示的 texture。

## 编码 Render Command

在第一部分中 command encoder 已经做了一些基础准备，这里来使用 Buffer 和 Pipeline 做真正的绘制：

{% highlight objc %}
passDescriptor.colorAttachments[0].clearColor = MTLClearColorMake(0.85, 0.85, 0.85, 1);

[commandEncoder setRenderPipelineState:self.pipeline];
[commandEncoder setVertexBuffer:self.vertexBuffer offset:0 atIndex:0];
[commandEncoder drawPrimitives:MTLPrimitiveTypeTriangle vertexStart:0 vertexCount:3];
{% endhighlight %}

setVertexBuffer 的 atIndex 0 对应 vertex shader 函数中的 [[buffer(0)]] 属性，drawPrimitives 来画三角形，vertexStart 指定从 Buffer 的开始位置，vertexCount 指定顶点数。

## 通过 CADisplayLink 保持刷新同步

{% highlight objc %}
- (void)didMoveToSuperview {
  [super didMoveToSuperview];
  if (self.superview) {
    self.displayLink = [CADisplayLink displayLinkWithTarget:self selector:@selector(displayLinkDidFire:)];
    [self.displayLink addToRunLoop:[NSRunLoop mainRunLoop] forMode:NSRunLoopCommonModes];
  } else {
    [self.displayLink invalidate];
    self.displayLink = nil;
  }
}

- (void)displayLinkDidFire:(CADisplayLink *)displayLink {
  [self redraw];
}
{% endhighlight %}

## 代码和效果

[danjiang / MetalByExample / 2D](https://github.com/danjiang/MetalByExample/tree/2d)

![Metal By Example 2D Result](/images/mbe-2d-result.png)
