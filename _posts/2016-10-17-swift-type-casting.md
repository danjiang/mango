---
title: Swift 类型转换
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: swift
---

从这篇文章，你将学习到 Swift 的类型转换，类型转换是类独有的特性，让我们开始吧。

![Swift Type Casting](/images/swift-type-casting.jpg)

## 用于类型转换的类层次结构

{% highlight swift %}
class MediaItem {
  var name: String
  init(name: String) {
    self.name = name
  }
}

class Movie: MediaItem {
  var director: String
  init(name: String, director: String) {
    self.director = director
    super.init(name: name)
  }
}

class Song: MediaItem {
  var artist: String
  init(name: String, artist: String) {
    self.artist = artist
    super.init(name: name)
  }
}

let library = [
  Movie(name: "Casablance", director: "Michael Curtiz"),
  Song(name: "Blue Suede Shoes", artist: "Elvis Presley"),
  Movie(name: "Citizen Kane", director: "Orson Welles")
]
{% endhighlight %}

## 类型检查

通过 **is** 来检查一个实例是不是确定的子类类型：

{% highlight swift %}
var movieCount = 0
var songCount = 0

for item in library {
  if item is Movie {
    movieCount += 1
  } else if item is Song {
    songCount += 1
  }
}
{% endhighlight %}

## 向下转换

通过 **as?** 或 **as** 来将实例转换为确定的子类实例：

{% highlight swift %}
for item in library {
  if let movie = item as? Movie {
    print("Movie: \(movie.name), dir. \(movie.director)")
  } else if let song = item as? Song {
    print("Song: \(song.name), by \(song.artist)")
  }
}
{% endhighlight %}

## Any 和 AnyObject

* **Any** 代表任何类型的实例，包括函数类型；
* **AnyObject** 代表任何类的实例。

一个 **Any** 的例子：

{% highlight swift %}
var things = [Any]()

things.append(0)
things.append(0.0)
things.append(42)
things.append(3.14159)
things.append("hello")
things.append((3.0, 5.0))
things.append(Movie(name: "Ghostbusters", director: "Ivan Reitman"))
things.append({ (name: String) -> String in "Hello, \(name)" })

for thing in things {
  switch thing {
  case 0 as Int:
    print("zero as an Int")
  case 0 as Double:
    print("zero as a Double")
  case let someInt as Int:
    print("an integer value of \(someInt)")
  case let someDouble as Double where someDouble > 0:
    print("a positive double value of \(someDouble)")
  case is Double:
    print("some other double value that I don`t want to print")
  case let someString as String:
    print("a string value of \"\(someString)\"")
  case let (x, y) as (Double, Double):
    print("an (x, y) point at \(x), \(y)")
  case let movie as Movie:
    print("a movie called \(movie.name), dir. \(movie.director)")
  case let stringConverter as (String) -> String:
    print(stringConverter("Michael"))
  default:
    print("something else")
  }
}
{% endhighlight %}