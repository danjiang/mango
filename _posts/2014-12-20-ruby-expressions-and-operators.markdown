---
title: Ruby 的表达式和运算
author: 但江
location: 成都
category: programming
tag: ruby
---

![Desk](/images/desk.jpg)

#### 变量

变量的定义方法

* 以 `$` 开头的是全局变量，任何程序可见
* 以 `@` 开头是实例变量，以 `@@` 开头是类变量
* 以 `_` 或小写字母开头是本地变量，在方法和块中定义
* Ruby 中实例变量默认都是私有作用域，外部访问都是通过方法

{% highlight ruby %}
o.m = v # 就是 o.m = (v)
o[x] = y # 就是 o.[] = (x, y)
{% endhighlight %}

变量的初始化

* 未初始化的全局变量就像未初始化的实例变量，返回 nil
* 类变量必须先要初始化，才能使用
* 如果引用没有初始化的实例变量，Ruby 返回 nil
* 引用未初始化的本地变量会报错

#### 常量

常量以大写字母开头，通常约定，大多数常量都全大写，以下划线分割，如 `LIKE_THIS`，Ruby 的类和模块名称也是常量，它们通常遵循骆驼命名规则（单词的第一个字母大写），如 `LikeThis`

常量可以在 Ruby 任何代码中引用，没有作用域的限制，引用方法如下

{% highlight ruby %}
Conversion::CM_PER_INCH
{% endhighlight %}

`::` 左边的部分如果忽略掉，这样的常量会在全局域中查找，其实就是在 `Object` 类中查找，`::ARGV` 也就是 `Object::ARGV`

#### 方法调用

方法名称和本地变量的表示方式没有什么区别，如果 Ruby 解析器看到了本地变量的赋值语句，它就知道是本地变量，否则当成方法来调用，方法调用通过 `.` 或 `::` ，如果只写方法名字，就是在 `self` 上调用

#### 赋值

基本方式

{% highlight ruby %}
x = 1
{% endhighlight %}

缩写方式

{% highlight ruby %}
x += y # 就是 x = x + y
x -= y # 就是 x = x - y
x ||= y # 就是 x = x || y
{% endhighlight %}

并行赋值

{% highlight ruby %}
x, y, z = 1, 2, 3
{% endhighlight %}

#### 运算符

值得注意的是 Ruby 中的运算符大都定义成类的方法

如果要定义一元的 `+` 和 `-`，在定义方法时使用方法名 `-@` 和 `+@`

`**` 是幂指数运算，如 `2 ** 3`，结果就是 `8`

`<<` 常用做追加运算，比如 `String`，`Array` 和 `IO` 类中都重写定义了此运算来完成追加的功能

{% highlight ruby %}
message = "hello"        # 一个字符串
messages = []            # 一个空数组
message << " world"      # 追加到到字符串
messages << message      # 追加到数组
STDOUT << message        # 打印到标准输出
{% endhighlight %}

`<=>` 运算符用作比较方法，运算符左边的与运算符右边的比较，大于返回 `-1`，等与返回 `0`，小于返回 `+1`，实现了此方法的类可以 `include Comparable` 来获取 `<=` `==` 等这些比较方法

`=~` 是正则表达式运算符，左边的字符串是否满足右边的正则表达式

`===` 运算符主要隐式的用于 `case` 表达式中
