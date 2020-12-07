---
title: Objective-C 对象模型和消息传递
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: objective-c
---

本文是对 Objective-C 机制的讲解，涉及到 Runtime、对象和消息传递，首先会回顾下 C 语言。随着 Swift 日益成熟，Objective-C 地位在日益下滑，但是并不会完全消失，一是很多已有的大应用都有很多 Objective-C 存量代码，如果要使用 C/C++ 的一些跨平台的库，使用 Objective-C 编写 Objective-C++ 做桥接是唯一的选择。多说一点 Swift 可以直接使用 C 的库。

![Objective C](/images/objective-c.png)

## Objective-C 对象模型

### Objective-C 对象的实质

Objective-C 中一切皆对象，如下的代码中，someString 是对象，NSString 是类对象。

{% highlight objc %}
NSString *someString = @"The string";
{% endhighlight %}

someString 对象会参照如下 id 定义的模版进行定义，其本质也是结构体指针：

{% highlight c %}
typedef struct objc_object {
    Class isa; // 是什么类
} *id;
{% endhighlight %}

Class 定义是如下的类对象结构体指针：

{% highlight c %}
typedef struct objc_class *Class;
struct objc_class {
    Class isa; // 是什么类
    Class super_class; // 父类
    const char *name;
    long version;
    long info;
    long instance_size;
    struct objc_ivar_list *ivars; // 实例变量
    struct objc_method_list **methodLists; // 方法列表
    struct objc_cache *cache;
    struct objc_protocol_list *protocols;
}
{% endhighlight %}

综合起来，类关系图如下：

![Objective-C Class Hierarchy](/images/objective-c-class-hierarchy.png)

由上图可见，类方法就是定义在 metaclass 中的实例方法。

检测类层级的方法：

{% highlight objc %}
- (BOOL)isKindOfClass:(Class)aClass;
- (BOOL)isMemberOfClass:(Class)aClass;
{% endhighlight %}

### Objective-C 对象的内存分布

{% highlight objc %}
NSString *someString = @"The string";
NSString *anotherString = someString;
{% endhighlight %}

上面的代码在内存中的表示如下图：

![NSString](/images/nsstring.png)

栈中的变量在压栈出栈的时候被自动管理，堆中的内存管理被 Objective-C 做了抽象，不需要使用 [开始学习 Objective-C - 动态存储分配](/programming/2020/12/01/get-started-with-objective-c#动态存储分配) 中描述的 C 语言的动态存储分配函数，而是 [Objective-C 内存管理](/programming/2020/12/04/objective-c-memory-management/) 中描述的自动引用计数 ARC 来管理。

## Objective-C 消息传递

### 方法派遣

先来看一下 Objective-C 和 C++ 中的方法调用：

{% highlight objc %}
Object *obj = [Object new];
[obj performWith:parameter1 and:parameter2];
{% endhighlight %}

{% highlight cpp %}
Object *obj = new Object();
obj->perform(parameter1, parameter2);
{% endhighlight %}

在 Objective-C 中，方法调用的本质是消息传递，都是在运行时决定那个实现代码被执行，编译器不关心被发送消息的对象类型，都是在运行时做决定。

而在 C++ 中，方法调用是在编译时决定的，[Virtual Function](https://www.learncpp.com/cpp-tutorial/122-virtual-functions/) 是通过 Virtual Table 在运行时决定那个实现代码被执行。

### objc_msgSend

{% highlight objc %}
Object *obj = [Object new];
[obj performWith:parameter1 and:parameter2];
{% endhighlight %}

上面的方法调用，都会被编译器转化为 C 函数 objc_msgSend：

{% highlight c %}
void objc_msgSend(id self, SEL cmd, ...)
{% endhighlight %}

id 就是消息的接收者，SEL 就是对方法的封装，... 就是参数值

objc_msgSend 根据 id 和 SEL 找到正确的方法然后调用，objc_msgSend 会把这个查找结果缓存起来，下次就不用找了。

### 动态方法解析

接着上面的说，objc_msgSend 根据 id 和 SEL 没有找到正确的方法，就会进行如下 3 步的动态方法解析：

{% highlight objc %}
+ (BOOL)resolveInstanceMethod:(SEL)sel;
{% endhighlight %}

{% highlight objc %}
- (id)forwardingTargetForSelector:(SEL)aSelector;
{% endhighlight %}

{% highlight objc %}
- (void)forwardInvocation:(NSInvocation *)anInvocation;
{% endhighlight %}

### Method Swizzling

Objective-C 中的类都有一个如下列表，表示方法名到方法实现的对应：

![Method Swizzling Before](/images/method-swizzling-before.png)

经过如下代码的转换，lowercaseString 和 uppercaseString 对应实现进行了交换：

{% highlight objc %}
Method originalMethod = class_getInstanceMethod([NSString string], @selector(lowercaseString));
Method swappedMethod = class_getInstanceMethod([NSString string], @selector(uppercaseString));

method_exchangeImplementations(originalMethod, swappedMethod);
{% endhighlight %}

![Method Swizzling After](/images/method-swizzling-after.png)