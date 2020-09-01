---
title: Objective-C 的分类，协议和预处理
author: 但江
avatar: danjiang
location: 成都 
category: programming
tag: objective-c
---

本文并不是一篇完整的教程，更像一篇快速笔记，讲解 Objective-C 中的分类，类扩展，协议和预处理。

![Objective C](/images/objective-c.png)

## 分类 Category

分类提供了一个简单的方式，用它可以将类的定义模块化到相关方法的分类中；它还提供了扩展现有类定义的简便方式，并且不必访问类的源代码，也无须创建子类。如下通过分类来扩展 **DTShape** 新增 **sum** 方法：

{% highlight objc %}
#import "DTShape.h"

@interface DTShape (MathOps)

- (int)sum:(int)length;

@end
{% endhighlight %}

在分类的实现中也可以访问类中属性和方法：

{% highlight objc %}
#import "DTShape+MathOps.h"

@implementation DTShape (MathOps)

- (int)sum:(int)length {
  return self.numberOfSides * length;
}

@end
{% endhighlight %}

要使用分类，需要导入相应的头文件：

{% highlight objc %}
#import "DTShape+MathOps.h"

DTNamedShape *shape = [[DTNamedShape alloc] initWithName:@"square"];
shape.numberOfSides = 4;
[shape sum:4]
{% endhighlight %}

分类中不支持添加实例变量，所以没有办法支撑属性，但是可以通过 Associated Object 来支撑属性，但是不推荐这么做：

{% highlight objc %}
#import <Foundation/Foundation.h>

@interface EOCPerson : NSObject
@property (nonatomic, copy, readonly) NSString *firstName;
@property (nonatomic, copy, readonly) NSString *lastName;

- (id)initWithFirstName:(NSString*)firstName 
            andLastName:(NSString*)lastName;
@end

@implementation EOCPerson
@end
{% endhighlight %}

{% highlight objc %}
@interface EOCPerson (Friendship)
@property (nonatomic, strong) NSArray *friends;
- (BOOL)isFriendsWith:(EOCPerson*)person;
@end


#import <objc/runtime.h>

static const char* kFriendsPropertyKey = "kFriendsPropertyKey";

@implementation EOCPerson (Friendship)

- (NSArray*)friends {
    return objc_getAssociatedObject(self, kFriendsPropertyKey);
}

- (void)setFriends:(NSArray*)friends {
    objc_setAssociatedObject(self, kFriendsPropertyKey, friends, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}

@end
{% endhighlight %}

## 类的扩展 Class Extension

创建一个未命名的分类，且在括号 **()** 之间不指定名字，这是一种特殊的情况，这种特殊的语法定义为类的扩展；未命名分类是非常有用的，因为它们的方法都是私有的，如果需要写一个类，而且属性和方法仅供这个类本身使用，未命名分类比较合适：

{% highlight objc %}
@interface PWAbility ()

@property (copy, nonatomic) NSDictionary *commands;

@end

@implementation PWAbility

- (instancetype)init {
  self = [super init];
  if (self) {
    _commands = @{PWTextCommand.msgType: PWTextCommand.class};
  }
  return self;
}

- (PWCommand *)commandWithData:(NSData *)data {
  NSDictionary *json = [NSJSONSerialization JSONObjectWithData:data options:0 error:nil];
  NSString *msgType = (NSString *)[json valueForKey:@"msgType"];
  if (!msgType) {
    return nil;
  } else {
    Class class = (Class)self.commands[msgType];
    PWCommand<PWCommandReceivable> *command = [class new];
    [command parseData:json];
    return command;
  }
}

@end
{% endhighlight %}

## 深入分类

要理解 Category 的实现，就需要深入研究 Runtime 这部分的源码，这里参考 [深入理解 Objective-C：Category](https://tech.meituan.com/2015/03/03/diveintocategory.html) 这遍文章进行了一些关键信息的摘录：

### 编译 Category

所有的 OC 类和对象，在 Runtime 层都是用 struct 表示的，Category 也不例外，其定义：

{% highlight c %}
typedef struct category_t {
    const char *name;
    classref_t cls;
    struct method_list_t *instanceMethods;
    struct method_list_t *classMethods;
    struct protocol_list_t *protocols;
    struct property_list_t *instanceProperties;
} category_t;
{% endhighlight %}

* Category 和 Class Extension 完全是两个东西，Class Extension 在编译期决议，它就是类的一部分，它伴随类的产生而产生，亦随之一起消亡，但是 Category 则完全不一样，它是在运行期决议的。
* 首先，编译器生成了 Category 中给类添加的实例方法的列表 instanceMethods，类方法的列表 classMethods，协议的列表 protocols，属性的列表 instanceProperties。
* 其次，编译器生成了 Category 本身 OBJC$_CATEGORYMyClass$_MyAddition，并用前面生成的列表来初始化 Category 本身。有一个需要注意到的事实就是 Category 的名字用来给各种列表以及后面的 Category 结构体本身命名，而且有 static 来修饰，所以**在同一个编译单元里我们的 Category 名不能重复，否则会出现编译错误**。
* 编译器在 DATA 段下的 objc_catlist section 里保存了一个大小为 1 的 category_t 的数组L_OBJC_LABELCATEGORY$，当然，如果有多个 Category，会生成对应长度的数组，用于运行期 Category 的加载。

### 加载 Category

* 在 Objective-C Runtime 库被加载过程中，前面编译阶段产生的 category_t 数组被加载和附加到类上，Category 的方法被**放到了新方法列表的前面**，而原来类的方法被放到了新方法列表的后面，这也就是我们平常所说的 Category 的方法会 "覆盖" 掉原来类的同名方法，这是因为运行时在查找方法的时候是顺着方法列表的顺序查找的。
* 附加 Category 到类的工作会先于 +load 方法的执行，所以在类的 +load 方法调用的时候，我们可以调用 Category 中声明的方法。
* +load 的执行顺序是先类，后 Category，而 Category 的 +load 执行顺序是根据编译顺序决定的。
* 但是对于 "覆盖" 掉的方法，则会先找到最后一个编译的 Category 里的对应方法。

### load 和 initialize

{% highlight objc %}
+ (void)load
+ (void)initialize
{% endhighlight %}

* load 在当每个类和 Category 被加载到 Runtime 时运行，且只运行一次。
* initialize 在类第一次被使用时，才会运行，且只运行一次。

## 协议

协议列出了一组方法，有些可以是选择实现，有些是必须实现，如果决定实现特定协议的所有方法，也就意味着遵守这项协议：

1. **@required** 后面的方法是必须实现，默认都是必须实现；
2. **@optional** 后面的方法是选择实现。

{% highlight objc %}
@class PWListener;

@protocol PWListenerDelegate <NSObject>

- (void)listenerDidStartSuccess:(PWListener *)listener;
- (void)listenerDidStartFailed:(PWListener *)listener;
- (void)listener:(PWListener *)listener didConnectDevice:(PWLocalDevice *)device;

@end

@interface PWListener : NSObject

@property (weak, nonatomic) id<PWListenerDelegate> delegate;

- (instancetype)initWithAbility:(PWAbility *)ability port:(NSInteger)port;

- (void)start;

@end
{% endhighlight %}

## 预处理

预处理程序语句使用 **#** 标记，这个符号必须是一行中的第一个非空字符，顾名思义，预处理程序实际上是在分析 Objective-C 程序之前处理这些语句。

**#define** 就是给符号名称指定程序常量：

{% highlight objc %}
#define PI 3.141592654
#define TWO_PI 2 * 3.141592654
{% endhighlight %}

**#import** 导入头文件，**""** 在包含源文件的目录中查找，**<>** 只在特殊的系统头文件目录中寻找包含文件：

{% highlight objc %}
#import "PWAPIController.h"
#import <AFNetworking/AFNetworking.h>
{% endhighlight %}

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
