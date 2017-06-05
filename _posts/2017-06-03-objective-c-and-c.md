---
title: Objective-C 和 C
author: 但江
avatar: danjiang
location: 成都 
category: programming
tag: objective-c
---

本文并不是一篇完整的教程，更像一篇快速笔记，讲解 Objective-C 中的 C 特性，以及 C 语言本身的一些内容，主要关于数组，字符串，函数，块，结构体和指针。

![Objective C](/images/objective-c.png)

## 数组

Objective-C 中用 **NSNumber** 封装了基本数字类型为对象的方式，**NSArray** 中只能装对象，所以要采用下面的形式：

{% highlight objc %}
NSArray *numbers = @[@1, @2, @3];

for (NSNumber *number in numbers) {
  NSLog(@"%d", number.intValue);
}
{% endhighlight %}

仍然可以使用 C 语言的数组：

{% highlight c %}
int numbers[3];

numbers[0] = 10;
numbers[1] = 20;
numbers[2] = 30;

for (int i = 0; i < 3; i++) {
  printf("%d\n", numbers[i]);
}
{% endhighlight %}

## 字符串

Objective-C 中用 **NSString** 封装了字符串类型为对象的方式：

{% highlight objc %}
NSString *name = @"John Smith";
{% endhighlight %}

仍然可以使用 C 语言中用 **指针指向字符数组** 的方法来使用字符串：

{% highlight c %}
char *name = "John Smith";
printf("My name is %s", name);
{% endhighlight %}

## 函数

Objective-C 声明和使用函数，和 C 相比没有什么区别：

{% highlight objc %}
int foo(int bar) {
  return bar * 2;
}

foo(1);
{% endhighlight %}

## 块

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

## 结构体

Objective-C 声明和使用结构体，和 C 相比没有什么区别：

{% highlight objc %}
struct MyPoint {
  int x;
  int y;
};
typedef struct MyPoint MyPoint;

MyPoint p = { .x = 10, .y = 20 };

printf("(%d, %d)\n", p.x, p.y);
{% endhighlight %}

## 指针

Objective-C 声明和使用指针，和 C 相比没有什么区别，指针就是一个整数变量，里面装着指向某个值的内存地址，**&** 是地址运算符，**\*** 是间接寻址运算符：

{% highlight objc %}
int count = 10;

int *intPtr = &count;

int x = *intPtr;
{% endhighlight %}