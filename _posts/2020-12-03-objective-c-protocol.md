---
title: Objective-C 的协议
author: 但江
avatar: danjiang
location: 深圳 
category: programming
tag: objective-c
---

本文并不是一篇完整的教程，更像一篇快速笔记，讲解 Objective-C 中的协议。

![Objective C](/images/objective-c.png)

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