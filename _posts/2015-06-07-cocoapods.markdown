---
title: iOS 实战 - CocoaPods
author: 但江
avatar: danjiang
location: 成都
category: programming
---

[CocoaPods][1] 是为了解决 Swift 和 Objective-C 项目的依赖关系，在没有 CocoaPods 以前，我们只能手动地拷贝第三方的代码，维护起来很麻烦，CocoaPods 是通过 Ruby 语言编写的，自然在设计和使用上非常类似于 RubyGems。

## 安装 CocoaPods

因为 CocoaPods 用 Ruby 语言编写的，所以先要安装 Ruby，参考文章 [开始学习 Ruby][2]，接下来通过 RubyGems 安装 CocoaPods。

{% highlight text %}
$ gem install cocoapods
{% endhighlight %}

## 维护依赖关系

首先在 Xcode 项目的根目录运行如下命令来初始化：

{% highlight text %}
$ pod init
{% endhighlight %}

编辑 Podfile，写入如下内容：

{% highlight ruby %}
platform :ios, '8.0'
use_frameworks!

target 'MyApp' do
  pod 'AFNetworking', '~> 2.5'
  pod 'ORStackView', '~> 2.0'
  pod 'SwiftyJSON', '~> 2.1'
end
{% endhighlight %}

运行如下命令安装依赖：

{% highlight text %}
$ pod install
{% endhighlight %}

通过打开 Xcode 工作空间来构建项目：

{% highlight text %}
$ open App.xcworkspace
{% endhighlight %}

现在就可以在项目中导入依赖：

{% highlight objc %}
#import <Reachability/Reachability.h>
{% endhighlight %}

[1]: https://cocoapods.org
[2]: /programming/2014/12/06/get-started-with-ruby/
