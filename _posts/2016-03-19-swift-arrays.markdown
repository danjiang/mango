---
title: Swift 数组
author: 但江
avatar: danjiang
location: 成都 
category: programming
tag: swift
---

从这篇文章，你将学习到如何使用 Swift 三种集合类型之一的数组，让我们开始吧。

![Swift Arrays](/images/swift-arrays.jpg)

## 可变和不可变

有 Objective-C 编程经验的朋友都知道有 NSMutableArray 和 NSArray，一个可变，另一个不可变，Swift 中语法很简化，用 **var** 就是变量，因此可变，可以增删改数组中的元素，用 **let** 就是常量，因此不可变，数组的长度和内容都不能改变。

## 创建数组

创建空数组

{% highlight swift %}
let numbers = [Int]()
let strings = [String]()
{% endhighlight %}

通过默认值来创建

{% highlight swift %}
var threeDoubles = [Double](repeating: 0.0, count: 3)
{% endhighlight %}

通过数组语义来创建

{% highlight swift %}
let evenNumbers = [2, 4, 6, 8]
var shoppingList = ["Eggs", "Milk"]
{% endhighlight %}

## 访问数组元素

{% highlight swift %}
var shoppingList = ["Eggs", "Milk", "Bread"]

// 检查是否为空数组
if shoppingList.isEmpty {
  print("Empty List")
}

// 检测有多少个数组元素
if shoppingList.count < 5 {
  print("Not Enough")
}

// 访问最后一个元素
print(shoppingList.first ?? "")
// 访问第一个元素
print(shoppingList.last ?? "")
// 通过下标来访问第二元素
print(shoppingList[1])
// 通过区间来访问第二个到第三个元素
print(shoppingList[1...2])


var numbers = [3, 1, 8]

// 访问值最小的元素
print(numbers.min() ?? 0)
// 访问值最大的元素
print(numbers.max() ?? 0)
{% endhighlight %}

## 操作数组元素

{% highlight swift %}
var shoppingList = ["Eggs", "Milk"]

// 末尾追加元素
shoppingList.append("Bread")
shoppingList += ["Banana"]

// 插入元素
shoppingList.insert("Avocado", at: 0)

// 删除元素
shoppingList.removeLast()
shoppingList.removeFirst()
shoppingList.remove(at: 1)

// 修改元素
shoppingList[1] = "Pecan"
shoppingList[0...1] = ["Grape", "Pear"]
{% endhighlight %}

## 遍历数组

{% highlight swift %}
var shoppingList = ["Eggs", "Milk", "Bread"]

// 不需要下标的遍历
for item in shoppingList {
  print("I want to buy \(item)")
}

// 需要下标的遍历
for (index, item) in shoppingList.enumerated() {
  print("\(index). \(item)")
}
{% endhighlight %}

## 函数式操作 

**reduce** 利用第一个参数做为初始值，通过第二个参数做为计算方法来累计每一次运算的结果。

{% highlight swift %}
var scores = [2, 3, 5]

// 函数式计算总和
var total = scores.reduce(0, +)

// 等价于下面的计算
var sum = 0
for score in scores {
  sum += score
}
{% endhighlight %}

**filter** 利用块中的运算结果做为依据过滤数组元素，为真就不会过滤掉，最终将剩下的元素做为新数组返回。

{% highlight swift %}
var scores = [2, 7, 10]

// 函数式计算总和
var finals = scores.filter({ $0 >= 5 })

// 等价于下面的计算
var rests = [Int]()
for score in scores {
  if score >= 5 {
    rests.append(score)
  }
}
{% endhighlight %}

**map** 利用块中的运算结果做为新数组中的元素，意思就是元素一一映射成为新数组。

{% highlight swift %}
var scores = [2, 7, 10]

// 函数式计算总和
var newScores = scores.map({ $0 - 1 })

// 等价于下面的计算
var freshScores = [Int]()
for score in scores {
  freshScores.append(score - 1)
}
{% endhighlight %}

废话一句，玩过 Ruby 同学应该很有印象吧，函数式操作数组是不是很简洁。
