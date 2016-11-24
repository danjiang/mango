---
title: 健康日记从想法到产品
author: 但江
avatar: danjiang
location: 成都 
category: business
---

之前就已经写过文章 [移动应用从想法到产品][1]，我在工作室设计和开发的移动应用都是按照文中的步骤来的，根据实践经验所总结的，不过最近在设计和开发新的应用 [健康日记][2]，所以记录下设计和开发此应用背后的故事，再对照那篇文章的内容，读者应该会有更深的体会，注意有很多图片。

![Health Diary V1.0](/images/health-diary-v1.0.png)

## 想法

首先，当然要先说说想法的由来，工作室之前已经出品了应用 [每日体重记录][3]，帮助您记录体重和减肥，也有为数不多的用户通过邮件反馈：

> 关于《每日体重记录》软件的小建议
> 
> 开发团队的工作人员你们好。
> 
> 我一直在用贵团队开发的《每日体重记录》。在使用时，发现有两个软件不具备的功能其实很需要。
> 
> 功能1，在日历界直接显示体重数，而不是一个黑点。这样体重的变化会一目了然。
> 
> 功能2，如果能加入一个线条图标分析图就好了。
> 
> 以上仅为愚见。

> 为什么不能像日记样记录每天的情况

用户的邮件反馈，再结合 Google 统计中访问 [每日体重记录][3] 主页所搜索的关键词，就会产生一些零星的想法，我平时都随身携带一个纸质的小笔记本，突然冒出一些想法我就会记录下来，渐渐地就会有一个清晰的思路，以下就是笔记本中的内容：

![Health Diary Idea](/images/health-diary-idea1.jpg)

想法都是很美好的，下面就要认真研究和分析一下那些是目前能够做成产品的，对于血糖，睡眠和咖啡因摄入，需要专业性和更定制化的功能，所以做成独立的应用是比较好的，空气质量和天气对于身体健康有影响，但不是记录和分析的来源，所以剩下的可以融合成产品的就是：体重，饮水，心情和目标达成。

## 故事

有了初步的想法，下面就要通过简要的文字描述用户的使用场景，要涵盖五大要素：人物，事件，时间，地点和起因，故事如下：

> 以日记的方式记录每一天的身体健康状况。
> 
> * 人物：关注身体状况的人。
> * 事件：记录身体状况，关注影响身体健康的外界因素。
> * 时间：随时。
> * 时间：随地。
> * 起因：身体有问题而不得不记录，有健康意识所以从简单的记录开始。
> 
> 故事：小欧是生活在省会城市的青年白领，因为肥胖的原因，导致个人感情和生活事业都不太顺利，小欧自从毕业工作后，饮食不规律，作息时间混乱，平时也是久坐族，缺乏体育锻炼，小欧从而下定决心准备减肥，过一种更健康的生活方式，焕发出新的生命活力，赢取白富美，走上人生巅峰。

## 绘制草图

接下来就通过纸和笔来绘制简单的草图界面，我是通过应用 Marvel 将这些草图串起来，体验下基本的流程和交互，[健康日记的草图界面][4]。

![Health Diary Paper Timeline and Calendar](/images/health-diary-paper-timeline-and-calendar.jpg)

![Health Diary Paper Chart and Settings](/images/health-diary-paper-chart-and-settings.jpg)

![Health Diary Paper Diary and Compose](/images/health-diary-paper-diary-and-compose.jpg)

![Health Diary Paper Edit and Camera](/images/health-diary-paper-edit-and-camera.jpg)

![Health Diary Paper Weight and Water](/images/health-diary-paper-weight-and-water.jpg)

![Health Diary Paper Mood and Target](/images/health-diary-paper-mood-and-target.jpg)

## 高保真

上面的草图是不是很让人很抓狂，接着就要开始用专业工具 Sketch 来设计应用图标和高保真界面。

![Health Diary Paper Icon](/images/health-diary-icon.jpg)

![Health Diary Timeline](/images/health-diary-timeline.jpg)

![Health Diary Calendar and Diary](/images/health-diary-calendar-and-diary.jpg)

![Health Diary Diary With Photo and Weight](/images/health-diary-diary-with-photo-and-weight.jpg)

![Health Diary Diary With Water and Mood](/images/health-diary-diary-with-water-and-mood.jpg)

![Health Diary Diary With Target](/images/health-diary-diary-with-target.jpg)

## 动效设计

针对一些界面的动画效果，通过 Framer Studio 来设计动效，你可以参看文章 [动效设计][5] 来了解更多，目前设计的效果如下：

1. [发布按钮动画效果][6]，点击圆形的加号就可以看到效果。
2. [日记中工具栏动画效果][7]，点击体重，饮水，心情或目标达成的工具栏按钮都可以看到效果。

## 编程和测试

如下的宣传页中所说，我正在用 Swift 编写代码，[关注我们][2]，了解应用的最新动向吧。

![Health Diary Website](/images/health-diary-website.jpg)

[1]: /business/2015/01/17/mobile-app-from-idea-to-product/
[2]: http://danthought.com/health
[3]: http://danthought.com/weight
[4]: http://marvl.in/562fgc
[5]: /design/2015/06/21/build-interaction-and-animation-prototypes/
[6]: http://share.framerjs.com/te7fp470scpf/
[7]: http://share.framerjs.com/lli9qjg9plr0/
