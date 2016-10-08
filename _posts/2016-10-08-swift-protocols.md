---
title: Swift 协议
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: swift
---

从这篇文章，你将学习到 Swift 的协议，协议声明了一套方法、属性和其他的要求，类、结构体和枚举可以实现协议的要求，满足这一套功能，让我们开始吧。

![Swift Protocols](/images/swift-protocols.jpg)

## 基本语法

类的父类放在第一个，后面依次跟上协议，多个协议用逗号隔开。

{% highlight swift %}
protocol FirstProtocol {

}

protocol AnotherProtocol {
    
}

class SomeSuperclass {

}

class SomeClass: SomeSuperclass, FirstProtocol, AnotherProtocol {

}
{% endhighlight %}

## 属性

属性需求需要在末尾表明是读写 **{ get set }**，还是只读 **{ get }**。下面示例说明协议只是声明你要实现的功能，你怎么实现都是可以的，**Person** 通过存储属性来实现，**Starship** 通过计算属性来实现。

{% highlight swift %}
protocol FullyNamed {
  var fullName: String { get }
}

struct Person: FullyNamed {
  var fullName: String
}

class Starship: FullyNamed {
  var prefix: String?
  var name: String
  init(name: String, prefix: String? = nil) {
    self.name = name
    self.prefix = prefix
  }
  var fullName: String {
    return (prefix != nil ? prefix! + " " : "") + name
  }
}
{% endhighlight %}

## 方法

方法需求的声明和在类、结构体中定义方法没有什么太大区别，你不需要实现方法是自然的事，但是不能给方法参数定义默认值。

{% highlight swift %}
protocol Greating {
    func hello() -> String
}

class Welcome: Greating {
    func hello() -> String {
        return "Hello"
    }
}
{% endhighlight %}

## 修改方法

前面的博文也提到过，对于结构体和枚举这种值类型，想修改值类型的属性，需要在方法前面添加 **mutating**，对于在协议中的方法声明也是这样。

{% highlight swift %}
protocol Togglable {
    mutating func toggle()
}

enum OnOffSwitch: Togglable {
    case off, on
    mutating func toggle() {
        switch self {
        case .off:
            self = .on
        case .on:
            self = .off
        }
    }
}
{% endhighlight %}

## 初始化器

初始化器也是和在类和结构体中声明一样，只是不需要实现。注意在类中满足协议中声明的初始化器，既可以是指定初始化器，也可以是便利初始化器，前面一定要加 **required** 修饰符。

{% highlight swift %}
protocol SomeProtocol {
    init()
}

class SomeSuperClass {
    init() {
        
    }
}

class SomeSubClass: SomeSuperClass, SomeProtocol {
    required override init() {
        
    }
}
{% endhighlight %}

## 协议也是一种类型

既然是一种类型，当然可以定义协议变量和常量，还有数组。

{% highlight swift %}
protocol Generator {
    init()
    func randomNumber() -> Int
}

let generator: Generator
let generators: [Generator]

struct Caculator {
    let generator: Generator
    
    func random() {
        generator.randomNumber()
    }
}
{% endhighlight %}

> 上面的示例中，**Caculator** 在生产随机数时并没有自己实现，而是将这个任务代理给了 **Generator**，这就是 Cocoa 编程中常见的代理模式 **Delegation**

## 通过添加扩展来满足协议

扩展可以用来给定义的类型增加功能，自然也可以用来满足协议。

{% highlight swift %}
struct Dice {
    var sides: Int
}

protocol TextRepresentable {
    var textualDescription: String { get }
}

extension Dice: TextRepresentable {
    var textualDescription: String {
        return "A \(sides)-sided dice"
    }
}
{% endhighlight %}

## 协议继承

继承其他协议，再新增更多需求。

{% highlight swift %}
protocol SomeProtocol {

}

protocol AnotherProtocol {

}

protocol InheritingProtocol: SomeProtocol, AnotherProtocol {
    // 新增其他需求
}
{% endhighlight %}

## 协议组合

可以要求一个类型同时满足多个协议。

{% highlight swift %}
protocol SomeProtocol {

}

protocol AnotherProtocol {

}

struct SomeStruct {
    func compose(p: SomeProtocol & AnotherProtocol) {
        
    }
}
{% endhighlight %}

## 限定只有类可以满足的协议

在协议继承列表中添加 **class**。

{% highlight swift %}
protocol SomeClassOnlyProtocol: class {
    
}
{% endhighlight %}

## 检查协议的满足情况

可以通过类型转换的方法来检查协议的满足情况：

* **is** 检查实例是否满足协议，返回 **true** 或者 **false**；
* **as?** 返回可选协议类型，如果不满足协议则是 **nil**；
* **as!** 强制转换为协议类型，如果不满足协议，运行时就会挂。

{% highlight swift %}
protocol HasArea {
  var area: Double { get }
}

class Country: HasArea {
  var area: Double
  init(area: Double) {
    self.area = area
  }
}

class Animal {
  var legs: Int
  init(legs: Int) {
    self.legs = legs
  }
}

let objects: [AnyObject] = [Country(area: 243_610), Animal(legs: 4)]

for object in objects {
  if let objectWithArea = object as? HasArea {
    print("Area is \(objectWithArea)")
  } else {
    print("Something that doesn`t have an area")
  }
}
{% endhighlight %}

## 可选协议需求

可选协议需求定义要加 **optional** 修饰符，注意协议和协议需求都要加 **@objc** 属性，**@objc** 属性表明协议只能被继承自 Objective-C 的类或其他标示了 **@objc** 属性的类满足。

{% highlight swift %}
@objc protocol CounterDataSource {
  @objc optional func increment(forCount count: Int) -> Int
  @objc optional var fixedIncrement: Int { get }
}
{% endhighlight %}

## 协议扩展

协议可以被扩展，来提供方法和属性的实现，这样所有满足了协议的类型都有这些方法和属性。

### 默认实现

协议扩展可以用来提供协议需求的默认实现。

{% highlight swift %}
protocol Generator {
  func randomNumber() -> Int
}

extension Generator {
  func randomNumber() -> Int {
    return 2
  }
}

let generator = MyGenerator()
generator.randomNumber()
{% endhighlight %}

### 添加约束

协议扩展还可以利用泛型 **where** 语句来约束满足协议的类型，只有在这些类型也满足这些约束，这些类型才会得到这些扩展的方法和属性。

{% highlight swift %}
protocol Generator {
  func randomNumber() -> Int
}

extension Collection where Iterator.Element: Generator {
  var randomNumber: Int {
    var sum = 0
    for element in self {
      sum += element.randomNumber()
    }
    return sum
  }
}

struct MyGenerator: Generator {
  func randomNumber() -> Int {
    return 1
  }
}

let generators = [MyGenerator(), MyGenerator()]
generators.randomNumber

let numbers = [1, 2]
// generators.randomNumber 没有此方法可以调用
{% endhighlight %}
