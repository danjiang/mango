---
title: Swift 类
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: swift
---

从这篇文章，你将学习到如何使用 Swift 的类，具有 Swift 的结构体的能力，而且还有更多，当然其涵盖的内容较多，这里先只讲解大概，更多细节可以查看下面类的能力。

![Swift Classes](/images/swift-classes.jpg)

## 类的能力

* [属性](/programming/2016/07/30/swift-properties/)
* [方法](/programming/2016/08/14/swift-methods/)
* [下标](/programming/2016/08/21/swift-subscripts/)
* [初始化](/programming/2016/08/29/swift-initialization/)
* [扩展](/programming/2016/09/12/swift-extensions/)
* 协议
* 继承
* 类型转换
* [反初始化](/programming/2016/08/29/swift-initialization/)
* ARC

## 定义

定义了类 **VideoMode**，有一个结构体属性 **resolution**、布尔属性 **interlaced**、浮点属性 **frameRate** 和字符串属性 **name**。

{% highlight swift %}
class VideoMode {
  var resolution = Resolution()
  var interlaced = false
  var frameRate = 0.0
  var name: String?
}
{% endhighlight %}

## 创建实例

创建了类 **VideoMode** 的实例，其所有属性都是默认值。

{% highlight swift %}
let someVideoMode = VideoMode()
{% endhighlight %}

## 访问属性

通过 **.** 就可以访问属性，当然还可以给属性赋值。

{% highlight swift %}
print("The width of someVideoMode is \(someVideoMode.resolution.width)")

someVideoMode.resolution.width = 1280
print("The width of someVideoMode is now \(someVideoMode.resolution.width)")
{% endhighlight %}

## 类是引用类型

引用类型，在赋值或者当作参数传递给函数时，只会传递实例的引用。

{% highlight swift %}
let hd = Resolution(width: 1920, height: 1080)

let tenEighty = VideoMode()
tenEighty.resolution = hd
tenEighty.interlaced = true
tenEighty.name = "1080i"
tenEighty.frameRate = 25.0

let alsoTenEighty = tenEighty
alsoTenEighty.frameRate = 30.0

print("The frameRate property of tenEighty is now \(tenEighty.frameRate)")
{% endhighlight %}

## 恒等运算符

因为类是引用类型，所以我们需要一个运算符来比较两个常量或变量是否指向同一个类的实例，这就是 **===** 和 **!==**。

{% highlight swift %}
if tenEighty === alsoTenEighty {
  print("same instance")
}

if tenEighty !== alsoTenEighty {
  print("not same instance")
}
{% endhighlight %}
