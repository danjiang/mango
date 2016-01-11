---
title: 中国首届 Swift 开发者大会
author: 但江
avatar: danjiang
location: 北京 
category: business
---

中国首届 Swift 开发者大会，昨天在北京北航新主楼会议中心召开，坦率地讲，我从来没有参见过这种会议，也是抱着去感受一下的心态，还因为我也用 Swift 编写过两款应用，一款 iPhone 应用 [健康日记][1]，一款即将上线的 Mac 应用 [热量助手][2]，也是抱着交流的心态，才大老远地飞到北京，还好没有冻成狗。

![China Swift Conference 2015]({{ site.image_base_url }}/china-swift-conference-2015.jpg)

地方不太好找，晚来了一会儿，错过了会议的开场白，搜寻了半天才找了个座位，然后第一个 Session 就开始了，大部分 Session 感受不深，函数式编程对我没有什么吸引力，用了较久的 Ruby，map 这些方法在 Ruby 里司空见惯，我不排斥什么编程方式、什么设计模式、什么编程原则，只是不要把这些东西说的太邪乎了，引用一下 **Metaprogramming Ruby** 书中的内容来解释一下，我们可以在意这些，但是不要太在意，较长耐心看。

> When I started learning metaprogramming, it looked like magic. I felt like leaving my usual programming behind to enter a new world—a world that was surprising, exciting, and sometimes a bit scary.
>
> As I finish revising this book, the feeling of magic is still there. However, I realize now that in practice there is no hard line separating metaprogramming from plain old vanilla programming. Metaprogramming is just another powerful set of coding tools that you can wield to write code that’s simple, clean, and well tested.
>
> I’ll go out on a limb to make a bolder assertion: with Ruby, the distinction between metaprogramming and regular code is fuzzy—and ultimately pointless. Once you have an in-depth understanding of the language, you’ll have a hard time deciding which techniques and idioms are “meta” and which ones are plain old programming.
>
> In fact, metaprogramming is so deeply ingrained in Ruby that you can barely write an idiomatic Ruby program without using a few metaprogramming spells. The language actually expects that you’ll tweak the object model, reopen classes, define methods dynamically, and manage scopes with blocks. As Bill might say in a Zen moment, “There is no such thing as metaprogram- ming. It’s just programming all the way down.”
>
> Metaprogramming Is Just Programming

接着说一下，我有感受的 Session。

Chris 通过 Live Coding 的方式展示了 Struct 值引用的有趣用法，比较新颖，Swift 刚出来的时候，看苹果出的语法书，里面介绍有 Struct，第一反应就是这是搞了个 C Struct 的升级版嘛，我去，然后从来没用过，只用 Class，Chris 说大家用类都用的很熟悉了，我们应该尝试探索 Struct 的用法，是个不错的想法。

Greg 讲解了 Swift Style，从第二个 Swift 应用开始我也采用了 [Swift Style Guide][3] ，我是个很在意代码风格的人，空格没有对齐，我都会很不爽，如果不是团队协作写应用，你对这个问题感受或许不太深，团队协作，自然会看别人的代码，你还傻逼地通过注释来解决这个问题吗，那真是傻逼透了，如果都遵循同样的 Code Style，就好比大家都说四川话，交流就会比较畅通了，不过实际实践中，还会有一些人际关系方面的问题。

Onevcat 讲解怎么写好一个第三方库，这是对我最受用的一个 Session，我正打算写一些通用的 UI 控件，对，重复造一些轮子，Onevcat 列举的武田君，不知道是真是假，反正跟我很像，我不太相信别人写的很多库，手机端不像服务端，用户用的手机版本不受你控制，你在应用中用了某个库，结果这丫没有及时地根据 iOS 版本升级库，导致你的应用某些环节在新版本上有问题，这时候你想自己来改库就会很麻烦，自己写的东西，你更熟悉嘛，改动也就灵活很多了，也是像例子中说的那样把做同样事情的代码，通过 CV 大法搞定，所以是否用这个库的问题还要归结到写库的人靠不靠谱，我通常都是因为了解这个人，才会用他写的库，好比 AFNetworking，因为写 Ruby 程序，自然就知道 Heroku，然后就知道了 Mattt Thompson，才知道了 AFNetworking，然后我才放心地开始用，程序员也是要看人的。

会议的总体感受比较像是个学生会搞的活动，经验上还是很欠缺，在寒冷的北京，感觉就像程序员写的代码一样冰冷，会议中交流的机会太少了，只是很短暂利用了 Onevcat 接受采访后的时间，很简短地交流了下，会议中同其他参会者交流完全没有，也不知道怎么开始，不熟嘛，会议举办方应该想明白大家 Face to Face 是为了什么，拍拍照，我去，大家还是多运动减减肥吧，最后说一点，有好些 Session 讲解太泛泛而谈了，这说明两个问题，一是讲解者对这个主题不是非常的熟悉，二是对受众不熟悉，对主题很熟悉，你才明白哪些是基础，哪些是进阶，哪些是关键，哪些自己下来看下就行了，对受众熟悉，你才知道你要讲进阶，还是基础，是实战，还是理论。希望下一次会议更多地促进交流，还有就是 Swift 在实战中运用，不要说太多的语法。

[1]: http://danthought.com/health
[2]: http://danthought.com/calorie
[3]: https://github.com/raywenderlich/swift-style-guide
