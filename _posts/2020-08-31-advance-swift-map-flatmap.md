---
title: Swift 进阶 - map 和 flatMap
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: swift
---

map 和 flatMap 主要分在集合上的使用和在可选类型上的使用，下面分别来看下。

![Advance Swift](/images/advance-swift.png)

## 集合上使用 map 和 flatMap

先看如下的代码：

{% highlight swift %}
func getInfos(by name: String) -> [String] {
    if name == "Jack" {
        return ["Male", "25", "New York"]
    } else if name == "Lucy" {
        return ["Female", "18", "London"]
    } else {
        return ["Unkown", "Unkown", "Unkown"]
    }
}

let names = ["Jack", "Lucy", "Nobody"]

let infos1 = names.map { getInfos(by: $0) }
print(infos1)

let infos2 = names.flatMap { getInfos(by: $0) }
print(infos2)
{% endhighlight %}

输入是一个一维数组，转换后 infos1 的结果是如下的一个二维数组，所以 **map 后有两层结构**：

{% highlight swift %}
[["Male", "25", "New York"], ["Female", "18", "London"], ["Unkown", "Unkown", "Unkown"]]
{% endhighlight %}

输入是一个一维数组，转换后 infos2 的结果是如下的一个一维数组，所以 **flatMap 后只有一层结构**：

{% highlight swift %}
["Male", "25", "New York", "Female", "18", "London", "Unkown", "Unkown", "Unkown"]
{% endhighlight %}

map 在 Array 上的实现大致如下：

{% highlight swift %}
extension Array {
    func map<T>(_ transform: (Element) -> T) -> [T] {
        var result: [T] = []
        for x in self {
            result.append(transform(x))
        }
        return result
    }
}
{% endhighlight %}

flatMap 在 Array 上的实现大致如下：

{% highlight swift %}
extension Array {
    func flatMap<T>(_ transform: (Element) -> [T]) -> [T] {
        var result: [T] = []
        for x in self {
            result.append(contentsOf: transform(x))
        }
        return result
    }
}
{% endhighlight %}

## 可选类型上使用 map 和 flatMap

如下代码中，输入是 stringNumbers.first，其类型是 String?

- 转换后 x 的类型是 Int??，所以 **map 后有两层 Optional**
- 转换后 y 的类型是 Int?，所以 **flatMap 后只有一层 Optional**

{% highlight swift %}
let stringNumbers = ["1", "2", "3", "foo"]
let x = stringNumbers.first.map { Int($0) } // Optional(Optional(1))
let y = stringNumbers.first.flatMap { Int($0) } // Optional(1)
{% endhighlight %}

map 在 Optional 上的实现大致如下：

{% highlight swift %}
extension Optional {
    func map<U>(transform: (Wrapped) -> U) -> U? {
        if let value = self  {
            return transform(value)
        }
        return nil
    }
}
{% endhighlight %}

flatMap 在 Optional 上的实现大致如下：

{% highlight swift %}
extension Optional {
    func flatMap<U>(transform: (Wrapped) -> U?) -> U? {
        if let value = self, let transformed = transform(value) {
            return transformed
        }
        return nil
    }
}
{% endhighlight %}
