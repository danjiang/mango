---
title: Swift 可选类型
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: swift
---

从这篇文章，你将学习到如何使用 Swift 可选类型，如果稍微有点其他语言的编程经验，都会大概使用过对空值的表示方法，nil、null 和 none 傻傻分不清楚，让我们开始吧。

![Swift Optionals](/images/swift-optionals.jpg)

## 什么是可选类型

Swift 中为解决对空值的表示采用了可选类型 Optional，表明所定义的变量要么有值，要么没有值，还要知道可选类型不仅可以用在引用类型，还可以用在像 Int 这样的值类型。

{% highlight swift %}
var choice: Int? // 定义了一个可选 Int 类型的变量
choice = 5 // 变量可以有值为 5
choice = nil // 变量可以没有任何值为空
choice = 0 // 0 也是有值，不是空
{% endhighlight %}

## 展开可选类型

为什么要展开可选类型，先看看下面这个例子：

{% highlight swift %}
var name: String? = "Jack"
var greeting = "Hello, " + name
{% endhighlight %}

在 greeting 这一行会报如下错误：

{% highlight swift %}
error: value of optional type 'String?' not unwrapped; did you mean to use '!' or '?'?
{% endhighlight %}

Swift 是门讲究少出错的语言，在使用可选类型进行运算前，要通过下面两种方式展开可选类型保证可选类型有值，再进行其他运算。

### 强制展开

你非常确定，可选类型在运算时，确实有值，你可以采用下面这种方式，但是，运行时如果没有值，程序就会崩溃。

{% highlight swift %}
var name: String? = "Jack"
var greeting = "Hello, " + name!
{% endhighlight %}

### if let 绑定

如果 name 有值，程序会走 if 这一段，并将 name 赋值给 n；如果 name 没有值，会走 else 这一段。

{% highlight swift %}
var name: String? = "Jack"
if let n = name {
  print("Hello, " + n)
} else {
  print("Hello, Stranger")
}
{% endhighlight %}

有多个可选类型的情况：

{% highlight swift %}
var age: Int? = 5
var name: String? = "Jack"
if let n = name, a = age {
  print("Hello \(n). You are \(a) years old.")
} else {
  print("Hello stranger. How old are you?")
}
{% endhighlight %}

## 隐式展开可选类型

有时候，一个变量在初始化时，可能为 nil，当被赋值后保证不会为 nil，这种情况下就应该使用隐式展开可选类型来简化调用时的代码。好绕口。

{% highlight swift %}
class LeftViewController: NSViewController {

  @IBOutlet weak var searchField: NSSearchField!

}
{% endhighlight %}

上面的示例中，**searchField** 在 **LeftViewController** 调用 **init** 方法初始化时还是 nil，但是 **LeftViewController** 从 **XIB** 中加载了相关视图过后，就不再为 nil，在后面逻辑代码处理中可以放心使用类似 **searchField.doSomething()** 这样的代码。

## 空合并运算符

在操作可选类型时候，一种简便的提供默认值的写法。

{% highlight swift %}
let defaultWeight = 60
var inputWeight: Int?

var weight = inputWeight ?? defaultWeight
// inputWeight 没有值，weight 结果是 60

inputWeight = 55
weight = inputWeight ?? defaultWeight
// inputWeight 有值，weight 结果是 55
{% endhighlight %}

## 可选链

可选链是一种在可选类型上访问属性、调用方法和访问下标的过程，如果可选类型有值，那么调用成功，如果可选类型没有值，整个过程将返回 nil，多个调用过程可以链接起来，整个链中如果任意一个可选类型没有值，则整个链返回 nil。

{% highlight swift %}
class Person {
  var residence: Residence?
}

class Residence {
  var numberOfRooms = 1
}

let john = Person()

if let roomCount = john.residence?.numberOfRooms {
  print("John`s residence has \(roomCount) room(s).")
} else {
  print("Unable to retrieve the number of rooms.")
}
{% endhighlight %}
