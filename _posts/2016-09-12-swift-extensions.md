---
title: Swift 扩展
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: swift
---

从这篇文章，你将学习到 Swift 的扩展，可以给已经存在的类、结构体、枚举和协议类型添加新功能，让我们开始吧。

![Swift Extensions](/images/swift-extensions.jpg)

## 基本语法

基本语法如下，注意扩展只能添加新功能，不能重写旧功能。

{% highlight swift %}
class Item {
  
}

extension Item {
  
}
{% endhighlight %}

## 计算属性

下面的示例扩展了 Swift 内置的 **Double** 类型，增加了一些计算属性来进行单位转换。注意扩展不能添加新的存储属性和属性观察者。

{% highlight swift %}
extension Double {
  var km: Double { return self * 1_000.0 }
  var m: Double { return self }
  var cm: Double { return self / 100.0 }
  var mm: Double { return self / 1_000.0 }
  var ft: Double { return self / 3.28084 }
}
let oneInch = 25.4.mm
print("One inch is \(oneInch) meters")
{% endhighlight %}

## 初始化器

下面的示例扩展了自定义结构体 **Rect**，增加了新的初始化器，通过一些计算最终调用了原有的初始化器。注意扩展只能给类增加便利初始化器，不能给类增加指定初始化器和反初始化器。

{% highlight swift %}
struct Size {
  var width = 0.0, height = 0.0
}
struct Point {
  var x = 0.0, y = 0.0
}
struct Rect {
  var origin = Point()
  var size = Size()
}

extension Rect {
  init(center: Point, size: Size) {
    let originX = center.x - (size.width / 2)
    let originY = center.y - (size.height / 2)
    self.init(origin: Point(x: originX, y: originY), size: size)
  }
}
{% endhighlight %}

## 方法

下面的示例扩展了 Swift 内置的 **Int** 类型，增加了一个实例方法。

{% highlight swift %}
extension Int {
  func repetitions(task: () -> Void) {
    for _ in 0..<self {
      task()
    }
  }
}

3.repetitions { 
  print("Hello!")
}
{% endhighlight %}

如果扩展新增的方法会改变 **self** 的值，需要在方法前面添加 **mutating** 关键词。

{% highlight swift %}
extension Int {
  mutating func square() {
    self = self * self
  }
}
var someInt = 3
someInt.square()
{% endhighlight %}

## 下标

下面的示例扩展了 Swift 内置的 **Int** 类型，增加了一个下标来获取整数在十进制上的每一位数字。

{% highlight swift %}
extension Int {
  subscript(digitIndex: Int) -> Int {
    var decimalBase = 1
    for _ in 0..<digitIndex {
      decimalBase *= 10
    }
    return (self / decimalBase) % 10
  }
}
123456789[0]
123456789[1]
{% endhighlight %}

## 嵌套类型

下面的示例扩展了 Swift 内置的 **Int** 类型，增加了一个嵌套类型。

{% highlight swift %}
extension Int {
  enum Kind {
    case negative, zero, positive
  }
  var kind: Kind {
    switch self {
    case 0:
      return .zero
    case let x where x > 0:
      return .positive
    default:
      return .negative
    }
  }
}

func printIntegerKinds(_ numbers: [Int]) {
  for number in numbers {
    switch number.kind {
    case .negative:
      print("- ", terminator: "")
    case .zero:
      print("0 ", terminator: "")
    case .positive:
      print("+ ", terminator: "")
    }
  }
}
printIntegerKinds([1, 0, -1])
{% endhighlight %}
