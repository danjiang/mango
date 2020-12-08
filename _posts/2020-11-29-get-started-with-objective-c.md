---
title: 开始学习 Objective-C
author: 但江
avatar: danjiang
location: 深圳
category: programming
tag: objective-c
---

随着 Swift 日益成熟，Objective-C 地位在日益下滑，但是并不会完全消失，一是很多已有的大应用都有很多 Objective-C 存量代码，如果要使用 C/C++ 的一些跨平台的库，使用 Objective-C 编写 Objective-C++ 做桥接是唯一的选择。多说一点 Swift 可以直接使用 C 的库。本文并不是一篇完整的教程，更像一篇快速笔记，既会回顾 C 语言，也会讲解 Objective-C 中的扩展部分：声明 declar 和定义 define，基本数据类型、对象封装的基本数据类型、集合类、面向对象和其他杂项。

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
	* 结构体
	* 联合
	* 枚举
	* 动态存储分配
* 程序的组成
	* 头文件和源文件
	* 预处理
	* 编译和链接

## 声明 declar 和定义 define

{% highlight c %}
声明说明符 声明符;
{% endhighlight %}

声明为编译器提供有关于标识符含义的信息，声明说明符分为 3 大类：

1. 存储类型。
2. 类型限定符。
3. 类型说明符，比如 char, short, int, long 等。

### 存储类型

存储类型可以用于变量以及较小范围的函数和形式参数的说明，C 和 Objective-C 程序中的每一个变量都具有以下 3 个性质：

1. 存储期限 Storage Duration：变量的存储期限决定了为变量预留和内存被释放的时间。
2. 作用域 Scope：变量的作用域是指可以引用变量的那部分程序文本。
3. 链接 Linkage：变量的链接确定了程序的不同部分可以共享此变量的范围。

#### auto 存储类型

**auto** 存储类型的变量具有自动存储期限，块作用域，并且无链接。

{% highlight c %}
int i; // 静态存储期限，文件作用域，外部链接

void f(void) {
  int j; // 自动存储期限，块作用域，无链接
}
{% endhighlight %}

#### static 存储类型

**static** 存储类型用在块外部时，说明变量具有内部链接，本质上使变量只在声明它的文件内可见；当用在块内部时，static 把变量的存储期限从自动的变成了静态的。

{% highlight c %}
static int i; // 静态存储期限，文件作用域，内部链接

void f(void) {
  static int j; // 静态存储期限，块作用域，无链接
}
{% endhighlight %}

#### extern 存储类型

**extern** 存储类型使几个源文件可以共享同一个变量，下面的语句不会导致编译器为变量 i 分配存储单元，它只是提示编译器需要访问定义在别处的变量，可能稍后在同一个文件中，更常见的是在另一个文件中。

{% highlight c %}
extern int i; // 静态存储期限，文件作用域，什么链接？
{% endhighlight %}

#### register 存储类型

**register** 存储类型就是要求编译器把变量存储在寄存器中。

### 类型限定符

#### const

**const** 用来声明一些变量是**只读的**。

通过 **static** 和 **const** 定义在此文件内部使用的常量：

{% highlight objc %}
static NSString * const PWHeaderCRLF = @"\r\n";
{% endhighlight %}

通过 **extern** 和 **const** 定义在此文件外部可使用的常量：

{% highlight objc %}
// .h
extern NSString * const HVBaseURL;

// .m
NSString * const HVBaseURL = @"https://example.com/api/";
{% endhighlight %}

## 基本数据类型

### C 定义的基本数据类型

{% highlight objc %}
int someInteger = 42;
float someFloatingPointNumber = 3.1415;
double someDoublePrecisionFloatingPointNumber = 6.02214199e23;
char ch = 'a';
{% endhighlight %}

### Objective-C 定义的基本数据类型

注意 NSInteger、NSUInteger 和 CGFloat 在不同 CPU 架构上精度会有所不同。

{% highlight objc %}
BOOL isGood = YES;
NSInteger negative = -123;
NSUInteger positive = 123;
CGFloat width = 100.0;
{% endhighlight %}

## 对象封装的基本数据类型

### 字符串

可以使用 C 语言中用 **指针指向字符数组** 的方法来使用字符串：

{% highlight c %}
char *name = "John Smith";
printf("My name is %s", name);
{% endhighlight %}

Objective-C 中用 **NSString** 封装了字符串类型为对象的方式：

{% highlight objc %}
NSString *firstString = [[NSString alloc] initWithCString:"Hello World!"
                                                 encoding:NSUTF8StringEncoding];
NSString *secondString = [NSString stringWithCString:"Hello World!"
                                            encoding:NSUTF8StringEncoding];
NSString *thirdString = @"Hello World!";
{% endhighlight %}

### 数字对象

NSNumber

{% highlight objc %}
NSNumber *magicNumber = @42;
NSNumber *unsignedNumber = @42u;
NSNumber *longNumber = @42l;

NSNumber *boolNumber = @YES;

NSNumber *simpleFloat = @3.14f;
NSNumber *betterDouble = @3.1415926535;

NSNumber *someChar = @'T';
{% endhighlight %}

{% highlight objc %}
int scalarMagic = [magicNumber intValue];
unsigned int scalarUnsigned = [unsignedNumber unsignedIntValue];
long scalarLong = [longNumber longValue];

BOOL scalarBool = [boolNumber boolValue];

float scalarSimpleFloat = [simpleFloat floatValue];
double scalarBetterDouble = [betterDouble doubleValue];

char scalarChar = [someChar charValue];
{% endhighlight %}

### 值对象

NSNumber 是 NSValue 的子类，NSValue 可以封装 C 基本数据类型，还可以封装指针和结构体。

{% highlight objc %}
NSString *mainString = @"This is a long string";
NSRange substringRange = [mainString rangeOfString:@"long"];
NSValue *rangeValue = [NSValue valueWithRange:substringRange];
{% endhighlight %}

{% highlight objc %}
typedef struct {
    int i;
    float f;
} MyIntegerFloatStruct;

struct MyIntegerFloatStruct aStruct;
aStruct.i = 42;
aStruct.f = 3.14;

NSValue *structValue = [NSValue value:&aStruct
                         withObjCType:@encode(MyIntegerFloatStruct)];
{% endhighlight %}

## 集合类

### 数组

可以使用 C 语言的数组：

{% highlight c %}
int numbers[3];

numbers[0] = 10;
numbers[1] = 20;
numbers[2] = 30;

for (int i = 0; i < 3; i++) {
  printf("%d\n", numbers[i]);
}
{% endhighlight %}

Objective-C 中使用 **NSArray** 实现数组，**NSArray** 中只能装对象，所以对于基本数字类型要用 **NSNumber** 封装为对象，所以要采用下面的形式：

{% highlight objc %}
NSArray *numbers = @[@1, @2, @3];

for (NSNumber *number in numbers) {
  NSLog(@"%d", number.intValue);
}

id firstObject = @"someString";
id secondObject = nil;
id thirdObject = @"anotherString";

NSArray *someArray = @[firstObject, secondObject, thirdObject];
{% endhighlight %}

### 集合

NSSet

{% highlight objc %}
NSSet *simpleSet = [NSSet setWithObjects:@"Hello, World!", @42, aValue, anObject, nil];
{% endhighlight %}

### 字典

NSDictionary

{% highlight objc %}
NSDictionary *dictionary = @{
    @"anObject" : someObject,
    @"helloString" : @"Hello, World!",
    @"magicNumber" : @42,
    @"aValue" : someValue
};
{% endhighlight %}

### 可变类

上面的 NSArray、NSSet 和 NSDictionary 都是创建的时候给定其内容，之后就不能再修改了，其可变类是 NSMutableArray、NSMutableSet 和 NSMutableDictionary。

{% highlight objc %}
NSMutableDictionary *dictionary = [[NSMutableDictionary alloc] init];
[dictionary setObject:@"another string" forKey:@"secondString"];
[dictionary removeObjectForKey:@"anObject"];
{% endhighlight %}

## 面向对象

### 类和对象

定义了一个 **DTShape** 类，有一个属性是 **numberOfSides**，还有一个实例方法 **simpleDescription**：

> 开头为 **+** 是类方法
>
> 开头为 **-** 是实例方法

{% highlight objc %}
#import <Foundation/Foundation.h>

@interface DTShape : NSObject

@property (nonatomic) int numberOfSides;

- (NSString *)simpleDescription;

@end
{% endhighlight %}

{% highlight objc %}
#import "DTShape.h"

@implementation DTShape

- (NSString *)simpleDescription {
  return [NSString stringWithFormat:@"A shape with %d sides", self.numberOfSides];
}

@end
{% endhighlight %}

使用 **new** 创建一个对象，设置属性 **numberOfSides** 为 **4**，再调用 **simpleDescription** 方法：

{% highlight objc %}
DTShape *shape = [DTShape new];
shape.numberOfSides = 4;
NSString *description = [shape simpleDescription];
{% endhighlight %}

### 属性

属性是对实例变量访问的抽象，类似于 Java 中 getter 和 setter，如下的代码会自动添加两个实例变量 _threshold 和 _quantizationLevels 来作为属性的实现：

{% highlight objc %}
@interface VideoToonFilter : VideoFilter

@property (nonatomic, assign) float threshold;
@property (nonatomic, assign) float quantizationLevels;

- (instancetype)init;

@end
{% endhighlight %}

#### @dynamic

@dynamic 关键字告诉编译器不要自动添加实例变量作为属性的实现，实现会在运行时找到。

#### 原子性

默认情况下，属性的实现都会加锁来保证原子性，也就自带 atomic，声明成 nonatomic 就不保证原子性。

#### 可读可写

默认情况下，属性的实现都是可读可写的，也就是自带 readwrite，声明成 readonly 就是只读的。

#### 内存管理

* assign：只是赋值。
* strong：retain 新值，release 旧值。
* weak：像 assign 只是赋值，但是指向的对象被销毁时，会赋值为 nil。
* unsafe_unretained：像 assign 只是赋值，但是指向的对象被销毁时，不会赋值为 nil。
* copy：拷贝一次。

#### 方法名

* getter=\<name\> 指定 getter 的名称。
* setter=\<name\> 指定 setter 的名称。

### 继承

通过继承 **DTShape** 来声明了类 **DTNamedShape**，其中新增了属性 **name**，新增了初始化方法 **initWithName**，覆盖了实例方法 **simpleDescription**：

{% highlight objc %}
#import "DTShape.h"

@interface DTNamedShape : DTShape

@property (nonatomic, copy) NSString *name;

- (instancetype)initWithName:(NSString *)name;

@end
{% endhighlight %}

{% highlight objc %}
#import "DTNamedShape.h"

@implementation DTNamedShape

- (instancetype)initWithName:(NSString *)name {
  self = [super init];
  if (self) {
    self.name = name;
  }
  return self;
}

- (NSString *)simpleDescription {
  return [NSString stringWithFormat:@"A %@ with %d sides", self.name, self.numberOfSides];
}

@end
{% endhighlight %}

### id

**id** 数据类型可存储任何类型的对象，它是一般对象类型。

{% highlight objc %}
id s = [DTShape new];
{% endhighlight %}

### @class

**@class** 指令提高了效率，因为编译器不需要引入和处理整个 **DTShape.h** 文件，只需要知道 **DTShape** 是一个类名：

{% highlight objc %}
@class DTShape;
{% endhighlight %}

## 其他

### typedef

使用 **typedef** 定义一个新类型名，可按照下面的步骤：

1. 像声明所需类型的变量那样写一条语句；
2. 在通常应该出现声明的变量名的地方，将其替换为新的类型名；
3. 在语句的前面加上关键字 **typedef**。

{% highlight objc %}
typedef NSNumber *NumberObject;

NumberObject myValue1, myValue2, myResult;
{% endhighlight %}

### 枚举

Objective-C 中通过 **NS_ENUM** 定义枚举，不过只能是整数类型：

{% highlight swift %}
typedef NS_ENUM(NSInteger, HVRegisterType) {
  HVRegisterTypeRegister = 0,
  HVRegisterTypeForgetPassword = 1,
  HVRegisterTypeResetPassword = 2
};
{% endhighlight %}

C 语言中定义枚举：

{% highlight c %}
typedef enum {CLUBS, DIAMONDS, HEARTS, SPADES} Suit;
Suit s1, s2;
{% endhighlight %}

### 联合

Objective-C 声明和使用的联合就是 C 的语法：

{% highlight c %}
union {
  int i;
  double d;
} u;
{% endhighlight %}

{% highlight c %}
struct {
  int i;
  double d;
} s;
{% endhighlight %}

内存中的分布：

![C Union Struct](/images/c-union-struct.png)

### 结构体

Objective-C 声明和使用的结构体就是 C 的语法：

{% highlight objc %}
struct MyPoint {
  int x;
  int y;
};
typedef struct MyPoint MyPoint;

MyPoint p = { .x = 10, .y = 20 };

printf("(%d, %d)\n", p.x, p.y);
{% endhighlight %}

### 块

块是 Objective-C 对 C 语言的一种扩展，块定义在函数或者方法内部，并能够访问在函数或者方法范围内块之外的任何变量，一般不能够改变这些变量的值，只有添加 **__block** 在变量前面才可以：

{% highlight objc %}
NSString* (^calculateTriangularNumber) (int) = ^NSString* (int n) {
  int i, triangularNumber = 0;
  
  for (i = 1; i <= n; i++) {
    triangularNumber += i;
  }
  
  return [NSString stringWithFormat:@"Triangular number %i is %i", n, triangularNumber];
};

NSString *result = calculateTriangularNumber(5);
{% endhighlight %}

通过 typedef 定义类型：

{% highlight objc %}
@interface DTAlertView : UIView

typedef void (^alertViewCompletion) (BOOL finished);

- (void)showInView:(UIView *)view completion:(alertViewCompletion)completion;

@end
{% endhighlight %}

### 函数

Objective-C 声明和使用的函数就是 C 的语法：

{% highlight objc %}
int foo(int bar) {
  return bar * 2;
}

foo(1);
{% endhighlight %}

### 指针

Objective-C 声明和使用的指针就是 C 的语法，指针就是一个整数变量，里面装着指向某个值的内存地址，**&** 是地址运算符，**\*** 是间接寻址运算符：

{% highlight objc %}
int count = 10;

int *intPtr = &count;

int x = *intPtr;
{% endhighlight %}

### 动态存储分配

C 语言中的动态存储分配：

* malloc 函数 —— 分配内存块，但是不对内存块进行初始化。
* calloc 函数 —— 分配内存块，并且对内存块进行清零。
* realloc 函数 —— 调整先前分配的内存块大小。

### 预处理

预处理程序语句使用 **#** 标记，这个符号必须是一行中的第一个非空字符，顾名思义，预处理程序实际上是在分析 Objective-C 或者 C 程序之前处理这些语句。

#### 宏定义

使用 **#define** 就是给符号名称指定程序常量：

{% highlight objc %}
#define PI 3.141592654
#define TWO_PI 2 * 3.141592654
{% endhighlight %}

#### 条件编译

**#ifdef**，**#ifndef**，**#endif** 是条件编译：

{% highlight objc %}
#ifdef __OBJC__
  #import <Foundation/Foundation.h>
  #import <UIKit/UIKit.h>
#endif

#ifndef __OBJC__
  #if USE_IL2CPP_PCH
    #include "il2cpp_precompiled_header.h"
  #endif
#endif
{% endhighlight %}

在 Xcode -> Target -> Build Settings 中设置：

![Objective C Preprocessor Macros](/images/objective-c-preprocessor-macros.png)

#### 头文件

Objective-C 中使用 **#import** 导入头文件，**\"\"** 在包含源文件的目录中查找，**<>** 在系统头文件目录中寻找包含文件：

{% highlight objc %}
#import "PWAPIController.h"
#import <AFNetworking/AFNetworking.h>
{% endhighlight %}

C 中使用 **#include** 导入头文件，**\"\"** 在包含源文件的目录中查找，**<>** 在系统头文件目录中寻找包含文件，通常把系统头文件保存在目录 **/usr/include** 中：

{% highlight c %}
#include "decode_audio.h"
#include <stdio.h>
{% endhighlight %}

## 参考资料

* [Programming with Objective-C](https://developer.apple.com/library/mac/#documentation/Cocoa/Conceptual/ProgrammingWithObjectiveC/Introduction/Introduction.html)
* [Google Objective-C Style Guide](http://google.github.io/styleguide/objcguide.html)
* [C 语言程序设计：现代方法](https://book.douban.com/subject/4279678/)

