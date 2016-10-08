---
title: Swift 结构体
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: swift
---

从这篇文章，你将学习到如何使用 Swift 的结构体，其涵盖的内容较多，这里先只讲解大概，更多细节可以查看下面结构体的能力。

![Swift Structs](/images/swift-structs.jpg)

## 结构体的能力

* [属性](/programming/2016/07/30/swift-properties/)
* [方法](/programming/2016/08/14/swift-methods/)
* [下标](/programming/2016/08/21/swift-subscripts/)
* [初始化](/programming/2016/08/29/swift-initialization/)
* [扩展](/programming/2016/09/12/swift-extensions/)
* [协议](/programming/2016/10/08/swift-protocols/)

## 定义

定义了结构体 **Resolution**，其有两个属性 **width** 和 **height**。

{% highlight swift %}
struct Resolution {
  var width = 0
  var height = 0
}
{% endhighlight %}

## 创建实例

创建了结构体 **Resolution** 的实例，其所有属性都是默认值。

{% highlight swift %}
let someResolution = Resolution()
{% endhighlight %}

## 访问属性

通过 **.** 就可以访问属性

{% highlight swift %}
print("The width of someResolution is \(someResolution.width)")
{% endhighlight %}

## 自动生成的初始化器

当没有给结构体定义初始化器时，会根据结构体的属性自动生成一个初始化器。

{% highlight swift %}
struct Resolution {
  var width = 0
  var height = 0
}

let someResolution = Resolution(width: 300, height: 400)
{% endhighlight %}

## 结构体是值类型

值类型，在赋值或者当作参数传递给函数时，其值都会被拷贝。值得注意的是，Swift 中，整型、浮点数、布尔、字符串、数组和字典都是值类型，都是通过结构体实现的。

{% highlight swift %}
struct Resolution {
  var width = 0
  var height = 0
}

let hd = Resolution(width: 1920, height: 1080)
var cinema = hd

cinema.width = 2048

print("cinema is now \(cinema.width) pixels wide")
print("hd is still \(hd.width) pixels wide")
{% endhighlight %}
