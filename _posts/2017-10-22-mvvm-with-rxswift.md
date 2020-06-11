---
title: MVVM 模式中结合 RxSwift
author: 但江
avatar: danjiang
location: 成都
category: programming
---

本文首先会分别讲解一下 MVVM 模式和 RxSwift，然后再讲解如何将 MVVM 模式和 RxSwift 进行结合。

![ReactiveX Filter Operator](/images/ReactiveX-Filter-Operator.png)

## MVVM 模式

### MVM 模式

MVC 模式大家应该不陌生了吧，我们来简单回顾一下，View 层渲染视图和处理 UI 事件，用户在 View 层点击了一个下单的按钮，View 层将此点击事件告诉 Controller 层的 ViewController，ViewController 会装配 Model 层的订单数据，然后访问网络接口，也许还需要处理本地存储，来完成下单的业务逻辑，根据处理结果，ViewController 再通知 View 层刷新视图来显示对应的结果。

> MVC 模式中业务逻辑主要放在 ViewController 中。

### MVVM 模式

MVC 模式带来了一些问题，ViewController 承担的任务过于繁重，既然叫 ViewController，我们是不是应该只让它负责控制 View 呢？业务逻辑是我们应用中最关键的部分，我们希望业务逻辑很容易地被测试，也希望业务逻辑能够跨平台复用，首先 ViewController 继承自 UIViewController，似乎不是很容易被测试，如果是 Plain Object 应该要更容易被测试，其次，iOS 平台 ViewController 继承自 UIViewController，macOS 平台 ViewControler 继承自 NSViewController，别忘了还有多 Window 的问题，代码复用起来也会有难度。

我们来看看 MVVC 模式怎么解决这些问题，MVVC 模式比 MVC 模式，多加了一个 ViewModel 层，ViewModel 层处理业务逻辑并且完全独立于视图展现，ViewController 层只负责控制 View，初始化 ViewModel 时传入必要的依赖类，将 ViewModel 和 View 之间绑定起来，View 层和 Model 层并没有什么特别的变化。

> MVVC 模式中业务逻辑主要放在 ViewModel 中。

ViewModel 只是一个 Plain Object，所以可以被更容易的测试，也能够跨平台复用。

## RxSwift

RxSwift 是 Reactive Programming 的一种实现，当然还有其他的实现啰，Reactive Programming 讲究的是数据流的传递，可以并行且独立处理这些数据流，这样可以更好地处理异步编程的问题。什么是数据流，用户输入的文字是一串数据流，从网站上几字节几字节地下载图片也是数据流，用户跑步时 GPS 位置的不断变化也是数据流。

RxSwift 三大基石：Observables，Operators，Schedulers。

### Observables

Observable<T> 是对发射数据流的抽象，只能发射三种数据：next（正常值），completed（结束值），error（错误值）：

{% highlight swift %}
API.download(file: "http://www...")
  .subscribe(onNext: { data in
    ... append data to temporary file
  },
  onError: { error in
    ... display error to user
  },
  onCompleted: {
    ... use downloaded file
  })
{% endhighlight %}

### Operators

Operators 是对数据流进行各种转换、过滤、合并等处理的抽象，通过各种 Operators 组合从而完成更复杂的业务，这种方式也可以更好地复用一条数据流的处理链：

{% highlight swift %}
UIDevice.rx.orientation
  .filter { value in
    return value != .landscape
  }
  .map { _ in
    return "Portrait is the best!"
  }
  .subscribe(onNext: { string in
    showAlert(text: string)
  })
{% endhighlight %}

### Schedulers

Schedulers 是对数据流发射和处理调度的抽象，RxSwift 已经预制了处理 Operation Queue, GCD Queue 和 Thread 的 Scheduler，但不仅仅只是这些，还可以进一步扩展。

## MVVM 模式和 RxSwift 进行结合

### ViewModel

使用 RxSwift 来完成 ViewModel 和 View 的绑定很方便，首先来看一下 ViewModel，其中声明了实现业务逻辑必须的依赖、输出、输入，要分清楚这三部分：

{% highlight swift %}
struct HomeViewModel {

  // 依赖：实现业务逻辑必须的

  let sceneCoordinator: SceneCoordinatorType
  let habitService: HabitService

  // 输出：列表

  var habits: Observable<[Habit]> {
    ...
  }

  // 输入：删除

  func onDelete(habit: Habit) -> CocoaAction {
    ...
  }

  // 输入：修改

  func onUpdateForm(habit: Habit) -> Action<HabitForm, Void> {
    ...
  }

  // 输入：创建

  func onCreateHabit() -> CocoaAction {
    ...
  }

}
{% endhighlight %}

传入必要的依赖来创建 ViewModel：

{% highlight swift %}
let service = HabitService()
let sceneCoordinator = SceneCoordinator(window: window!)
let homeViewModel = HomeViewModel(sceneCoordinator: sceneCoordinator, habitService: service)
{% endhighlight %}

### ViewController

关注 ViewController 中的 bindViewModel 方法如何将 ViewModel 和 View 绑定：

{% highlight swift %}
class HomeViewController: UIViewController, BindableType {

  var viewModel: HomeViewModel!

  fileprivate let progressView = ProgressView()
  fileprivate let tableView = UITableView()
  fileprivate let habitCellIdentifier = "cell"
  fileprivate let bag = DisposeBag()

  override func loadView() {
    super.loadView()

    // 布局的代码
  }

  // 关注这里如何将 ViewModel 和 View 绑定

  func bindViewModel() {
    navigationItem.rightBarButtonItem!.rx.action = viewModel.onCreateHabit()
    viewModel.habits
      .bind(to: tableView.rx.items) { tableView, ip, habit in
        let cell = tableView.dequeueReusableCell(withIdentifier: self.habitCellIdentifier) as! HabitCell
        cell.update(with: habit)
        return cell
    }
    .addDisposableTo(bag)
  }

}
{% endhighlight %}

上面的实现中使用了 RxSwift 中的 Observable，Action 中的 CocoaAction，Action，RxCocoa 中对 UITableView 的扩展，这些工具都是 RxSwift 提供给你的。

### 其他类

MVVM 模式并没有说明如何划分所有代码的职责，比如实现网络请求，界面导航，缓存等代码应该放在哪里呢？都放在 ViewModel 吗？这样不就回到最初那个 Massive View Controller 的问题，你可以根据自己的经验，分离这些代码到 Manager，Service，Helper 等，不知道你怎么取这个名字，然后将这些依赖在 ViewModel 中声明为属性，你可以初始化的时候传入，也可以在生命周期其他阶段传入，ViewModel 充当大脑的角色，指挥和协调这些依赖来完成业务逻辑。

RxSwift 提供的工具当然可以用到这些依赖的实现当中，不是只能用在 ViewModel 和 View 的绑定中。

## 一点感想

MVVM 模式，其他 XX 模式，需要这些编程模式的目的为了分清代码的职责，让它们各司其职，完成工作，也便于我们理解这些代码，一个小组的程序员如果不能对这个模式达成共识，有共同的认知，最终会乱成一团，有些东西是教不会的，只能自己成长和领悟，这也是程序员职业素养的一部分，整体协同能力。

RxSwift 给代码实现方式的冲击还是很大，以前某些功能用 Delegate 可以轻松完成，但是为了 Reactive，不得不做一些额外的工作。
