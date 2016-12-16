---
title: Swift 变量和常量
author: 但江
avatar: danjiang
location: 成都 
category: programming
tag: swift
---

从这篇文章，你将学习到如何使用 Swift 变量和常量，和其他任何编程语言一样，编程所使用的变量和常量都是要指定类型的，所以也会谈到类型相关的问题，最后会说一下元组 Tuples，让我们开始吧。

![Swift Variables and Constnats](/images/swift-variables-and-constants.jpg)

## 变量

**var** 来声明变量，意思是在给了变量初始值过后，还可以再次修改

{% highlight swift %}
var greeting = "早上好，张三"
greeting = "晚上好，张三"
{% endhighlight %}

## 常量

**let** 来声明常量，意思是在给了常量初始值过后，也就不能再修改了，否侧会有编译错误。

{% highlight swift %}
let apple = "苹果"
{% endhighlight %}

![Swift Constnats Error](/images/swift-constants-error.jpg)

> 编写 Swift 代码时，最好方式是一开始都声明为常量，在写过程中，如果发现确实需要改变其值，再将声明修改为变量。

## 命名

任何 Unicode 字符都可以作为变量名和常量名，最好用全单词且首字母小写。

{% highlight swift %}
let π = 3.14159
let userAge = 12
var personName = "Mick"
{% endhighlight %}

## 类型注解

类型注解 Type Annotation 意思是在声明变量或常量时，在 **:** 后面指明类型。

{% highlight swift %}
var greeting: String // 指明 greeting 是 String 类型
greeting = "你好" // 所以只能给 greeting 赋 String 类型的值
{% endhighlight %}

## 类型推断

你应该会注意到以上很多示例代码中没有使用类型注解，Swift 是强类型编辑语言，每一个变量或常量在声明时都要明确类型，那么这又是怎么做到的呢？这就是通过类型推断 Type Inference 来明确的。

{% highlight swift %}
let age = 42 // age 被推断为 Int 类型
let pi = 3.14159 // pi 被推断为 Double 类型
{% endhighlight %}

## 类型转换

Swift 是强类型语言，所以值不能自动地转换为其他类型，需要自己来做类型转换 Type Conversion，否侧会有编译错误。

![Swift Type Conversion Error](/images/swift-type-conversion-error.jpg)

{% highlight swift %}
let integer = 11 // integer 被推断为 Int 类型
var double = 5.5 // double 被推断为 Double 类型
double = Double(integer)
{% endhighlight %}

## 元组

元组 Tuple 可以让你创建和传递一组值，作为函数的返回值来返回多个值很方便。

{% highlight swift %}
func currentLocation() -> (Double, Double) {
  return (104.06667, 30.66667)
}

let location = currentLocation()
let latitude = location.0
let longitude = location.1
{% endhighlight %}
