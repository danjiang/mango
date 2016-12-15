---
title: Swift 方法
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: swift
---

从这篇文章，你将学习到如何使用 Swift 的方法，方法可以定义在类、结构体和枚举中，方法简单来说就是和特定类型关联的函数。

![Swift Methods](/images/swift-methods.jpg)

## 实例方法

### 基本概念

下面的示例中定义了 3 个实例方法 **increment**、**incrementBy(_:)** 和 **reset**，它们都操作了实例变量 **count**。

{% highlight swift %}
class Counter {
  var count = 0
  func increment() {
    count += 1
  }
  func increment(by amount: Int) {
    count += amount
  }
  func reset() {
    count = 0
  }
}

let counter = Counter()
counter.increment()
counter.increment(by: 2)
counter.reset()
{% endhighlight %}

关于方法的参数和返回值，可以更多参考 [Swift 函数](/programming/2016/04/08/swift-functions/)，都是一样的。

### self

每一个实例都有一个隐含属性叫 **self**，它就代表实例本身，下面示例中 **self.x** 指就是实例的属性 **x**，和方法的参数 **x** 区分开来。

{% highlight swift %}
struct Point {
  var x = 0.0, y = 0.0
  func isToTheRightOf(x: Double) -> Bool {
    return self.x > x
  }
}
{% endhighlight %}

### 通过实例方法修改值类型

结构体和枚举都是值类型，默认来说，不能通过实例方法修改值类型的属性，但是可以在方法前面添加 **mutating** 来打破这个规则，搞得这么麻烦就是不鼓励大家这么用。

{% highlight swift %}
struct Point {
  var x = 0.0, y = 0.0
  mutating func moveBy(x deltaX: Double, y deltaY: Double) {
    x += deltaX
    y += deltaY
  }
}

struct Point {
  var x = 0.0, y = 0.0
  mutating func moveBy(x deltaX: Double, y deltaY: Double) {
    self = Point(x: x + deltaX, y: y + deltaY)
  }
}
{% endhighlight %}

## 类型方法

前面介绍的实例方法，在某一个实例上调用。当然，还可以定义在类型上调用的方法，也就是类型方法。在 **func** 前面加 **static** 来定义类型方法；在 **func** 前面加 **class** 也可以定义类型方法，不过子类就可以覆盖其实现。

{% highlight swift %}
struct LevelTracker {
  static var highesetUnlockedLevel = 1
  var currentLevel = 1
  
  static func unlock(_ level: Int) {
    if level > highesetUnlockedLevel {
      highesetUnlockedLevel = level
    }
  }
  
  static func isUnlocked(_ level: Int) -> Bool {
    return level <= highesetUnlockedLevel
  }
  
  mutating func advance(to level: Int) -> Bool {
    if LevelTracker.isUnlocked(level) {
      currentLevel = level
      return true
    } else {
      return false
    }
  }
}
{% endhighlight %}

注意，类型方法中引用类型属性，不需要通过类型名称。
