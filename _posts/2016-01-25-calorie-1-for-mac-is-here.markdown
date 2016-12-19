---
title: 热量助手 Mac 版来了
author: 但江
avatar: danjiang
location: 成都 
category: business
---

经过一段时间的努力，热量助手 Mac 版正式上线了，你可以在 Mac 电脑上查询食物热量和运动消耗，同时也可以记录每天的食物热量，当然还可以配合 iPhone 和 iPad 版本的热量助手一起使用。

## 看看长什么样吧

每日热量统计和记录，还有每日所需基本热量。

![Calorie Mac V1 Log](/images/calorie-mac-v1-log.jpg)

浏览食物分类、锻炼分类和品牌食物。

![Calorie Mac V1 Browse](/images/calorie-mac-v1-browse.jpg)

通过名称来搜索食物或锻炼，搜索记录会保留。

![Calorie Mac V1 search](/images/calorie-mac-v1-search.jpg)

记录食物热量和运动消耗。

![Calorie Mac V1 Create](/images/calorie-mac-v1-create.jpg)

在所有设备中同步你的热量记录。

![Calorie Mac V1 iCloud](/images/calorie-mac-v1-icloud.jpg)

## 背后的小故事

### 应用图标

Mac 应用的图标并不是完全的扁平化感觉，在 Yosemite 过后，OS X 整体的风格更向 iOS 风格靠近，看下 OS X 系统中自带应用的图标。

![OS X HIG Desiging App Icons](/images/OS-X-HIG-Designing-App-Icons.jpg)

健康日记的图标。

![Calorie Mac iOS Icon](/images/calorie-mac-ios-icon.png)

### UI 设计

这是我第一次制作 Mac 应用，你可以看下面三张图，我在用纸和笔绘制草图时候，可以看出基本上还是套用的 iOS 界面设计，但是在实际开发过程中发现这样是行不通的，然后根据 OS X Human Interface Guidelines 中的指导，再在 Xcode 中不断地调整，最后才有应用界面的整体结构，算不上非常满意，但还是蛮不错啦。

![Calorie Mac Draft Browse](/images/calorie-mac-draft-browse.jpg)

![Calorie Mac Draft Search](/images/calorie-mac-draft-search.jpg)

![Calorie Mac Draft Logs](/images/calorie-mac-draft-logs.jpg)

### Cocoa 编程

Cocoa 编程在 UI 组件的使用上和 Cocoa Touch 还是有很多不同的，总体来说要更 Low Level 一些，所以在 iOS 中用的很习以为常的功能，在 OS X 上并没有，比如 backgroundColor，Global Tint Color 之类。Core Data 和 iCloud 的使用并没有非常大的不同。

当然印象最深自然是 Bindings 的使用，举个小例子，在下面的 Controller 中我们有一个 date 表示当前日期。

{% highlight swift %}
import Cocoa

protocol CalendarViewControllerDelegate: class {
  func calendarViewControllerDidChoose(viewController: CalendarViewController)
}

class CalendarViewController: NSViewController {

  weak var delegate: CalendarViewControllerDelegate? = nil

  dynamic var date: NSDate? {
    didSet {
      delegate?.calendarViewControllerDidChoose(self)
    }
  }

}
{% endhighlight %}

下面的日期选择控件中日期应该反映到上面 Controller 的 date 中，date 的值也应该反映到日期选择控件中，比如在日期选择控件刚开始显示的时候，这一切都是由 Bindings 来完成，看下图右侧的配置。

![XIB CalendarViewController](/images/XIB-CalendarViewController.jpg)

## 赶紧下载体验吧

讲了这么多，希望你 [下载体验热量助手 Mac 版][1]。

[1]: http://danthought.com/footprints/cmsn 
