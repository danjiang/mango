---
title: iOS 基础 - KVO
author: 但江
avatar: danjiang
location: 成都 
category: programming
tag: ios-basic
---

一些 iOS 基础知识，业务开发中经常用到，面试时也常会被问到，这里总结一下，此篇文章讲解 KVO。

![Apple Develop Xcode](/images/apple-develop-xcode.jpg)

## Key-Path Expressions

对于实例的属性访问可以采用 [keyPath: \.属性名] 这样的方式，后面讲 KVO 时，可以看到这种表达式的用处。

{% highlight swift %}
struct Person {
    var name: String
}

struct Book {
    var title: String
    var authors: [Person]
    var primaryAuthor: Person {
        return authors.first!
    }
}

let abelson = Person(name: "Harold Abelson")
let sussman = Person(name: "Gerald Jay Sussman")
let book = Book(title: "Structure and Interpretation of Computer Programs", authors: [abelson, sussman])

print(book[keyPath: \Book.title])
print(book[keyPath: \.primaryAuthor.name])
{% endhighlight %}

## KVO

KVO 主要应用场景是观察对象属性的变化，能被观察的对象属性需满足下面两个条件：

1. 首先要利用 Objective-C Runtime 的特性，所以需要继承自 NSObject。
2. 属性需要是 @objc dynamic。

{% highlight swift %}
class Child: NSObject {
    
    let name: String

    @objc dynamic var age: Int

    init(name: String, age: Int) {
        self.name = name
        self.age = age
        super.init()
    }

    func celebrateBirthday() {
        age += 1
    }

}
{% endhighlight %}

通过在要观察的对象上调用 observe，通过前面讲的 Key-Path Expressions 来指定属性 \.age，方法会返回一个 NSKeyValueObservation，当不再需要监听变化时，可以通过 invalidate 方法来结束监听：

{% highlight swift %}
let mia = Child(name: "Mia", age: 5)
let observation = mia.observe(\.age, options: [.initial, .old]) { (child, change) in
    if let oldValue = change.oldValue {
        print("\(child.name)’s age changed from \(oldValue) to \(child.age)")
    } else {
        print("\(child.name)’s age is now \(child.age)")
    }
}

mia.celebrateBirthday()

observation.invalidate()
{% endhighlight %}

示例 1：监听摄像头闪光灯模式的变化

{% highlight swift %}
private var flashModeObservation: NSKeyValueObservation?

flashModeObservation = videoDevice.observe(\.flashMode) { [weak self] device, _ in
    self?.flashModeChanged(device)
}
{% endhighlight %}

示例 2：监听 AVPlayerItem 的状态变化

{% highlight swift %}
private var statusObservation: NSKeyValueObservation?

// item is AVPlayerItem
statusObservation = item.observe(\.status) { [weak self] item, _ in
    self?.itemStatusChanged(item)
}
{% endhighlight %}

## Objective-C 版本的 KVO

[Key-Value Coding and Observing · objc.io](https://www.objc.io/issues/7-foundation/key-value-coding-and-observing/)