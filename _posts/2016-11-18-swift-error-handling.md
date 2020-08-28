---
title: Swift 错误处理
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: swift
---

从这篇文章，你将学习到 Swift 的错误处理，如何抛出、捕获和传播错误，让我们开始吧。

![Swift Error Handling](/images/swift-error-handling.jpg)

## 表示和抛出错误

任何满足 **Error** 协议的值类型都可以表示错误，Swift 枚举通过关联值很适合表示一组相关的错误：

{% highlight swift %}
enum VendingMachineError: Error {
  case invalidSelection
  case insufficientFunds(coinsNeeded: Int)
  case outOfStock
}
{% endhighlight %}

使用 **throw** 来抛出错误：

{% highlight swift %}
throw VendingMachineError.insufficientFunds(coinsNeeded: 5)
{% endhighlight %}

## 处理错误

先说一下，表明一个函数、方法或初始化器可以抛出错误，在定义时要使用 **throws** 关键词：

{% highlight swift %}
func canThrowErrors() throws -> String
{% endhighlight %}

下面通过自动售货机售货时会抛出几种错误的示例，分别说明下处理错误的 4 种方法：

{% highlight swift %}
struct Item {
  var price: Int
  var count: Int
}

class VendingMachine {
  var inventory = [
    "Candy Bar": Item(price: 12, count: 7),
    "Chips": Item(price: 10, count: 4),
    "Pretzels": Item(price: 7, count: 11)
  ]
  var coinsDeposited = 0
  
  func vend(itemNamed name: String) throws {
    guard let item = inventory[name] else {
      throw VendingMachineError.invalidSelection
    }
    
    guard item.count > 0 else {
      throw VendingMachineError.outOfStock
    }
    
    guard item.price <= coinsDeposited else {
      throw VendingMachineError.insufficientFunds(coinsNeeded: item.price - coinsDeposited)
    }
    
    coinsDeposited -= item.price
    
    var newItem = item
    newItem.count -= 1
    inventory[name] = newItem
    
    print("Dispensing \(name)")
  }
}
{% endhighlight %}

### 传播错误

{% highlight swift %}
let favoriteSnacks = [
  "Alice": "Chips",
  "Bob": "Licorice",
  "Eve": "Pretzels"
]

func buyFavoriteSnack(person: String, vendingMachine: VendingMachine) throws {
  let snackName = favoriteSnacks[person] ?? "Candy Bar"
  try vendingMachine.vend(itemNamed: snackName)
}
{% endhighlight %}

### 捕获错误

{% highlight swift %}
var vendingMachine = VendingMachine()
vendingMachine.coinsDeposited = 8

do {
  try buyFavoriteSnack(person: "Alice", vendingMachine: vendingMachine)
} catch VendingMachineError.invalidSelection {
  print("Invalid Selection.")
} catch VendingMachineError.outOfStock {
  print("Out of Stock.")
} catch VendingMachineError.insufficientFunds(let coinsNeeded) {
  print("Insufficient funds. Please insert an additional \(coinsNeeded) coins.")
}
{% endhighlight %}

### 转换错误为可选类型值

{% highlight swift %}
func someThrowingFunction() throws -> Int {
}

let x = try? someThrowingFunction()
{% endhighlight %}

### 禁止传播错误

{% highlight swift %}
let photo = try! loadImage(atPath: "./Resources/John Appleseed.jpg")
{% endhighlight %}

## 指定一些清理操作

**defer** 可以表明一些在离开当前代码块必须执行的代码：

{% highlight swift %}
func processFile(filename: String) throws {
  if exists(filename) {
    let file = open(filename)
    defer {
      close(file)
    }
    while let line = try file.readline() {
      // 操作文件内容
    }
    // close(file) 在这里被调用
  }
}
{% endhighlight %}

> Deferred actions are executed in the reverse of the order that they’re written in your source code. That is, the code in the first defer statement executes last, the code in the second defer statement executes second to last, and so on. The last defer statement in source code order executes first.
>
> —- Quote from [The Swift Programming Language](https://docs.swift.org/swift-book/)