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

## 分类

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

## 类的扩展

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
