---
title: Ruby 的方法，Proc，Lambda 和闭包
author: 但江
location: 成都
category: programming
tag: ruby
---

![Typewriter](/images/typewriter.jpg)

#### 方法

通过 `def` 来定义方法

{% highlight ruby %}
def factorial(n)
  if n < 1
    raise "argument must be > 0"
  elsif n == 1
    1
  else
    n * factorial(n - 1)
  end
end
{% endhighlight %}

如果一个方法正常的结束，最后一个表达式的值就是方法的返回值，Ruby 可以有多个返回值

{% highlight ruby %}
def polar(x, y)
  return Math.hypot(y, x), Math.atan2(y, x)
end

distance, theta = polar(x, y)
{% endhighlight %}

定义单例方法，在单个对象上定义的方法

{% highlight ruby %}
o = "message"

def o.printme
  puts self
end

o.printme
{% endhighlight %}

`alias` 定义一个已有方法的别名

{% highlight ruby %}
alias aka also_known_as
{% endhighlight %}

可以像如下给方法参数添加默认值

{% highlight ruby %}
def prefix(s, len = 1)
  s[0, len]
end
{% endhighlight %}

默认值可以是任意的表达式，默认值是方法在被调用的时候才赋值给参数的

{% highlight ruby %}
def suffix(s, index = s.size - 1)
  s[index, s.size - index]
end
{% endhighlight %}

在方法参数名前添加 `*`，可以接受任意长度的参数，这些参数都会变成参数数组

{% highlight ruby %}
def max(first, *rest)
  max = first
  rest.each { |x| max = x if x > max }
  max
end

max(1)       # first = 1, rest=[]   
max(1, 2)    # first = 1, rest=[2]  
max(1, 2, 3) # first = 1, rest=[2, 3]
{% endhighlight %}

可以用 `*` 来打散数组，范围，或枚举变成独立的方法参数

{% highlight ruby %}
data = [3, 2, 1]
m = max(*data) # first = 3, rest=[2, 1] => 3
{% endhighlight %}

用哈希来做命名参数

{% highlight ruby %}
def sequence(args)
  n = args[:n] || 0
  m = args[:m] || 1
  c = args[:c] || 0

  a = []
  n.times { |i| a << m * i + c }
  a
end

sequence({ :n => 3, :m => 5 }) # => [0, 5, 10]
{% endhighlight %}

块参数

在方法定义的最后一个参数前加 `&`，这个参数会指向块

{% highlight ruby %}
def sequence3(n, m, c, &b)
  i = 0
  while(i < n)
    b.call(i * m + c) 
    i += 1
  end
end

sequence3(5, 2, 2) { |x| puts x }
{% endhighlight %}

在方法调用时，& 用在 Proc 对象前，就像普通方法调用后面跟的块一样

{% highlight ruby %}
a, b = [1, 2, 3], [4, 5]
sum = a.inject(0) { |total, x| total + x } # => 6
sum = b.inject(sum) { |total, x| total + x } # => 15

# 上的代码，可以转换为下面这样

a, b = [1, 2, 3], [4, 5]
summation = Proc.new { |total, x| total + x }
sum = a.inject(0, &summation) # => 6
sum = b.inject(sum, &summation) # => 15
{% endhighlight %}

#### Proc，Lambda 和 闭包

创建 `Proc`

{% highlight ruby %}
p = Proc.new { |x, y| x + y }
{% endhighlight %}

创建 `lambda`

{% highlight ruby %}
is_positive = lambda { |x| x > 0 }

succ = ->(x){ x + 1 } # Ruby 1.9 中的新定义方法
{% endhighlight %}

通过 Proc 类定义了 call 方法来调用

{% highlight ruby %}
succ.call(2) # => 3
{% endhighlight %}

如果在一个内部函数里，对在外部作用域（但不是在全局作用域）的变量进行引用，那么内部函数就被定义为`闭包` 。定义在外部函数内的但由内部函数引用或者使用的变量被称为`自由变量`，注意 `lambda` 和 `proc` 中使用的变量不是在创建 `lambda` 和 `proc` 的时候就静态绑定了，而是动态绑定的，变量的值是在 `lambda` 和 `proc` 被调用的时候才进行查找的

{% highlight ruby %}
def accessor_pair(initialValue = nil)
  value = initialValue
  getter = lambda { value }
  setter = lambda { |x| value = x }
  return getter, setter
end

getX, setX = accessor_pair(0)
puts getX[] # Prints 0
setX[10]
puts getX[] # Prints 10
{% endhighlight %}
