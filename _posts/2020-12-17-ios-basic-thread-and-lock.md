---
title: iOS 基础 - 线程和锁
author: 但江
avatar: danjiang
location: 深圳 
category: programming
tag: ios-basic
---

一些 iOS 基础知识，业务开发中经常用到，面试时也常会被问到，这里总结一下，此篇文章讲解线程和锁。

![Apple Develop Xcode](/images/apple-develop-xcode.jpg)

## NSThread

NSThread 是 Foundation 对线程的封装，具体实现一个线程可以继承 NSThread 实现 main 方法，如下示例中 main 方法会读写共享变量 self.shareData.count：

{% highlight objc %}
@interface MyThread : NSThread

@property (nonatomic, strong) MyShareData *shareData;

@end
{% endhighlight %}

{% highlight objc %}
@implementation MyThread

- (void)main {
    for (int i = 0; i < 1000; i++) {
        self.shareData.count++;
    }
    NSLog(@"End in %@ Thread. Count = %d", self.name, self.shareData.count);
}

@end
{% endhighlight %}

共享变量 MyShareData 的实现：

{% highlight objc %}
@interface MyShareData : NSObject

@property (nonatomic, assign) int count;

@end
{% endhighlight %}

{% highlight objc %}
@implementation MyShareData

@end
{% endhighlight %}

如下代码线程 A 和线程 B 各自累加共享变量 1000 次：

{% highlight swift %}
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        MyShareData *shareData = [MyShareData new];
        
        MyThread *aThread = [MyThread new];
        aThread.name = @"A";
        aThread.shareData = shareData;
        
        MyThread *bThread = [MyThread new];
        bThread.name = @"B";
        bThread.shareData = shareData;

        [aThread start];
        [bThread start];
        
        sleep(5);
        
        NSLog(@"End in Main Thread. Count = %d", shareData.count);
    }
    return 0;
}
{% endhighlight %}

输出结果应该是 2000，结果却如下不是 2000，是因为产生了 Race Condition：

{% highlight text %}
NSThreadAndLock[2630:89151] End in A Thread. Count = 1352
NSThreadAndLock[2630:89152] End in B Thread. Count = 1780
NSThreadAndLock[2630:88696] End in Main Thread. Count = 1780
{% endhighlight %}

![Concurrency Race Condition](/images/concurrency_race_condition.png)

## 属性修饰符 atomic

将属性修饰符从 nonatomic 改为 atomic：

{% highlight objc %}
@interface MyShareData : NSObject

@property (atomic, assign) int count;

@end
{% endhighlight %}

运行一下，发现没有解决 Race Condition 的问题：

{% highlight text %}
NSThreadAndLock[2695:93969] End in A Thread. Count = 1159
NSThreadAndLock[2695:93970] End in B Thread. Count = 1885
NSThreadAndLock[2695:93493] End in Main Thread. Count = 1885
{% endhighlight %}

> Properties are **atomic** by default so that synthesized accessors provide robust access to properties in a multithreaded environment—that is, the value returned from the getter or set via the setter is always fully retrieved or set regardless of what other threads are executing concurrently.
>
> ---- Qutoe from [Declared Properties - The Objective-C Programming Language](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjectiveC/Chapters/ocProperties.html)

由此可见，只能保证在读或者在写的过程是原子性的。

## @synchronized

> The @synchronized directive is a convenient way to **create mutex locks on the fly** in Objective-C code. The @synchronized directive does what any other mutex lock would do—it prevents different threads from acquiring the same lock at the same time.
>
> ---- Qutoe from [Synchronization - Threading Programming Guide](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/ThreadSafety/ThreadSafety.html#//apple_ref/doc/uid/10000057i-CH8-SW1)

修改 main 方法如下：

{% highlight objc %}
@implementation MyThread

- (void)main {
    for (int i = 0; i < 1000; i++) {
        @synchronized (self.shareData) {
            self.shareData.count++;
        }
    }
    NSLog(@"End in %@ Thread. Count = %d", self.name, self.shareData.count);
}

@end
{% endhighlight %}

运行一下，发现解决了 Race Condition 的问题：

{% highlight text %}
NSThreadAndLock[2826:104341] End in A Thread. Count = 1656
NSThreadAndLock[2826:104342] End in B Thread. Count = 2000
NSThreadAndLock[2826:103834] End in Main Thread. Count = 2000
{% endhighlight %}

## NSLock

> An NSLock object implements **a basic mutex** for Cocoa applications. The interface for all locks (including NSLock) is actually defined by the NSLocking protocol, which defines the lock and unlock methods. You use these methods to acquire and release the lock just as you would any mutex.
>
> ---- Qutoe from [Synchronization - Threading Programming Guide](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/ThreadSafety/ThreadSafety.html#//apple_ref/doc/uid/10000057i-CH8-SW1)

修改共享变量 MyShareData，在累加前需要获取互斥锁，在累加后需要释放互斥锁：

{% highlight objc %}
@interface MyShareData ()

@property (nonatomic, strong) NSLock *theLock;

@end

@implementation MyShareData

- (instancetype)init {
    self = [super init];
    if (self) {
        _theLock = [[NSLock alloc] init];
    }
    return self;
}

- (void)addSafe {
    [self.theLock lock];
    self.count++;
    [self.theLock unlock];
}

@end
{% endhighlight %}

修改 main 方法如下：

{% highlight objc %}
@implementation MyThread

- (void)main {
    for (int i = 0; i < 1000; i++) {
        [self.shareData addSafe];
    }
    NSLog(@"End in %@ Thread. Count = %d", self.name, self.shareData.count);
}

@end
{% endhighlight %}

运行一下，发现解决了 Race Condition 的问题：

{% highlight text %}
NSThreadAndLock[3057:118234] End in B Thread. Count = 1715
NSThreadAndLock[3057:118233] End in A Thread. Count = 2000
NSThreadAndLock[3057:117777] End in Main Thread. Count = 2000
{% endhighlight %}

## NSRecursiveLock

> The NSRecursiveLock class defines **a lock that can be acquired multiple times by the same thread** without causing the thread to deadlock. A recursive lock keeps track of how many times it was successfully acquired. Each successful acquisition of the lock must be balanced by a corresponding call to unlock the lock. Only when all of the lock and unlock calls are balanced is the lock actually released so that other threads can acquire it.
>
> ---- Qutoe from [Synchronization - Threading Programming Guide](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/ThreadSafety/ThreadSafety.html#//apple_ref/doc/uid/10000057i-CH8-SW1)

{% highlight objc %}
NSRecursiveLock *theLock = [[NSRecursiveLock alloc] init];
 
void MyRecursiveFunction(int value)
{
    [theLock lock];
    if (value != 0)
    {
        --value;
        MyRecursiveFunction(value);
    }
    [theLock unlock];
}
 
MyRecursiveFunction(5);
{% endhighlight %}

## NSConditionLock

> An NSConditionLock object defines **a mutex lock that can be locked and unlocked with specific values.** You should not confuse this type of lock with a condition (see Conditions). The behavior is somewhat similar to conditions, but is implemented very differently.
>
> Typically, you use an NSConditionLock object when threads need to perform tasks in a specific order, such as when one thread produces data that another consumes. While the producer is executing, the consumer acquires the lock using a condition that is specific to your program. (The condition itself is just an integer value that you define.) When the producer finishes, it unlocks the lock and sets the lock condition to the appropriate integer value to wake the consumer thread, which then proceeds to process the data.
>
> The locking and unlocking methods that NSConditionLock objects respond to can be used in any combination. For example, you can pair a lock message with unlockWithCondition:, or a lockWhenCondition: message with unlock. Of course, this latter combination unlocks the lock but might not release any threads waiting on a specific condition value.
>
> ---- Qutoe from [Synchronization - Threading Programming Guide](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/ThreadSafety/ThreadSafety.html#//apple_ref/doc/uid/10000057i-CH8-SW1)

如下代码通过 NSConditionLock 来实现 **生产者消费者问题**：

{% highlight objc %}
typedef NS_ENUM(NSInteger, MyFlag) {
    MyFlagNoData = 0,
    MyFlagHasData = 1
};

@interface MyQueue : NSObject

- (void)putItem:(int)item;
- (int)getItem;

@end
{% endhighlight %}

{% highlight objc %}
@interface MyQueue ()

@property (nonatomic, strong) NSConditionLock *condLock;
@property (nonatomic, strong) NSMutableArray *array;

@end

@implementation MyQueue

- (instancetype)init {
    self = [super init];
    if (self) {
        _array = [NSMutableArray new];
        _condLock = [[NSConditionLock alloc] initWithCondition:MyFlagNoData];
    }
    return self;
}

- (void)putItem:(int)item {
    [self.condLock lock];
    [self.array addObject:@(item)];
    NSLog(@"Put Item %d", item);
    [self.condLock unlockWithCondition:MyFlagHasData];
}

- (int)getItem {
    int item = -1;
    [self.condLock lockWhenCondition:MyFlagHasData];
    if (self.array.count > 0) {
        NSNumber *number = (NSNumber *)self.array.firstObject;
        item = number.intValue;
        [self.array removeObjectAtIndex:0];
    }
    BOOL isEmpty = self.array.count == 0;
    [self.condLock unlockWithCondition:(isEmpty ? MyFlagNoData : MyFlagHasData)];
    return item;
}

@end
{% endhighlight %}

{% highlight objc %}
@interface MyProducer : NSThread

@property (nonatomic, strong) MyQueue *queue;

@end
{% endhighlight %}

{% highlight objc %}
@implementation MyProducer

- (void)main {
    for (int i = 1; i <= 5; i++) {
        sleep(1);
        [self.queue putItem:i];
    }
}

@end
{% endhighlight %}

{% highlight objc %}
@interface MyConsumer : NSThread

@property (nonatomic, strong) MyQueue *queue;

@end
{% endhighlight %}

{% highlight objc %}
@implementation MyConsumer

- (void)main {
    for (int i = 1; i <= 5; i++) {
        NSLog(@"Get Item %d", [self.queue getItem]);
    }
}

@end
{% endhighlight %}

{% highlight objc %}
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        MyQueue *queue = [MyQueue new];
        
        MyConsumer *consumer = [MyConsumer new];
        consumer.name = @"Consumer";
        consumer.queue = queue;
        
        MyProducer *producer = [MyProducer new];
        producer.name = @"Producer";
        producer.queue = queue;

        [consumer start];
        [producer start];
        
        sleep(8);
        
        NSLog(@"End in Main Thread");
    }
    return 0;
}
{% endhighlight %}

得到正确的输出结果：

{% highlight text %}
NSThreadAndLock[5159:279114] Put Item 1
NSThreadAndLock[5159:279113] Get Item 1
NSThreadAndLock[5159:279114] Put Item 2
NSThreadAndLock[5159:279113] Get Item 2
NSThreadAndLock[5159:279114] Put Item 3
NSThreadAndLock[5159:279113] Get Item 3
NSThreadAndLock[5159:279114] Put Item 4
NSThreadAndLock[5159:279113] Get Item 4
NSThreadAndLock[5159:279114] Put Item 5
NSThreadAndLock[5159:279113] Get Item 5
NSThreadAndLock[5159:278682] End in Main Thread
{% endhighlight %}

## NSCondition

> The NSCondition class provides the same semantics as POSIX conditions, but **wraps both the required lock and condition data structures in a single object**. The result is an object that you can lock like a mutex and then wait on like a condition.
>
> ---- Qutoe from [Synchronization - Threading Programming Guide](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/ThreadSafety/ThreadSafety.html#//apple_ref/doc/uid/10000057i-CH8-SW1)

如下代码通过 NSCondition 来实现 **生产者消费者问题**，只改写了 MyQueue 的实现：

{% highlight objc %}
@interface MyQueue ()

@property (nonatomic, strong) NSCondition *cocoaCondition;
@property (nonatomic, strong) NSMutableArray *array;

@end

@implementation MyQueue

- (instancetype)init {
    self = [super init];
    if (self) {
        _array = [NSMutableArray new];
        _cocoaCondition = [NSCondition new];
    }
    return self;
}

- (void)putItem:(int)item {
    [self.cocoaCondition lock];
    [self.array addObject:@(item)];
    NSLog(@"Put Item %d", item);
    [self.cocoaCondition signal];
    [self.cocoaCondition unlock];
}

- (int)getItem {
    [self.cocoaCondition lock];
    while (self.array.count == 0) {
        [self.cocoaCondition wait];
    }
    int item = -1;
    NSNumber *number = (NSNumber *)self.array.firstObject;
    item = number.intValue;
    [self.array removeObjectAtIndex:0];
    [self.cocoaCondition unlock];
    return item;
}

@end
{% endhighlight %}

运行程序，同样能得到正确的输出结果。

通过 POSIX Conditions 来实现 **生产者消费者问题**，请参考 [博文 - iOS 基础 - Posix Thread](/programming/2020/06/17/pthread/)。

## 读写锁

通过 NSLock 和 NSCondition 来实现读写锁，请参考 [博文 - iOS 基础 - 使用互斥锁和条件变量实现读写锁](/programming/2020/12/16/pthread-implement-read-write-lock-with-mutex-and-condition/)。

通过 Dispatch Barrier 解决修改共享数据的 Race Condition，也是一种读写锁的实现，请参考 [博文 - iOS 基础 - GCD 和 Operation](/programming/2020/12/16/pthread-implement-read-write-lock-with-mutex-and-condition/)。