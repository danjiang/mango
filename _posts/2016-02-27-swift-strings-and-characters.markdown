---
title: Swift 字符串和字符
author: 但江
avatar: danjiang
location: 成都 
category: programming
tag: swift
---

从这篇文章，你将学习到如何使用 Swift 字符串和字符，当然一定会讲到字符编码的问题，让我们开始吧。

![Swift Strings and Characters](/images/swift-strings-and-characters.jpg)

## 创建字符串

一段由 **"** **"** 包围起来的字符也就是字符串了。

{% highlight swift %}
let greeting = "早上好"
{% endhighlight %}

## 可变和不可变

用过 Objective-C 的朋友应该知道很多 Foundation 的基础类都有可变和不可变两种类型，针对字符串就是 NSString 和 NSMutableString，Swift 没有那么复杂，像其他类型一样通过 **let** 和 **var** 来区分。

{% highlight swift %}
var welcome = "欢迎"
welcome += "，马克"

let name = "马克"
name += " 安德森" // 会报错的
{% endhighlight %}

## 字符串是值类型

请记住字符串是值类型，所以赋值或者方法的参数传递时，会重新拷贝一份，再传递，不用担心会改变原值。

{% highlight swift %}
func greeting(_ name: String) {
  var username = name
  username = "汤姆"
  print(username)
}

var name = "马克"
greeting(name)

print(name) // name 仍然是马克
{% endhighlight %}

## 遍历字符串

字符就是字符串中的每一个字符，我们可以遍历一个字符串来看看。

{% highlight swift %}
let greeting = "早上好"
for character in greeting.characters {
  print(character)
}
{% endhighlight %}

## 拼接字符串

**+** 就可以拼接两个字符串，字符串的 **append** 方法可以拼接字符。

{% highlight swift %}
let greeting = "早上好"
let name = "马克"
var welcome = greeting + "，" + name

let exclamationMark: Character = "!"
welcome.append(exclamationMark)
{% endhighlight %}

## 字符串插值

**\\()** 可以更方便地在字符串中插入其他基础值。

{% highlight swift %}
let two = 2
let three = 3
let result = "\(two) 加上 \(three) 的结果是 \(two + three)"
{% endhighlight %}

## 访问和修改字符串

需要通过一些特别的下标语法来访问字符串中每一个字符，不能直接通过 Int，有点蛋疼。

{% highlight swift %}
var greeting = "早上好"

greeting[greeting.startIndex] // 早
greeting[greeting.index(before: greeting.endIndex)] // 好
greeting[greeting.index(after: greeting.startIndex)] // 上

let index = greeting.index(greeting.startIndex, offsetBy: 2)
greeting[index] // 好

greeting.insert("！", at: greeting.endIndex) // 早上好！

let range = greeting.startIndex..<greeting.index(greeting.startIndex, offsetBy: 1)
greeting.removeSubrange(range)
greeting.insert("晚", at: greeting.startIndex) // 晚上好！
{% endhighlight %}

## 比较字符串

如果两个字符串（或者两个字符）的可扩展的字形群集是标准相等的，那就认为它们是相等的，在这个情况下，即使可扩展的字形群集是有不同的 Unicode 标量构成的，只要它们有同样的语言意义和外观，就认为它们标准相等，

听起来好复杂的样子，绝大多数情况不用深究，只要知道 **==** 来比较字符串的值，长的一样就会是相等的。

![Swift Strings Equal](/images/swift-strings-equal.jpg)

## Unicode

计算机只能存储 1 和 0，怎么知道你定义的字符串是什么内容呢，办法就是把每一个字符通过一串数字来表示，这就是所谓字符集编码。

Unicode 是一个国际标准，用于文本的编码和表示，Unicode 是其中一种方式，还有 UTF-8 和 UTF-16 这样的方式，Swift 的字符串类型是基于 Unicode 标量建立的，Unicode 标量是对应字符或者修饰符的唯一的 21 位数字。

{% highlight swift %}
let dollarSign = "\u{24}" // $，Unicode scalar U+0024
let blackHeart = "\u{2665}" // ♥，Unicode scalar U+2665
{% endhighlight %}
