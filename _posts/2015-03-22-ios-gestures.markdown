---
title: iOS 手势
author: 但江
location: 成都 
category: design
---

iOS 设备从诞生之初就是触摸屏幕，不同于之前通过物理按键来操作的手持设备，也不同于个人电脑上通过物理键盘和鼠标来操作的交互方式，通过手指来触碰屏幕是一种全新的交互方式，手指和鼠标相比略显笨拙，但是直接触碰的方式更容易触动人心，本文将全面介绍 iOS 系统中推荐的标准手势，以及在设计手势操作时需要注意的问题。

#### 标准手势

**单击** —— 点击控件或者选择条目。

![iOS Gestures Tap](/images/ios-gestures-tap.gif)

**拖拽** —— 把一个控件从一个地方拖动到另一个地方。

![iOS Gestures Drag](/images/ios-gestures-drag.gif)

**轻拂** —— 快速地滑动。

![iOS Gestures Flick](/images/ios-gestures-flick.gif)

**滑动** —— 单个手指的滑动可以返回之前的界面，可以在分割视图中显示隐藏视图，或者在表格视图中显示删除按钮，四个手指的滑动可以在 iPad 中切换应用。

![iOS Gestures Swipe](/images/ios-gestures-swipe.gif)

**双击** —— 可以图片或者一块内容上放大或者缩小，还可以居中。

![iOS Gestures Double Tap](/images/ios-gestures-double-tap.gif)

**捏合** —— 向外捏合来放大，向内捏合来缩小。

![iOS Gestures Pinch](/images/ios-gestures-pinch.gif)

**长按** —— 在可编辑或者可选择的文本中，在光标位置显示放大镜。

![iOS Gestures Touch hold](/images/ios-gestures-touch-hold.gif)

**摇动** —— 用在撤销或重做。

![iOS Gestures Shake](/images/ios-gestures-shake.gif)

#### 请注意这些问题

* 尽量避免将标准手势用来表现不标准行为。
* 尽量避免自定义手势来替代标准手势所表现的行为。
* 在不是特别需要的情况下不要自定义手势。
* 为没有发现你应用手势的人，准备好备用方案，所有的手势操作都应该可以通过可见的控件来完成，好比手势是一种键盘快捷方式，不通过快捷方式也能完成任务。
* 对于一些破坏性的行为，应该给手势增加难度，以此来防止意外的误点击发生。
* 为每次点击和操作都提供视觉反馈，然后有节制地加上声效。
