---
title: Swift 下标
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: swift
---

从这篇文章，你将学习到如何使用 Swift 的下标功能，可以给自己定义的类型添加类似 **Array** 和 **Dictionary** 的能力。

![Swift Subscripts](/images/swift-subscripts.jpg)

## 语法

通过 **subscript** 关键词来定义，然后就可以在所定义类型的实例上通过 **[]** 来调用。

{% highlight swift %}
subscript(index: Int) -> Int {
  get {

  }
  set(newInt) {

  }
}
{% endhighlight %}

**newValue** 的类型和 **subscript** 返回值的类型一样，注意，**set** 中参数名可以不指定，如果不指定，就有一个默认的叫 **newValue** 的参数，类似于计算属性。

## 使用

可以参照内置的 **Array** 和 **Dictionary** 使用方法，下标定义可以只读，如下面的示例。

{% highlight swift %}
struct TimesTable {
  let multiplier: Int
  subscript(index: Int) -> Int {
    return multiplier * index
  }
}

let threeTimesTable = TimesTable(multiplier: 3)
print("six times three is \(threeTimesTable[6])")
{% endhighlight %}

## 深入

下标定义有多个输入参数，这些参数也可以是任意类型，返回值也可以是任意类型，一个类型中可以定义多个下标。一个输入参数是最为常见的，看一下下面这个关于 **Matrix** 的示例，有两个输入参数。

{% highlight swift %}
struct Matrix {
  let rows: Int, columns: Int
  var grid: [Double]

  init(rows: Int, columns: Int) {
    self.rows = rows
    self.columns = columns
    grid = Array(count: rows * columns, repeatedValue: 0.0)
  }

  func indexIsValid(row: Int, column: Int) -> Bool {
    return row >= 0 && row < rows && column >= 0 && column < columns
  }

  subscript(row: Int, column: Int) -> Double {
    get {
      assert(indexIsValid(row, column: column), "Index out of range")
      return grid[(row * columns) + column]
    }
    set {
      assert(indexIsValid(row, column: column), "Index out of range")
      grid[(row * columns) + column] = newValue
    }
  }
}

var matrix = Matrix(rows: 2, columns: 2)
matrix[0, 1] = 1.5
matrix[1, 0] = 3.2
{% endhighlight %}
