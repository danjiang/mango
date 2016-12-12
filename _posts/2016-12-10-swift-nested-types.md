---
title: Swift 嵌套类型
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: swift
---

从这篇文章，你将学习到 Swift 的嵌套类型，让我们开始吧。

![Swift Nested Types](/images/swift-nested-types.jpg)

## 一个实际的例子

对于音乐所属的流派应该是几个固定值，下面定义的 **Music** 类，在其中还定义了嵌套的枚举类型 **Genre**，**Genre** 是针对 **Music** 而言的，所以不放在全局，做为嵌套类型放在 **Music** 中很合适。

{% highlight swift %}
struct Music {
  
  enum Genre: String {
    case pop
    case rock
    case jazz
  }
  
  enum AudioFormat: String {
    case mp3
    case wav
  }
  
  let name: String
  let singer: String
  let genre: Genre
  let audioFormat: AudioFormat
  
}

let blankSpace = Music(name: "Blank Space", singer: "Tyler Swift",
                       genre: .pop, audioFormat: .mp3)
{% endhighlight %}

## 引用嵌套类型

当然，不仅仅在 **Music** 中可以使用 **Genre**，在外部也是可以的。

{% highlight swift %}
let pop = Music.Genre.pop.rawValue
{% endhighlight %}