---
title: Metal 示例之光照
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: metal
---

本文主要通过自己对 [Metal By Example](https://gumroad.com/l/metalbyexample) 理解编写，这一篇文章讲解 Metal 如何处理光照。

![Metal By Example Cover](/images/mbe-cover.png)

## 加载 OBJ 模型数据

OBJ 模型数据中包含顶点坐标（vertex positions），法线（normals），纹理坐标（texture coordinates）：

{% highlight objc %}
- (void)makeResources {
  NSURL *modelURL = [[NSBundle mainBundle] URLForResource:@"teapot" withExtension:@"obj"];
  MBEOBJModel *model = [[MBEOBJModel alloc] initWithContentsOfURL:modelURL generateNormals:YES];
  MBEOBJGroup *group = [model groupForName:@"teapot"];
  self.mesh = [[MBEOBJMesh alloc] initWithGroup:group device:self.device];
  self.uniformBuffer = [self.device newBufferWithLength:sizeof(MBEUniforms) * MBEInFlightBufferCount
                                                options:MTLResourceOptionCPUCacheModeDefault];
  self.uniformBuffer.label = @"Uniforms";
}
{% endhighlight %}

## 光照的理论

这里通过下面的公式来模拟光照，每一个像素的颜色由这三项决定：

![Metal By Example Light](/images/mbe-light.png)

### 环境光（Ambient Light）

环境光就是光源照到其他物体上反弹到目标物体上的光：

![Metal By Example Ambient Light](/images/mbe-ambient-light.png)

I 是亮度（Intensity）， L 是光源（Light Source），M 是材质（Material）

### 漫射光（Diffuse Light）

投射在粗糙表面上的光向各个方向反射的现象，遵循朗伯特氏余弦定律（Lambert’s cosine law），反射光的强度与入射光和平面法线的角度余弦值成比例：

![Metal By Example Diffuse Light](/images/mbe-diffuse-light1.png)

![Metal By Example Diffuse Light](/images/mbe-diffuse-light2.png)

### 镜面反射光（Specular Light）

镜面反射光描述了材质向特定方向反射光的倾向，而不是散射到各个方向，明亮度由高光强度（specular power）这个参数来控制，高光强度 5 接近于无泽面，高光强度 50 接近于发光面。

这里采用 Blinn-Phong approximation 来计算 Specular Term，D 代表光源照射方向，V 代表平面被观看的方向：

![Metal By Example Specular Term](/images/mbe-specular-term.png)

带入前面的 Specular Power 和 Specular Term 就可以计算出镜面反射光：

![Metal By Example Specular Light](/images/mbe-specular-light.png)

## Metal 中的光照

编写 shader 函数：

{% highlight cpp %}
#include <metal_stdlib>
#include <metal_matrix>

using namespace metal;

struct Light
{
  float3 direction;
  float3 ambientColor;
  float3 diffuseColor;
  float3 specularColor;
};

constant Light light = {
  .direction = { 0.13, 0.72, 0.68 },
  .ambientColor = { 0.05, 0.05, 0.05 },
  .diffuseColor = { 0.9, 0.9, 0.9 },
  .specularColor = { 1, 1, 1 }
};

struct Material
{
  float3 ambientColor;
  float3 diffuseColor;
  float3 specularColor;
  float specularPower;
};

constant Material material = {
  .ambientColor = { 0.9, 0.1, 0 },
  .diffuseColor = { 0.9, 0.1, 0 },
  .specularColor = { 1, 1, 1 },
  .specularPower = 100
};

struct Vertex
{
  float4 position;
  float4 normal;
};

struct ProjectedVertex
{
  float4 position [[position]];
  float3 eye;
  float3 normal;
};

struct Uniforms
{
  float4x4 modelViewProjectionMatrix;
  float4x4 modelViewMatrix;
  float3x3 normalMatrix;
};

vertex ProjectedVertex vertex_project(device Vertex *vertices [[buffer(0)]],
                             constant Uniforms &uniforms [[buffer(1)]],
                             uint vid [[vertex_id]])
{
  ProjectedVertex vertexOut;
  vertexOut.position = uniforms.modelViewProjectionMatrix * vertices[vid].position;
  vertexOut.eye = -(uniforms.modelViewMatrix * vertices[vid].position).xyz;
  vertexOut.normal = uniforms.normalMatrix * vertices[vid].normal.xyz;
  return vertexOut;
}

fragment float4 fragment_light(ProjectedVertex vertexIn [[stage_in]],
                               constant Uniforms &uniforms [[buffer(0)]])
{
  float3 ambientTerm = light.ambientColor * material.ambientColor;
  
  float3 normal = normalize(vertexIn.normal);
  float diffuseIntensity = saturate(dot(normal, light.direction));
  float3 diffuseTerm = light.diffuseColor * material.diffuseColor * diffuseIntensity;
  
  float3 specularTerm(0);
  if (diffuseIntensity > 0)
  {
    float3 eyeDirection = normalize(vertexIn.eye);
    float3 halfway = normalize(light.direction + eyeDirection);
    float specularFactor = pow(saturate(dot(normal, halfway)), material.specularPower);
    specularTerm = light.specularColor * material.specularColor * specularFactor;
  }
  
  return float4(ambientTerm + diffuseTerm + specularTerm, 1);
}
{% endhighlight %}

## 代码和效果

[danjiang / MetalByExample / Lighting](https://github.com/danjiang/MetalByExample/tree/lighting)

{% youtube Zwe-MxsrLjk %}
