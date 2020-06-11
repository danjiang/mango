---
title: iOS 基础 - 处理屏幕点击事件
author: 但江
avatar: danjiang
location: 成都 
category: programming
tag: ios-basic
---

一些 iOS 基础知识，业务开发中经常用到，面试时也常会被问到，这里总结一下，此篇文章讲解处理屏幕点击事件。

![Apple Develop Xcode](/images/apple-develop-xcode.jpg)

## 概述

点击事件的处理分成两个部分，第一部分是找到包含点击事件的 Responder，第二部分是决定处理点击事件的第一个 Responder。能够处理点击事件的 Responder 都是 UIResponder 的实例。

## 找到包含点击事件的 Responder

UIKit 通过调用 UIView 的 hitTest(_:with:) 方法来找到最深层包含点击事件的 subview：

{% highlight swift %}
func hitTest(_ point: CGPoint, with event: UIEvent?) -> UIView?
{% endhighlight %}

hitTest(_:with:) 的实现是调用每一个 subview 的 point(inside:with:) 确认点击事件是否包含在 subview 中，此方法还会忽略 isHidden = true，isUserInteractionEnabled = false 和 alpha < 0.01 的 subview。

{% highlight swift %}
func point(inside point: CGPoint, with event: UIEvent?) -> Bool
{% endhighlight %}

所以如果要改变找到包含点击事件的 Responder 的流程，需要实现的方法是 point(inside:with:)。

示例 1：扩大按钮的点击区域

{% highlight swift %}
class SearchHistoryExpandButton: UIButton {
    
    override func point(inside point: CGPoint, with event: UIEvent?) -> Bool {
        let newArea = CGRect(x: bounds.origin.x - 5,
                             y: bounds.origin.y - 10,
                             width: bounds.size.width + 10,
                             height: bounds.size.height + 20)
        return newArea.contains(point)
    }
    
}
{% endhighlight %}

示例 2：点击区域限制在四个角

![Square with Corners](/images/square-with-corners.png)

{% highlight swift %}
class PhotoEditorResizeView: UIView {

    override func point(inside point: CGPoint, with event: UIEvent?) -> Bool {
        if bounds.contains(point) {
            for area in caculateThumbAreas().values { // 四个角的 rect
                if area.contains(point) {
                    return true
                }
            }
        }
        return false
    }

}
{% endhighlight %}

## 决定处理点击事件的第一个 Responder

![iOS Responder Chains](/images/ios-responder-chains.png)

决定处理点击事件的第一个 Responder 的流程会顺着上图的实线走，如果 Text Field 不处理这个事件，UIKit 再发给 Text Field 的父 UIView，再给父父 UIView，然后是父父 UIView 的 UIViewController，此 UIViewController 是 UIWindow 的 rootViewController，所以下一步就是此 UIWindow，再下一步就是 UIApplication，如果实现 UIApplicationDelegate 的对象，也是 UIResponder 的实例，那么就会再传递给此 Delegate。

UIButton 即便没有添加 addTarget，也属于处理了点击事件的 Responder：

{% highlight swift %}
button.addTarget(self, action: #selector(onTap), for: .touchUpInside)
{% endhighlight %}

一个子 UIView 添加了 UITapGestureRecognizer，就算是处理了点击事件的 Responder，不会再传递给父 UIView：

{% highlight swift %}
addGestureRecognizer(UITapGestureRecognizer(target: self, action: #selector(onTap)))
{% endhighlight %}