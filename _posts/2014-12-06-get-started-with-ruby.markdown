---
title: 开始学习 Ruby 
author: 但江
location: 成都 
category: programming
---

> Ruby 设计理念是为了让程序员使用时感到快乐

#### 安装 Ruby

针对 Mac 或者 Linux 用户来说，安装 RVM（RVM 可以让你在系统上安装不同版本的 Ruby 和管理不同的 Gemsets）

	$\curl -sSL https://get.rvm.io | bash -s stable

通过 RVM 安装 Ruby

	$rvm install 2.0.0

在终端中运行 `irb` 可以运行交互式的 Ruby Shell，敲一敲代码来感受真实 Ruby 代码

#### Ruby 的特性

一切都是对象

{% highlight ruby %}
1.class # => Fixnum 类的对象
0.0.class # => Float 类的对象
true.class # => TrueClass 的单例实例
false.class # => FalseClass
nil.class # => NilClass
{% endhighlight %}

块和迭代器

{% highlight ruby %}
3.times { print "Ruby! " } # 输出 "Ruby! Ruby! Ruby! "
1.upto(9) {|x| print x } # 输出 "123456789"
{% endhighlight %}

数组

{% highlight ruby %}
a = [3, 2, 1] # 创建数组 
a[3] = a[2] - 1 # []来访问数组中的值
a.each do |elt| # each 是一个迭代器，elt 是传递到块中的参数
  print elt+1 # 输出 "4321"
end
{% endhighlight %}

哈希

{% highlight ruby %}
h = { :one => 1, :two => 2 } # 创建哈希
h[:one] # 通过键访问值
h[:three] = 3 # 添加一个新的键值对到哈希中 
h.each do |key,value| # 遍历哈希中的键值
  print "#{value}:#{key}; " # 输出 "1:one; 2:two; 3:three; " 
end
{% endhighlight %}

表达式和运算符

{% highlight ruby %}
1 + 2 # => 3: 加法
1 * 2 # => 2: 乘法
1 + 2 == 3 # => true: == 测试相等
2 ** 1024 # 2的1024次幂，Ruby 有任意长度的整数
"Ruby" + " rocks!" # => "Ruby rocks!": 字符串相接
"Ruby! " * 3 # => "Ruby! Ruby! Ruby! ": 重复
{% endhighlight %}

方法

{% highlight ruby %}
def square(x) # 定义一个名为 square 的方法，只有一个参数 x
  x*x
end
{% endhighlight %}

赋值

{% highlight ruby %}
x, y = 1, 2 # 同样的效果: x = 1; y = 2
a, b = b, a # 交换两个变量的值 
x, y, z = [1, 2, 3] # 数组的元素自动赋给三个变量
{% endhighlight %}

标点符号后缀和前缀

`Array` 和 `Hash` 都有 `empty?` 方法，`?` 表明方法的返回值是布尔值，`Array` 有定义 `sort` 和 `sort!` 方法，`sort` 会返回排序后的数组，但是不会改变原来的数组，`sort!` 会将数组本身顺序改变，`!` 表明调用方法时要明确其后果。

正则和区间

{% highlight ruby %}
/[Rr]uby/ # 匹配 "Ruby" 或 "ruby"
/\d{5}/ # 匹配5个连续的数字
1..3 # 1到3，包含3
1...3 # 1到3，不包含3 
{% endhighlight %}

类和模块

{% highlight ruby %}
class Sequence # 定义一个类
  include Enumerable # 在此类中包含 Enumerable 模块中的方法

  def initialize(from, to, by) # 初始化方法，创建类实例时自动调用
    @from, @to, @by = from, to, by # @ 前缀的是实例变量，用参数值赋值给实例变量
  end

  def each
    x = @from # 从起始值开始
    while x <= @to # 还没有结束值
      yield x # 传递 x 给块
      x += @by # x 增加 @by
    end
  end

  def length
    return 0 if @from > @to # 注意 if 的用法 
    Integer((@to - @from) / @by) + 1 
    # 计算序列的长度，最后一行表达式的值作为方法的返回值
  end

  def[](index) # 重写数组访问操作符
    return nil if index < 0
    v = @from + index*@by
    if v <= @to
      v
    else
      nil
    end
  end

  def *(factor) # 重写算术运算符
    Sequence.new(@from * factor, @to * factor, @by * factor)
  end

  def +(offset) # 重写算术运算符
    Sequence.new(@from + offset, @to + offset, @by)
  end
end

# 如何使用定义的 Sequence 类
s = Sequence.new(1, 10, 2) # 从1到10，步长是2
s.each { |x| print x } # 输出 "13579"
print s[s.size-1] # 输出 9
t = (s + 1) * 2 # 从4到22，步长是4
{% endhighlight %}
