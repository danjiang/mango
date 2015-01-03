---
title: Ruby 的流程控制
author: 但江
location: 成都
category: programming
---

![Crossroad](/images/crossroad.jpg)

#### 条件

`if else` 的写法

{% highlight ruby %}
if data
  data << x
else
  data = [x]
end
{% endhighlight %}

`if elsif` 的写法

{% highlight ruby %}
if x == 1
  name = "one"
elsif x == 2
  name = "two"
elsif x == 3 then name = "three"
elsif x == 4; name = "four"
else
  name = "many"
end
{% endhighlight %}

`if` 语句的返回值来做为赋值

{% highlight ruby %}
name = if x == 1 then "one"
       elsif x == 2 then "two"
       elsif x == 3 then "three"
       elsif x == 4 then "four"
       else              "many"
       end
{% endhighlight %}

`if` 简短写法

{% highlight ruby %}
puts message if message # 输出 message，如果有定义 message
{% endhighlight %}

`unless` 当表达式是 `false` 或 `nil`，就执行下面的内容，用法类似于 `if`，除了没有 `elsif` 这样的写法

{% highlight ruby %}
unless condition
  code 
end

unless condition
  code
else
  code
end

code unless condition
case
{% endhighlight %}

`if/elsif/else` 的另一种写法

{% highlight ruby %}
name = case x
       when 1
         "one"
       when 2 then "two"
       when 3; "three"
       else "many"
       end
{% endhighlight %}

需要注意一点，`case` 中的表达式怎么和 `when` 中值进行比较的，是通过 `===` 运算符来完成的，上面的写法实际就是按下面这种方式运行的

{% highlight ruby %}
name = case
       when 1 === x then "one"
       when 2 === x then "two"
       when 3 === x then "three"
       else "many"
       end
{% endhighlight %}

`Class` 类中定义 `===` 方法来检测对象是否某个类的实例，所以就可以写如下的代码

{% highlight ruby %}
puts case x
     when String then "string"
     when Numeric then "number"
     when TrueClass, FalseClass then "boolean"
     else "other"
     end
{% endhighlight %}

`?:` 是一种更简洁的条件表达式

{% highlight ruby %}
def how_many_messages(n)
  "You have " + n.to_s + ( n == 1 ? " message." : " messages.")
end
{% endhighlight %}

#### 循环

`while` 基本的写法

{% highlight ruby %}
x = 10
while x >= 0 do
  puts x
  x = x - 1
end
{% endhighlight %}

`until` 基本的写法

{% highlight ruby %}
x = 0
until x > 10 do
  puts x
  x = x + 1
end
{% endhighlight %}

`while` 简洁的写法

{% highlight ruby %}
x = 0
puts x = x + 1 while x < 10
{% endhighlight %}

`until` 简洁的写法

{% highlight ruby %}
a = [1, 2, 3]
puts a.pop until a.empty?
{% endhighlight %}

`for in` 循环每一次都调用对象的 `each` 方法得到值

{% highlight ruby %}
hash = { :a => 1, :b => 2, :c => 3 }
for key,value in hash
  puts "#{key} => #{value}"
end
{% endhighlight %}

还比如直接用 `each` 来的帅

{% highlight ruby %}
hash = { :a => 1, :b => 2, :c => 3 }
hash.each do |key, value|
  puts "#{key} => #{value}"
end
{% endhighlight %}

#### 块

块就是跟在方法后面在 `{ }` 或 `do end` 中的代码块

{% highlight ruby %}
1.upto(10) { |x| puts x }
1.upto(10) do |x|
  puts x
end
{% endhighlight %}

在定义方法的时候可以通过 `yield` 调用块，如下面的自定义方法

{% highlight ruby %}
def sequence(n, m, c)
  i = 0
  while (i < n)
    yield m * i + c # 调用块，传递值给块
    i += 1
  end
end
{% endhighlight %}

在方法定义中可以通过 `block_given?` 来判断是否有块

{% highlight ruby %}
def sequence(n, m, c)
  i, s = 0, []
  while (i < n)
    y = m * i + c
    yield y if block_given?
    s << y
    i += 1
  end
  s
end
{% endhighlight %}

块中最后一行表达式的值就是块的返回值，从上面的例子可以看到方法定义中可以利用 `yield` 调用获取到块的返回值来进行相应的操作

{% highlight ruby %}
words.sort! { |x, y| y <=> x }
{% endhighlight %}

块中定义的变量的作用域就在块中

#### 其他的流程控制的

`return` 对于无论嵌套的块有多深，总是将最近的包围的方法返回

{% highlight ruby %}
def find(array, target)
  array.each_with_index do |element, index|
    return index if (element == target) # 找到了就作为 find 的返回值
  end
  nil # 没有找到，就返回 nil
end
{% endhighlight %}

`break` 会结束循环，执行循环块后的第一条语句

{% highlight ruby %}
while (line = gets.chop) # 循环开始
  break if line == "quit" # 退出循环
  puts eval(line)
end
puts "Good bye" # 然后从这里继续执行
{% endhighlight %}

`next` 会结束当前这一步的循环执行，进行下一步的循环执行

{% highlight ruby %}
while(line = gets.chop) # 循环开始
  next if line[0, 1] == "#" # 如果是注释符号，到下一步循环
  puts eval(line)
  # 然后从这里继续执行
end
{% endhighlight %}

`redo` 会重新执行当前的这一步循环

{% highlight ruby %}
i = 0
while (i < 3) # 输出 "0123" 而不是 "012"
  # redo 执行后会从这里继续执行
  print i
  i += 1
  redo if i == 3
end
{% endhighlight %}

#### 异常

异常对象是 `Exception` 类的实例或其子类的实例。需要知道的是大多数的异常子类都是继承于 `StandardError` 类，这是 Ruby 程序需要处理的异常

`Exception` 类定义了两个方法来获取异常的相关信息

* `message` 方法返回易于人阅读的异常信息
* `backtrace` 方法返回异常是在什么地方开始的，以及方法调用的层级引用

调用 `raise` 创建异常，若只有一个字符串参数时，会创建一个新的 `RuntimeError` 对象，字符串参数就是异常的 `message`

{% highlight ruby %}
def factorial(n)
  raise "bad argument" if n < 1
  return 1 if n == 1
  n * factorial(n - 1)
end
{% endhighlight %}

调用 `raise` 时，若第一个参数是类，并且该类有一个 `exception` 方法，`raise` 就会调用这个 `exception` 方法然后抛出此方法返回的异常对象，因为 `Exception` 类定义了 `exception` 方法，所以你可以将任何异常类作为 `raise` 的第一个参数，还可以添加第二个参数传递给 `exception` 方法作为异常的 `message`

{% highlight ruby %}
raise RuntimeError, "bad argument" if n < 1
{% endhighlight %}

调用 `raise` 时，若第一个参数是异常对象，那就直接将异常对象抛出

{% highlight ruby %}
raise RuntimeError.new("bad argument") if n < 1
raise RuntimeError.exception("bad argument") if n < 1
{% endhighlight %}

自定义异常类，通常继承 `StandardError`

{% highlight ruby %}
class MyError < StandardError
end
{% endhighlight %}

捕获异常，下面写法中 `rescue` 捕获任何 `StandardError` 的异常，其他的异常会被忽略掉

{% highlight ruby %}
begin
  # 任意的 Ruby 表达式
rescue
  # 如果有异常抛出，流程就会转到这里
end
{% endhighlight %}

在 `rescue` 中，全局变量 `$!` 代表被处理的异常对象，可以通过下面的方式来为异常对象指定一个变量名称

{% highlight ruby %}
begin
  x = factorial(-1)
rescue => ex
  puts "#{ex.class}: #{ex.message}"
end
{% endhighlight %}

捕获特定类型的异常

{% highlight ruby %}
begin
  x = factorial(1)
rescue ArgumentError => ex
  puts "Try again with a value >= 1"
rescue TypeError => ex
  puts "Try again with an integer"
end
{% endhighlight %}

如果在 `rescue` 语句又发生了异常，原来被捕获的异常会被抛弃，新的异常传播会从这里开始

在异常捕获时，`else` 中的语句在 `begin end` 语句运行没有出现异常的时候才会执行

在异常捕获时，`ensure` 中的语句都会执行，无论前面异常捕获语句中的代码发生了什么

一个典型的方法定义中使用异常语句

{% highlight ruby %}
def method_name(x)
  # 任意的 Ruby 表达式
rescue 
  # 如果有异常抛出，流程就会转到这里
else
  # 如果没有异常抛出，流程就会转到这里
ensure
  # 无论有没有异常抛出，流程就会转到这里
end
{% endhighlight %}

`rescue` 的一种简洁用法，不能指定异常类名，只能捕获 `StandardError` 异常

{% highlight ruby %}
y = factorial(x) rescue 0
{% endhighlight %}
