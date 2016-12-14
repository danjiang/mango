---
title: Swift 泛型
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: swift
---

从这篇文章，你将学习到 Swift 的泛型，泛型可以说是 Swift 最强大的功能，想写出复用度高、又简洁的代码就靠它了，让我们开始吧。

![Swift Generic](/images/swift-generic.jpg)

## 泛型函数

如下面定义的两个函数，前一个用来交换两个整数变量的值，后一个用来交换两个字符串变量的值：

{% highlight swift %}
func swapTwoInts(_ a: inout Int, _ b: inout Int) {
  let temporaryA = a
  a = b
  b = temporaryA
}

func swapTwoStrings(_ a: inout String, _ b: inout String) {
  let temporaryA = a
  a = b
  b = temporaryA
}
{% endhighlight %}

上面两个函数的内部实现一模一样，唯一不同的就是所交换变量类型，一个是整数，一个是字符串，我可以通过下面的 **泛型函数** 来解决这个问题：

{% highlight swift %}
func swapTwoValues<T>(_ a: inout T, _ b: inout T) {
  let temporaryA = a
  a = b
  b = temporaryA
}
{% endhighlight %}

上面的 **泛型函数** 中我们用 **T** 来做为实际类型名称的占位符，这样就可以传入任何类型并且保证参数 **a** 和 **b** 是相同类型。

**T** 就是 **类型参数** 的一个示例，在函数名称后面由 **<>** 包围起来，一旦定义了 **类型参数**，不仅可以用在函数参数上，还可以用在函数返回类型，**类型参数** 在函数被调用时就会被实际类型所替换，**<>** 可以定义多个 **类型参数**，由逗号分隔。

通常情况而言，**类型参数** 的名称应该有一定的描述意义，如 **Array\<Element\>** 描述了 **泛型类型** 和 **类型参数** 之间的关系，如果找不到什么关系，就用单字母 **T, U, V**，要遵循首字母大写的驼峰命名法。

## 泛型类型

除了 **泛型函数**，还可以定义 **泛型类型**，类似于 **Array** 和 **Dictionary**，下面通过定义一个 **Stack** 来作为示例，队列是一种先进先出的数据结构，就像排队买东西一样。

{% highlight swift %}
struct Stack<Element> {
  
  var items = [Element]()
  
  mutating func push(_ item: Element) {
    items.append(item)
  }
  
  mutating func pop() -> Element {
    return items.removeLast()
  }
  
}
{% endhighlight %}

## 扩展泛型类型

在扩展时，不需要再提供 **类型参数**，可以使用原类型中的 **类型参数**。

{% highlight swift %}
extension Stack {
  
  var topItem: Element? {
    return items.isEmpty ? nil : items[items.count - 1]
  }
  
}
{% endhighlight %}

## 类型约束

很多时候，需要给 **类型参数** 加上约束，比如 **Dictionary** 中 **Key** 必须要能被哈希，字典数据结构才能正常地插入、删除和替换值，下面是 **Dictionary** 在文档中的声明：

{% highlight swift %}
struct Dictionary<Key: Hashable, Value>
{% endhighlight %}

我们定义一个在数组中查找某个值的下标的函数，但是现在还不能编译通过：

{% highlight swift %}
func findIndex<T>(of valueToFind: T, in array: [T]) -> Int? {
  for (index, value) in array.enumerated() {
    if value == valueToFind {
      return index
    }
  }
  return nil
}
{% endhighlight %}

不能编译通过的原因是 Swift 中不是任何类型都可以通过 **==** 比较，能够比较的类型需要满足 **Equatable** 协议，所以我们需要在 **T** 后面加上约束 **Equatable**：

{% highlight swift %}
func findIndex<T: Equatable>(of valueToFind: T, in array: [T]) -> Int? {
  for (index, value) in array.enumerated() {
    if value == valueToFind {
      return index
    }
  }
  return nil
}
{% endhighlight %}

## 关联类型

在定义协议时，可以定义 **关联类型**，用来占位一个类型，满足这个协议的类型需要指明这个占位类型是什么，下面示例定义了 **Container** 协议，其中由 **associatedtype** 指明了 **ItemType** 为 **关联类型**，**Stack** 声明满足 **Container** 协议，其中由 **typealias** 指明了 **ItemType** 为 **Element**：

{% highlight swift %}
protocol Container {
  associatedtype ItemType
  mutating func append(_ item: ItemType)
  var count: Int { get }
  subscript(i: Int) -> ItemType { get }
}

struct Stack<Element>: Container {
  
  var items = [Element]()
  
  mutating func push(_ item: Element) {
    items.append(item)
  }
  
  mutating func pop() -> Element {
    return items.removeLast()
  }
  
  // conformance to the Container protocol
  
  typealias ItemType = Element
  
  mutating func append(_ item: Element) {
    push(item)
  }
  
  var count: Int {
    return items.count
  }
  
  subscript(i: Int) -> Element {
    return items[i]
  }
  
}
{% endhighlight %}

## 泛型 Where 子句

类似于 **类型约束**，我们可以通过 **where** 来给 **关联类型** 添加约束，在类型或函数开始花括号 **{** 前面，如下面的示例约束了两个 **Container** 的 **关联类型** 必须是相同类型且满足 **Equatable** 协议：

{% highlight swift %}
func allItemsMatch<C1: Container, C2: Container>
  (_ someContainer: C1, _ anotherContainer: C2) -> Bool
  where C1.ItemType == C2.ItemType, C1.ItemType: Equatable {
  if someContainer.count != anotherContainer.count {
    return false
  }
  for i in 0..<someContainer.count {
    if someContainer[i] != anotherContainer[i] {
      return false
    }
  }
  return true
}
{% endhighlight %}
