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
