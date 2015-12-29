---
title: 设计方案：热量助手的日历 
author: 但江
avatar: danjiang
location: 成都 
category: design
---

现有版本的 [热量记录][calorie] 是通过表格分段来显示每一天的热量记录，当记录时间周期已经很长时，要导航到某一天的记录非常的不灵活，很多根据日期来记录信息的应用，为了方便找到特定日期的信息，使用日历是一种很常用的方式。

![Calorie No Calnedar]({{ site.image_base_url }}/calorie1.1-no-calendar.png)

该如何设计这样一个日历呢，首先要明确要解决的问题：

1. 轻松查看最近几天的记录，因为这是最常见的行为；
2. 能够浏览跨月，甚至跨年的记录，可以花费一些努力才能完成，因为不是很频繁地操作，但是要用到时也要很清楚如何完成；
3. 总的来说，界面上要能**显示当前日期**，能够**回到今天**，有**前一天**和**后一天**的导航，有**前一月**和**后一月**的导航

## 第一次尝试

![Calorie Calendar Beta1]({{ site.image_base_url }}/calorie1.1-calendar-beta1.png)

问题：

1. 日历中没有月份的选择
2. 没有回到今天的方式

优点：

1. 很容易地浏览前一天和后一天
2. 标题中显示了当前日期

## 第二次尝试

![Calorie Calendar Beta2]({{ site.image_base_url }}/calorie1.1-calendar-beta2.png)

问题：

1. 日历中没有前一天和后一天的选择
2. 日历中显示的当前日期不明显，没有一目了然的文字
3. 显示和隐藏日历的按钮，其选中状态不明显

优点：

1. 日历中有月份的选择
2. 能够很容易地回到今天
3. 日历中有不同的色块区分今天和当前选中的日期

## 最终方案

![Calorie Calendar Final]({{ site.image_base_url }}/calorie1.1-calendar-final.png)

问题：

1. 导航条缺少标题
2. 显示和隐藏日历的按钮，其选中状态还是不明显

优点：

1. 界面中显示了当前日期
2. 导航条中有4个控件：显示和隐藏日历，回到今天，前一天，后一天
3. 日历中容易选择前一月和后一月，也有不同的色块区分今天和当前选中的日期

[calorie]: http://danthought.com/calorie
