---
title: Swift 闭包
author: 但江
avatar: danjiang
location: 成都 
category: programming
tag: swift
---

从这篇文章，你将学习到如何使用 Swift 的闭包，和函数有着千丝万缕的关系，让我们开始吧。

![Swift Closures](/images/swift-closures.jpg)

## 三种定义方式

1. 全局函数就是有名字的闭包，但是不捕获任何值，不常用。
2. 嵌套函数也是有名字的闭包，从外围函数捕获值，较常用。
3. 闭包表达式是没有名字的闭包，从外围环境捕获值，很常用，主要讲解。

## 闭包表达式

闭包表达式写法很像函数定义，下面 **greeting** 函数的第二个参数 **closure** 就是一个闭包表达式，**(_ name: String)** 是闭包的参数列表，**-> String** 是闭包的返回值。

{% highlight swift %}
func greeting(name: String, closure: (_ name: String) -> String) {
  print(closure(name))
}
{% endhighlight %}

普通调用方式

{% highlight swift %}
greeting(name: "Jack", closure: { (name: String) -> String
  in
  return "Hello, " + name
})
{% endhighlight %}

因为 **closure** 是函数 **greeting** 的最后一个参数，所以还可以采用下面更简洁的调用方式，推荐这样做。

{% highlight swift %}
greeting(name: "Mick") { (name) -> String in
  return "Goodbye, " + name
}
{% endhighlight %}

## 捕获值

闭包可以从外围环境中捕获常量和变量，可以在闭包内部使用这些常量和变量，即使定义这些常量和变量的作用域已经不存在。

{% highlight swift %}
func makeIncrementer(forIncrement amount: Int) -> () -> Int {
    var runningTotal = 0
    func incrementer() -> Int {
        runningTotal += amount
        return runningTotal
    }
    return incrementer
}
{% endhighlight %}

调用 **makeIncrementer** 创建闭包 **incrementByTen** 后，再调用 **incrementByTen** 仍然可以使用 **runningTotal** 和 **amount**，再次调用 **makeIncrementer** 创建闭包 **incrementBySeven**，其所引用的 **runningTotal** 和 **amount** 与 **incrementByTen** 没有联系，互不干扰。

{% highlight swift %}
let incrementByTen = makeIncrementer(forIncrement: 10)
print(incrementByTen()) // 10
print(incrementByTen()) // 20

let incrementBySeven = makeIncrementer(forIncrement: 7)
print(incrementBySeven()) // 7
print(incrementByTen()) // 30
{% endhighlight %}

## 闭包是引用类型

从赋值就可以看出来，下面示例紧接着上面的示例。

{% highlight swift %}
let alsoIncrementByTen = incrementByTen
print(alsoIncrementByTen()) // 40
{% endhighlight %}