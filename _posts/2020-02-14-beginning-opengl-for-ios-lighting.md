---
title: iOS OpenGL ES 编程入门之光照
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: opengl
---

之前学习的 iOS OpenGL ES 编程的资料和知识比较零散，现在希望通过观看 [Ray Wenderlich - Beginning OpenGL for iOS](https://www.youtube.com/playlist?list=PL23Revp-82LL_XoQEiTT6zsgHHrpjr1D9) 的同时，把目前掌握的 OpenGL ES 知识点整理一下，第五篇文章会讲 Lighting 这些基础内容。

![OpenGL](/images/opengl.jpg)

## Phone Lighting

冯氏光照模型，当然还有其他光照模型，这个光照模型比较简单，分为下面 3 部分，后面分别讲解：

![OpenGL Phone Lighting Model](/images/opengl-phone-lighting-model.jpg)

> When we see the world around us, we are really seeing the cumulative effect of trillions upon trillions of tiny little particles called photons. These photons are emitted by energy sources like the sun; and after traveling a long distance, they will bounce off some objects and get refracted by others, until they finally strike the retinas in the backs of our eyes. Our eyes and brain take it from there and reconstruct the activity of all of these photons into the world that we can see around us.
>
> Computer graphics have historically simulated the effects of light by either simulating the behavior of actual photons or by using shortcuts to fake that behavior. One way of simulating the behavior of the actual photons is with a ray tracer. A ray tracer simulates the photons by shooting rays into the scene and calculating how those rays interact with the objects in the scene. This technique is quite powerful, and it can lend itself well to really good reflections and refractions and other special effects like caustics (the patterns you see when light passes through water, for example).
>
> Unfortunately, ray tracing is usually too expensive to use for real-time render- ing. Instead, most games and apps simplify things and approximate the way that light works at a higher level, rather than simulating it directly. Simple lighting algorithms can go a long way, and there are also ways of faking reflections, refractions, and more. These techniques can use OpenGL to put most of the workload on the GPU and run blazing fast, even on a mobile phone.
>
> ---- Quote from [OpenGL ES 2 for Android](https://pragprog.com/book/kbogla/opengl-es-2-for-android)

## Ambient Light 环境光

<em class="fab fa-github"></em> [示例代码](https://github.com/danjiang/LearningOpenGLES2/tree/master/08-1.AmbientLight "08-1.AmbientLight")

环境光就是光源照到其他物体上反弹到目标物体上的光，对于当前环境来说是一个整体值，汇聚了各种反弹到目标物体上的光：

* **Ambient Color = Light Color × Ambient Intensity**
* **Light Color** 光源颜色值
* **Ambient Intensity** 环境光强度，明暗程度，越大越亮

* **Color = Texture Color × Ambient Color**
* **Texture Color** 物体自身的颜色

fragment shader 中按照前面说的公式进行计算：

{% highlight glsl %}
uniform sampler2D u_Texture;

varying lowp vec4 frag_Color;
varying lowp vec2 frag_TexCoord;

struct Light {
    lowp vec3 Color;
    lowp float AmbientIntensity;
};
uniform Light u_Light;

void main(void) {
    // Ambient
    lowp vec4 AmbientColor = vec4(u_Light.Color, 1.0) * u_Light.AmbientIntensity;
    gl_FragColor = texture2D(u_Texture, frag_TexCoord) * AmbientColor;
}
{% endhighlight %}

## Diffuse Light 漫射光

<em class="fab fa-github"></em> [示例代码](https://github.com/danjiang/LearningOpenGLES2/tree/master/08-2.DiffuseLight "08-2.DiffuseLight")

光源照射到粗糙表面上的光向各个方向反射的现象，首先要理解下面两个概念：

1. Normals 法线，和平面上的每一条直线都垂直的直线就是该平面的法线。
2. [Vectors Dot Product 向量点积](https://www.mathsisfun.com/algebra/vectors-dot-product.html)

![OpenGL Normals](/images/opengl-normals.jpg)

* **Diffuse Factor = -(Normal dot Light Direction)**，比如正对着光源自然更亮，n dot d = -1，然后再取反
* **Normal** 法线
* **Light Direction** 光源的方向

![OpenGL Diffuse Color](/images/opengl-diffuse-color.png)

* **Diffuse Color = Light Color × Diffuse Intensity × Diffuse Factor**
* **Light Color** 光源颜色值
* **Diffuse Intensity** 漫射光强度，明暗程度，越大越亮

* **Color = Texture Color × (Ambient Color + Diffuse Color)**

每一个 vertex 都新增了法向量的值：

{% highlight swift %}
class Cube : Model {
    let vertexList : [Vertex] = [
        
        // Front
        Vertex( 1, -1, 1,  1, 0, 0, 1,  1, 0,  0, 0, 1), // 0
        Vertex( 1,  1, 1,  0, 1, 0, 1,  1, 1,  0, 0, 1), // 1
        Vertex(-1,  1, 1,  0, 0, 1, 1,  0, 1,  0, 0, 1), // 2
        Vertex(-1, -1, 1,  0, 0, 0, 1,  0, 0,  0, 0, 1), // 3
        
        // Back
        Vertex(-1, -1, -1, 0, 0, 1, 1,  1, 0,  0, 0,-1), // 4
        Vertex(-1,  1, -1, 0, 1, 0, 1,  1, 1,  0, 0,-1), // 5
        Vertex( 1,  1, -1, 1, 0, 0, 1,  0, 1,  0, 0,-1), // 6
        Vertex( 1, -1, -1, 0, 0, 0, 1,  0, 0,  0, 0,-1), // 7
        
        // Left
        Vertex(-1, -1,  1, 1, 0, 0, 1,  1, 0, -1, 0, 0), // 8
        Vertex(-1,  1,  1, 0, 1, 0, 1,  1, 1, -1, 0, 0), // 9
        Vertex(-1,  1, -1, 0, 0, 1, 1,  0, 1, -1, 0, 0), // 10
        Vertex(-1, -1, -1, 0, 0, 0, 1,  0, 0, -1, 0, 0), // 11
        
        // Right
        Vertex( 1, -1, -1, 1, 0, 0, 1,  1, 0,  1, 0, 0), // 12
        Vertex( 1,  1, -1, 0, 1, 0, 1,  1, 1,  1, 0, 0), // 13
        Vertex( 1,  1,  1, 0, 0, 1, 1,  0, 1,  1, 0, 0), // 14
        Vertex( 1, -1,  1, 0, 0, 0, 1,  0, 0,  1, 0, 0), // 15
        
        // Top
        Vertex( 1,  1,  1, 1, 0, 0, 1,  1, 0,  0, 1, 0), // 16
        Vertex( 1,  1, -1, 0, 1, 0, 1,  1, 1,  0, 1, 0), // 17
        Vertex(-1,  1, -1, 0, 0, 1, 1,  0, 1,  0, 1, 0), // 18
        Vertex(-1,  1,  1, 0, 0, 0, 1,  0, 0,  0, 1, 0), // 19
        
        // Bottom
        Vertex( 1, -1, -1, 1, 0, 0, 1,  1, 0,  0,-1, 0), // 20
        Vertex( 1, -1,  1, 0, 1, 0, 1,  1, 1,  0,-1, 0), // 21
        Vertex(-1, -1,  1, 0, 0, 1, 1,  0, 1,  0,-1, 0), // 22
        Vertex(-1, -1, -1, 0, 0, 0, 1,  0, 0,  0,-1, 0), // 23
        
    ]
}
{% endhighlight %}

vertex shader 中要将法向量转换到 camera coordinates：

{% highlight glsl %}
uniform highp mat4 u_ModelViewMatrix;

attribute vec3 a_Normal;

varying lowp vec3 frag_Normal;

void main(void) {
    frag_Normal = (u_ModelViewMatrix * vec4(a_Normal, 0.0)).xyz;
}
{% endhighlight %}

fragment shader 中按照前面说的公式进行计算：

{% highlight glsl %}
uniform sampler2D u_Texture;

varying lowp vec4 frag_Color;
varying lowp vec2 frag_TexCoord;
varying lowp vec3 frag_Normal;

struct Light {
    lowp vec3 Color;
    lowp float AmbientIntensity;
    lowp float DiffuseIntensity;
    lowp vec3 Direction;
};
uniform Light u_Light;

void main(void) {
    // Ambient
    lowp vec3 AmbientColor = u_Light.Color * u_Light.AmbientIntensity;
    
    // Diffuse
    lowp vec3 Normal = normalize(frag_Normal);
    lowp float DiffuseFactor = max(-dot(Normal, u_Light.Direction), 0.0);
    lowp vec3 DiffuseColor = u_Light.Color * u_Light.DiffuseIntensity * DiffuseFactor;
    
    gl_FragColor = texture2D(u_Texture, frag_TexCoord) * vec4((AmbientColor + DiffuseColor), 1.0);
}
{% endhighlight %}

## Specular Light 镜面反射光

<em class="fab fa-github"></em> [示例代码](https://github.com/danjiang/LearningOpenGLES2/tree/master/08-3.SpecularLight "08-3.SpecularLight")

镜面反射就是光源照射到物体表面发生反射，然后进入人眼被看到，不同的材质反射光的能力不同：

* **Reflection = reflect(Light Direction, Normal)**，反射光根据图示的规则计算出来
* **Light Direction** 光源的方向
* **Normal** 法线

![OpenGL Reflection](/images/opengl-reflection.png)

* **Specular Factor = pow(-(Reflection dot Eye), Shininess)**，比如眼睛正对着反射光源自然更亮，r dot eye = -1，然后再取反
* **Eye** 物体上每个 fragment 的 camera coordinates
* **Shininess** 高光强度，物体材质反射光的能力

![OpenGL Specular Color](/images/opengl-specular-color.png)

* **Specular Color = Light Color × Specular Intensity × Specular Factor**
* **Light Color** 光源颜色值
* **Specular Intensity** 镜面反射光强度，明暗程度，越大越亮

* **Color = Texture Color × (Ambient Color + Diffuse Color + Specular Color)**

vertex shader 中计算每个 vertex 位置坐标的 camera coordinates：

{% highlight glsl %}
uniform highp mat4 u_ModelViewMatrix;

attribute vec4 a_Position;

varying lowp vec3 frag_Position;

void main(void) {
    frag_Position = (u_ModelViewMatrix * a_Position).xyz;
}
{% endhighlight %}

fragment shader 中按照前面说的公式进行计算：

{% highlight glsl %}
uniform sampler2D u_Texture;

varying lowp vec4 frag_Color;
varying lowp vec2 frag_TexCoord;
varying lowp vec3 frag_Normal;
varying lowp vec3 frag_Position;

struct Light {
    lowp vec3 Color;
    lowp float AmbientIntensity;
    lowp float DiffuseIntensity;
    lowp vec3 Direction;
    highp float SpecularIntensity;
    highp float Shininess;
};
uniform Light u_Light;

void main(void) {
    // Ambient
    lowp vec3 AmbientColor = u_Light.Color * u_Light.AmbientIntensity;
    
    // Diffuse
    lowp vec3 Normal = normalize(frag_Normal);
    lowp float DiffuseFactor = max(-dot(Normal, u_Light.Direction), 0.0);
    lowp vec3 DiffuseColor = u_Light.Color * u_Light.DiffuseIntensity * DiffuseFactor;
    
    // Specular
    lowp vec3 Eye = normalize(frag_Position);
    lowp vec3 Reflection = reflect(u_Light.Direction, Normal);
    lowp float SpecularFactor = pow(max(0.0, -dot(Reflection, Eye)), u_Light.Shininess);
    lowp vec3 SpecularColor = u_Light.Color * u_Light.SpecularIntensity * SpecularFactor;
    
    gl_FragColor = texture2D(u_Texture, frag_TexCoord) * vec4((AmbientColor + DiffuseColor + SpecularColor), 1.0);
}
{% endhighlight %}