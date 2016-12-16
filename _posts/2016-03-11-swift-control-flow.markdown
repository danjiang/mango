---
title: Swift 流程控制
author: 但江
avatar: danjiang
location: 成都 
category: programming
tag: swift
---

从这篇文章，你将学习到如何使用 Swift 流程控制，当然有常见的 **for** 循环和 **if** 条件，还有 **continue** 和 **break** 等流程控制语句，最后还会讲到 **guard** 和 **\#available**，让我们开始吧。

![Swift Control Flow](/images/swift-control-flow.jpg)

## 循环

### For-In

**for-in** 可以用来遍历序列，比如数字区间，数组，还有字典。

{% highlight swift %}
for index in 1...3 {
  print("\(index) times 3 is \(index * 3)")
}

let brands = ["Nike", "New Balance", "Under Armour", "Adidas"];
for brand in brands {
  print("Do you like \(brand)?")
}

let numberOfLegs = ["spider": 8, "ant": 6, "cat": 4]
for (animalName, legCount) in numberOfLegs {
  print("\(animalName)s have \(legCount) legs")
}
{% endhighlight %}

### While

判断条件的循环。

{% highlight swift %}
var n = 2
while n < 100 {
  n = n * 2
}
print(n)
{% endhighlight %}

### Repeat-While

先循环，后判断条件。

{% highlight swift %}
var m = 2
repeat {
  m = m * 2
} while m < 100
print(m)
{% endhighlight %}

## 条件

### If

熟悉的条件判断。

{% highlight swift %}
let babyGender = "boy"
if babyGender == "boy" {
  print("是个男孩")
} else {
  print("是个女孩")
}
{% endhighlight %}

### Switch

**switch** 中每一个 **case** 中的语句运行完后不会再运行下一个 **case** 中的语句，还可以像下例中用 **where** 作进一步判断。

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

还可以用于判断值是否在一个区间。

{% highlight swift %}
let approximateCount = 62
var naturalCount: String
switch approximateCount {
case 0:
  naturalCount = "no"
case 1..<5:
  naturalCount = "a few"
case 5..<12:
  naturalCount = "several"
case 12..<100:
  naturalCount = "dozens of"
case 100..<1000:
  naturalCount = "hundreds of"
default:
  naturalCount = "many"
}
{% endhighlight %}

还有一种值绑定的用法，看下面这个例子就明白了。

{% highlight swift %}
let anotherPoint = (2, 0)
switch anotherPoint {
case (let x, 0):
  print("on the x-axis with an x value of \(x)")
case (0, let y):
  print("on the y-axis with a y value of \(y)")
case let (x, y):
  print("somewhere else at (\(x), \(y))")
}
{% endhighlight %}

## 控制转移语句

### Continue

**continue** 用于循环中，跳过本次循环执行，跳到下一次循环开始。

{% highlight swift %}
let puzzleInput = "great minds think alike"
var puzzleOutput = ""
for character in puzzleInput.characters {
  switch character {
  case "a", "e", "i", "o", "u", " ":
      continue
  default:
    puzzleOutput.append(character)
  }
}
print(puzzleOutput)
{% endhighlight %}

### Break

**break** 用于结束整个流程控制语句，可以用于 **switch** 中，也可以用于循环中。

{% highlight swift %}
let numberSymbol: Character = "三"
var possibleIntegerValue: Int?
switch numberSymbol {
case "1", "一":
  possibleIntegerValue = 1
case "2", "二":
  possibleIntegerValue = 2
case "3", "三":
  possibleIntegerValue = 3
default:
  break
}
{% endhighlight %}

## Guard

在使用 **guard** 时，要保证条件判断必须为真来继续执行后面的代码，在 **guard** 中使用可选类型绑定来赋值的变量或常量，可以在 **guard** 后面继续使用（不是 **guard** 后面的块，而是 **guard** 同级后面的代码），**guard** 总是带有 **else** 段，在 **else** 段通常会使用 **return**、**break**、**continue**、**throw** 和 **fatalError()** 来进行控制转移语句结束后面代码的运行。

{% highlight swift %}
func greet(person: [String: String]) {
  guard let name = person["name"] else {
    return
  }
  
  print("Hello \(name)!")
  
  guard let location = person["location"] else {
    print("I hope the weather is nice near you.")
    return
  }
  
  print("I hope the weather is nice in \(location).")
}

greet(person: ["name": "John"])
// Hello John!
// I hope the weather is nice near you.

greet(person: ["name": "Jane", "location": "Cupertino"])
// Hello Jane!
// I hope the weather is nice in Cupertino.
{% endhighlight %}

## API 可用性检测

**\#available** 是内置检测 API 可用性的方法，当然是针对 deployment target，也就是你配置的应用可以执行的最低系统版本。

{% highlight swift %}
if #available(iOS 9, OSX 10.0, *) {
  // Use iOS 9 APIs on iOS, and use OS X v10.10 APIs on OS X
} else {
  // Fall back to earlier iOS and OS X APIs
}
{% endhighlight %}
