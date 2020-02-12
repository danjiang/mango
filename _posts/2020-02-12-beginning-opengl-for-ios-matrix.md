---
title: iOS OpenGL ES 编程入门之矩阵
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: opengl
---

之前学习的 iOS OpenGL ES 编程的资料和知识比较零散，现在希望通过观看 [Ray Wenderlich - Beginning OpenGL for iOS](https://www.youtube.com/playlist?list=PL23Revp-82LL_XoQEiTT6zsgHHrpjr1D9) 的同时，把目前掌握的 OpenGL ES 知识点整理一下，第三篇文章会讲 Matrix 和坐标转换这些基础内容。

![OpenGL](/images/opengl.jpg)

## Matrix

![OpenGL Matrix](/images/opengl-matrix.png)

**1. Model Matrix**

对场景中的一个模型进行转换，所以 model matrix 应该在每一个 model 中：

{% highlight swift %}
class Model {
    // ModelView Transformation
    var position : GLKVector3 = GLKVector3(v: (0.0, 0.0, 0.0))
    var rotationX : Float = 0.0
    var rotationY : Float = 0.0
    var rotationZ : Float = 0.0
    var scale : Float = 1.0
    
    func modelMatrix() -> GLKMatrix4 {
        var modelMatrix : GLKMatrix4 = GLKMatrix4Identity
        modelMatrix = GLKMatrix4Translate(modelMatrix, self.position.x, self.position.y, self.position.z)
        modelMatrix = GLKMatrix4Rotate(modelMatrix, self.rotationX, 1, 0, 0)
        modelMatrix = GLKMatrix4Rotate(modelMatrix, self.rotationY, 0, 1, 0)
        modelMatrix = GLKMatrix4Rotate(modelMatrix, self.rotationZ, 0, 0, 1)
        modelMatrix = GLKMatrix4Scale(modelMatrix, self.scale, self.scale, self.scale)
        return modelMatrix
    }    
}
{% endhighlight %}

> A model matrix is used to place objects into world-space coordinates. For example, we might have our puck model and our mallet model initially centered at (0, 0, 0). Without a model matrix, our models will be stuck there: if we wanted to move them, we’d have to update each and every vertex ourselves. Instead of doing that, we can use a model matrix and transform our vertices by multiplying them with the matrix. If we want to move our puck to (5, 5), we just need to prepare a model matrix that will do this for us.
>
> Quote from [OpenGL ES 2 for Android](https://pragprog.com/book/kbogla/opengl-es-2-for-android)

**2. View Matrix**

对场景中的每个模型进行转换，不会放在 model 中，但会对 model 产生影响所以需要传入到 model 中，View Matrix 可以有多个，当前有效的只有一个，切回 View Matrix 就类似于切换视角，这里说下矩阵的乘法跟相乘顺序有关系，View Matrix × Model Matrix 和 Model Matrix × View Matrix 会产生不同的结果，正确的应该是 View Matrix × Model Matrix，最右边的会先对坐标进行转换：

{% highlight swift %}
class ViewController: GLKViewController {
    override func glkView(_ view: GLKView, drawIn rect: CGRect) {
        let viewMatrix : GLKMatrix4 = GLKMatrix4MakeTranslation(0, -1, -5)
        self.cube.renderWithParentMoelViewMatrix(viewMatrix)
    }
}

class Model {
    func renderWithParentMoelViewMatrix(_ parentModelViewMatrix: GLKMatrix4) {
        let modelViewMatrix : GLKMatrix4 = GLKMatrix4Multiply(parentModelViewMatrix, modelMatrix())
   }
}
{% endhighlight %}

> A view matrix is used for the same reasons as a model matrix, but it equally affects every object in the scene. Because it affects everything, it is functionally equivalent to a camera: move the camera around, and you’ll see things from a different viewpoint.
>
> The advantage of using a separate matrix is that it lets us prebake a bunch of transformations into a single matrix. As an example, imagine we wanted to rotate the scene around and move it a certain amount into the distance. One way we could do this is by issuing the same rotate and translate calls for every single object. While that works, it’s easier to just save these transformations into a separate matrix and apply that to every object.
>
> Quote from [OpenGL ES 2 for Android](https://pragprog.com/book/kbogla/opengl-es-2-for-android)

**3. Projection Matrix**

先要理解一点就是透视，比如下图中的铁轨实际上几乎都是等宽的，离你越远的，你视觉上会觉得更窄：

![OpenGL Rails](/images/opengl-rails.png)

OpenGL 在这一步会执行 perspective division，就是将坐标 x, y, z 除以 w，从而得到区间在 [-1, 1] 之间的 normalized device coordinates，如下图所示，离的越远 w 值越大，就可以实现前面所说的透视：

![OpenGL Perspective Division](/images/opengl-perspective-division.png)

> To create the illusion of 3D on the screen, OpenGL will take each gl_Position and divide the x, y, and z components by the w component. When the w component is used to represent distance, this causes objects that are further away to be moved closer to the center of the rendering area, which then acts like a vanishing point. This is how OpenGL fools us into seeing a scene in 3D, using the same trick that artists have been using for centuries.
>
> Quote from [OpenGL ES 2 for Android](https://pragprog.com/book/kbogla/opengl-es-2-for-android)

可以采用 Projection Matrix 来自动计算 w 的值：

{% highlight swift %}
extension ViewController {
    func setupScene() {
        self.shader = BaseEffect(vertexShader: "SimpleVertexShader.glsl", fragmentShader: "SimpleFragmentShader.glsl")
        
        self.shader.projectionMatrix = GLKMatrix4MakePerspective(
            GLKMathDegreesToRadians(85.0),
            GLfloat(self.view.bounds.size.width / self.view.bounds.size.height),
            1,
            150)

        self.cube = Cube(shader: self.shader)        
    }
}
{% endhighlight %}

GLKMatrix4MakePerspective 参数的含义：

| fovyRadians | 如下图中 FOV 表示的垂直视角的角度 |
| aspect | 可视区域的长宽比率，通常就是屏幕的长宽比 |
| nearZ | This should be set to the distance to the far plane and must be pos- itive and greater than the distance to the near plane. |
| farZ | This should be set to the distance to the near plane and must be positive. For example, if this is set to 1, the near plane will be located at a z of -1. |

![OpenGL Perspective Frustum](/images/opengl-perspective-frustum.png)

**4. Viewport Transformation**

这一步是 OpenGL 自动完成的，通常我们只需要设置 Viewport：

{% highlight swift %}
glViewport(0, 0, self.view.bounds.size.width, self.view.bounds.size.height)
{% endhighlight %}

> Before we can see the final result, OpenGL needs to map the x and y components of the normalized device coordinates to an area on the screen that the operating system has set aside for display, called the viewport; these mapped coordinates are known as window coordinates. We don’t really need to be too concerned about these coordinates beyond telling OpenGL how to do the mapping. We’re currently doing this in our code with a call to glViewport() in onSurfaceChanged().
>
> Quote from [OpenGL ES 2 for Android](https://pragprog.com/book/kbogla/opengl-es-2-for-android)

## Depth Buffer

Depth Buffer 是 Frame Buffer 中包含的另一种 Buffer，之前说过的还有 Color Buffer，Depth Buffer 存储的 Depth 信息就是为了解决物体渲染时的遮挡问题，最前面的物体渲染，后面被遮挡的物体被遗弃掉：

{% highlight swift %}
glClear(GLbitfield(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT))

glEnable(GLenum(GL_DEPTH_TEST))
{% endhighlight %}

> OpenGL gives us a better solution in the form of the depth buffer, a special buffer that records the depth of every fragment on the screen. When this buffer is turned on, OpenGL performs a depth test for every fragment: if the fragment is closer than what’s already there, draw it; otherwise discard it.
>
> Quote from [OpenGL ES 2 for Android](https://pragprog.com/book/kbogla/opengl-es-2-for-android)

## Culling

OpenGL 绘制的几何面都有两面，顶点的绘制顺序是逆时针就是正面，顶点的绘制顺序是顺时针就是背面，OpenGL 设置为只绘制正面：

{% highlight swift %}
glEnable(GLenum(GL_CULL_FACE))
{% endhighlight %}

顶点 Index 的数值顺序就很关键了：

![OpenGL Culling](/images/opengl-culling.png)

{% highlight swift %}
let indexList : [GLubyte] = [
    // Front
    0, 1, 2,
    2, 3, 0,
    
    // Back
    4, 5, 6,
    6, 7, 4,
    
    // Left
    3, 2, 5,
    5, 4, 3,
    
    // Right
    7, 6, 1,
    1, 0, 7,
    
    // Top
    1, 6, 5,
    5, 2, 1,
    
    // Bottom
    3, 4, 7,
    7, 0, 3
]
{% endhighlight %}

## Shader Uniform 变量

vertex shader 中定义的 matrix 变量都是 uniform 修饰符：

{% highlight glsl %}
uniform highp mat4 u_ModelViewMatrix;
uniform highp mat4 u_ProjectionMatrix;

attribute vec4 a_Position;
attribute vec4 a_Color;

varying lowp vec4 frag_Color;

void main(void) {
    frag_Color = a_Color;
    gl_Position = u_ProjectionMatrix * u_ModelViewMatrix * a_Position;
}
{% endhighlight %}

通过下面的方法来获取 uniform 变量的 Index：

{% highlight swift %}
self.modelViewMatrixUniform = glGetUniformLocation(self.programHandle, "u_ModelViewMatrix")
self.projectionMatrixUniform = glGetUniformLocation(self.programHandle, "u_ProjectionMatrix")
{% endhighlight %}

通过下面的方法来传递值给 uniform 变量：

{% highlight swift %}
glUniformMatrix4fv(self.projectionMatrixUniform, 1, GLboolean(GL_FALSE), self.projectionMatrix.array)
glUniformMatrix4fv(self.modelViewMatrixUniform, 1, GLboolean(GL_FALSE), self.modelViewMatrix.array)
{% endhighlight %}

> 不同类型的 uniform 变量要使用不同名称的 OpenGL ES Functions，方法名上就看的出来参数类型，之前错把 glUniform1i 写成了 glUniform1f，找了很多天才发现了这个问题 😤。

* [danjiang / LearningOpenGLES2 / 06-1.ModelTransformation](https://github.com/danjiang/LearningOpenGLES2/tree/master/06-1.ModelTransformation)
* [danjiang / LearningOpenGLES2 / 06-2.ModelTransformation-Animation](https://github.com/danjiang/LearningOpenGLES2/tree/master/06-2.ModelTransformation-Animation)
* [danjiang / LearningOpenGLES2 / 06-3.ViewTransformation](https://github.com/danjiang/LearningOpenGLES2/tree/master/06-3.ViewTransformation)
* [danjiang / LearningOpenGLES2 / 06-4.ProjectionTransformation](https://github.com/danjiang/LearningOpenGLES2/tree/master/06-4.ProjectionTransformation)
* [danjiang / LearningOpenGLES2 / 06-5.AnimateCube](https://github.com/danjiang/LearningOpenGLES2/tree/master/06-5.AnimateCube)
* [danjiang / LearningOpenGLES2 / 06-6.DepthAndCulling](https://github.com/danjiang/LearningOpenGLES2/tree/master/06-6.DepthAndCulling)
