---
title: Swift 内存管理
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: swift
---

从这篇文章，你将学习到 Swift 的内存管理，和 Objective-C 一样，主要也是通过 ARC 机制来实现的，还会讲到类实例之间的循环强引用和闭包的循环强引用，问题是怎么来的，以及通过弱引用或无主引用来解决问题的办法，让我们开始吧。

![Swift Memory Management](/images/swift-memory-management.jpg)

## ARC

ARC 也就是自动引用计数，当然只是针对类，当创建了一个类的新实例，针对这个实例的引用计数就是一，当将这个实例的引用赋值给属性、常量或变量，引用计数就加一，这些属性、常量或变量不再引用这个实例，引用计数就减一，引用计数为零的时候，这个实例在内存中的空间就会被回收掉，引用计数不为零，可以保证代码能获取到实例中的属性值等，也就是所谓的强引用。

{% highlight swift %}
class Person {
  let name: String
  init(name: String) {
    self.name = name
    print("\(name) is being initialized")
  }
  deinit {
    print("\(name) is being deinitialized")
  }
}

var reference1: Person?
var reference2: Person?
var reference3: Person?

reference1 = Person(name: "John Appleseed")
// Prints "John Appleseed is being initialized"

reference2 = reference1
reference3 = reference1

reference1 = nil
reference2 = nil

reference3 = nil
// Prints "John Appleseed is being deinitialized"
{% endhighlight %}

## 类实例之间的循环强引用

### 问题的由来

{% highlight swift %}
class Person {
  let name: String
  init(name: String) {
    self.name = name
  }
  var apartment: Apartment?
  deinit {
    print("\(name) is being deinitialized")
  }
}

class Apartment {
  let unit: String
  init(unit: String) {
    self.unit = unit
  }
  var tenant: Person?
  deinit {
    print("Apartment \(unit) is being deinitialized")
  }
}

var john: Person?
var unit4A: Apartment?

john = Person(name: "John Appleseed")
unit4A = Apartment(unit: "4A")
{% endhighlight %}

![Reference Cycle 01](/images/referenceCycle01_2x.png)

{% highlight swift %}
john!.apartment = unit4A
unit4A!.tenant = john
{% endhighlight %}

![Reference Cycle 02](/images/referenceCycle02_2x.png)

{% highlight swift %}
john = nil
unit4A = nil
{% endhighlight %}

![Reference Cycle 03](/images/referenceCycle03_2x.png)

### 弱引用

弱引用不会让引用计数加一，也就不会阻止实例在内存中的空间被回收掉，表示弱引用，只需要在属性或变量定义前添加 **weak**，因为弱引用在 **ARC** 回收掉实例后会被赋值为 **nil**，所以弱引用只能用在变量且是可选类型。

{% highlight swift %}
class Person {
  let name: String
  init(name: String) {
    self.name = name
  }
  var apartment: Apartment?
  deinit {
    print("\(name) is being deinitialized")
  }
}

class Apartment {
  let unit: String
  init(unit: String) {
    self.unit = unit
  }
  
  // 👇 Pay attention to this line
  weak var tenant: Person?
  
  deinit {
    print("Apartment \(unit) is being deinitialized")
  }
}

var john: Person?
var unit4A: Apartment?

john = Person(name: "John Appleseed")
unit4A = Apartment(unit: "4A")

john!.apartment = unit4A
unit4A!.tenant = john
{% endhighlight %}

![Weak Reference 01](/images/weakReference01_2x.png)

{% highlight swift %}
john = nil
{% endhighlight %}

![Weak Reference 02](/images/weakReference02_2x.png)

{% highlight swift %}
unit4A = nil
{% endhighlight %}

![Weak Reference 03](/images/weakReference03_2x.png)

### 无主引用

无主引用也不会让引用计数加一，也就不会阻止实例在内存中的空间被回收掉，表示无主引用，只需要在属性或变量定义前添加 **unowned**，因为无主引用在 **ARC** 回收掉实例后不会被赋值为 **nil**，并且无主引用被假定总是有值，所以无主引用只能用在非可选类型。

{% highlight swift %}
class Customer {
  let name: String
  var card: CreditCard?
  init(name: String) {
    self.name = name
  }
  deinit {
    print("\(name) is being deinitialized")
  }
}

class CreditCard {
  let number: UInt64
  
  // 👇 Pay attention to this line
  unowned let customer: Customer
  
  init(number: UInt64, customer: Customer) {
    self.number = number
    self.customer = customer
  }
  deinit {
    print("Card #\(number) is being deinitialized")
  }
}

var john: Customer?

john = Customer(name: "John Appleseed")
john!.card = CreditCard(number: 1234_5678_9012_3456, customer: john!)
{% endhighlight %}

![Unowned Reference 01](/images/unownedReference01_2x.png)

{% highlight swift %}
john = nil
{% endhighlight %}

![Unowned Reference 01](/images/unownedReference02_2x.png)

### 无主引用和隐式展开可选属性

**弱引用**的方式展示了两个属性都可以为 **nil** 的情况；**无主引用**的方式展示了一个属性可以为 **nil**，另一个属性不可以为 **nil** 的情况；下面要介绍的就是如何处理两个属性都不可以为 **nil** 的情况：

{% highlight swift %}
class Country {
  let name: String
  var capitalCity: City!
  init(name: String, capitalName: String) {
    self.name = name
    self.capitalCity = City(name: capitalName, country: self)
  }
}

class City {
  let name: String
  unowned let country: Country
  init(name: String, country: Country) {
    self.name = name
    self.country = country
  }
}

var country = Country(name: "Canada", capitalName: "Ottawa")
print("\(country.name)`s capital city is called \(country.capitalCity.name)")
{% endhighlight %}

## 闭包的循环强引用

### 问题的由来

{% highlight swift %}
class HTMLElement {
  let name: String
  let text: String?
  
  lazy var asHTML: () -> String = {
    if let text = self.text {
      return "<\(self.name)>\(text)</\(self.name)>"
    } else {
      return "<\(self.name)/>"
    }
  }
  
  init(name: String, text: String? = nil) {
    self.name = name
    self.text = text
  }
  
  deinit {
    print("\(name) is being deinitialized")
  }
}

var paragraph: HTMLElement? = HTMLElement(name: "p", text: "hello, world")
print(paragraph!.asHTML())

paragraph = nil
{% endhighlight %}

![Closure Reference Cycle 01](/images/closureReferenceCycle01_2x.png)

### 占用列表

占用列表定义了闭包捕获引用的规则，可以是弱引用和无主引用。

1. 当闭包和它捕获的实例总是互相引用，且同时被回收时，采用无主引用的方式；
2. 当捕获的引用可能为 **nil** 时，采用弱引用的方式。

根据上面的规则，**HTMLElement** 中的问题应该采用无主引用的方式：

{% highlight swift %}
class HTMLElement {
  let name: String
  let text: String?
  
  lazy var asHTML: () -> String = {
    // 👇 Pay attention to this line
    [unowned self] in
    if let text = self.text {
      return "<\(self.name)>\(text)</\(self.name)>"
    } else {
      return "<\(self.name)/>"
    }
  }
  
  init(name: String, text: String? = nil) {
    self.name = name
    self.text = text
  }
  
  deinit {
    print("\(name) is being deinitialized")
  }
}

var paragraph: HTMLElement? = HTMLElement(name: "p", text: "hello, world")
print(paragraph!.asHTML())

paragraph = nil
// Prints "p is being deinitialized"
{% endhighlight %}

![Closure Reference Cycle 02](/images/closureReferenceCycle02_2x.png)