---
title: Objective-C 数据类型
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: objective-c
---

本文是对 Objective-C 数据类型的讲解。随着 Swift 日益成熟，Objective-C 地位在日益下滑，但是并不会完全消失，一是很多已有的大应用都有很多 Objective-C 存量代码，如果要使用 C/C++ 的一些跨平台的库，使用 Objective-C 编写 Objective-C++ 做桥接是唯一的选择。多说一点 Swift 可以直接使用 C 的库。

![Objective C](/images/objective-c.png)

## 基本数据类型

### C 中的基本数据类型

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

### NSNumber

数字对象。

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

### NSString

字符串对象。

{% highlight objc %}
NSString *firstString = [[NSString alloc] initWithCString:"Hello World!"
                                                 encoding:NSUTF8StringEncoding];
NSString *secondString = [NSString stringWithCString:"Hello World!"
                                            encoding:NSUTF8StringEncoding];
NSString *thirdString = @"Hello World!";
{% endhighlight %}

### NSValue

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

### NSArray

数组。

{% highlight objc %}
id firstObject = @"someString";
id secondObject = nil;
id thirdObject = @"anotherString";

NSArray *someArray = @[firstObject, secondObject, thirdObject];
{% endhighlight %}

### NSSet 

集合。

{% highlight objc %}
NSSet *simpleSet = [NSSet setWithObjects:@"Hello, World!", @42, aValue, anObject, nil];
{% endhighlight %}

### NSDictionary

字典。

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

## 参考资料

* [Programming with Objective-C](https://developer.apple.com/library/mac/#documentation/Cocoa/Conceptual/ProgrammingWithObjectiveC/Introduction/Introduction.html)
* [Google Objective-C Style Guide](http://google.github.io/styleguide/objcguide.html)