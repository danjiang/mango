---
title: Swift 继承
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: swift
---

从这篇文章，你将学习到 Swift 的继承，继承是类独有的特性，让我们开始吧。

![Swift Inheritance](/images/swift-inheritance.jpg)

## 定义基类

不继承于其他类的类就是基类，注意 Swift 中定义的基类不会再自动继承于一个统一的基类，很多其他编程语言会这样。

{% highlight swift %}
class Vehicle {
  
  var currentSpeed = 0.0
  var description: String {
    return "traveling at \(currentSpeed) miles per hour"
  }
  func makeNoise() {}
  
}
{% endhighlight %}

## 定义子类

子类继承了来自父类的功能，还可以再添加自己的功能。

{% highlight swift %}
class Bicycle: Vehicle {
  var hasBasket = false
}
{% endhighlight %}

## 覆盖

覆盖父类的方法，属性和下标，需要在定义前面添加 **override**，如果覆盖了父类方法，却没有写 **override**，编译器会报错，避免不小心覆盖父类方法的失误。

### 访问父类的方法，属性和下标

在覆盖父类的功能，添加新的功能时，还可以复用父类的功能。

{% highlight swift %}
super.someMethod()
super.someProperty
super[someIndex]
{% endhighlight %}

### 覆盖方法

{% highlight swift %}
class Train: Vehicle {
  override func makeNoise() {
    print("Choo Choo")
  }
}
{% endhighlight %}

### 覆盖属性的 get 和 set

无论继承的属性是通过存储属性，还是计算属性来实现的，都可以通过提供 **get** 和 **set** 来覆盖。继承的只读属性可以变成读写属性，但是继承的读写属性不可以变成只读属性。

{% highlight swift %}
class Car: Vehicle {
  var gear = 1
  override var description: String {
    return super.description + " in gear \(gear)"
  }
}
{% endhighlight %}

### 覆盖属性观察者

无论继承的属性是通过存储属性，还是计算属性来实现的，都还可以提供属性观察者。

{% highlight swift %}
class AutomaticCar: Car {
  override var currentSpeed: Double {
    didSet {
      gear = Int(currentSpeed / 10.0) + 1
    }
  }
}
{% endhighlight %}

## 避免覆盖

在定义父类时，不希望子类覆盖的功能，可以通过修饰符 **final** 来实现。

{% highlight swift %}
final var
final func
final subscript
final class
{% endhighlight %}
