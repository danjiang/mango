---
title: Swift 枚举
author: 但江
avatar: danjiang
location: 成都 
category: programming
tag: swift
---

从这篇文章，你将学习到如何使用 Swift 的枚举，如果你需要定义一系列明确的范围值，比如月份、性别和方向，那么枚举就是最佳选择，让我们开始吧。

![Swift Enums](/images/swift-enums.jpg)

## 基本定义

**enum** 定义枚举的名称，**case** 定义枚举的每一种情况。

{% highlight swift %}
enum Direction {
  case Up
  case Down
  case Left
  case Right
}

var direction = Direction.Down
direction = .Left // 已经知道 direction 的类型，推荐使用这种方式来赋值
{% endhighlight %}

## Switch

使用 **switch** 语句可以很方便地判断枚举的各种情况，如果 **switch** 语句中的 **case** 能覆盖枚举的每一种情况，也就不需要 **default**，否则就需要 **default** 来处理没有覆盖到的情况。

{% highlight swift %}
var direction = Direction.Left

switch direction {
case .Up:
  print("Keep going")
case .Down:
  print("Go back")
case .Left:
  print("Go left")
case .Right:
  print("Go right")
}
// prints "Go right"

switch direction {
case .Down:
  print("Go back")
default:
  print("You are lost")
}
// prints "You are lost"
{% endhighlight %}

## 关联值

可变

## 原始值

不可变