---
title: Objective-C Runtime、对象和消息传递
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: objective-c
---

本文是对 Objective-C 机制的讲解，涉及到 Runtime、对象和消息传递，首先会回顾下 C 语言。随着 Swift 日益成熟，Objective-C 地位在日益下滑，但是并不会完全消失，一是很多已有的大应用都有很多 Objective-C 存量代码，如果要使用 C/C++ 的一些跨平台的库，使用 Objective-C 编写 Objective-C++ 做桥接是唯一的选择。多说一点 Swift 可以直接使用 C 的库。

![Objective C](/images/objective-c.png)

## 回顾 C 语言

Objective-C 是对 C 语言的扩展，主要增加了面向对象的功能，这里罗列一下 C 语言的内容，要详细学习 C 语言，推荐 [C 语言程序设计：现代方法](https://book.douban.com/subject/4279678/)。

* 语法
	* 声明和定义
	* 算术运算符、赋值运算符、位运算符
	* 选择语句、循环语句
	* 基本类型：整数、浮点数、字符，类型转换
	* 数组
	* 指针
	* 字符串
	* 函数
	* 结构
	* 联合
	* 枚举
	* 动态存储分配
* 程序的组成
	* 头文件和源文件
	* 预处理
	* 编译和链接

## Objective-C Runtime

### 消息传递

先来看一下如下 Objective-C 和 C++ 中的方法调用：

{% highlight objc %}
Object *obj = [Object new];
[obj performWith:parameter1 and:parameter2];
{% endhighlight %}

{% highlight cpp %}
Object *obj = new Object();
obj->perform(parameter1, parameter2);
{% endhighlight %}

在 Objective-C 中，方法调用其实是消息传递，都是在 runtime 决定那个实现代码被执行，编译器不关心被发送消息的对象的类型，都是在 runtime 做决定。

而在 C++ 中，方法调用是在编译时决定的，[virtual function](https://www.learncpp.com/cpp-tutorial/122-virtual-functions/) 是通过 virtual table 在运行时决定那个实现代码被执行。

### 内存管理

C 语言中的动态存储分配：

* malloc 函数 —— 分配内存块，但是不对内存块进行初始化。
* calloc 函数 —— 分配内存块，并且对内存块进行清零。
* realloc 函数 —— 调整先前分配的内存块大小。

{% highlight objc %}
NSString *someString = @"The string";
NSString *anotherString = someString;
{% endhighlight %}

上面的代码在内存中的表示如下图：

![NSString](/images/nsstring.png)

栈中的变量在压栈出栈的时候被自动管理，堆中的内存管理被 Objective-C 做了抽象，不需要使用前面描述的 C 语言中的动态存储分配函数，Objective-C runtime 通过自动引用计数（ARC）来管理。

### 通过关联对象给类添加数据

有时候一些类并不是自己定义实现的，没有源代码，不能通过修改源代码的方式添加新实例变量，这里可以通过关联对象来添加：

{% highlight objc %}
void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy);
id objc_getAssociatedObject(id object, const void *key);
void objc_removeAssociatedObjects(id object);
{% endhighlight %}

objc_AssociationPolicy 的值如下，可以和属性中内存管理相对应：

* OBJC_ASSOCIATION_ASSIGN
* OBJC_ASSOCIATION_COPY
* OBJC_ASSOCIATION_COPY_NONATOMIC
* OBJC_ASSOCIATION_RETAIN
* OBJC_ASSOCIATION_RETAIN_NONATOMIC

## Objective-C 对象

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

## Objective-C 消息传递

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
