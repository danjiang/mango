---
title: iOS 导航
author: 但江
avatar: danjiang
location: 成都 
category: design
---

导航的作用是将应用中不同界面有效地组织起来，对于用户而言，查看内容和完成任务才是最重要的，导航不应该过多地引起用户的注意，而是辅助用户完成自己的目标，本文将全面介绍 iOS 系统中内置的导航方式。

## 不要让用户迷失方向

无论采用那种导航方式，最基本的原则是让用户知道，他在哪里，想去的地方应该怎么去，最好还可以知道是从哪里来的。

## 层级结构

> Use a navigation bar to give users an easy way to traverse a hierarchy of data.

层级结构就如同我们在电脑中经常看到的树形菜单一样，如果界面之间的组织关系是按层级逐渐细分的，就可以采用这种导航方式，如下界面中从所有食物分类，然后浏览到水果及制品，最后浏览到奇异果干，就是逐级深入的方式。

![Calorie Browse Navigation]({{ site.image_base_url }}/calorie-browse1.png)

## 标签栏

> Use a tab bar to display several peer categories of content or functionality.

运用标签栏来区分不同类别的内容或者功能，如下图中内置的时钟应用，提供了四种与时间有关的不同功能，每种功能对应着不同的标签：世界时钟、闹钟、秒表、计时器，每个标签中的功能互不干扰，好像是在运行一个独立的应用。

![Clock]({{ site.image_base_url }}/clock1.png)

## 平铺界面

> Use a page control when each app screen represents an individual instance of the same type of item or page.

适用于用来查看同样类型不同内容的界面，界面数量也不是很多的情况，如下图中内置的 Passbook 应用，展示不同店家的代金券，每张代金券的视觉表现都是相同，但是其中的内容肯定不同，通常每个用户的代金券也不会有很多，所以很适合使用平铺界面的导航方式。

![Passbook]({{ site.image_base_url }}/passbook.png)

## 模态视图

> In general, it`s best to give users one path to each screen.

模态视图不同于前三种导航方式，它会临时地出现在界面上，引起用户的注意，用户还不能分心，必须根据情况进行操作，所以要在合理的情况下使用，否则会让用户心烦意乱，如下图中内置的警告框，模态视图和分享界面，就抓住了用户的注意力，需要用户进行操作。

![Modal Contexts]({{ site.image_base_url }}/modal-contexts.png)
