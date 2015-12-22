---
title: Ruby 的类和模块
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: ruby
---

## 类

### 定义类

**class** 关键字创建了一个常量来代表这个类，常量的名称和类的名称一样。在类定义中，在实例方法外，**self** 指代定义的类

{% highlight ruby %}
class Point
end
{% endhighlight %}

**Point** 常量代表我们的新类，所有的类都有一个 **new** 方法，用来创建实例

{% highlight ruby %}
p = Point.new
{% endhighlight %}

**new** 方法创建一个新的对象实例，然后自动调用 **initialize** 初始化实例，将 **new** 方法参数都传递给 **initialize**。**initialize** 是自动被设置成私有方法的。在实例方法内部，**self** 指代定义该方法的类的实例。一个对象的实例变量只能在对象的实例方法中被访问

{% highlight ruby %}
class Point
  def initialize(x,y)
    @x, @y = x, y
  end

  def to_s
    "(#@x,#@y)"
  end
end
{% endhighlight %}

### 定义访问器和属性

{% highlight ruby %}
class Point
  def initialize(x, y)
    @x, @y = x, y
  end

  def x
    @x
  end

  def y
    @y
  end
end
{% endhighlight %}

有了上面定义的访问方法，我们就可以通过对象的实例方法来访问实例变量

{% highlight ruby %}
p = Point.new(1, 2)
q = Point.new(p.x * 2, p.y * 3)
{% endhighlight %}

赋值表达式只会在通过对象调用时，才调用 **setter** 方法，所以在实例方法中写 **x = 2** 不会调用 **setter** 方法，只会创建新的本地变量

**Module** 类定义了 **attr_reader** 和 **attr_accessor** 方法来简化创建访问器，**Class** 类继承了 **Module**，所以所有的类都有这两个方法

{% highlight ruby %}
class Point
  attr_accessor :x, :y # 定义了实例变量的 getter 和 setter 方法
end
{% endhighlight %}

定义一个不可变版本的类

{% highlight ruby %}
class Point
  attr_reader :x, :y # 定义实例变量的只读方法
end
{% endhighlight %}

### 定义运算符

Ruby 的运算符也是方法，所以也可以自定义，**–@** 是一元运算符，反向的意思

{% highlight ruby %}
class Point
  attr_reader :x, :y

  def initialize(x, y)
    @x, @y = x, y
  end

  def +(other)
    Point.new(@x + other.x, @y + other.y)
  end

  def -@
    Point.new(-@x, -@y)
  end

  def *(scalar)
    Point.new(@x * scalar, @y * scalar)
  end
end
{% endhighlight %}

上面定义的 **+** 运算符，没有检查 **other** 是否是 **Point**，有一种检查方法不是检查是否是特定的类，而是检查是否有类似于 **Point** 的行为，也就是检查 **other** 是否是有 **x** 和 **y** 方法，这就是 **duck type** ，如果 *它走路和叫像鸭子，那它就是鸭子*

{% highlight ruby %}
def +(other)
  raise TypeError, "Point-like argument expected" unless
    other.respond_to? :x and other.respond_to? :y
  Point.new(@x + other.x, @y + other.y)
end
{% endhighlight %}

### 定义数组或哈希访问

通过 **[]** 的数组或哈希访问

{% highlight ruby %}
def [](index)
  case index
  when 0 then @x
  when 1 then @y
  when :x then @x
  when :y then @y
  else nil
  end
end
{% endhighlight %}

### 定义遍历

如果要遍历只需要定义 **each**

{% highlight ruby %}
def each
  yield @x
  yield @y
end
{% endhighlight %}

有了这个方法就可以像下面这样遍历

{% highlight ruby %}
p = Point.new(1,2)
p.each { |x| print x } # Prints "12"
{% endhighlight %}

定义了 **each**，只需要**include Enumerable**，就可以拥有20多个遍历的方法，像这样判断一个坐标是不是在原点

{% highlight ruby %}
p.all? { |x| x == 0 }
{% endhighlight %}

### 定义比较

实现 **==** 方法来比较 **Point**

{% highlight ruby %}
def ==(o) 
  if o.is_a? Point
    @x == o.x && @y == o.y
  elsif 
    false 
  end
end
{% endhighlight %}

通常情况下 **eql?** 方法和 **==** 方法相同，但是我们想用 **eql?** 方法来进行更严格的比较

{% highlight ruby %}
def eql?(o)             
  if o.instance_of? Point      
    @x.eql?(o.x) && @y.eql?(o.y)
  elsif
    false
  end
end
{% endhighlight %}

如果你想用自己定义的类的实例来做哈希的键值，就需要实现 **eql?** 方法，因为哈希用此方法来比较两个哈希的键值，但想要用作键值的类还需要根据 **eql?** 比较方法来实现 **hash** 方法

{% highlight ruby %}
def hash
  code = 17
  code = 37 * code + @x.hash
  code = 37 * code + @y.hash
  code
end
{% endhighlight %}

类的实例对象需要排序，就需要实现 **<=>** 方法，左边与右边的比较，大 **-1**，等 **0**，小 **+1**，然后只需要 **include Comparable**，就会获得 **<=** **==** 等这些比较方法

{% highlight ruby %}
def <=>(other)
  return nil unless other.instance_of? Point
  @x ** 2 + @y ** 2 <=> other.x ** 2 + other.y ** 2
end
{% endhighlight %}

### 定义类方法

{% highlight ruby %}
class Point
  attr_reader :x, :y

  def self.sum(*points)
    x = y = 0
    points.each { |p| x += p.x; y += p.y }
    Point.new(x, y)
  end
end
{% endhighlight %}

通过 **class << Point** 来打开一个类的 **eigenclass** 来添加类方法

{% highlight ruby %}
class << Point 
  def sum(*points)
    x = y = 0
    points.each { |p| x += p.x; y += p.y }
    Point.new(x, y)
  end
end
{% endhighlight %}

### 定义常量

{% highlight ruby %}
class Point
  ORIGIN = Point.new(0, 0)
  UNIT_X = Point.new(1, 0)
  UNIT_Y = Point.new(0, 1)
end
{% endhighlight %}

### 定义类变量

**类变量**在类方法，实例方法和类定义本身中可见

{% highlight ruby %}
class Point
  @@n = 0 
  @@totalX = 0
  @@totalY = 0

  def initialize(x, y)
    @x, @y = x, y

    @@n += 1
    @@totalX += x
    @@totalY += y
  end

  def self.report
    puts "Number of points created: #@@n"
    puts "Average X coordinate: #{@@totalX.to_f / @@n}"
    puts "Average Y coordinate: #{@@totalY.to_f / @@n}"
  end
end
{% endhighlight %}

### 定义类实例变量

类也是对象，可以像其他对象一样有实例变量，类的实例变量叫**类实例变量**，在类定义中且在实例方法定义外的实例变量就是类实例变量

{% highlight ruby %}
class Point
  @n = 0
  @totalX = 0
  @totalY = 0

  def initialize(x, y)
    @x, @y = x, y
  end

  def self.new(x, y)
    @n += 1
    @totalX += x
    @totalY += y

    super
  end

  def self.report
    puts "Number of points created: #@n"
    puts "Average X coordinate: #{@totalX.to_f / @n}"
    puts "Average Y coordinate: #{@totalY.to_f / @n}"
  end
end
{% endhighlight %}

## 方法可见度

### Public

* 没有显示的标明 **protected**，**private** 的方法都是 **public**，除了 **initialize** 方法被隐式地标明 **private**
* 定义在类定义之外的 **global** 方法都是定义成 **Object** 的私有实例方法
* **public** 方法可以在任何地方调用

### Private

* 只能在本类和子类的实例方法中调用 **private** 的方法
* 如果有一个 **private** 的方法 **m**，不能这样调用 **o.m** 或 **self.m**

### Protected

* 在本类和子类中调用
* 可以通过类的实例来显示调用

{% highlight ruby %}
class Point
  # Public 方法写在这里

  protected
  # Protected 方法写在这里

  private
  # Private 方法写在这里
end
{% endhighlight %}

**public**，**protected** 和 **private** 只是针对方法，实例变量和类变量都是私有的，常量都是公共的，可以通过 **private_class_method** 将类方法定义为私有方法，比如构建工厂方法的时候

{% highlight ruby %}
private_class_method :new
{% endhighlight %}

## 子类和继承

如果类没有显示的说明父类，那么就是继承于 **Object** 类，在 Ruby 中，**BasicObject** 是最顶层的类

{% highlight ruby %}
class Point3D < Point
{% endhighlight %}

子类会继承父类的**方法**，也有可能重写父类的方法，方法的调用遵循方法名称查找

子类会继承父类的**私有方法**，所以也有可能重写父类的私有方法，通常类定义私有方法是想作为内部的辅助方法，只想自己类用，所以在不了解父类的私有方法实现的时候，最好不要重写父类的私有方法

**super** 调用当前方法同名的父类方法

{% highlight ruby %}
class Point3D < Point
  def initialize(x, y, z)
    super(x, y)
    @z = z
  end
end
{% endhighlight %}

子类会继承父类的**类方法**，也可以重写父类的类方法，通常类方法的调用都通过类名来显示调用，最好不要依赖于继承，就通过定义该类方法的类来调用

**实例变量**不是通过类定义的，所以和子类继承机制没有关系，当赋值给实例变量时，它们就被创建,，同样，**类实例变量**只是代表类的实例变量，也和子类继承机制没有关系

**类变量**被所有类和其子类共享，如果一个子类给已经在父类中存在的类变量赋值，会改变类变量，所有的类和其子类都会使用新的值

{% highlight ruby %}
class A
  @@value = 1
  def A.value
    @@value
  end
end

print A.value # => 1

class B < A
  @@value = 2
end

print A.value # => 2

class C < A
  @@value = 3
end

print B.value # => 3
{% endhighlight %}

常量可以被继承，常量被改变的时候会出现警告，当我们在子类中 **ORIGIN = Point3D.new(0, 0, 0)** 定义相同名称的常量的时候，不是重写了父类的变量，而是定义了一个新的常量，现在就有两个常量 **Point::ORIGIN** 和 **Point3D::ORIGIN**

## 对象的创建和初始化

**new** 方法主要有两个任务，先是给对象分配空间，然后调用 **initialize** 来初始化

工厂方法是以多种方式来创建对象

{% highlight ruby %}
class Point
  def initialize(x, y)
    @x, @y = x, y
  end

  private_class_method :new

  def Point.cartesian(x, y)
    new(x, y)
  end

  def Point.polar(r, theta)
    new(r * Math.cos(theta), r * Math.sin(theta))
  end
end
{% endhighlight %}

**dup**，**clone** 分配一个新的对象，拷贝旧对象的实例变量和 **tainted** 的变量到新对象，并且 **clone** 方法还会拷贝旧对象的单例方法和 **frozen** 的变量，如果定义了 **initialize_copy** ，还会在拷贝对象后，调用此方法

**marshal_dump** 序列化对象的实例变量，**marshal_load** 反序列化

{% highlight ruby %}
class Point
  def initialize(*coords)
    @coords = coords
  end

  def marshal_dump
    @coords.pack("w*")
  end

  def marshal_load(s)
    @coords = s.unpack("w*")
  end
end
{% endhighlight %}

单例模式是一个类只有一个实例对象，通过引用 **Singleton** 模块来构建单例非常方便，会定义一个类方法 **instance** 来返回类的单例对象

{% highlight ruby %}
require 'singleton'

class PointStats
  include Singleton

  def initialize
    @n, @totalX, @totalY = 0, 0.0, 0.0
  end

  def record(point)
    @n += 1
    @totalX += point.x
    @totalY += point.y
  end

  def report
    puts "Number of points created: #@n"
    puts "Average X coordinate: #{@totalX / @n}"
    puts "Average Y coordinate: #{@totalY / @n}"
  end
end

def initialize(x, y)
  @x, @y = x, y
  PointStats.instance.record(self)
end
{% endhighlight %}

## 模块

模块不能被实例化，不能被继承，独立的。因为类是 **Class** 类的实例，模块是 **Module** 类的实例，**Class** 类是 **Module** 类的子类，意味着所有类都是模块

模块用来作为命名空间，模块中可以定义常量

{% highlight ruby %}
module Base64
  DIGITS = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/'

  def self.encode
  end

  def self.decode
  end
end
{% endhighlight %}

嵌套命名空间，嵌套一个类到另外一个类中，只会影响内部类的命名空间，并不会给内部类访问外部类的方法和变量的权利

{% highlight ruby %}
module Base64
  DIGITS = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/'

  class Encoder
    def encode
    end
  end

  class Decoder
    def decode
    end
  end

  # 两个类都可以访问的工具方法
  def Base64.helper
  end
end
{% endhighlight %}

模块中可以定义实例方法，这些实例方法可混合到其他类中。比如说，如果你实现了 **<=>** 方法，混合 **Comparable** 模块就能得到 **<**, **<=** 等方法，**include** 会影响类型检测方法 **is_a?** 和 **===** 的结果，**include** 只能用于模块

{% highlight ruby %}
class Point
  include Comparable
end
{% endhighlight %}

模块可以包含另外一个模块

{% highlight ruby %}
module Iterable
  include Enumerable
  def each
    loop { yield self.next }
  end
end
{% endhighlight %}

**Object.extend** 可以将模块的实例方法变为对象的单例方法，如果接收对象是 **Class** 类的实例，将变为那个类的类方法

{% highlight ruby %}
countdown = Object.new
def countdown.each
  yield 3
  yield 2
  yield 1
end
countdown.extend(Enumerable)
print countdown.sort # 输出 "[1, 2, 3]"
{% endhighlight %}

## 方法的查找

实例方法的查找，针对 **o.m** 的方法查找过程是

1. 首先检测 **o** 是否有单例方法 **m**
2. 检测 **o** 的类是否有实例方法 **m**
3. 检测 **o** 的类所混合的模块是否有实例方法 **m**，按照 **include** 的倒序来查找
4. 按照第 2，3 步的方法来检测 **o** 的所有父类
5. 调用 **method_missing** 方法，查找以上所有路径找到第一个就调用，没有，只有调用 **Kernel** 模块的 **method_missing** 方法

一个详细例子 **message = "hello"; message.world** 我们想在 **String** 实例 **"hello"** 上调用名称为 **world** 的方法，方法查找过程是

1. 检测单例方法，这里没有定义任何单例方法
2. 检测 **String** 类，也没有名称为 **world** 的实例方法
3. 检测 **String** 类所混合的 **Comparable** 和 **Enumerable** 模块，是否有名称为 **world** 方法，仍然没有找到
4. 检测 **String** 的父类 **Object**，**Object** 类也没有 **world** 方法
5. 检测 **Object** 类所混合的 **Kernel** 模块，也没有找到 **world** 方法，继而我们开始切换到查找 **method_missing** 方法
6. 通过上述路径查找 **method_missing** 方法，最后在 **Kernel** 模块中找到该方法，调用其 **method_missing** 方法，会抛出异常 **NoMethodError: undefined method 'world' for "hello":String**

类方法的查找和实例方法的查找很相似，Ruby 在查找一个对象的 **eigenclass** 的单例方法时，会查找 **eigenclass** 的所有父类的单例方法，所以查找 **Fixnum** 的类方法，会先查找 **Fixnum**，**Integer**，**Numeric** 和 **Object** 的单例方法，因为类是 **Class** 的实例，所以再查找 **Class**，**Module**，**Object** 和 **Kernel** 的实例方法

## 常量的查找

{% highlight ruby %}
module Kernel
  # Kernel 中定义的常量
  A = B = C = D = E = F = "defined in kernel"
end

# 全局中定义的常量，也就是定义在 Object 中
A = B = C = D = E = "defined at toplevel"

class Super
  # 父类中定义的常量
  A = B = C = D = "defined in superclass"
end

module Included
  # 模块中定义的常量
  A = B = C = "defined in included module"
end

module Enclosing
  # 外围模块中定义的常量
  A = B = "defined in enclosing module"

  class Local < Super
    include Included

    # 本地中定义的常量
    A = "defined locally"

    # 常量查找的顺序 
    # [Enclosing::Local, Enclosing, Included, Super, Object, Kernel]
    search = (Module.nesting + self.ancestors + Object.ancestors).uniq

    puts A  # 输出 "defined locally"
    puts B  # 输出 "defined in enclosing module"
    puts C  # 输出 "defined in included module"
    puts D  # 输出 "defined in superclass"
    puts E  # 输出 "defined at toplevel"
    puts F  # 输出 "defined in kernel"
  end
end
{% endhighlight %}
