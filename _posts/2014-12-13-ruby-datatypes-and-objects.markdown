---
title: Ruby 的数据类型和对象 
author: 但江
location: 成都 
category: programming
tag: ruby
---

![Cans](/images/cans.jpg)

#### 数字

当用 `Fixnum` 来表示数字，长度不够用时会自动转换为 `Bignum`，不必担心这些问题，还有一点就是要知道 `Float` 是用二进制的方式来表示浮点数，所以会损失精度，要准确的存储浮点数就要用 `BigDecimal`

{% highlight ruby %}
0.4 - 0.3 == 0.1 # 两者不相等
{% endhighlight %}

#### 字符串

单引号的表示

{% highlight ruby %}
'This is a simple Ruby string literal'
{% endhighlight %}

双引号的表示，#{} 中可以插入 Ruby 表达式

{% highlight ruby %}
"360 degrees=#{2 * Math::PI} radians"
# 上面的结果是 "360 degrees=6.28318530717959 radians"
{% endhighlight %}

Kernel.`，此方法可以调用系统命令

{% highlight ruby %}
if windows
  listcmd = 'dir'
else
  listcmd = 'ls'
end
listing = `#{listcmd}`
{% endhighlight %}

Ruby 中的字符串是可修改的，每出现一个字符串表示新创建一个字符串，下面的代码会输出不同的 object_id

{% highlight ruby %}
10.times { puts "test".object_id }
{% endhighlight %}

在 Ruby 中，字符串就是字符的序列，这些字符不必只是在 ASCII 字符集中，字符串的每一个元素都是字符，每一个字符串都有一个编码来决定这些字节和字节所代表的字符之间的转换，如 UTF-8 这样的编码使用变化的字节数来表示每一个字符，字节与字符之间不再是一对一或者二对一。方法 `length` 和 `size` 返回字符串中的字符数，新的方法 `bytesize` 返回字节数

{% highlight ruby %}
# -*- coding: utf-8 -*- # 明确编码格式为 UTF-8
s = "2×2=4" # 注意字符串中 x，包含 6 个字节，编码了 5 个字符
s.length   # => 5: 字符: '2'   '×'   '2'   '='   '4'
s.bytesize # => 6: 字节: 32   c3 97  32    3d    34
{% endhighlight %}

#### 数组

每个数组元素的类型可以不相同，可以是任意类型，元素是可修改的

{% highlight ruby %}
[1, 2, 3] # 有三个数字的数组
[-10...0, 0..10,] # 有两个区间的数组，末尾的逗号是允许的
[[1, 2], [3, 4], [5]] # 嵌套数组
[x + y, x - y, x * y] # 数组可以包含任意的表达式
[] # 空数组
{% endhighlight %}

#### 哈希

哈希就是 Map，键值对

{% highlight ruby %}
numbers = { :one => 1, :two => 2 }
numbers = { one: 1, two: 2, three: 3 } # Ruby 1.9
{% endhighlight %}

#### 区间

一段区间，数字或者字符

{% highlight ruby %}
1..10      # 从 1 到 10 的整数，包含 10
1.0...10.0 # 从 1.0 到 10.0 的浮点数，不包含 10.0
{% endhighlight %}

对区间调用方法要括起来

{% highlight ruby %}
(1..3).to_a
{% endhighlight %}

检测值是否在区间中

{% highlight ruby %}
cold_war = 1945..1989
cold_war.include? birthdate.year
{% endhighlight %}

#### 符号

Ruby 会维护一个 `Symbol` 表，存储所有的类名，方法名和变量名，这样可以避免字符串比较，因为可以通过比较 `Symbol` 在表中的位置来比较 `Symbol`，也就是转换成数字的比较

{% highlight ruby %}
:symbol # 符号的表示方法
o.respond_to? :each # 检测对象是否有某个方法
{% endhighlight %}

#### True False Nil

`true` 是 TrueClass 的单例，`false` 是 FalseClass 的单例，`false` 或者 `nil` 就是 `false` ，其他值都是 `true`，`0`, `0.0` 和 `""` 也是 `true`，`nil` 是 `NilClass` 的单例

{% highlight ruby %}
o == nil # 判断是否为空 
o.nil?
{% endhighlight %}

#### 对象

对于 Ruby 中的对象需要知道这些

* Ruby 中所有的值都是对象，没有基本类型的说法
* Ruby 的每一个对象都有一个 `Fixnum` 的对象标示符 `object_id`，`Object` 类对 `hash` 方法的实现就是直接返回 `object_id`
* Ruby 有垃圾回收机制

Ruby 操作的都是对象的引用，比如调用方法的参数传递的是引用

{% highlight ruby %}
s = "Ruby" 
t = s      
t[-1] = ""
print s # 输出 "Rub"
t = "Java"
print s, t # 输出 "RubJava"
{% endhighlight %}

`Fixnum` 和 `Symbol` 没有改变其值的方法，所以 `Fixnum` 和 `Symbol` 的对象是不可变的，虽然传的是引用，但你不能在方法中改变其值，感觉就像是传值一样

获取对象的类

{% highlight ruby %}
o = "test"  
o.class     
{% endhighlight %}

判断对象是不是类的实例

{% highlight ruby %}
o.class == String
o.instance_of? String   
{% endhighlight %}

判断对象是类的子类的实例

{% highlight ruby %}
x = 1                    
x.is_a? Fixnum           
x.is_a? Integer          
x.is_a? Numeric          
x.is_a? Comparable       
x.is_a? Object           
{% endhighlight %}

判读对象是否有实现某个方法，只是检测方法名，会忽略掉方法的参数

{% highlight ruby %}
o.respond_to? :"<<"  
{% endhighlight %}

`equal?` 比较两个对象是不是相同的引用，子类不会重写此方法

{% highlight ruby %}
a = "Ruby"       
b = c = "Ruby"   
a.equal?(b) # false
b.equal?(c) # true
{% endhighlight %}

`==` 在 `Object` 类中的实现同 `equal?`，但是子类一般会重写此方法来比较内容

{% highlight ruby %}
a = "Ruby"
b = "Ruby"    
a.equal?(b) # false
a == b # true
{% endhighlight %}

`eql?` 在 `Object` 类中的实现是 `equal?` 的别名方法，通常子类会重写此方法来进行不转换类型的严格比较

{% highlight ruby %}
1 == 1.0 # true
1.eql?(1.0) # false
{% endhighlight %}

`===` 隐式地用在 `case`, `when` 语句中，测试 `case` 中的表达式怎么和 `when` 中值进行比较的

{% highlight ruby %}
(1..10) === 5    # true
/\d+/ === "123"  # true
String === "s"   # true
:s === "s"       # true
{% endhighlight %}

`=~` 是否满足正则表达式

`<=>` 比较方法，运算符左边的与运算符右边的比较，大于返回-1，等与返回0，小于返回+1，实现了此方法的类可以 `include Comparable` 来获取 `<=` `==` 等这些比较方法

`dup` 和 `clone` 都是浅拷贝。`dup` 和 `clone` 分配一个新的对象，拷贝旧对象的实例变量和 `tainted` 的变量到新对象，并且 `clone` 方法还会拷贝旧对象的单例方法和 `frozen` 的变量

`frozen` 后的对象变成不可变对象，不能改变其值

{% highlight ruby %}
s = "ice"      
s.freeze       
s.frozen?      
s.upcase!      # TypeError: 不能修改 frozen 字符串
s[0] = "ni"    # TypeError: 不能修改 frozen 字符串
{% endhighlight %}

当一个对象通过 `taint` 方法被标记为 `tainted`，从该对象派生的任何对象都是 `tainted`

{% highlight ruby %}
s = "untrusted"   
s.taint           
s.tainted?        # true
s.upcase.tainted? # true: 派生的对象也是 tainted
s[3, 4].tainted?  # true: substrings 也是 tainted
{% endhighlight %}
