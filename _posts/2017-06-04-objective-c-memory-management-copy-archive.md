---
title: Objective-C 内存管理，复制和归档
author: 但江
avatar: danjiang
location: 成都
category: programming
tags: objective-c featured
---

本文并不是一篇完整的教程，更像一篇快速笔记，讲解 Objective-C 中的内存管理，复制对象和归档对象。

![Objective C](/images/objective-c.png)

## 内存管理

### MRC

#### 手工管理内存计数

* 当创建对象时，**[Object new]**，初始的引用计数为 1；
* 当创建引用到对象，**[object retain]**，引用计数加 1；
* 当不再需要对象时，**[object release]**，引用计数减 1。

#### 自动释放池

使用 **alloc** 创建一个对象，然后将它作为方法调用的结果返回，尽管方法不再使用这个对象，但是并不能释放它，因为需要将这个对象作为方法的返回值，**NSAutoreleasePool** 类创建的目的就是希望能够解决这个问题，自动释放池可以帮助追踪需要延迟一些时间释放的对象：

{% highlight objc %}
NSAutoreleasePool *pool = [[NSAutoreleasePool alloc] init];
// Code benefitting from a local autorelease pool.
[pool release];
{% endhighlight %}

{% highlight objc %}
return [result autorelease];
{% endhighlight %}

#### 属性的赋值特性

在声明如下的属性时可以采用 **assign**，**retain**，**copy** 这些特性：

{% highlight objc %}
@property (nonatomic, retain) NSMutableArray *data;
{% endhighlight %}

{% highlight objc %}
self.property = newValue;

// assign 直接指向对象引用
property = newValue

// retain 直接指向对象引用，引用计数加 1
if (property != newValue) {
  [property release];
  property = [newValue retain];
}

// copy 拷贝成新的对象，指向这个对象引用
if (property != newValue) {
  [property release];
  property = [newValue copy];
}
{% endhighlight %}

### ARC

ARC 仍然采用了引用计数，不过系统会检测出何时需要保持对象，何时需要自动释放对象，何时需要释放对象。

#### strong 和 weak

**strong** 让编译器保证在事件循环中通过对赋值执行保持操作强属性能够存活下来：

{% highlight objc %}
@property (nonatomic, strong) NSMutableArray *data;
{% endhighlight %}

**weak** 表明当引用的对象释放时，弱变量会被自动设置为 nil：

{% highlight objc %}
@property (nonatomic, weak) NSMutableArray *data;
{% endhighlight %}

注意：属性默认特性是 unsafe_unretained

#### @autoreleasepool

**@autoreleasepool** 这个指令围住的语句块定了自动释放池的上下文，任何在这个上下文中创建的对象都是自动释放的，在自动释放池块结束的时候销毁这些对象：

{% highlight objc %}
int main(int argc, char * argv[]) {
  @autoreleasepool {
    return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
  }
}
{% endhighlight %}

#### 事件循环

iOS 应用运行在事件循环中，为了处理新的事件，系统会创建一个新的自动释放池上下文，调用到应用中的一些方法用于处理事件，再从方法返回，系统会继续等待下一个事件的发生，然而，做这些事件之前，自动释放池上下文已经结束，意味着自动释放池的对象可能会被销毁。

## 复制对象

复制对象需要实现 **NSCopying** 协议：

{% highlight objc %}
#import <Foundation/Foundation.h>

@interface HVUser : NSObject <NSCopying>

@property (nonatomic, copy) NSString *name;
@property (nonatomic, copy) NSString *mobile;
@property (nonatomic) int gender;

@end
{% endhighlight %}

{% highlight objc %}
#import "HVUser.h"

@implementation HVUser

- (id)copyWithZone:(NSZone *)zone {
  HVUser *user = [[[self class] allocWithZone:zone] init];
  user.name = self.name;
  user.mobile = self.mobile;
  user.gender = self.gender;
  return user;
}

@end
{% endhighlight %}

{% highlight objc %}
HVUser *user1 = [HVUser new];
user1.name = @"Jack";

HVUser *user2 = [user1 copy];
user2.name = @"Mick";
{% endhighlight %}

如果需要通过拷贝得到可变实例和不可变实例，还可以再实现：

{% highlight objc %}
- (id)mutableCopyWithZone:(NSZone *)zone {
{% endhighlight %}

**copy** 始终返回不可变实例，**mutableCopy** 始终返回可变实例：

{% highlight objc %}
-[NSMutableArray copy] => NSArray
-[NSArray copy] => NSMutableArray
{% endhighlight %}

## 归档对象

归档对象需要实现 **NSCoding** 协议：

{% highlight objc %}
#import <Foundation/Foundation.h>

@interface HVUser : NSObject <NSCoding>

@property (nonatomic, copy) NSString *name;
@property (nonatomic, copy) NSString *mobile;
@property (nonatomic) int gender;

@end
{% endhighlight %}

{% highlight objc %}
#import "HVUser.h"

@implementation HVUser

- (void)encodeWithCoder:(NSCoder *)coder {
  [coder encodeObject:self.name forKey:@"name"];
  [coder encodeObject:self.mobile forKey:@"mobile"];
  [coder encodeInt:self.gender forKey:@"gender"];
}

- (id)initWithCoder:(NSCoder *)aDecoder {
  self = [super init];
  if (self) {
    self.name = [aDecoder decodeObjectForKey:@"name"];
    self.mobile = [aDecoder decodeObjectForKey:@"mobile"];
    self.gender = [aDecoder decodeIntForKey:@"gender"];
  }
  return self;
}

@end
{% endhighlight %}

{% highlight objc %}
HVUser *user = [HVUser new];
user.name = @"Jack";
user.mobile = @"18722223333";
user.gender = 1;

NSUserDefaults *userDefaults = [NSUserDefaults standardUserDefaults];
[userDefaults setObject:[NSKeyedArchiver archivedDataWithRootObject:user] forKey:@"user"];
[userDefaults synchronize];

NSData *data = [userDefaults valueForKey:@"user"];
HVUser *savedUser = (HVUser *)[NSKeyedUnarchiver unarchiveObjectWithData:data];
{% endhighlight %}
