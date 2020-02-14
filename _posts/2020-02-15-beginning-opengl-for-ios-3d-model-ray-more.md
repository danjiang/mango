---
title: iOS OpenGL ES 编程入门之加载模型、射线和更多探索
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: opengl
---

之前学习的 iOS OpenGL ES 编程的资料和知识比较零散，现在希望通过观看 [Ray Wenderlich - Beginning OpenGL for iOS](https://www.youtube.com/playlist?list=PL23Revp-82LL_XoQEiTT6zsgHHrpjr1D9) 的同时，把目前掌握的 OpenGL ES 知识点整理一下，第六篇文章会讲 3D Model 和 Ray 这些基础内容，最后会说一下 OpenGL ES 可以探索的更多内容。

![OpenGL](/images/opengl.jpg)

## 3D Model

### Import Objects From Blender

<em class="fab fa-github"></em> [示例代码](https://github.com/danjiang/LearningOpenGLES2/tree/master/09-1.ObjModelLoader "09-1.ObjModelLoader")

通常情况下 3D 模型都是在如 Blender 这种专业软件里面制作的，然后从 Blender 中导出 OBJ 格式，当然也有其他格式可用，然后再解析 OBJ 格式为我们自己定义的数据格式。

一个 OBJ 文件的内容如下：

{% highlight swift %}
# Blender v2.62 (sub 0) OBJ File: 'cube.blend'
# www.blender.org
mtllib cube.mtl
v 1.000000 -1.000000 -1.000000
v 1.000000 -1.000000 1.000000
v -1.000000 -1.000000 1.000000
v -1.000000 -1.000000 -1.000000
v 1.000000 1.000000 -0.999999
v 0.999999 1.000000 1.000001
v -1.000000 1.000000 1.000000
v -1.000000 1.000000 -1.000000
vt 0.126874 0.998126
vt 0.126874 0.749375
vt 0.375625 0.749375
vt 0.375625 0.998126
vt 0.375624 0.500625
vt 0.375625 0.251875
vt 0.624374 0.251874
vt 0.624375 0.500624
vt 0.624375 0.749375
vt 0.624375 0.998126
vt 0.375624 0.003126
vt 0.624373 0.003126
vt 0.873126 0.749375
vt 0.873126 0.998126
vn 0.000000 0.000000 -1.000000
vn 1.000000 0.000000 0.000000
vn -1.000000 -0.000000 -0.000000
vn 0.000000 1.000000 0.000000
vn -0.000000 -0.000000 1.000000
vn 0.000000 -1.000000 0.000000
usemtl MaterialDiffuseR
s off
f 5/1/1 1/2/1 4/3/1
f 5/1/1 4/3/1 8/4/1
usemtl MaterialDiffuseM
f 1/5/2 5/6/2 6/7/2
f 1/5/2 6/7/2 2/8/2
usemtl MaterialSpecularG
f 3/9/3 7/10/3 8/4/3
f 3/9/3 8/4/3 4/3/3
usemtl MaterialSpecularY
f 5/6/4 8/11/4 7/12/4
f 5/6/4 7/12/4 6/7/4
usemtl MaterialPhongB
f 2/13/5 6/14/5 7/10/5
f 2/13/5 7/10/5 3/9/5
usemtl MaterialPhongC
f 1/5/6 2/8/6 3/9/6
f 1/5/6 3/9/6 4/3/6
{% endhighlight %}

OBJ 文件中数据的含义：

> * *Vertex (v)*: The position of the vertex in XYZ space.
> * *Texture Coordinates (vt)*: The texel (texture element) to sample in UV space. You can think of this as a way to map each vertex to the position on the texture where it should get its color value from. These values range from (0, 0) (bottom left of texture) to (1, 1) (upper right of texture).
> * *Normals (vn)*: The surface normal of the vertex plane (triangle) in XYZ space. You can think of this as the vector that points “straight out” from the front of the plane at the vertex. This value is needed to ensure proper lighting.
> * *Faces (f)*: A plane triangle defined by three vertices, texture coordinates and normals.
>
> ---- Qutoe from [How To Export Blender Models to OpenGL ES: Part 1/3](https://www.raywenderlich.com/2604-how-to-export-blender-models-to-opengl-es-part-1-3)

如何解析可以参考：

* <em class="fab fa-youtube"></em> [Importing Models - Beginning OpenGL ES and GLKit](https://www.youtube.com/watch?v=EgtKKveoYSU&list=PL23Revp-82LL_XoQEiTT6zsgHHrpjr1D9&index=14&t=40s)
* [How To Export Blender Models to OpenGL ES: Part 1/3](https://www.raywenderlich.com/2604-how-to-export-blender-models-to-opengl-es-part-1-3)
* [How To Export Blender Models to OpenGL ES: Part 2/3](https://www.raywenderlich.com/2603-how-to-export-blender-models-to-opengl-es-part-2-3)
* [How To Export Blender Models to OpenGL ES: Part 3/3](https://www.raywenderlich.com/2602-how-to-export-blender-models-to-opengl-es-part-3-3)
* <em class="fab fa-github"></em> [How To Export Blender Models to OpenGL ES 的示例代码](https://github.com/danjiang/ModelViewer "danjiang / ModelViewer")

其他可参考的 3D 文件解析代码和库：

* <em class="fab fa-github"></em> [SwiftObjLoader](https://github.com/danjiang/SwiftObjLoader "danjiang / SwiftObjLoader")
* <em class="fab fa-github"></em> [obj2opengl](https://github.com/HBehrens/obj2opengl "HBehrens / obj2opengl")
* <em class="fab fa-github"></em> [assimp](https://github.com/assimp/assimp "assimp / assimp")

### Building Simple Objects

可以通过几何知识来自己计算出 3D 模型的顶点坐标来绘制：

![OpenGL Puck](/images/opengl-puck.png)

可以参考 [OpenGL ES 2 for Android - Chapter 8](https://pragprog.com/book/kbogla/opengl-es-2-for-android)

## Screen Touch

### Ray

<em class="fab fa-github"></em> [示例代码](https://github.com/danjiang/AirHockeyAndroid "danjiang / AirHockeyAndroid")

射线是为了解决通过手机屏幕上的触控来操作 3D 模型，在手机屏幕上触控可以得到在屏幕上的 2D 坐标，也就是 Normalized Device Coordinates：

![OpenGL Touch 2D](/images/opengl-touch-2d.png)

有了 2D 坐标 (x, y)，再加上在 3D Frustum 的近端和远端分别得到不同的 z，就可以得到两个 3D 坐标（normalizedX, normalizedY, -1）和 （normalizedX, normalizedY, 1），两点就是一条直线，这就是我们的射线：

![OpenGL Touch 3D](/images/opengl-touch-3d.png)

之前关于 [iOS OpenGL ES 编程入门之矩阵](/programming/2020/02/12/beginning-opengl-for-ios-matrix/) 的文章讲到了 Projection Matrix 将 Camera Coordinates 转换为 Normalized Device Coordinates，通过 Projection Matrix 的逆矩阵，就可以将 Projection Normalized Device 转换为 Camera Coordinates：

![OpenGL Inverse Matrix](/images/opengl-inverse-matrix.png)

> We also need to undo the perspective divide. There’s an interesting property of the inverted view projection matrix: after we multiply our vertices with the inverted view projection matrix, nearPointWorld and farPointWorld will actually contain an inverted w value. This is because normally the whole point of a projection matrix is to create different w values so that the perspective divide can do its magic; so if we use an inverted projection matrix, we’ll also get an inverted w. All we need to do is divide x, y, and z with these inverted w’s, and we’ll undo the perspective divide.
>
> ---- Quote from [OpenGL ES 2 for Android](https://pragprog.com/book/kbogla/opengl-es-2-for-android)

{% highlight java %}
private Ray convertNormalized2DPointToRay(float normalizedX, float normalizedY) {
	final float[] nearPointNdc = {normalizedX, normalizedY, -1, 1};
	final float[] farPointNdc = {normalizedX, normalizedY, 1, 1};

	final float[] nearPointWorld = new float[4];
	final float[] farPointWorld = new float[4];

	multiplyMV(nearPointWorld, 0, invertedViewProjectionMatrix, 0, nearPointNdc, 0);
	multiplyMV(farPointWorld, 0, invertedViewProjectionMatrix, 0, farPointNdc, 0);

	divideByW(nearPointWorld);
	divideByW(farPointWorld);

	Point nearPointRay = new Point(nearPointWorld[0], nearPointWorld[1], nearPointWorld[2]);
	Point farPointRay = new Point(farPointWorld[0], farPointWorld[1], farPointWorld[2]);

	return new Ray(nearPointRay, Geometry.vectorBetween(nearPointRay, farPointRay));
}

private void divideByW(float[] vector) {
	vector[0] /= vector[3];
	vector[1] /= vector[3];
	vector[2] /= vector[3];
}
{% endhighlight %}

最后就是通过几何计算来确认直线是否和 3D 模型交汇，得出是否点击中 3D 模型，可以将 3D 模型简化成外层有一个球体包裹住，也就是直线是否和球体交汇的问题：

![OpenGL Line Sphere](/images/opengl-line-sphere.png)

下面两篇文章讲解了更简单触控操作 3D 模型的方式：

* [How To Rotate a 3D Object Using Touches with OpenGL](https://www.raywenderlich.com/2914-how-to-rotate-a-3d-object-using-touches-with-opengl)
* [OpenGL ES Transformations with Gestures](https://www.raywenderlich.com/2591-opengl-es-transformations-with-gestures)

## The Next Step

通过目前这[几篇文章](/#opengl)，已经比较完整介绍了 OpenGL ES 的基础部分，下面列出更多可以探索的方向：

### Image Processing		

这是我目前学习 OpenGL ES 的主要目的，图像处理主要用在音视频和计算机视觉上，我目前也主要集中在音视频的视频效果器的学习上，会在 <em class="fab fa-github"></em> [DTLiving](https://github.com/danjiang/DTLiving "danjiang / DTLiving") 上进行实践，后面有更多的实践体会后再来说说，下面是些参考资料：

* [OpenGL ES Pixel Shaders Tutorial](https://www.raywenderlich.com/2323-opengl-es-pixel-shaders-tutorial)
* <em class="fab fa-github"></em> [OpenGL ES Pixel Shaders Tutorial 的示例代码](https://github.com/danjiang/PixelShader "danjiang / PixelShader")
* [音视频开发进阶指南: 基于 Android 与 iOS 平台的实践](https://book.douban.com/subject/30124646/)
* <em class="fab fa-github"></em> [GPUImage](https://github.com/BradLarson/GPUImage "BradLarson / GPUImage")

### Particle System

什么下雪、烟花、爆炸这些效果都可以用 Particle System 来做：

* [OpenGL ES Particle System Tutorial: Part 1/3](https://www.raywenderlich.com/2704-opengl-es-particle-system-tutorial-part-1-3)
* [OpenGL ES Particle System Tutorial: Part 2/3](https://www.raywenderlich.com/2703-opengl-es-particle-system-tutorial-part-2-3)
* <em class="fab fa-github"></em> [OpenGL ES Particle System Tutorial 的示例代码](https://github.com/danjiang/GLParticles "danjiang / GLParticles")
* [OpenGL ES 2 for Android - Live Wallpaper](https://pragprog.com/book/kbogla/opengl-es-2-for-android)
* <em class="fab fa-github"></em> [LiveWallpaper 的示例代码](https://github.com/danjiang/LiveWallpaper "danjiang / LiveWallpaper")

### Game

做一个打砖块的游戏：

* <em class="fab fa-youtube"></em> [Making Games in Open GL: Part 1 - Beginning OpenGL ES and GLKit](https://www.youtube.com/watch?v=98cd9Eya5Jc&list=PL23Revp-82LL_XoQEiTT6zsgHHrpjr1D9&index=9)
* <em class="fab fa-youtube"></em> [Making Games in Open GL: Part 2 - Beginning OpenGL ES and GLKit](https://www.youtube.com/watch?v=I3J2qIimBqA&list=PL23Revp-82LL_XoQEiTT6zsgHHrpjr1D9&index=10)
* <em class="fab fa-youtube"></em> [Making Games in Open GL: Part 3 - Beginning OpenGL ES and GLKit](https://www.youtube.com/watch?v=x0UJiMehcP4&list=PL23Revp-82LL_XoQEiTT6zsgHHrpjr1D9&index=11)
* <em class="fab fa-youtube"></em> [Making Games in Open GL: Part 4 - Beginning OpenGL ES and GLKit](https://www.youtube.com/watch?v=629tijNMPgU&list=PL23Revp-82LL_XoQEiTT6zsgHHrpjr1D9&index=12)
* <em class="fab fa-github"></em> [Making Games in Open GL 的示例代码](https://github.com/danjiang/LearningOpenGLES2/tree/master/09.BrickBreakerGame "09.BrickBreakerGame")

在 Android 上做一个 Air Hockey 的游戏：

* [OpenGL ES 2 for Android - Air Hockey](https://pragprog.com/book/kbogla/opengl-es-2-for-android)
* <em class="fab fa-github"></em> [Air Hockey 的示例代码](https://github.com/danjiang/AirHockeyAndroid "danjiang / AirHockeyAndroid")

用 SceneKit 做一个简易 3D 模型版的水果忍者： 

* [SceneKit Tutorial with Swift Part 1: Getting Started](https://www.raywenderlich.com/146175/scene-kit-tutorial-swift-part-1-getting-started)
* [SceneKit Tutorial with Swift Part 2: Nodes](https://www.raywenderlich.com/146176/scene-kit-tutorial-swift-part-2-nodes-2)
* [SceneKit Tutorial with Swift Part 3: Physics](https://www.raywenderlich.com/146177/scene-kit-tutorial-swift-part-3-physics-2)
* [SceneKit Tutorial with Swift Part 4: Render Loop](https://www.raywenderlich.com/146178/scene-kit-tutorial-swift-part-4-render-loop-2)
* [SceneKit Tutorial with Swift Part 5: Particle Systems](https://www.raywenderlich.com/146179/scene-kit-tutorial-swift-part-5-particle-systems-2)

用 Unity 做游戏：

* [Unity Games by Tutorials](https://store.raywenderlich.com/products/unity-games-by-tutorials)