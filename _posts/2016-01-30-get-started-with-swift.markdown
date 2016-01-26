---
title: 开始学习 Swift 
author: 但江
avatar: danjiang
location: 成都 
category: programming
tag: swift
---

苹果公司在 2014 年 6 月 2 日发布了 Swift 编程语言，你可以使用 Swift 编写在 OS X、iOS、watchOS 和 tvOS 上运行的应用，2015 年 12 月 4 日，苹果公司宣布 Swift 编程语言开放源代码，让 Swift 有更广阔的想象空间。

请注意，示例代码都来自 **The Swift Programming Language - A Swift Tour**，如果你有更多疑问可以参考这本苹果官方教程，我都是自己敲的代码，不是复制粘贴的，所以你最好也一起跟着这篇文章敲代码来开始学 Swift，所谓 Muscle Memory。

![Swift Opensource]({{ site.image_base_url }}/swift-opensource.jpg)

## Playground

目前来看，大家学习 Swift 主要是为了开发苹果平台的应用，你应该有一台 Mac，Mac 上应该装了 Xcode，打开 Xcode，在欢迎界面点击 **Get started with a plaground**。

![Get Started with a Playground]({{ site.image_base_url }}/get-started-with-a-playground.jpg)

在下面的界面中，输入你新建的 Playground 的 **名称** 和 **平台**。

![New Playground]({{ site.image_base_url }}/new-playground.jpg)

从上一步点击 **Next** 过后，你就会看到 Playground 的主界面了，在这里敲代码学习 Swift，可以不用操心很多你不熟悉的平台 SDK，学习语法，打好基础。

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

## 函数和闭包

通过 **func** 来定义函数，**()** 里面的是参数列表，**->** 是返回类型

{% highlight swift %}
func greet(name: String, day: String) -> String {
  return "Hello \(name), today is \(day)."
}
greet("Bob", day: "Tuesday")
{% endhighlight %}

函数可以通过元组 tuple 返回多个值

{% highlight swift %}
func calculateStatistics(scores: [Int]) -> (min: Int, max: Int, sum: Int) {
  var min = scores[0]
  var max = scores[1]
  var sum = 0
  
  for score in scores {
    if score > max {
      max = score
    } else if score < min {
      min = score
    }
    sum += score
  }
  
  return (min, max, sum)
}
let statistics = calculateStatistics([5, 3, 100, 3, 9])
print(statistics.sum)
print(statistics.2)
{% endhighlight %}

函数可以将任意数量参数收集为数组

{% highlight swift %}
func sumOf(numbers: Int...) -> Int {
  var sum = 0
  for number in numbers {
    sum += number
  }
  return sum
}
sumOf()
sumOf(11, 22, 33)
{% endhighlight %}

函数可以嵌套，里面的函数可以访问外面函数中声明的变量

{% highlight swift %}
func returnFifteen() -> Int {
  var y = 10
  func add() {
    y += 5
  }
  add()
  return y
}
returnFifteen()
{% endhighlight %}

函数作为变量值

{% highlight swift %}
func makeIncrementer() -> ((Int) -> Int) {
  func addOne(number: Int) -> Int {
    return 1 + number
  }
  return addOne
}
var increment = makeIncrementer()
increment(7)
{% endhighlight %}

函数作为参数

{% highlight swift %}
func hasAnyMatches(list: [Int], condition: (Int) -> Bool) -> Bool {
  for item in list {
    if condition(item) {
      return true
    }
  }
  return false
}
func lessThanTen(number: Int) -> Bool {
  return number < 10
}
var numbers = [20, 19, 7, 12]
hasAnyMatches(numbers, condition: lessThanTen)
{% endhighlight %}

闭包就是一段可以稍后再执行的代码，闭包可以继续访问在其被创建时的外部变量，函数这些

{% highlight swift %}
numbers.map({
  (number: Int) -> Int in
  let result = 3 * number
  return result
})
{% endhighlight %}

更简洁的方式

{% highlight swift %}
let mappedNumbers = numbers.map({ number in 3 * number })
print(mappedNumbers)
{% endhighlight %}

超简洁的方式

{% highlight swift %}
let sortedNumbers = numbers.sort({ $0 > $1 })
print(sortedNumbers)
{% endhighlight %}

## 对象和类

通过 **class** 定义一个类来瞧瞧

{% highlight swift %}
class Shape {
  var numberOfSides = 0
  func simpleDescription() -> String {
    return "A shape with \(numberOfSides) sides."
  }
}
{% endhighlight %}

定义一个 Shape 对象来瞧瞧

{% highlight swift %}
var shape = Shape()
shape.numberOfSides = 7
var shapeDescription = shape.simpleDescription()
{% endhighlight %}

加上初始化方法 **init** 来瞧瞧，所有的属性都需要初始化，可以像 numberOfSides 通过初始值，还是像 name 通过 init 方法，这里为了和参数 name 区分，所以用了 **self** 表明属性 name

{% highlight swift %}
class NamedShape {
  var numberOfSides: Int = 0
  var name: String
  
  init(name: String) {
    self.name = name
  }
  
  func simpleDescription() -> String {
    return "A shape with \(numberOfSides) sides."
  }
}
{% endhighlight %}

定义一个子类，注意重写了父类的方法需要加上 **override**，通过 **super** 可以调用父类的方法

{% highlight swift %}
class Square: NamedShape {
  var sideLength: Double
  
  init(sideLength: Double, name: String) {
    self.sideLength = sideLength
    super.init(name: name)
    numberOfSides = 4
  }
  
  func area() -> Double {
    return sideLength * sideLength
  }
  
  override func simpleDescription() -> String {
    return "A square with sides of length \(sideLength)."
  }
}
let test = Square(sideLength: 5.2, name: "my test square")
test.area()
test.simpleDescription()
{% endhighlight %}

一个有 **getter** 和 **setter** 的子类

{% highlight swift %}
class EquilateralTriangle: NamedShape {
  var sideLength: Double = 0.0
  
  init(sideLength: Double, name: String) {
    self.sideLength = sideLength
    super.init(name: name)
    numberOfSides = 3
  }
  
  var perimeter: Double {
    get {
      return 3.0 * sideLength
    }
    set {
      sideLength = newValue / 3.0
    }
  }
  
  override func simpleDescription() -> String {
    return "An equilateral triangle with sides of length \(sideLength)."
  }
}
var triangle = EquilateralTriangle(sideLength: 3.1, name: "a triangle")
print(triangle.perimeter)
triangle.perimeter = 9.9
print(triangle.sideLength)
{% endhighlight %}

还可以通过 **willSet** 和 **didSet** 在属性值改变的前或后来执行一些代码，类似于监听属性值变化

{% highlight swift %}
class TriangleAndSquare {
  var triangle: EquilateralTriangle {
    willSet {
      square.sideLength = newValue.sideLength
    }
  }
  
  var square: Square {
    willSet {
      triangle.sideLength = newValue.sideLength
    }
  }
  
  init(size: Double, name: String) {
    square = Square(sideLength: size, name: name)
    triangle = EquilateralTriangle(sideLength: size, name: name)
  }
}
var triangleAndSquare = TriangleAndSquare(size: 10, name: "another test shape")
print(triangleAndSquare.square.sideLength)
print(triangleAndSquare.triangle.sideLength)
triangleAndSquare.square = Square(sideLength: 50, name: "larger square")
print(triangleAndSquare.triangle.sideLength)
{% endhighlight %}

## 枚举和结构体

用 **enum** 定义一个枚举来瞧瞧，可以有方法的哦，还有一点需要注意，可以指明枚举的 Raw Value 的类型，下例中就是 Int 类型

{% highlight swift %}
enum Rank: Int {
  case Ace = 1
  case Two, Three, Four, Five, Six, Seven, Eight, Nine, Ten
  case Jack, Queen, King
  func simpleDescription() -> String {
    switch self {
    case .Ace:
      return "ace"
    case .Jack:
      return "jack"
    case .Queen:
      return "queen"
    case .King:
      return "king"
    default:
      return String(self.rawValue)
    }
  }
}
let ace = Rank.Ace
let aceRawValue = ace.rawValue
{% endhighlight %}

通过 Raw Value 来创建枚举

{% highlight swift %}
if let covertedRank = Rank(rawValue: 3) {
  let threeDescription = covertedRank.simpleDescription()
}
{% endhighlight %}

定义枚举时 Raw Value 的类型不是必须的，没有也行

{% highlight swift %}
enum Suit {
  case Spades, Hearts, Diamonds, Clubs
  func simpleDescription() -> String {
    switch self {
    case .Spades:
      return "spades"
    case .Hearts:
      return "hearts"
    case .Diamonds:
      return "diamonds"
    case .Clubs:
      return "clubs"
    }
  }
}
let hearts = Suit.Hearts
let heartsDescription = hearts.simpleDescription()
{% endhighlight %}

用 **struts** 定义结构体，结构体和类很像，记住最大的一点不同是，在传递时，结构体是整体拷贝一份，也就是值传递，而类是引用传递

{% highlight swift %}
struct Card {
  var rank: Rank
  var suit: Suit
  func simpleDescription() -> String {
    return "The \(rank.simpleDescription()) of \(suit.simpleDescription())"
  }
}
let threeOfSpades = Card(rank: .Three, suit: .Spades)
let threeOfSpadeDescription = threeOfSpades.simpleDescription()
{% endhighlight %}

枚举的实例可以有关联值 Associated Value，要区别于 Raw Value，针对一个枚举 case，Raw Value 都是一样的，Associated Value 可以不同

{% highlight swift %}
enum ServerResponse {
  case Result(String, String)
  case Error(String)
}
let success = ServerResponse.Result("6:00 am", "8:09 pm")
let failure = ServerResponse.Error("Out of cheese.")

switch success {
case let .Result(sunrise, sunset):
  print("Sunrise is at \(sunrise) and sunset is at\(sunset).")
case let .Error(error):
  print("Failure... \(error)")
}
{% endhighlight %}

## 协议和扩展

使用 **protocol** 来定义协议
 
{% highlight swift %}
protocol ExampleProtocol {
  var simpleDescription: String { get }
  mutating func adjust()
}
{% endhighlight %}

类，枚举和结构体都可以遵循协议

{% highlight swift %}
class SimpleClass: ExampleProtocol {
  var simpleDescription: String = "A very simple class."
  var anotherProperty: Int = 68105
  func adjust() {
    simpleDescription += " Now 100% adjusted."
  }
}
var a = SimpleClass()
a.adjust()
let aDescription = a.simpleDescription

struct SimpleStructure: ExampleProtocol {
  var simpleDescription: String = "A simple structure"
  mutating func adjust() {
    simpleDescription += " (adjusted)"
  }
}
var b = SimpleStructure()
b.adjust()
let bDescription = b.simpleDescription
{% endhighlight %}

使用 **extension** 可以给已有的类型添加功能，加一些方法，运算属性等等

{% highlight swift %}
extension Int: ExampleProtocol {
  var simpleDescription: String {
    return "The number \(self)"
  }
  mutating func adjust() {
    self += 42
  }
}
print(7.simpleDescription)
{% endhighlight %}

## 泛型

在 **<>** 中定义类型的名称来创建泛型方法或类型

{% highlight swift %}
func repeatItem<Item>(item: Item, numberOfTimes: Int) -> [Item] {
  var result = [Item]()
  for _ in 0..<numberOfTimes {
    result.append(item)
  }
  return result
}
repeatItem("knock", numberOfTimes: 4)
{% endhighlight %}

泛型还可以用在定义类，枚举和结构体中

{% highlight swift %}
enum OptionalValue<Wrapped> {
  case None
  case Some(Wrapped)
}
var possibleInteger: OptionalValue<Int> = .None
possibleInteger = .Some(100)
{% endhighlight %}

使用 **where** 可以给泛型限制一些条件，如必须实现某些协议，或者必须有特定的父类等

{% highlight swift %}
func anyCommonElements <T: SequenceType, U: SequenceType
  where T.Generator.Element: Equatable,
  T.Generator.Element == U.Generator.Element>
  (lhs: T, _ rhs: U) -> Bool {
    for lhsItem in lhs {
      for rhsItem in rhs {
        if lhsItem == rhsItem {
          return true
        }
      }
    }
    return false
}
anyCommonElements([1, 2, 3], [3])
{% endhighlight %}

## 一点感受

如果你有其他编程语言的开发经验，或多或少你都可以在 Swift 中看到它们的影子，map 这些函数在 Ruby 中是不是也有啊，所谓的函数式编程，protocol 这种面向协议编程方式，像不像早年 Java 中强调的面向接口编程，看来 Swift 是取众家所长。
