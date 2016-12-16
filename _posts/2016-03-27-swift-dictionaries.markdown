---
title: Swift 字典
author: 但江
avatar: danjiang
location: 成都 
category: programming
tag: swift
---

从这篇文章，你将学习到如何使用 Swift 三种集合类型之一的字典，让我们开始吧。

![Swift Dictionaries](/images/swift-dictionaries.jpg)

## 简单说明

字典是由很多个无序的键值对组成，一种键必须唯一，不能重复。

## 创建字典

创建空字典

{% highlight swift %}
var namesOfIntegers = [Int: String]()
var pairs = [String: String]()
{% endhighlight %}

通过字典语义来创建

{% highlight swift %}
var airports = ["YYZ": "Toronto Pearson", "DUB": "Dublin"]
{% endhighlight %}

## 访问字典

{% highlight swift %}
var colors = ["Red": "红色", "Green": "绿色", "Blue": "蓝色"]

// 检查是否为空字典
colors.isEmpty

// 检测有多少个字典元素
colors.count

// 通过键来获取值 
colors["Green"]
{% endhighlight %}

## 操作字典

{% highlight swift %}
var colors = ["Red": "红色", "Green": "绿色", "Blue": "蓝色"]

// 新增键值对
colors["Yellow"] = "黄色"

// 修改值
colors["Red"] = "大红色"

// 删除键值对
colors["Yellow"] = nil
{% endhighlight %}

## 遍历字典

{% highlight swift %}
var colors = ["Red": "红色", "Green": "绿色", "Blue": "蓝色"]

// 遍历所有键值对
for (key, value) in colors {
  print("\(key) - \(value)")
}

// 遍历所有键
for key in colors.keys {
  print(key)
}

// 遍历所有值
for value in colors.values {
  print(value)
}
{% endhighlight %}

## 函数式操作

**reduce** 利用第一个参数做为初始值，通过第二个参数做为计算方法来累计每一次运算的结果。

{% highlight swift %}
var colors = ["Red": "红色", "Green": "绿色", "Blue": "蓝色"]

// 函数式拼接键
let keys = colors.reduce("", { $0 + "\($1.0), "})

// 等价于下面的计算
var allKeys = ""
for key in colors.keys {
  allKeys += "\(key), "
}
{% endhighlight %}

**filter** 利用块中的运算结果做为依据过滤键值对，为真就不会过滤掉，最终将剩下的键值对做为新字典返回。

{% highlight swift %}
var colors = ["Red": "红色", "Green": "绿色", "Blue": "蓝色"]

// 函数式拼接键
let redAndBlue = colors.filter({ $0.1 == "红色" || $0.1 == "蓝色"})

// 等价于下面的计算
var blueAndRed = [(String, String)]()
for (key, value) in colors {
  if value == "红色" || value == "蓝色" {
    blueAndRed.append((key, value))
  }
}
{% endhighlight %}
