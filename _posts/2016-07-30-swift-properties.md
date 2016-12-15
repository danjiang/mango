---
title: Swift 属性
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: swift
---

从这篇文章，你将学习到如何使用 Swift 的属性，属性可以定义在类、结构体和枚举中。

![Swift Properties](/images/swift-properties.jpg)

## 存储属性

### 基本概念

就是定义在类或者结构体中的变量或常量

{% highlight swift %}
struct Person {
  let name: String
  var age: Int
}

var tom = Person(name: "Tom", age: 22)
tom.age += 1
{% endhighlight %}

### 常量结构体实例中的存储属性

当定义了常量结构体 **Person** 的实例 **mick**，就不能再改变 **mick** 的变量属性 **age**，记住结构体是值类型，常量结构体实例中每一个属性都不能改变值，即便这个属性是个变量属性。类是引用类型，常量类实例中每一个属性会根据其是常量属性，还是变量属性来决定是否可以被改变，因为类实例指向的引用一直没有变化，符合常量类实例规则。

{% highlight swift %}
let mick = Person(name: "Mick", age: 18)
mick.age += 1 // error
{% endhighlight %}

### 延迟加载的存储属性

存储属性前面加上 **lazy**，当这个属性第一次被使用到时，才进行加载。某些属性在初始化后且需要一些第三方条件的情况下才知道怎么加载，或者其加载条件很消耗资源，都可以采用延迟加载的策略，所以你应该能想到，**lazy** 只能用于变量属性。

{% highlight swift %}
class DataImporter {
  
  var fileName = "data.txt" // time consuming for IO
  
}

class DataManager {
  
  lazy var importer = DataImporter()
  var data = [String]()
  
}
{% endhighlight %}

## 计算属性

计算属性通常都是由其他存储属性或值计算出来的，这样可以暴露一些更简洁的接口，封装一些内部逻辑实现。如下的 **Rect** 中根据存储属性 **origin** 和 **size** 来计算的计算属性 **center**

{% highlight swift %}
struct Point {
  var x = 0.0, y = 0.0
}

struct Size {
  var width = 0.0, height = 0.0
}

struct Rect {
  var origin = Point()
  var size = Size()
  var center: Point {
    get {
      let centerX = origin.x + size.width / 2
      let centerY = origin.y + size.height / 2
      return Point(x: centerX, y: centerY)
    }
    set {
      origin.x = newValue.x - size.width / 2
      origin.y = newValue.y - size.height / 2
    }
  }
}
{% endhighlight %}

还可以不写 **set**，只提供 **get**，类似于提供了只读的情况

{% highlight swift %}
struct Rect {
  ...
  var maxX: Double {
    return origin.x + size.width
  }
  var maxY: Double {
    return origin.y + size.height
  }
  ...
}
{% endhighlight %}

## 属性观察者

属性观察者可以用于观察属性值的变化，但是有一点，设置的新属性值与旧属性值是否有变化都会被调用。

* **willSet** 在属性值被存储前调用，新属性值会作为参数 **newValue** 传入，如果不设置其他参数名；
* **didSet** 在属性值被存储后调用，旧属性值会作为参数 **oldValue** 传入，如果不设置其他参数名。

{% highlight swift %}
class StepCounter {
  var totalSteps: Int = 0 {
    willSet(newTotalSteps) {
      print("About to set totalSteps to \(newTotalSteps)")
    }
    didSet {
      if totalSteps > oldValue {
        print("Added \(totalSteps - oldValue) steps")
      }
    }
  }
}
{% endhighlight %}

## 类型属性

上面介绍的实例属性，属于某一个实例，每创建一个新的实例，它都有其自己的实例属性。类型属性属于这个类型，当需要定义对于所有实例属性都一样的常量或变量时，类型属性就很有用。由 **static** 来定义类型属性；**class** 还可以用在类的类型计算属性上，这样子类就可以覆盖其实现。

{% highlight swift %}
struct SomeStructure {
  static var storedTypeProperty = "Some value."
  static var computedTypeProperty: Int {
    return 1
  }
}

enum SomeEnumeration {
  static var storedTypeProperty = "Some value."
  static var computedTypeProperty: Int {
    return 6
  }
}

class SomeClass {
  static var storedTypeProperty = "Some value."
  static var computedTypeProperty: Int {
    return 27
  }
  class var overrideableComputedTypeProperty: Int {
    return 107
  }
}
{% endhighlight %}
