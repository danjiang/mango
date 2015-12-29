---
title: iOS 设计原则 
author: 但江
avatar: danjiang
location: 成都 
category: design
---

总结的一些 Photoshop 设计 iOS 应用的原则，还有一些资源和工具。

## 设计原则

1. Photoshop design document is 72ppi all the time and no color management
2. Design for 1x, and scale up to 2x, no half pixel problem
3. Photoshop object will be export as a image, should be build with large size and convert to smart object, so resize to fit with screen, then add layer style and other generated effects
4. Photoshop save for web with the PNG-24 format
5. No need to use ImageOptim
6. Font size should not below 13px
7. 44pt minimum touch size
8. Set Global Light to 90°/-90°. Sets the light source used for Layer Styles to directly above
9. Resize button with pen tool

## 资源和工具

1. [Colour management and UI design](https://bjango.com/articles/photoshop/)
2. [The iOS Design Guidelines](http://iosdesign.ivomynttinen.com)
3. [iOS 8 GUI PSD](http://www.teehanlax.com/tools/iphone/)
4. [The Ultimate Guide To iPhone Resolutions](http://www.paintcodeapp.com/news/ultimate-guide-to-iphone-resolutions)
5. [App Icon Template](http://appicontemplate.com/ios8)
6. [Retinize It](http://retinize.it)
7. [Skala Preview](https://bjango.com/mac/skalapreview/)

## Sketch

我现在已经完全使用 [Sketch][1] 来做设计一年多的时间了，很多上面提到在用 Photoshop 做设计需要注意的原则和辅助工具，[Sketch][1] 都有内置的功能，使用起来效率更高，所以强烈推荐。

[1]: http://www.bohemiancoding.com/sketch/
