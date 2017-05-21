---
title: Swift 数字和运算符
author: 但江
avatar: danjiang
location: 成都 
category: programming
tag: swift
---

从这篇文章，你将学习到如何使用 Swift 数字和运算符，还会说到布尔值，让我们开始吧。

![Swift Numbers and Operators](/images/swift-numbers-and-operators.jpg)

## 赋值运算符

**=** 赋值运算符可以初始化或者改变变量的值。

{% highlight swift %}
let b = 13
var a = 7
a = b // a 的值变成 13
{% endhighlight %}

## 算数运算符

**+** 加法、**-** 减法、**\*** 乘法和 **/** 除法。

{% highlight swift %}
2 + 2 // 等于 4
3 - 2 // 等于 1
3 * 3 // 等于 9
8 / 4 // 等于 2
{% endhighlight %}

**%** 取余运算符，只能是整数。

{% highlight swift %}
9 % 4 // 9 = 4 * 2 + 1，所以还剩 1
{% endhighlight %}

## 复合赋值运算符

一种简便的运算和赋值在一起的方式，下面示例的是加法，其他算术运算符类似。

{% highlight swift %}
var a = 2
a += 3 // a = a + 3，等于 5
{% endhighlight %}

## 比较运算符

**==** 相等、**!=** 不相等、**>** 大于、**<** 小于、**>=** 大于等于和 **<=** 小于等于。

{% highlight swift %}
1 == 1 // ture
2 != 1 // true
2 > 1 // true
1 < 2 // true
1 >= 1 // true
2 <= 1 // false
{% endhighlight %}

比较运算符的结果都是布尔值，布尔值在 Swift 中也是一种数据类型，注意只有两种值 **true** 和 **false**。

{% highlight swift %}
var flag: Bool = true
flag = false
{% endhighlight %}

## 三元条件运算符

一种简便的 **if else** 运算的写法。

{% highlight swift %}
var distance = 80
var result = distance > 60 ? "far" : "nearby"
{% endhighlight %}

等价于

{% highlight swift %}
var result = "unkown"
if distance > 60 {
  result = "far"
} else {
  result = "nearby"
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

## 逻辑运算符

组合布尔逻辑的运算符，**!** 非，**&&** 且，**\|\|** 或。

{% highlight swift %}
let allowed = false
if !allowed {
  print("Denied")
}

let young = true
let handsome = false
if young && handsome {
  print("Good Boy")
}

let poor = true
let lazy = true
if poor || lazy {
  print("Bad Boy")
}
{% endhighlight %}

## 自定义运算符

Swift 中支持自定义运算符，如下是重载 Swift 中已经存在的运算符：

{% highlight swift %}
struct Point {
  var x = 0
  var y = 0
}

func +(left: Point, right: Point) -> Point {
  return Point(x: left.x + right.x, y: left.y + right.y)
}

let p1 = Point(x: 2, y: 5)
let p2 = Point(x: 4, y: 3)
let p3 = p1 + p2
{% endhighlight %}

定义不存在的 Swift 中间运算符，这里使用 **infix** 关键词，不过还有 **prefix** 和 **postfix** 关键词：

{% highlight swift %}
infix operator >>>

func >>>(filter1: @escaping Filter, filter2: @escaping Filter) -> Filter {
  return { image in filter2(filter1(image)) }
}
{% endhighlight %}