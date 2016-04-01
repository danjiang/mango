---
title: Swift 集合
author: 但江
avatar: danjiang
location: 成都 
category: programming
tag: swift
---

从这篇文章，你将学习到如何使用 Swift 三种集合类型之一的集合，让我们开始吧。

![Swift Sets](/images/swift-sets.jpg)

## 简单说明

集合是一组无序的值，并且是同类型和没有重复的，如果你要保证一组值没有重复，但是不关心顺序，那你应该使用集合。

## 针对集合的哈希值 

集合中存储的类型一定是可以哈希的，此类型要有提供哈希值的运算方法，集合通过提供的哈希值来比较集合中的值，以判断没有重复的情况发生，自定义的类型要符合 **Hashable** 协议，需要提供名为 **hashValue** 的 **Int** 属性，因为 **Hashable** 符合了 **Equatable** 协议，所以还要提供 **==** 运算方法。

## 创建集合

创建空集合

{% highlight swift %}
let genres = Set<String>()
let types = Set<Int>()
{% endhighlight %}

通过集合语义来创建

{% highlight swift %}
var genres: Set<String> = ["Rock", "Pop", "Jazz"]
{% endhighlight %}

## 访问集合

{% highlight swift %}
var genres: Set<String> = ["Rock", "Pop", "Jazz"]

// 检测有多少个值
print("I have \(genres.count) favorite music genres.")

// 检测是否为空集合
if genres.isEmpty {
  print("As far as music goes, I`m not picky.")
} else {
  print("I have particular music preferences.")
}

// 检测集合是否包含某值
if genres.contains("Funk") {
  print("I get up on the good foot.")
} else {
  print("It`s too funky in here.")
}
{% endhighlight %}

## 操作集合

{% highlight swift %}
var genres: Set<String> = ["Rock", "Pop", "Jazz"]

// 集合中加入一个值
genres.insert("Classical")

// 集合中删除一个值
genres.remove("Jazz")
{% endhighlight %}

## 遍历字典

{% highlight swift %}
var genres: Set<String> = ["Rock", "Pop", "Jazz"]

for genre in genres.sort() {
  print(genre)
}
{% endhighlight %}

## 集合操作

![Swift Sets Venn](/images/swift-sets-venn.jpg)

{% highlight swift %}
let oddDigits: Set = [1, 3, 5, 7, 9]
let evenDigits: Set = [0, 2, 4, 6, 8]
let singleDigitPrimeNumbers: Set = [2, 3, 5, 7]

oddDigits.union(evenDigits).sort()
// [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]

oddDigits.intersect(evenDigits).sort()
// []

oddDigits.subtract(singleDigitPrimeNumbers).sort()
// [1, 9]

oddDigits.exclusiveOr(singleDigitPrimeNumbers).sort()
// [1, 2, 9]
{% endhighlight %}

![Swift Sets Euler](/images/swift-sets-euler.jpg)

{% highlight swift %}
let houseAnimals: Set = ["Dog", "Cat"]
let farmAnimals: Set = ["Cow", "Chick", "Sheet", "Dog", "Cat"]
let cityAnimals: Set = ["Pigeon", "Rat"]

// 子集
houseAnimals.isSubsetOf(farmAnimals)

// 超集
farmAnimals.isSubsetOf(houseAnimals)

// 没有共同值
farmAnimals.isDisjointWith(cityAnimals)
{% endhighlight %}
