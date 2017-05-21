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
  case up
  case down
  case left
  case right
}

var direction = Direction.down
direction = .left // 已经知道 direction 的类型，推荐使用这种方式来赋值
{% endhighlight %}

## 枚举是值类型

值类型，在赋值或者当作参数传递给函数时，其值都会被拷贝。

{% highlight swift %}
enum CompassPoint {
  case north, south, east, west
}
var currentDirection = CompassPoint.west
let rememberedDirection = currentDirection
currentDirection = .east
if rememberedDirection == .west {
  print("The remembered direction is still .West")
}
{% endhighlight %}

## Switch

使用 **switch** 语句可以很方便地判断枚举的各种情况，如果 **switch** 语句中的 **case** 能覆盖枚举的每一种情况，也就不需要 **default**，否则就需要 **default** 来处理没有覆盖到的情况。

{% highlight swift %}
var direction = Direction.left

switch direction {
case .up:
  print("Keep going")
case .down:
  print("Go back")
case .left:
  print("Go left")
case .right:
  print("Go right")
}
// prints "Go right"

switch direction {
case .down:
  print("Go back")
default:
  print("You are lost")
}
// prints "You are lost"
{% endhighlight %}

## 关联值

在定义枚举时，每一种情况都可以附加关联值，比如像下面这样定义一个条码的枚举，一个条码代表一个唯一的物品，条码既可以是条形码，也可以是二维码。

{% highlight swift %}
enum Barcode {
  case upca(Int, Int, Int, Int)
  case qrCode(String)
}

var barcode = Barcode.upca(8, 112233, 45678, 3)
barcode = .qrCode("ABCDEFGHIJKLMNOP")

switch barcode {
case .upca(let numberSystem, let manufacturer, let product, let check):
  print("UPC-A: \(numberSystem), \(manufacturer), \(product), \(check).")
case .qrCode(let productCode):
  print("QR code: \(productCode).")
}
{% endhighlight %}

## 原始值

在定义枚举时，每一种情况都可以有默认值，也叫原始值，从名字也看得出来，原始值和关联值不同在于不能改变，并且都是同一类型。

### 显示地定义原始值

{% highlight swift %}
enum ASCIIControlCharacter: Character {
  case tab = "\t"
  case lineFeed = "\n"
  case carriageReturn = "\r"
}
{% endhighlight %}

### 隐式地定义原始值

{% highlight swift %}
enum Planet: Int {
  case mercury = 1, venus, earth, mars
}

print(Planet.venus.rawValue) // 2

enum Direction: String {
  case left, right, forward, back
}

print(Direction.right.rawValue) // Right
{% endhighlight %}

### 从原始值初始化

{% highlight swift %}
if let home = Direction(rawValue: "Back") {
  print(home) // Back
}

if let lost = Direction(rawValue: "Lost") {
  print(lost)
} else {
  print("Be lost") // Be lost
}
{% endhighlight %}

## 递归枚举

枚举定义中还有另一个枚举的实例，需要使用 **indirect** 关键词来告诉编译器为了间接插入必要中间层：

{% highlight swift %}
indirect enum ArithmeticExpression {
  case number(Int)
  case addition(ArithmeticExpression, ArithmeticExpression)
  case multiplication(ArithmeticExpression, ArithmeticExpression)
}
{% endhighlight %}