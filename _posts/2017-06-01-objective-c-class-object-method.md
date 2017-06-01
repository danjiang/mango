---
title: Objective-C 的类，对象和方法
author: 但江
avatar: danjiang
location: 成都 
category: programming
tag: objective-c
---

本文并不是一篇完整的教程，更像一篇快速笔记，讲解 Objective-C 中的类，对象和方法，当然还有继承，以及 id，static，@class，const，NS_ENUM 和 typedef。

![Objective C](/images/objective-c.png)

## 类和对象

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

## 继承

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

## 其他

### id

**id** 数据类型可存储任何类型的对象，它是一般对象类型。

{% highlight objc %}
id s = [DTShape new];
{% endhighlight %}

### static

**static** 可以用来定义局部静态变量，针对所有的对象都是使用同一个变量，而不是像实例变量一样，每一个对象都有一个副本。注意：只能在定义静态变量和局部变量的方法中访问这些变量。

{% highlight objc %}
@implementation DTShape

...

- (int)count {
  static int instancesCount = 0;
  instancesCount++;
  return instancesCount;
}

...

@end
{% endhighlight %}

### @class

**@class** 指令提高了效率，因为编译器不需要引入和处理整个 **DTShape.h** 文件，只需要知道 **DTShape** 是一个类名：

{% highlight objc %}
@class DTShape;
{% endhighlight %}

### const

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

### NS_ENUM

定义枚举，不过只能是整数类型：

{% highlight swift %}
typedef NS_ENUM(NSInteger, HVRegisterType) {
  HVRegisterTypeRegister = 0,
  HVRegisterTypeForgetPassword = 1,
  HVRegisterTypeResetPassword = 2
};
{% endhighlight %}

### typedef

使用 **typedef** 定义一个新类型名，可按照下面的步骤：

1. 像声明所需类型的变量那样写一条语句；
2. 在通常应该出现声明的变量名的地方，将其替换为新的类型名；
3. 在语句的前面加上关键字 **typedef**。

{% highlight objc %}
typedef NSNumber *NumberObject;

NumberObject myValue1, myValue2, myResult;
{% endhighlight %}