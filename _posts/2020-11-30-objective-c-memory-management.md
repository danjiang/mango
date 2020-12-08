---
title: Objective-C 内存管理
author: 但江
avatar: danjiang
location: 深圳
category: programming
tag: objective-c
---

本文并不是一篇完整的教程，更像一篇快速笔记，讲解 Objective-C 中的内存管理。

![Objective C](/images/objective-c.png)

## MRC

### 手动引用计数

当创建对象时，初始的引用计数为 1

{% highlight objc %}
[Object new]
{% endhighlight %}

当如下持有对象时，引用计数加 1

{% highlight objc %}
someObject = [object retain]
{% endhighlight %}

当如下不再需要对象时，引用计数减 1

{% highlight objc %}
[object release]
{% endhighlight %}

当使用 **alloc** 创建一个对象，然后将它作为方法调用的结果返回，尽管此方法不再使用这个对象，但是并不能释放它，因为需要将这个对象作为方法的返回值，创建 **NSAutoreleasePool** 类的目的就是希望能够解决这个问题，自动释放池可以帮助追踪需要延迟一些时间释放的对象：

{% highlight objc %}
NSAutoreleasePool *pool = [[NSAutoreleasePool alloc] init];

// Code benefitting from a local autorelease pool.
id result = [self doSomething];

[pool release];
{% endhighlight %}

{% highlight objc %}
- (id)doSomething {
  return [result autorelease];
}
{% endhighlight %}

### 属性

声明属性时使用 **assign**：

{% highlight objc %}
@property (nonatomic, assign) int i;

self.property = newValue;

// assign 直接指向对象引用
property = newValue
{% endhighlight %}

声明属性时使用 **retain**：

{% highlight objc %}
@property (nonatomic, retain) NSMutableArray *data;

// retain 直接指向对象引用，引用计数加 1
if (property != newValue) {
  [property release];
  property = [newValue retain];
}
{% endhighlight %}

声明属性时使用 **copy**：

{% highlight objc %}
@property (nonatomic, copy) NSString *text;

// copy 拷贝成新的对象，指向这个对象引用
if (property != newValue) {
  [property release];
  property = [newValue copy];
}
{% endhighlight %}

## ARC

### 自动引用计数

ARC 仍然采用了引用计数，不过编译器会检测出何时需要保持对象，何时需要自动释放对象，补上这些代码，不能也不需要使用 **retain** 和 **release**，以及 **autorelease**。

**__strong** 修饰符，强引用：

{% highlight objc %}
id __strong obj = [[NSObject alloc] init];
{% endhighlight %}

**__weak** 修饰符，弱引用，弱引用不能持有对象实例，所以下面的代码，编译器会发出警告：

{% highlight objc %}
id __weak obj = [[NSObject alloc] init];
{% endhighlight %}

可以改写成，当指向的对象被释放，obj1 **被置为 nil**：

{% highlight objc %}
id __strong obj0 = [[NSObject alloc] init];
id __weak obj1 = obj0;
{% endhighlight %}

**__unsafe_unretained** 修饰符，不安全的所有权修饰符，同弱引用一样不能持有对象实例，所以下面的代码，编译器会发出警告：

{% highlight objc %}
id __unsafe_unretained obj = [[NSObject alloc] init];
{% endhighlight %}

可以改写成，当指向的对象被释放，obj1 **不会被置为 nil**，所以使用 obj1 时，对象有可能已经被释放，引起 Crash：

{% highlight objc %}
id __strong obj0 = [[NSObject alloc] init];
id __unsafe_unretained obj1 = obj0;
{% endhighlight %}

**__autoreleasing** 和 **@autoreleasepool** 结合使用，**@autoreleasepool** 这个指令围住的语句块决定了自动释放池的上下文，任何在这个上下文中创建的对象都是自动释放的，在自动释放池块结束的时候销毁这些对象：

{% highlight c %}
int main(int argc, char * argv[]) {
  @autoreleasepool {
    id __autoreleasing obj = [[NSObject alloc] init];
  }
}
{% endhighlight %}

### 属性

声明属性时新增 **strong** 会 retain 新值，release 旧值，让编译器保证在事件循环中通过对赋值执行保持操作，强属性能够存活下来：

{% highlight objc %}
@property (nonatomic, strong) NSMutableArray *data;
{% endhighlight %}

声明属性时新增 **weak** 像 assign 只是赋值，但是当引用的对象释放时，弱变量会被自动设置为 nil：

{% highlight objc %}
@property (nonatomic, weak) NSMutableArray *data;
{% endhighlight %}

声明属性时使用 **unsafe_unretained**，注意属性默认特性是 **unsafe_unretained**：

{% highlight objc %}
@property (nonatomic) NSMutableArray *data;

self.property = newValue;

// 像 assign 只是赋值，但是指向的对象被销毁时，不会赋值为 nil
property = newValue
{% endhighlight %}

## 事件循环

iOS 应用运行在事件循环中，为了处理新的事件，系统会创建一个新的自动释放池上下文，调用到应用中的一些方法用于处理事件，再从方法返回，系统会继续等待下一个事件的发生，然而，做这些事件之前，自动释放池上下文已经结束，意味着自动释放池的对象可能会被销毁。参考 [博文 - iOS 基础 - RunLoop 和 定时器](/programming/2020/06/13/ios-basic-runloop-timer)。