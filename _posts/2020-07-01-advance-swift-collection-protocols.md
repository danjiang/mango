---
title: Swift 进阶 - 集合协议
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: swift
---

深入 Swift 中集合协议，这些协议是数组、字典、集合和字符串实现的基础，有一些数据结构和算法的知识，理解这部分内容更容易一些。

![Advance Swift](/images/advance-swift.png)

## Sequence 和 Iterator

**Sequence** 就是**一串相同类型的值**，可以在其上遍历迭代：

{% highlight swift %}
protocol Sequence {
    associatedtype Element // 序列中的元素
    associatedtype Iterator: IteratorProtocol // 迭代器协议
    
    func makeIterator() -> Iterator // 生成迭代器
}
{% endhighlight %}

**迭代器**协议声明就比较简单，next() 方法**获取下一个值**，**不能后退**，**也不能随机访问**，next() 返回 nil 时表明迭代结束，所以也可以一直返回非 nil 的值就永远不结束：

{% highlight swift %}
protocol IteratorProtocol {
    associatedtype Element
    mutating func next() -> Element?
}
{% endhighlight %}

编译器将 for loop 转换为下面的迭代器代码：

{% highlight swift %}
var iterator = someSequence.makeIterator()
while let element = iterator.next() {
    doSomething(with: element)
}
{% endhighlight %}

如下的迭代器，输入 "abc"，每次迭代生成 "a"、"ab" 和 "abc"：

{% highlight swift %}
struct PrefixIterator: IteratorProtocol {
    let string: String
    var offset: String.Index
    
    init(string: String) {
        self.string = string
        offset = string.startIndex
    }
    
    mutating func next() -> Substring? {
        guard offset < string.endIndex else { return nil }
        string.formIndex(after: &offset)
        return string[..<offset]
    }
}
{% endhighlight %}

如下的 Sequence 使用了上面的迭代器，这个 Sequence 保存了输入的字符串，makeIterator() 都是根据保存的字符串创建迭代器，然后在这个基础进行一次遍历迭代，可以看出 Sequence 和迭代器的关系，迭代器就只是负责给出下一个值，有可能会改变一些初始值，Sequence 保存了初始值，以便创建迭代器时使用：

{% highlight swift %}
struct PrefixSequence: Sequence {
    let string: String
    
    func makeIterator() -> PrefixIterator {
        return PrefixIterator(string: string)
    }
}
{% endhighlight %}

Sequence 通常是只能单次遍历，也有可以多次遍历的情况，其语义并不保证可以多次遍历，尽量实现成只能单次遍历。

## Collection

**Collection** 是可以**无破坏多次遍历**、**有限**的序列，还可以通过**下标访问**。

Collection 协议是基于 Sequence 协议，满足 Collection 协议需要实现内容很多，但是因为有默认实现，最基本的需要实现的内容如下，满足了 Collection 协议后，就可以拥有很多功能，查看 [Collection - Apple Developer Documentation](https://developer.apple.com/documentation/swift/collection#)：

{% highlight swift %}
protocol Collection: Sequence {
    associatedtype Element // 序列中的元素
    associatedtype Index: Comparable // 下标
    var startIndex: Index { get } // 第一个元素的下标
    var endIndex: Index { get } // 最后一个元素的下标 + 1
    func index(after i: Index) -> Index // 当前下标 + 1
    subscript(position: Index) -> Element { get } // 下标获取元素
}
{% endhighlight %}

Array Literals 是数组语义，[Swift 数组](/programming/2016/03/19/swift-arrays/) 和 [Swift 集合](/programming/2016/04/03/swift-sets/) 中有关于通过如下数组语义创建的代码：

{% highlight swift %}
let evenNumbers = [2, 4, 6, 8]
var shoppingList = ["Eggs", "Milk"]

var genres: Set<String> = ["Rock", "Pop", "Jazz"]
{% endhighlight %}

通过实现 ExpressibleByArrayLiteral 协议，你也可以拥有：

{% highlight swift %}
extension FIFOQueue: ExpressibleByArrayLiteral {
    public init(arrayLiteral elements: Element...) {
        self.init(left: elements.reversed(), right: [])
    }
}
{% endhighlight %}

值得注意的 Associated Types：

* Iterator - 前面 Sequence 中已经讲过。
* Indices - Index 不一定只是通过 Int 实现，满足 Comparable 协议都可以，注意不能有指向 Collection 的引用，否则会有性能问题。查看 [Advance Swift - Indices](https://www.objc.io/books/advanced-swift/) 了解更多。
* SubSequence - 也是 Collection，和原 Collection 共享所有元素，只是通过两个下标来限定范围，可以通过 String(substring) 或 Array(arraySlice) 来拷贝元素创建新的 Collection。查看 [Advance Swift - Subsequences](https://www.objc.io/books/advanced-swift/) 了解更多。

## 集合协议层级

![Swift Collection Protocols](/images/swift-collection-protocols.png)

### BidirectionalCollection

**BidirectionalCollection** 支持**向后遍历**和**向前遍历**。

{% highlight swift %}
protocol BidirectionalCollection: Collection {
    func index(before i: Index) -> Index // 当前下标 - 1
}
{% endhighlight %}

### RandomAccessCollection

**RandomAccessCollection** 支持**常量时间的下标计算**。

{% highlight swift %}
protocol RandomAccessCollection: BidirectionalCollection {
    func index(_ i: Index, offsetBy distance: Int) -> Index // 根据距离计算下标
    func distance(from start: Index, to end: Index) -> Int // 计算两下标的距离
}
{% endhighlight %}

### MutableCollection

**MutableCollection** 支持**原址元素修改**。

{% highlight swift %}
protocol MutableCollection: Collection {
    subscript(position: Index) -> Element { set } // 下标修改元素
}
{% endhighlight %}

### RangeReplaceableCollection

**RangeReplaceableCollection** 支持**元素新增**和**元素删除**。

{% highlight swift %}
protocol RangeReplaceableCollection: Collection {
    init() // 创建空集合
    mutating func replaceSubrange<C>(_ subrange: Range<Index>, with newElements: C) // 替换下标范围的元素
        where C : Collection, Element == C.Element
}
{% endhighlight %}

### LazySequenceProtocol 和 LazyCollectionProtocol

**LazySequenceProtocol** 和 **LazyCollectionProtocol** 支持**当需要时才计算**。

{% highlight swift %}
let filtered = standardIn.lazy.filter {
    $0.split(separator: " ").count > 3
}
for line in filtered {
    print(line)
}
{% endhighlight %}

{% highlight swift %}
(1..<100).lazy.map { $0 * $0 }.filter { $0 > 10 }.map { "\($0)"}
{% endhighlight %}
