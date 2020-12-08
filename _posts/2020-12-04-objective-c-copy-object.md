---
title: Objective-C 复制对象
author: 但江
avatar: danjiang
location: 深圳
category: programming
tag: objective-c
---

本文并不是一篇完整的教程，更像一篇快速笔记，讲解 Objective-C 中的复制对象。

![Objective C](/images/objective-c.png)

## 复制对象

### 基础实现

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

如果需要通过拷贝区分得到可变实例和不可变实例，还需要再实现 **NSMutableCopying** 协议：

{% highlight objc %}
- (id)mutableCopyWithZone:(NSZone *)zone
{% endhighlight %}

另外，还需要注意拷贝的实现是**深拷贝**，还是**浅拷贝**。

### 常见问题

- **copy 始终返回不可变实例**
- **mutableCopy 始终返回可变实例**

{% highlight objc %}
-[NSMutableArray copy] => NSArray
-[NSArray mutableCopy] => NSMutableArray
{% endhighlight %}

先看下面的代码：

{% highlight objc %}
@interface MyPerson : NSObject

@property (nonatomic, copy) NSMutableArray *friends;

@end
{% endhighlight %}

{% highlight objc %}
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        MyPerson *person = [MyPerson new];
        
        NSMutableArray *oldFriends = [NSMutableArray arrayWithObjects:@"Mick", @"Lucy", @"Tom", nil];
        person.friends = oldFriends;
        
        [person.friends addObject:@"Jack"]; // crash here
    }
    return 0;
}
{% endhighlight %}

运行后，会 crash：

{% highlight text %}
2020-09-01 09:57:33.425804+0800 OCDemo[1147:20848] -[__NSArrayI addObject:]: unrecognized selector sent to instance 0x10059b890
2020-09-01 09:57:33.430715+0800 OCDemo[1147:20848] *** Terminating app due to uncaught exception 'NSInvalidArgumentException', reason: '-[__NSArrayI addObject:]: unrecognized selector sent to instance 0x10059b890'
{% endhighlight %}

其原因是 oldFriends 赋值给 person.friends 时，因为属性声明为 copy，对 oldFriends 先执行 copy 方法，再赋值，前面说了 copy 始终返回不可变实例，所以这里的 person.friends 底层是 NSArray，再其上执行 addObject 就会 Crash：

{% highlight objc %}
person.friends = [oldFriends copy];
{% endhighlight %}

再修改一下代码，这样的语义对使用者来说没有什么问题，但这可能并不满足需求：

{% highlight objc %}
@interface MyPerson : NSObject

@property (nonatomic, strong) NSMutableArray *friends;

@end
{% endhighlight %}

再执行下面的代码会输出 4，是因为 person.friends 和 oldFriends 指向同一个 NSMutableArray：

{% highlight objc %}
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        MyPerson *person = [MyPerson new];
        
        NSMutableArray *oldFriends = [NSMutableArray arrayWithObjects:@"Mick", @"Lucy", @"Tom", nil];
        person.friends = oldFriends;
        
        [person.friends addObject:@"Jack"];
        
        NSLog(@"%ld", oldFriends.count);
    }
    return 0;
}
{% endhighlight %}

再修改一下代码，这样的语义对使用者来说没有什么问题，也就不能改变 friends，但这可能并不满足需求，可能还需要修改 friends：

{% highlight objc %}
@interface MyPerson : NSObject

@property (nonatomic, copy) NSArray *friends;

@end
{% endhighlight %}

最后再修改一下代码，setFriends 方法中 friends 的底层实现是 NSArray，因为被 copy 了一次，在其上再调用 mutableCopy 返回可变实例 NSMutableArray，再将其赋值给 _friends，这样也就满足了语义，也不会带来什么问题：

{% highlight objc %}
@interface MyPerson : NSObject

@property (nonatomic, copy) NSMutableArray *friends;

@end
{% endhighlight %}

{% highlight objc %}
@implementation MyPerson

- (void)setFriends:(NSMutableArray *)friends {
    _friends = [friends mutableCopy];
}

@end
{% endhighlight %}