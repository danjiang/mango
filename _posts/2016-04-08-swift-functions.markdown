---
title: Swift 函数
author: 但江
avatar: danjiang
location: 成都 
category: programming
tag: swift
---

从这篇文章，你将学习到如何定义和使用 Swift 函数，让我们开始吧。

![Swift Functions](/images/swift-functions.jpg)

## 定义和使用函数

下面定义名称 greeting 的函数，只有一个 String 类型的 name 参数，函数返回值为 String 类型。

{% highlight swift %}
func greeting(name: String) -> String {
  let welcome = "Hello, " + name
  return welcome
}

print(greeting(name: "Jack"))
print(greeting(name: "Lucy"))
{% endhighlight %}

## 函数参数和返回值

### 没有参数的函数

{% highlight swift %}
func greeting() -> String {
  return "Hello World"
}
{% endhighlight %}

### 有多个参数的函数

参数之间用逗号分割。

{% highlight swift %}
func greeting(firtName: String, lastName: String) -> String {
  return "Hello, " + firtName + " " + lastName
}
{% endhighlight %}

### 没有返回值的函数

{% highlight swift %}
func greeting(firtName: String, lastName: String) {
  print("Hello, " + firtName + " " + lastName)
}
{% endhighlight %}

### 有多个返回值的函数

函数通过返回一个元组来返回多个值。

{% highlight swift %}
func greeting(name: String) -> (hello: String, goodbye: String) {
  let hi = "Hello, " + name
  let bye = "Goodbye, " + name
  return (hi, bye)
}
{% endhighlight %}

## 函数参数名称

函数的每一个参数都有外部名称和内部名称，外部名称在调用函数时使用，内部名称在函数内部实现中使用。

### 默认情况

函数参数的外部名称和内部名称一致，如下面的示例。

{% highlight swift %}
func greeting(firtName: String, lastName: String) {
  print("Hello, " + firtName + " " + lastName)
}

greeting(firtName: "Lincoln", lastName: "Park")
{% endhighlight %}

### 指定外部名称

下面示例中 firstName 和 lastName 是外部名称，first 和 last 是内部名称，调用函数时也要写明外部名称。

{% highlight swift %}
func greeting(firtName first: String, lastName last: String) {
  print("Hello, " + first + " " + last)
}

greeting(firtName: "Lincoln", lastName: "Park")
{% endhighlight %}

### 忽略外部名称

用下划线来忽略外部名称。

{% highlight swift %}
func greeting(_ firstName: String, _ lastName: String) {
  print("Hello, " + firstName + " " + lastName)
}

greeting("Lincoln", "Park")
{% endhighlight %}

### 默认参数值

函数参数可以指定默认值，在没有传入参数值时，此参数就使用默认值。

{% highlight swift %}
func greeting(name: String = "Unkown") {
  print("Hello, " + name)
}

greeting(name: "James")
greeting()
{% endhighlight %}

### 可变参数

函数接受零到多个参数值。

{% highlight swift %}
func sum(numbers: Int...) -> Int {
  var total = 0
  for number in numbers {
    total += number
  }
  return total
}

sum(numbers: 2, 3)
sum(numbers: 5, 6, 7)
{% endhighlight %}

### 传入传出参数

{% highlight swift %}
func swap(_ a: inout Int, _ b: inout Int) {
  let temporaryA = a
  a = b
  b = temporaryA
}

var someInt = 3
var anotherInt = 10
swap(&someInt, &anotherInt)
print("someInt is now \(someInt), and anotherInt is now \(anotherInt)")
{% endhighlight %}

## 函数类型

### 基本含义

函数类型由参数类型和返回值类型构成，如下示例的函数类型就是 (String, String) -> String

{% highlight swift %}
func greeting(firtName: String, lastName: String) -> String {
  return "Hello, " + firtName + " " + lastName
}
{% endhighlight %}

### 使用函数类型

既然是类型，因此也可以像其它类型一样，定义变量和常量，还有赋值。

{% highlight swift %}
func hello(name: String) {
  print("Hello, " + name)
}

func goodbye(name: String) {
  print("Goodbye, " + name)
}

var greeting: (String) -> Void = hello

greeting("Jack")

greeting = goodbye

greeting("Jack")
{% endhighlight %}

### 函数类型做为参数

{% highlight swift %}
func hello(name: String) -> String {
  return "Hello, " + name
}

func goodbye(name: String) -> String {
  return "Goodbye, " + name
}

func printGreeting(_ greeting: (String) -> String, name: String) {
  print(greeting(name))
}

printGreeting(hello, name: "Jack")
printGreeting(goodbye, name: "Jack")
{% endhighlight %}

### 函数类型做为返回值

{% highlight swift %}
func hello(name: String) -> String {
  return "Hello, " + name
}

func goodbye(name: String) -> String {
  return "Goodbye, " + name
}

func greeting(leave: Bool) -> (String) -> String {
  if leave {
    return goodbye
  } else {
    return hello
  }
}

var welcome = greeting(leave: true)

print(welcome("Jack"))

welcome = greeting(leave: false)

print(welcome("Jack"))
{% endhighlight %}

## 嵌套函数

上面的示例中定义的函数都是全局函数，也就是任何地方都可以调用，如果想隐藏一些实现的细节，可以使用嵌套函数，当然还可以通过函数返回值将内部函数返回来给外部使用。

{% highlight swift %}
func greeting(leave: Bool) -> (String) -> String {
  func hello(name: String) -> String {
    return "Hello, " + name
  }
  
  func goodbye(name: String) -> String {
    return "Goodbye, " + name
  }
  
  if leave {
    return goodbye
  } else {
    return hello
  }
}

var welcome = greeting(leave: true)

print(welcome("Jack"))

welcome = greeting(leave: false)

print(welcome("Jack"))
{% endhighlight %}
