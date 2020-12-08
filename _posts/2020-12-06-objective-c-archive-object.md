---
title: Objective-C 归档对象
author: 但江
avatar: danjiang
location: 深圳
category: programming
tag: objective-c
---

本文并不是一篇完整的教程，更像一篇快速笔记，讲解 Objective-C 中的归档对象。

![Objective C](/images/objective-c.png)

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
