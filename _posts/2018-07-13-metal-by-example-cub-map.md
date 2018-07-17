---
title: Metal 示例之天空盒子中的反射和折射
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: metal
---

本文主要通过自己对 [Metal By Example](https://gumroad.com/l/metalbyexample) 理解编写，这一篇文章讲解 Metal 中实现天空盒子，还有在模型上的反射和折射。

![Metal By Example Cover](/images/mbe-cover.png)

## 天空盒子

天空盒子（Skybox）就是一个在立方体内部装饰了纹理的场景，在立方体中心设置一个相机来看到的视角。

这里的立方体纹理不同于之前提到过 2D 纹理，立方体纹理在 Metal 是左手系坐标：

![Metal By Example Coordinate](/images/mbe-coordinate.jpg)

立方体纹理由 6 张图片组成，其中负 y (ny or negative y) 在最中心：

![Metal By Example Unwrapped Cube Map](/images/mbe-unwrapped-cube-map.png)

{% highlight objc %}
MTLTextureDescriptor *textureDescriptor = [MTLTextureDescriptor textureCubeDescriptorWithPixelFormat:MTLPixelFormatRGBA8Unorm
                                                                                                size:cubeSize
                                                                                           mipmapped:NO];

id<MTLTexture> texture = [device newTextureWithDescriptor:textureDescriptor];
{% endhighlight %}

Slice 和 Cube Texture 对应关系：

<table>
	<tr>
		<td>Face number</td>
		<td>Cube texture face</td>
	</tr>
	<tr>
		<td>0</td>
		<td>Positive X</td>
	</tr>
	<tr>
		<td>1</td>
		<td>Negative X</td>
	</tr>
	<tr>
		<td>2</td>
		<td>Positive Y</td>
	</tr>
	<tr>
		<td>3</td>
		<td>Negative Y</td>
	</tr>
	<tr>
		<td>4</td>
		<td>Positive Z</td>
	</tr>
	<tr>
		<td>5</td>
		<td>Negative Z</td>
	</tr>
</table>

{% highlight objc %}
for (size_t slice = 0; slice < 6; ++slice) {
  NSString *imageName = imageNameArray[slice];
  UIImage *image = [UIImage imageNamed:imageName];
  uint8_t *imageData = [self dataForImage:image];
  
  [texture replaceRegion:region
             mipmapLevel:0
                   slice:slice
               withBytes:imageData
             bytesPerRow:bytesPerRow
           bytesPerImage:bytesPerImage];
  free(imageData);
}
{% endhighlight %}

天空盒子的 Vertex Shader，纹理坐标就是立方体的物体空间：

{% highlight cpp %}
vertex ProjectedVertex vertex_skybox(Vertex inVertex             [[stage_in]],
                                     constant Uniforms &uniforms [[buffer(1)]],
                                     uint vid                    [[vertex_id]])
{
    float4 position = inVertex.position;
    
    ProjectedVertex outVert;
    outVert.position = uniforms.modelViewProjectionMatrix * position;
    outVert.texCoords = position;
    return outVert;
}
{% endhighlight %}

Fragment Shader 并没有什么特别的，像以前一样将纹理和纹理坐标进行 Sample，只是将 z 坐标取反来适配 Sampler：

{% highlight cpp %}
fragment half4 fragment_cube_lookup(ProjectedVertex vert          [[stage_in]],
                                    constant Uniforms &uniforms   [[buffer(0)]],
                                    texturecube<half> cubeTexture [[texture(0)]],
                                    sampler cubeSampler           [[sampler(0)]])
{
    float3 texCoords = float3(vert.texCoords.x, vert.texCoords.y, -vert.texCoords.z);
    return cubeTexture.sample(cubeSampler, texCoords);
}
{% endhighlight %}

## 反射

光在两种物质分界面上改变传播方向又返回原来物质中的现象，叫做光的反射。

Metal 中实现，可以逆向的来想，有一条射线从摄像机发射到物体表面，反射后到立方体纹理上的交汇处。

Metal 中使用内置函数 **reflect** 来实现此功能，注意纹理坐标基于世界坐标来计算：

![Metal By Example Reflection](/images/mbe-reflection.png)

{% highlight cpp %}
vertex ProjectedVertex vertex_reflect(Vertex inVertex             [[stage_in]],
                                      constant Uniforms &uniforms [[buffer(1)]],
                                      uint vid                    [[vertex_id]])
{
    float4 modelPosition = inVertex.position;
    float4 modelNormal = inVertex.normal;
    
    float4 worldCameraPosition = uniforms.worldCameraPosition;
    float4 worldPosition = uniforms.modelMatrix * modelPosition;
    float4 worldNormal = normalize(uniforms.normalMatrix * modelNormal);
    float4 worldEyeDirection = normalize(worldPosition - worldCameraPosition);
    
    ProjectedVertex outVert;
    outVert.position = uniforms.modelViewProjectionMatrix * modelPosition;
    outVert.texCoords = reflect(worldEyeDirection, worldNormal);
    
    return outVert;
}
{% endhighlight %}

## 折射

光从一种透明介质斜射入另一种透明介质时，传播方向一般会发生变化，这种现象叫光的折射，光在发生折射时入射角与折射角符合斯涅尔定律（Snell'sLaw），入射角 θI 与折射角 θT 的正弦之比叫做介质的绝对折射率，简称折射率（Index of Refraction）。

Metal 中使用内置函数 **refract** 来实现此功能，采用从空气进入玻璃的折射率来计算：

![Metal By Example Refraction](/images/mbe-refraction.png)

{% highlight cpp %}
// some common indices of refraction
constant float kEtaAir = 1.000277;
//constant float kEtaWater = 1.333;
constant float kEtaGlass = 1.5;

constant float kEtaRatio = kEtaAir / kEtaGlass;


vertex ProjectedVertex vertex_refract(Vertex inVertex             [[stage_in]],
                                      constant Uniforms &uniforms [[buffer(1)]],
                                      uint vid                    [[vertex_id]])
{
    float4 modelPosition = inVertex.position;
    float4 modelNormal = inVertex.normal;

    float4 worldCameraPosition = uniforms.worldCameraPosition;
    float4 worldPosition = uniforms.modelMatrix * modelPosition;
    float4 worldNormal = normalize(uniforms.normalMatrix * modelNormal);
    float4 worldEyeDirection = normalize(worldPosition - worldCameraPosition);

    ProjectedVertex outVert;
    outVert.position = uniforms.modelViewProjectionMatrix * modelPosition;
    outVert.texCoords = refract(worldEyeDirection, worldNormal, kEtaRatio);

    return outVert;
}
{% endhighlight %}

## Core Motion

利用 Core Motion 来获取设备的方向信息来转换场景的位置：

{% highlight objc %}
self.motionManager = [[CMMotionManager alloc] init];
if (self.motionManager.deviceMotionAvailable)
{
  self.motionManager.deviceMotionUpdateInterval = 1 / 60.0;
  CMAttitudeReferenceFrame frame = CMAttitudeReferenceFrameXTrueNorthZVertical;
  [self.motionManager startDeviceMotionUpdatesUsingReferenceFrame:frame];
}
{% endhighlight %}

{% highlight objc %}
CMDeviceMotion *motion = self.motionManager.deviceMotion;
CMRotationMatrix m = motion.attitude.rotationMatrix;

// permute rotation matrix from Core Motion to get scene orientation
vector_float4 X = { m.m12, m.m22, m.m32, 0 };
vector_float4 Y = { m.m13, m.m23, m.m33, 0 };
vector_float4 Z = { m.m11, m.m21, m.m31, 0 };
vector_float4 W = {     0,     0,     0, 1 };

matrix_float4x4 orientation = { X, Y, Z, W };
self.renderer.sceneOrientation = orientation;
{% endhighlight %}

## 代码和效果

[danjiang / mbe-sample-code / CubeMapping](https://github.com/danjiang/mbe-sample-code/tree/master/objc/08-CubeMapping)

{% youtube zJS7cMvYnfo %}