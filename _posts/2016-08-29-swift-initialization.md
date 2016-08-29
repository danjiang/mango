---
title: Swift 初始化
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: swift
---

从这篇文章，你将学习到 Swift 的初始化和反初始化，针对结构体和类，有些不同，让我们开始吧。

![Swift Initialization](/images/swift-initialization.jpg)

## 初始化

初始化的过程就是让实例在使用前完成必要的准备，比较主要的就是给存储属性赋值，赋值可以通过定义属性的时候给默认值，也可以通过初始化器来赋值。

## 结构体的初始化

结构体因为没有继承的能力，所以初始化还是比较简单的。

{% highlight swift %}
struct Fahrenheit {
  var temperature: Double
  init() {
    temperature = 32.0
  }
}
{% endhighlight %}

{% highlight swift %}
struct Fahrenheit {
  var temperature = 32.0
}
{% endhighlight %}

## 类的初始化

### 指定初始化器

{% highlight swift %}
init() {
}
{% endhighlight %}

指定初始化器是类的主要初始化器，一个指定初始化器应该初始化类声明的所有属性，调用父类的初始化器来将初始化通过父类链传递下去。

###  便利初始化器

{% highlight swift %}
convenience init() {
}
{% endhighlight %}

便利初始化器是类的支持初始化器，你可以定义便利初始化器来调用相同类中定义的指定初始化器，简化指定初始化器中需要的一些参数，设定默认值或者通过计算。

###  继承

![Swift Class Initialization](/images/swift-class-initialization.png)

- 指定初始化器，如果有父类，必须调用父类的指定初始化器。
- 便利初始化器必须调用相同类中的指定初始化器。

{% highlight swift %}
class Food {
  var name: String
  
  init (name: String) {
    self.name = name
  }
  
  convenience init() {
    self.init(name: "[Unnamed]")
  }
}

class RecipeIngredient: Food {
  var quantity: Int
  
  init (name: String, quantity: Int) {
    self.quantity = quantity
    super.init(name: name)
  }
  
  override convenience init(name: String) {
    self.init(name: name, quantity: 1)
  }
}
{% endhighlight %}

###  反初始化

{% highlight swift %}
deinit {
}
{% endhighlight %}

**deinit** 在类实例被回收前调用，可以做一些资源释放的工作，只有类才有。你不能自己调用 **deinit**，根据 **ARC**，系统会决定什么时候需要回收类实例调用 **deinit**。父类的 **deinit** 会在子类的 **deinit** 结束时被自动调用。