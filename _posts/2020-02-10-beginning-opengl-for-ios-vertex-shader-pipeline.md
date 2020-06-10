---
title: iOS OpenGL ES 编程入门之顶点、着色器和 Pipeline
author: 但江
avatar: danjiang
location: 成都
category: programming
tags: opengl featured
---

之前学习的 iOS OpenGL ES 编程的资料和知识比较零散，现在希望通过观看 [Ray Wenderlich - Beginning OpenGL for iOS](https://www.youtube.com/playlist?list=PL23Revp-82LL_XoQEiTT6zsgHHrpjr1D9) 的同时，把目前掌握的 OpenGL ES 知识点整理一下，第二篇文章会讲 Vertex、Shader 和 Pipeline 这些基础内容。

![OpenGL](/images/opengl.jpg)

## Vertex

Vertex 就是顶点，是一种数据结构，每一个顶点可以携带的各种信息，常见的就是 Position 和 Color，这里先说一下 OpenGL 可以绘制的类型有 Point、Line Segment 和 Triangle，比如说要绘制一个长方形，它由 2 个三角形组成，我们就需要 6 个 Vertex，每一个三角形 3 个 Vertex，每个 Vertex 有 XYZ 坐标：

{% highlight swift %}
struct Vertex {
    var x : GLfloat = 0.0
    var y : GLfloat = 0.0
    var z : GLfloat = 0.0
    
    init(_ x : GLfloat, _ y : GLfloat, _ z : GLfloat) {
        self.x = x
        self.y = y
        self.z = z 
    }
}
{% endhighlight %}

OpenGL 坐标系的原点在中间：

![OpenGL 2D Coordinate](/images/opengl-2d-coordinate.jpg)

一个三角形的坐标如下：

{% highlight swift %}
let vertices : [Vertex] = [
    Vertex( 0.0,  0.25, 0.0), // TOP
    Vertex(-0.5, -0.25, 0.0), // LEFT
    Vertex( 0.5, -0.25, 0.0), // RIGHT
]
{% endhighlight %}

## Shader

<em class="fab fa-github"></em> [示例代码](https://github.com/danjiang/LearningOpenGLES2/blob/master/01.HelloOpenGL2/HelloOpenGL2/ShaderProgram.swift "danjiang / LearningOpenGLES2 / 01.HelloOpenGL2 / HelloOpenGL2 /ShaderProgram.swift")

在 GPU 上跑的小程序，分为 vertex shader 和 fragment shader，GPU 小程序通过 GLSL 来编写，可以先简单理解 vertex shader 处理顶点，fragment shader 处理像素。

一个典型的 vertex shader：

{% highlight glsl %}
attribute vec4 a_position;
attribute mediump vec4 a_texcoord;

varying mediump vec2 v_texcoord;

void main(void) { 
    v_texcoord = a_texcoord.xy;
    gl_Position = a_position;
}
{% endhighlight %}

一个典型的 fragment shader：

{% highlight glsl %}
precision mediump float;

uniform sampler2D u_texture;

varying mediump vec2 v_texcoord;

void main(void) {
    gl_FragColor = texture2D(u_texture, v_texcoord);
}
{% endhighlight %}

### 这里简单的说一下 GLSL 的语法

* const：用于声明非可写的编译时常量变量。
* attribute：用于经常更改的信息，只能在顶点着色器中使用。
* uniform：用于不经常更改的信息，可用于顶点着色器和片元着色器。
* varying：用于修饰从顶点着色器向片元着色器传递的变量。

基本数据类型 int、float、bool 这些与 C 语言都是一致的，这里面的 float 是有一个修饰符的，常在 fragment shader 中使用，fragment shader 被调用的次数远大于 vertex shader，对于性能影响比较关键，所以需要指定精度：

* highp：32bit，一般用于顶点坐标（vertex coordinate）。
* medium：16bit，一般用于纹理坐标（texture coordinate）。
* lowp：8bit，一般用于颜色表示（color）。

### 将 Shader 编译、连接成 Program

![OpenGL Shader Program](/images/opengl-shader-program.png)

{% highlight swift %}
import UIKit

class ShaderProgram {
    
    private var program: GLuint = 0
    
    init(vertexShaderName: String, fragmentShaderName: String) {
        let vertexShader = compileShader(name: vertexShaderName, with: GLenum(GL_VERTEX_SHADER))
        let fragmentShader = compileShader(name: fragmentShaderName, with: GLenum(GL_FRAGMENT_SHADER))
        
        // 创建 shader program
        program = glCreateProgram()
        // 附加 vertex shader 给 program
        glAttachShader(program, vertexShader)
        // 附加 fragment shader 给 program
        glAttachShader(program, fragmentShader)
        // 连接
        glLinkProgram(program)
        
        var linkStatus = GLint()
        glGetProgramiv(program, GLenum(GL_LINK_STATUS), &linkStatus)
        if linkStatus == GL_FALSE {
            let bufferLength: GLsizei = 1024
            let info: [GLchar] = Array(repeating: GLchar(0), count: Int(bufferLength))
            glGetProgramInfoLog(program, bufferLength, nil, UnsafeMutablePointer(mutating: info))
            print("Could not link shader file \(vertexShaderName) and \(fragmentShaderName): \(String(validatingUTF8: info) ?? "")")
            exit(1)
        }
        
        if vertexShader != 0 {
            glDeleteShader(vertexShader)
        }
        if fragmentShader != 0 {
            glDeleteShader(fragmentShader)
        }
    }
    
    func use() {
        glUseProgram(program)
    }
    
    func delete() {
        if program != 0 {
            glDeleteProgram(program)
            program = 0
        }
    }
    
    func attributeLocation(for name: String) -> GLuint {
        return GLuint(glGetAttribLocation(program, name))
    }
    
    func uniformLocation(for name: String) -> GLint {
        return glGetUniformLocation(program, name)
    }
    
    // bundle 中读取 shader 文件，转换成 C 字符串，丢给 OpenGL 编译
    private func compileShader(name: String, with type: GLenum) -> GLuint {
        do {
            guard let shaderPath = Bundle.main.path(forResource: name, ofType: "glsl") else {
                print("Could not find shader file \(name)")
                exit(1)
            }
            let shaderString = try NSString(contentsOfFile: shaderPath, encoding: String.Encoding.utf8.rawValue)
            var shaderCString = shaderString.utf8String
            var shaderStringLength = GLint(shaderString.length)
            let shader = glCreateShader(type)
            glShaderSource(shader, 1, &shaderCString, &shaderStringLength)
            
            glCompileShader(shader)
            
            var compileStatus = GLint()
            glGetShaderiv(shader, GLenum(GL_COMPILE_STATUS), &compileStatus)
            if compileStatus == GL_FALSE {
                let bufferLength: GLsizei = 1024
                let info: [GLchar] = Array(repeating: GLchar(0), count: Int(bufferLength))
                glGetShaderInfoLog(shader, bufferLength, nil, UnsafeMutablePointer(mutating: info))
                print("Could not compile shader file \(name): \(String(validatingUTF8: info) ?? "")")
                exit(1)
            }
            
            return shader
        } catch {
            print("Could not load shader file \(name)")
            exit(1)
        }
    }

}
{% endhighlight %}

这里重点说下下面的两个方法，想要给 shader 中的 attribute 和 uniform 变量传递数据，当然需要知道变量的 Index，下面就是获取 Index 的方法：

{% highlight swift %}
func attributeLocation(for name: String) -> GLuint {
    return GLuint(glGetAttribLocation(program, name))
}

func uniformLocation(for name: String) -> GLint {
    return glGetUniformLocation(program, name)
}
{% endhighlight %}

## Pipeline

### 整体流程

![OpenGL Pipeline Overview](/images/opengl-pipeline-overview.png)
![OpenGL Pipeline Overview 2](/images/opengl-pipeline-overview2.png)

**1. Transfer Vertex Data**

第一步，给 GPU 传递 Vertex 数据，我们创建 Vertex 数据是在 CPU 的内存中，要将 Vertex 的数据发给 GPU，后面会详细讲怎么做。

**2. Execute Vertex Shader**

执行编写的 vertex shader。

**3. Primitive Assemble**

把几何图形拆解为最基本的三角形。

![OpenGL Pipeline Assemble Primitives](/images/opengl-pipeline-assemble-primitives.jpg)

**4. Rasterization**

将最基本的三角形转换为像素，还需要根据顶点中的位置坐标和颜色来决定每个像素的位置坐标和颜色：

![OpenGL Pipeline Rasterize Primitives](/images/opengl-pipeline-rasterize-primitives.png)

如果是画直线，像素的颜色由离两个顶点位置坐标的距离作为比率，来混合两个顶点的颜色，比如中间位置就是 **0.5 * red + 0.5 * green**：

![OpenGL Pipeline Linear Interpolation](/images//opengl-linear-interpolation.png)

如果是画三角形，比率就按面积来算：

![OpenGL Pipeline Triangle Interpolation](/images//opengl-triangle-interpolation.png)

**5. Execute Fragment Shader**

执行编写的 fragment shader。

**6. Per-Fragment Operations**

针对每个 fragment 的操作。

**7. Whole Framebuffer Operations**

针对整个 framebuffer 的操作。

**8. Frame Buffer**

将结果写到 framebuffer。

### 一次完整的绘制

#### Pass vertex as ordered with vertex buffer objects

<em class="fab fa-github"></em> [示例代码](https://github.com/danjiang/LearningOpenGLES2/tree/master/02.Triangle "danjiang / LearningOpenGLES2 / 02.Triangle")

![OpenGL Vertex Buffer Objects](/images/opengl-vertex-buffer-objects.jpg)

**1. CPU 传递数据给 GPU**

{% highlight swift %}
func setupVertexBuffer() {
    // 在 GPU 中创建一个 Buffer
    glGenBuffers(GLsizei(1), &vertexBuffer)
    // GL_ARRAY_BUFFER 指向上面创建的 Buffer
    glBindBuffer(GLenum(GL_ARRAY_BUFFER), vertexBuffer)
    // 给 GPU 提供 Buffer 数据，就是 GL_ARRAY_BUFFER 当前指向的 Buffer
    // 参数依次为 target、数据内存大小、数据、使用方式（因为不会变化，所以这里是 GL_STATIC_DRAW）
    let count = vertices.count
    let size =  MemoryLayout<Vertex>.size
    glBufferData(GLenum(GL_ARRAY_BUFFER), count * size, vertices, GLenum(GL_STATIC_DRAW))
}
{% endhighlight %}

**2. Shader 如何理解 Vertex 中的数据**

{% highlight glsl %}
attribute vec4 a_Position;
{% endhighlight %}

对于上面 vertex shader 中的属性 a_Position，Swift 中通过下面的方法可以得到属性 a_Position 的 Index，这个 Index 就代表 a_Position，后面传递数据时就会用到：

{% highlight swift %}
GLuint(glGetAttribLocation(program, "a_Position"))
{% endhighlight %}

也可以将一个 Index 绑定给 vertex shader 中的属性，当我更偏向上面的方法：

{% highlight swift %}
glBindAttribLocation(self.programHandle, VertexAttributes.vertexAttribPosition.rawValue, "a_Position")
{% endhighlight %}

下面就通过属性的 Index 让 Shader 理解 Vertex 数据结构：

{% highlight swift %}
// turn attribute on
glEnableVertexAttribArray(VertexAttributes.vertexAttribPosition.rawValue)
// where is it in the vertex buffer
glVertexAttribPointer(
    VertexAttributes.vertexAttribPosition.rawValue, // Attribute Index
    3, // 数据大小
    GLenum(GL_FLOAT), // 数据类型
    GLboolean(GL_FALSE), // 是否标准化
    GLsizei(MemoryLayout<Vertex>.size), // 步进长度也就是一个 Vertex 数据的大小
    nil) // 在 Vertex 数据结构中的 Offset
{% endhighlight %}

**3. 绘制**

{% highlight swift %}
glBindBuffer(GLenum(GL_ARRAY_BUFFER), vertexBuffer)
glDrawArrays(GLenum(GL_TRIANGLES), 0, 3)

// turn attribute off
glDisableVertexAttribArray(VertexAttributes.vertexAttribPosition.rawValue)
{% endhighlight %}
                          
#### Pass vertex as indexed with vertex buffer objects

<em class="fab fa-github"></em> [示例代码](https://github.com/danjiang/LearningOpenGLES2/tree/master/03.IndexedSquare "danjiang / LearningOpenGLES2 / 03.IndexedSquare")

之前传递 Vertex 方式是，绘制一个长方形，它由 2 个三角形组成，我们就需要 6 个 Vertex，由几何的知识，我们知道这 2 个三角形会有共同的 Vertex，我们完全可以只传递 4 个 Vertex，并告知 2 个三角形由 4 个 Vertex 中的哪 3 个组成：

{% highlight swift %}
let vertices : [Vertex] = [
    Vertex( 0.8, -0.8, 0),
    Vertex( 0.8,  0.8, 0),
    Vertex(-0.8,  0.8, 0),
    Vertex(-0.8, -0.8, 0)
]

let indices : [GLubyte] = [
    0, 1, 2,
    2, 3, 0
]
{% endhighlight %}

现在需要 Vertex Buffer 和 Index Buffer：

{% highlight swift %}
glGenBuffers(GLsizei(1), &vertexBuffer)
glBindBuffer(GLenum(GL_ARRAY_BUFFER), vertexBuffer)
let count = vertices.count
let size =  MemoryLayout<Vertex>.size
glBufferData(GLenum(GL_ARRAY_BUFFER), count * size, vertices, GLenum(GL_STATIC_DRAW))

glGenBuffers(GLsizei(1), &indexBuffer)
glBindBuffer(GLenum(GL_ELEMENT_ARRAY_BUFFER), indexBuffer)
glBufferData(GLenum(GL_ELEMENT_ARRAY_BUFFER), indices.count * MemoryLayout<GLubyte>.size, indices, GLenum(GL_STATIC_DRAW))
{% endhighlight %}

绘制的时候要将改为：

{% highlight swift %}
glBindBuffer(GLenum(GL_ARRAY_BUFFER), vertexBuffer)
glBindBuffer(GLenum(GL_ELEMENT_ARRAY_BUFFER), indexBuffer)
// 根据 indices 拆分出三角形需要的 vertices
glDrawElements(GLenum(GL_TRIANGLES), GLsizei(indices.count), GLenum(GL_UNSIGNED_BYTE), nil)
{% endhighlight %}

#### Pass vertex with postion and color

<em class="fab fa-github"></em> [示例代码](https://github.com/danjiang/LearningOpenGLES2/tree/master/03.ColoredSquare "danjiang / LearningOpenGLES2 / 03.ColoredSquare")

Vextex 数据结构有坐标位置和颜色：

{% highlight swift %}
let vertices : [Vertex] = [
    Vertex( 1.0, -1.0, 0, 1.0, 0.0, 0.0, 1.0),
    Vertex( 1.0,  1.0, 0, 0.0, 1.0, 0.0, 1.0),
    Vertex(-1.0,  1.0, 0, 0.0, 0.0, 1.0, 1.0),
    Vertex(-1.0, -1.0, 0, 1.0, 1.0, 0.0, 1.0)
]

let indices : [GLubyte] = [
    0, 1, 2,
    2, 3, 0
]
{% endhighlight %}

主要是要告知 Vertex 中数据对应的 Offset：

{% highlight swift %}
glEnableVertexAttribArray(VertexAttributes.position.rawValue)
glVertexAttribPointer(
    VertexAttributes.position.rawValue,
    3,
    GLenum(GL_FLOAT),
    GLboolean(GL_FALSE),
    GLsizei(MemoryLayout<Vertex>.size), BUFFER_OFFSET(0))

glEnableVertexAttribArray(VertexAttributes.color.rawValue)
glVertexAttribPointer(
    VertexAttributes.color.rawValue,
    4,
    GLenum(GL_FLOAT),
    GLboolean(GL_FALSE),
    GLsizei(MemoryLayout<Vertex>.size), BUFFER_OFFSET(3 * MemoryLayout<GLfloat>.size)) // x, y, z | r, g, b, a :: offset is 3*sizeof(GLfloat)
    
func BUFFER_OFFSET(_ n: Int) -> UnsafeRawPointer? {
    return UnsafeRawPointer(bitPattern: n)
}
{% endhighlight %}

#### Pass vertex with vertex array objects

<em class="fab fa-github"></em> [示例代码](https://github.com/danjiang/LearningOpenGLES2/tree/master/04.VertexArrayObject "danjiang / LearningOpenGLES2 / 04.VertexArrayObject")

可以通过 vertex array objects 将下面两步所做的事情放在一起：

1. CPU 传递数据给 GPU
2. Shader 如何理解 Vertex 中的数据

{% highlight swift %}
glGenVertexArraysOES(1, &vao)
glBindVertexArrayOES(vao)

glGenBuffers(GLsizei(1), &vertexBuffer)
glBindBuffer(GLenum(GL_ARRAY_BUFFER), vertexBuffer)
let count = vertices.count
let size =  MemoryLayout<Vertex>.size
glBufferData(GLenum(GL_ARRAY_BUFFER), count * size, vertices, GLenum(GL_STATIC_DRAW))

glGenBuffers(GLsizei(1), &indexBuffer)
glBindBuffer(GLenum(GL_ELEMENT_ARRAY_BUFFER), indexBuffer)
glBufferData(GLenum(GL_ELEMENT_ARRAY_BUFFER), indices.count * MemoryLayout<GLubyte>.size, indices, GLenum(GL_STATIC_DRAW))

glEnableVertexAttribArray(VertexAttributes.position.rawValue)
glVertexAttribPointer(
    VertexAttributes.position.rawValue,
    3,
    GLenum(GL_FLOAT),
    GLboolean(GL_FALSE),
    GLsizei(MemoryLayout<Vertex>.size), BUFFER_OFFSET(0))


glEnableVertexAttribArray(VertexAttributes.color.rawValue)
glVertexAttribPointer(
    VertexAttributes.color.rawValue,
    4,
    GLenum(GL_FLOAT),
    GLboolean(GL_FALSE),
    GLsizei(MemoryLayout<Vertex>.size), BUFFER_OFFSET(3 * MemoryLayout<GLfloat>.size))

glBindVertexArrayOES(0)
glBindBuffer(GLenum(GL_ARRAY_BUFFER), 0)
glBindBuffer(GLenum(GL_ELEMENT_ARRAY_BUFFER), 0)
{% endhighlight %}

绘制的时候要做的就要简单一些了：

{% highlight swift %}
glBindVertexArrayOES(vao)
glDrawElements(GLenum(GL_TRIANGLES), GLsizei(indices.count), GLenum(GL_UNSIGNED_BYTE), nil)
glBindVertexArrayOES(0)
{% endhighlight %}

#### Pass vertex via a pointer

<em class="fab fa-github"></em> [示例代码](https://github.com/danjiang/LearningOpenGLES2/tree/master/02.Triangle2 "danjiang / LearningOpenGLES2 / 02.Triangle2")

不通过 vertex buffer objects 传递数据，通过指针传递数据，如果数据结构复杂，通过 vertex buffer objects 更好：

{% highlight swift %}
var vertices : [GLfloat] = [
    0.0,  0.25, // TOP
    -0.5, -0.25, // LEFT
    0.5, -0.25, // RIGHT
]

glVertexAttribPointer(
    VertexAttributes.vertexAttribPosition.rawValue, // Attribute Index
    2, // 数据大小
    GLenum(GL_FLOAT), // 数据类型
    GLboolean(GL_FALSE), // 是否标准化
    GLsizei(0),
    &vertices) // 通过指针传递数据
{% endhighlight %}

> *void VertexAttribPointer(uint index, int size, enum type,
> boolean normalized, sizei stride, const void *pointer);*
> 
> Vertex data may be sourced from arrays that are stored in application memory (via a pointer) or faster GPU memory (in a buffer object). 
> 
> If an ARRAY_BUFFER is bound, the attribute will be read from the bound buffer, and pointer is treated as an offset within the buffer.
>
> ---- Quote from OpenGL ES 2.0 Quick Reference Card