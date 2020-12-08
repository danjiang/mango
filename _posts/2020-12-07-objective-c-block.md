---
title: Objective-C Block
author: 但江
avatar: danjiang
location: 深圳
category: programming
tag: objective-c
---

本文并不是一篇完整的教程，更像一篇快速笔记，讲解 Objective-C 中的 Block。

![Objective C](/images/objective-c.png)

## 什么是 Block

Blocks 是 C 语言的扩充功能，也就是带有自动变量（局部变量）的匿名函数。

### 匿名函数

匿名函数就是不带有名称的函数，C 语言的标准中函数和函数指针都要用到函数名称，所以不允许存在这样的函数：

{% highlight c %}
int func(int count);
int (*funcptr)(int) = &func;
{% endhighlight %}

### 带有自动变量值

C 语言的函数中可能使用的变量：

* 自动变量（局部变量）
* 函数的参数
* 静态变量（静态局部变量）
* 静态全局变量
* 全局变量

其中在函数的多次调用之间能够传递值的变量有：

* 静态变量（静态局部变量）
* 静态全局变量
* 全局变量

响应按钮点击回调的示例：

{% highlight c %}
int buttonId = 0;

void buttonCallback(int event) {
  printf("buttonId:%d event=%d\n", buttonId, event);
}
{% endhighlight %}

如果只有一个按钮，上面代码就没有问题，但是有多个按钮，下面代码就有问题：

{% highlight c %}
void setButtonCallbacks() {
  for (int i = 0; i < BUTTON_MAX; ++i) {
    buttonId = i;
    setButtonCallback(BUTTON_IDOFFSET + i, &buttonCallback);
  }
}
{% endhighlight %}

可以通过创建一个类来封装 buttonId 和回调函数：

{% highlight objc %}
@interface ButtonCallbackObject : NSObject {
    int buttonId_;
}

@end

@implementation ButtonCallbackObject

- (id)initWithButtonId:(int)buttonId {
    self = [super init];
    if (self) {
        buttonId_ = buttonId;
    }
    return self;
}

- (void)callback:(int)event {
    NSLog(@"buttonId:%d event=%d\n", buttonId_, event);
}

@end
{% endhighlight %}

{% highlight objc %}
void setButtonCallbacks() {
    for (int i = 0; i < BUTTON_MAX; ++i) {
        ButtonCallbackObject *callbackObj = [[ButtonCallbackObject alloc] initWithButtonId:i];
        setButtonCallbackUsingObject(BUTTON_IDOFFSET + i, callbackObj);
    }
}
{% endhighlight %}

创建一个类来解决这个问题就搞复杂了，用 Block 更适合解决这个问题，Block 中会捕获外部局部变量，也就是 \"带有自动变量值\" 的含义：

{% highlight objc %}
void setButtonCallbacks() {
    for (int i = 0; i < BUTTON_MAX; ++i) {
        setButtonCallbackUsingBlock(BUTTON_IDOFFSET + i, ^(int event) {
            printf("buttonId:%d event=%d\n", i, event);
        });
    }
}
{% endhighlight %}

## Block 语法

### Block 表达式语法

完整形式：

{% highlight text %}
^ 返回值类型 参数列表 表达式
{% endhighlight %}

{% highlight objc %}
^void (int event) {
    printf("buttonId:%d event=%d\n", i, event);
}
{% endhighlight %}

省略返回值类型：

{% highlight objc %}
^(int event) {
    printf("buttonId:%d event=%d\n", i, event);
}
{% endhighlight %}

省略返回值类型和参数列表

{% highlight objc %}
^ {
    printf("Blocks\n");
}
{% endhighlight %}

### Block 类型变量

声明 Block 类型变量：

{% highlight objc %}
void (^simpleBlock)(void);
{% endhighlight %}

给 Block 类型变量赋值：

{% highlight objc %}
simpleBlock = ^{
    NSLog(@"This is a block");
};
{% endhighlight %}

声明和赋值放在一起：

{% highlight objc %}
void (^simpleBlock)(void) = ^{
    NSLog(@"This is a block");
};
{% endhighlight %}

### 使用 typedef 简化语法

{% highlight objc %}
typedef void (^XYZSimpleBlock)(void);

XYZSimpleBlock anotherBlock = ^{
    ...
};

- (void)beginFetchWithCallbackBlock:(XYZSimpleBlock)callbackBlock {
    ...
    callbackBlock();
}
{% endhighlight %}

### 截获自动变量值

\"带有自动变量值\" 的含义就是 \"截获自动变量值\"，如下示例 Block 会截获外部的 anInteger 为 Block 自己的自动变量，截获后，再修改外部的自动变量，对已截获的自动变量没有影响。

{% highlight objc %}
- (void)testMethod {
    int anInteger = 42;
 
    void (^testBlock)(void) = ^{
        NSLog(@"Integer is: %i", anInteger);
    };
    
    anInteger = 84;
 
    testBlock();
}
{% endhighlight %}

{% highlight text %}
Integer is: 42
{% endhighlight %}

### __block

上面一个示例中截获的自动变量在 Block 内部不能修改其值，要想修改，就需要给截获的自动变量前添加 __block 说明符，说明此自动变量在 Block 的外部和内部共享存储空间，在 Block 外部可以修改：

{% highlight objc %}
__block int anInteger = 42;

void (^testBlock)(void) = ^{
    NSLog(@"Integer is: %i", anInteger);
};

anInteger = 84;

testBlock();
{% endhighlight %}

{% highlight text %}
Integer is: 84
{% endhighlight %}

在 Block 内部也可以修改：

{% highlight objc %}
__block int anInteger = 42;

void (^testBlock)(void) = ^{
    NSLog(@"Integer is: %i", anInteger);
    anInteger = 100;
};

testBlock();
NSLog(@"Value of original variable is now: %i", anInteger);
{% endhighlight %}

{% highlight text %}
Integer is: 42
Value of original variable is now: 100
{% endhighlight %}

### 截获 Objective-C 对象

截获一个可变数组实例的指针，通过指针修改可变数组的内容，并没有改指针，所以没有问题。

{% highlight objc %}
id array = [[NSMutableArray alloc] init];
void (^blk)(void) = ^{
    id obj = [[NSObject alloc] init];
    [array addObject:obj];
}
{% endhighlight %}

但是，如下的赋值就会改变指针的值，就有问题。

{% highlight objc %}
id array = [[NSMutableArray alloc] init];
void (^blk)(void) = ^{
    array = [[NSMutableArray alloc] init];
}
{% endhighlight %}

加上 __block 说明符就没有问题了。

{% highlight objc %}
__block id array = [[NSMutableArray alloc] init];
void (^blk)(void) = ^{
    array = [[NSMutableArray alloc] init];
}
{% endhighlight %}

## Block 实现

### Block 实质

Block 也是 Objective-C 对象，看下图：
* Objective-C 对象都有一个 void *isa 说明对象是什么类；
* invoke 是 C 语言的函数指针，也就 Block 中包裹的代码；
* descriptor 是用来描述这个 Block，其中 size 表明了 Block 的结构体实例的大小。

![Objective-C Block](/images/objective-c-block.png)

\"截获自动变量值\" 意味着执行 Block 语法时，Block 语法表达式所使用的自动变量值被保存到 Block 的结构体实例（即 Block 自身）中，所以不能改动到原有的自动变量值。

添加了 __block 说明符的变量，会变成结构体实例，上图中的 Block 结构体会含有此结构体实例的指针，这样就可以改动到原有的自动变量值。

### Block 存储域

如下图的一个程序的内存空间分布，有 text 程序段、data 数据段、stack 栈和 heap 堆，Block 可以放在如下三个存储域：

* 数据段：Global Block。
* 堆：Heap Block。
* 栈：Stack Block。

![Program Memory Layout](/images/program-memory-layout.png)

Global Block 的产生时机：

* 记述全局变量的地方有 Block 语法。
* Block 语法的表达式中不使用截获的自动变量。

栈上的 Block 会复制到堆的时机：

* 调用 Block 的 copy 实例方法。
* Block 作为函数返回值时。
* 将 Block 赋值给附有 __strong 修饰符 id 类型的类或 Block 类型成员变量时。
* 在方法名中含有 usingBlock 的 Cocoa 框架方法或 GCD 的 API 中传递 Block 时。

### Block 循环引用

如下代码中 self 是一个 UIViewController 的实例，此 UIViewController 强引用 HVUserAgreementViewController，HVUserAgreementViewController 强引用 confirmBlock，confirmBlock 强引用 self，这样形成了一个循环引用，通过 __weak 修饰符来转换 self 为弱引用，解除掉了循环引用关系。

{% highlight objc %}
- (void)openEULAViewController {
    __weak typeof(self) weakSelf = self;
    HVUserAgreementViewController *viewController = [HVUserAgreementViewController new];
    viewController.confirmBlock = ^{
        weakSelf.eulachecker.check = YES;
    };
    [self presentViewController:viewController animated:YES completion:nil];
}
{% endhighlight %}

再来看一个情况，如下的 Block 中有多条语句执行，在语句执行的过程中，weakSelf 都有可能变成 nil：

{% highlight objc %}
- (void)openEULAViewController {
    __weak typeof(self) weakSelf = self;
    HVUserAgreementViewController *viewController = [HVUserAgreementViewController new];
    viewController.confirmBlock = ^{
        [weakSelf doA];
        [weakSelf doB];
    };
    [self presentViewController:viewController animated:YES completion:nil];
}
{% endhighlight %}

进一步修改代码，如下的 Block 中有多条语句执行，在语句执行的过程中，strongSelf 不会变成 nil，因为 __strong 修饰符修饰的变量，再赋值时使引用计数 +1，在 Block 执行结束后，变量被销毁，使引用计数 -1，所以，在语句执行的过程中，strongSelf 不会变成 nil：

{% highlight objc %}
- (void)openEULAViewController {
    __weak typeof(self) weakSelf = self;
    HVUserAgreementViewController *viewController = [HVUserAgreementViewController new];
    viewController.confirmBlock = ^{
        __strong typeof(self) strongSelf = weakSelf;
        [strongSelf doA];
        [strongSelf doB];
    };
    [self presentViewController:viewController animated:YES completion:nil];
}
{% endhighlight %}
