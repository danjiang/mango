---
title: 开始学习 Swift 
author: 但江
avatar: danjiang
location: 成都 
category: programming
tag: swift
---

苹果公司在 2014 年 6 月 2 日发布了 Swift 编程语言，你可以使用 Swift 编写在 OS X、iOS、watchOS 和 tvOS 上运行的应用，2015 年 12 月 4 日，苹果公司宣布 Swift 编程语言开放源代码，让 Swift 有更广阔的想象空间。废话少说，一起跟着这篇文章敲代码来开始学习 Swift 吧。

![Swift Opensource]({{ site.image_base_url }}/swift-opensource.jpg)

## Playground

![Get Started with a Playground]({{ site.image_base_url }}/get-started-with-a-playground.jpg)

![New Playground]({{ site.image_base_url }}/new-playground.jpg)

![Tour Playground]({{ site.image_base_url }}/tour-playground.jpg)

## 基础值

{% highlight swift %}
var myVariable = 42 // 一个变量
myVariable = 50 // 所以可以改变它的值
let myConstant = 42 // 一个常量，所以不能改变它的值
{% endhighlight %}

变量和常量都需要明确类型

{% highlight swift %}
let implicitInteger = 70 // 编译器从初始值知道 implicitInteger 是整数类型
let implicitDouble = 70.0 // 编译器从初始值知道 implicitDouble 是浮点类型
let explicitDouble: Double = 70 // 编译器不能从初始值知道 explicitDouble 是浮点类型，所以显示地声明类型
{% endhighlight %}

值不能自动地转换为其他类型，需要自己来转换

{% highlight swift %}
let label = "The width is "
let width = 94
let widthLabel = label + String(width)
{% endhighlight %}

采用 **\()** 将不同类型值拼接成字符串很方便

{% highlight swift %}
let apples = 3
let oranges = 5
let appleSummary = "I have \(apples) apples."
let fruitSummary = "I have \(apples + oranges) pieces of fruit."
{% endhighlight %}

创建数组和词典

{% highlight swift %}
var shoppingList = ["catfish", "water", "tulips", "blue paint"]
shoppingList[1] = "bottle of water"

var occupations = [
  "Malcolm": "Captain",
  "Kaylee": "Mechanic"]
occupations["Jayne"] = "Public Relations"

let emptyArray = [String]()
let emptyDictionary = [String: Float]()
{% endhighlight %}

## 流程控制

**if** 和 **switch** 作为条件控制，**for-in**、**for**、**while** 和 **repeat-while** 作为循环控制

{% highlight swift %}
let individualScores = [75, 43, 103, 87, 12]
var teamScore = 0
for score in individualScores {
  if score > 50 {
    teamScore += 3
  } else {
    teamScore += 1
  }
}
print(teamScore)
{% endhighlight %}

可选值要么有值、要么没有值

{% highlight swift %}
var optionalString: String? = "Hello" // ? 表明可选
print(optionalString == nil)

var optionalName: String? = "John Appleseed"
var greeting = "Hello!"
if let name = optionalName { // optionalName 有值才会运行 {} 中的内容
  greeting = "Hello, \(name)"
}
{% endhighlight %}

**??** 可以为可选值提供默认值

{% highlight swift %}
let nickName: String? = nil
let fullName: String = "John Appleseed"
let informalGreeting = "Hi \(nickName ?? fullName)"
{% endhighlight %}

注意 **switch** 中每一个 **case** 中的语句运行完后不会再运行下一个 **case** 中的语句

{% highlight swift %}
let vegetable = "red pepper"
switch vegetable {
case "celery":
  print("Add some raisins and make ants on a log.")
case "cucumber", "watercress":
  print("That would make a good tea sandwich.")
case let x where x.hasSuffix("pepper"):
  print("Is it a spicy \(x)?")
default:
  print("Everything tastes good in soup.")
}
{% endhighlight %}

**for in** 遍历数组和词典很方便

{% highlight swift %}
let interestingNumbers = [
  "Prime": [2, 3, 5, 7, 11, 13],
  "Fibonacci": [1, 1, 2, 3, 5, 8],
  "Square": [1, 4, 9, 16, 25]
]
var largest = 0
for (kind, numbers) in interestingNumbers {
  for number in numbers {
    if number > largest {
      largest = number
    }
  }
}
print(largest)
{% endhighlight %}

**while** 和 **repeat-while**

{% highlight swift %}
var n = 2
while n < 100 {
  n = n * 2
}
print(n)

var m = 2
repeat {
  m = m * 2
} while m < 100
print(m)
{% endhighlight %}

**for**，注意 **..<** 不包含最大值，**...** 包含最大值

{% highlight swift %}
var firstForLoop = 0
for i in 0..<4 {
  firstForLoop += i
}
print(firstForLoop)

var secondForLoop = 0
for var i = 0; i < 4; ++i {
  secondForLoop += i
}
print(secondForLoop)
{% endhighlight %}
