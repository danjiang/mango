---
title: IGListKit
author: 但江
avatar: danjiang
location: 成都
category: programming
---

本文首先会讲解如何使用 IGListKit，然后讲解 IGListKit 中最核心的两点，一是如何继承 UICollectionViewLayout 实现 UICollectionView 的自定义布局和动态变化的动画，二是如何从 Model 变化中计算出 IndexPath 变化，也就是 Diffing 算法，这样才好 batch updates。

![IGListKit Logo](/images/iglistkit-logo.gif)

## 基础使用

通过以下三篇资料就能够掌握基础的使用，这里主要讲解一些注意事项。

* [Ray Wenderlich's IGListKit Tutorial: Better UICollectionViews](https://www.raywenderlich.com/147162/iglistkit-tutorial-better-uicollectionviews)
* [IGListKit Getting Started guide](https://instagram.github.io/IGListKit/getting-started.html)
* [IGListKit Modeling and Binding](https://instagram.github.io/IGListKit/modeling-and-binding.html)

### 一个 Model 对应一个 Section 中一个 Cell

首先这种方式能够 Diffing 的最小单位是一个 UICollectionView 的 Section，所以一个 Model 数据对应一个 Section，在 UI 布局上，自然也可以给 Section 加上 Supplementary View，但是要注意 Supplementary View 和 Section 是一个整体，Diffing 动画中同时出现和消失。

#### ListSectionController

实现 ListAdapterDataSource 的 func listAdapter(_ listAdapter: ListAdapter, sectionControllerFor object: Any) -> ListSectionController 需要返回 ListSectionController。

{% highlight swift %}
class LabelSectionController: ListSectionController {
  override func sizeForItem(at index: Int) -> CGSize {
    return CGSize(width: collectionContext!.containerSize.width, height: 55)
  }

  override func cellForItem(at index: Int) -> UICollectionViewCell {
    return collectionContext!.dequeueReusableCell(of: MyCell.self, for: self, at: index)
  }
}
{% endhighlight %}

#### ListSingleSectionController

快速地实现单个 Cell 的 Section 很方便。

{% highlight swift %}
func listAdapter(_ listAdapter: ListAdapter, sectionControllerFor object: Any) -> ListSectionController {
  let configureBlock = { (item: Any, cell: UICollectionViewCell) in
    guard let cell = cell as? StoryboardCell, let number = item as? Int else { return }
    cell.text = "Cell: \(number + 1)"
  }
  let sizeBlock = { (item: Any, context: ListCollectionContext?) -> CGSize in
    guard let context = context else { return .zero }
    return CGSize(width: context.containerSize.width, height: 44)
  }
  let sectionController = ListSingleSectionController(storyboardCellIdentifier: "cell",
                                                      configureBlock: configureBlock,
                                                      sizeBlock: sizeBlock)
  sectionController.selectionDelegate = self
  return sectionController
}
{% endhighlight %}

### 一个 Model 精细化地对应一个 Section 中多个 Cell

#### ListBindingSectionController

实现如下界面中 Model 到 Section 的映射很方便，一个 Section 中从上到下涵盖了 UserCell，ImageCell，ActionCell，CommentCell：

![ListBindingSectionController](/images/ListBindingSectionController.png)

对于如下的一个 Post：

{% highlight swift %}
data.append(Post(
  username: "@janedoe",
  timestamp: "15min",
  imageURL: URL(string: "https://placekitten.com/g/375/250")!,
  likes: 384,
  comments: [
    Comment(username: "@ryan", text: "this is beautiful!"),
    Comment(username: "@jsq", text: "😱"),
    Comment(username: "@caitlin", text: "#blessed"),
    ]
))
{% endhighlight %}

将其拆解为精细的 ListDiffable 数据，可以看到和上面的 Cell 是一一对应的：

{% highlight swift %}
func sectionController(
  _ sectionController: ListBindingSectionController<ListDiffable>,
  viewModelsFor object: Any
  ) -> [ListDiffable] {
  guard let object = object as? Post else { fatalError() }
  let results: [ListDiffable] = [
    UserViewModel(username: object.username, timestamp: object.timestamp),
    ImageViewModel(url: object.imageURL),
    ActionViewModel(likes: localLikes ?? object.likes)
  ]
  return results + object.comments
}
{% endhighlight %}

以下是 GitHub 上的代码：

* [rnystrom / IGListKit-Binding-Guide](https://github.com/rnystrom/IGListKit-Binding-Guide)

### ListDiffable

实现 ListDiffable 的 Model 必须是 Class，diffIdentifier() 用来辨识不同的 Model，isEqual(toDiffableObject:) 用来辨识这个 Model 是否有变化，因为 Model 是 Class，属于引用类型，如果要实现更新某个 Model，然后刷新 UI，如果直接改动 Class 中的属性，ListDiffable 就检查不到变化，因为你的数据和 IGListKit 中的数据指向同一地址，必须要拷贝一次，再改动才能有变化，如果数据从网络获取，或者从本地数据库加载，就不存在此问题，只要是不同内存地址。

{% highlight swift %}
class CartAddress {
  
  let receiver: String
  let contactPhone: String
  let detailAddress: String
  
  init() {
    receiver = ""
    contactPhone = ""
    detailAddress = ""
  }
  
}

extension CartAddress: ListDiffable {
  
  func diffIdentifier() -> NSObjectProtocol {
    return "address" as NSObjectProtocol
  }
  
  func isEqual(toDiffableObject object: ListDiffable?) -> Bool {
    if let object = object as? CartAddress {
      return receiver == object.receiver
        && contactPhone == object.contactPhone
        && detailAddress == object.detailAddress
    }
    return false
  }
  
}
{% endhighlight %}


如果想让 Swift structs diffable？[尝试这个](https://github.com/Instagram/IGListKit/issues/35#issuecomment-277503724
)。

## UICollectionViewLayout

UICollectionViewFlowLayout 并没有什么有趣的变化动画，要实现一些有趣的变化动画，就需要继承 UICollectionViewFlowLayout 或者 UICollectionViewLayout 来实现，通过以下四篇文章来学习：

* [UICollectionView Custom Layout Tutorial: A Spinning Wheel](https://www.raywenderlich.com/107687/uicollectionview-custom-layout-tutorial-spinning-wheel)
* [UICollectionView Custom Layout Tutorial: Pinterest](https://www.raywenderlich.com/164608/uicollectionview-custom-layout-tutorial-pinterest-2)
* [Custom UICollectionViewLayout Tutorial With Parallax](https://www.raywenderlich.com/156794/custom-uicollectionviewlayout-tutorial-parallax)
* [Animating Items in a UICollectionView](https://www.raizlabs.com/dev/2014/02/animating-items-in-a-uicollectionview/)

以下是 GitHub 上的代码：

* [danjiang / CollectionViewItemAnimations](https://github.com/danjiang/CollectionViewItemAnimations)
* [danjiang / CircularCollectionView](https://github.com/danjiang/CircularCollectionView)
* [danjiang / Pinterest](https://github.com/danjiang/Pinterest)
* [danjiang / JungleCup](https://github.com/danjiang/JungleCup)

### 继承 UICollectionViewLayout 来实现基本的变化

![Layout Lifecycle](/images/layout-lifecycle.png)

{% highlight swift %}
override func prepare()
{% endhighlight %}

UICollectionView 第一次显示在屏幕上或者调用 UICollectionViewLayout 的 invalidateLayout() 就会调用 prepare()。

{% highlight swift %}
override var collectionViewContentSize: CGSize
{% endhighlight %}

然后 UICollectionView 需要通过上面的方法来决定内容的整体大小，不只是可见的部分。

{% highlight swift %}
override func layoutAttributesForElements(in rect: CGRect) -> [UICollectionViewLayoutAttributes]?
override func layoutAttributesForItem(at indexPath: IndexPath) -> UICollectionViewLayoutAttributes?
{% endhighlight %}

这两个方法会在 UICollectionView 的布局过程中被频繁调用，layoutAttributesForElements(in rect: CGRect) 需要根据 rect 来返回在其中的 layout attributes。

{% highlight swift %}
override class var layoutAttributesClass: Swift.AnyClass

override func apply(_ layoutAttributes: UICollectionViewLayoutAttributes)
{% endhighlight %}

如果在给 UICollectionViewLayout 通过上面第一个方法提供了自定义的 layout attributes，UICollectionViewCell 需要重写上面第二个方法来应对 layout attributes 中自定义的属性，比如 anchorPoint。

{% highlight swift %}
override func shouldInvalidateLayout(forBoundsChange newBounds: CGRect) -> Bool
{% endhighlight %}

如果需要根据 bounds 的变化来重新计算 layout，上面方法需要返回 true。

### 接下来才是实现有趣的动态变化

触发 UICollectionView 的 Sections 或 Items 新增，删除，刷新和移动：

{% highlight swift %}
collectionView.insertSections(IndexSet(integer: 0))
collectionView.insertItems(at: [IndexPath(item: 0, section: 0)])
collectionView.deleteSections(IndexSet(integer: 0))
collectionView.deleteItems(at: [IndexPath(item: 0, section: 0)])
collectionView.reloadSections(IndexSet(integer: 0))
collectionView.reloadItems(at: [IndexPath(item: 0, section: 0)])
collectionView.moveSection(0, toSection: 1)
collectionView.moveItem(at: IndexPath(item: 0, section: 0),
                        to: IndexPath(item: 1, section: 0))
{% endhighlight %}

要批量处理就需要放到 performBatchUpdates() 的块中：

{% highlight swift %}
collectionView.performBatchUpdates({
  // all kinds of changes
}) { finished in
  // finished block
}
{% endhighlight %}

当前 batch of updates 的各项变化可以从 updateItems 中获取：

{% highlight swift %}
- (void)prepareForCollectionViewUpdates:(NSArray *)updateItems
{% endhighlight %}

计算新增，删除，移动的 Item 的 layout attribute：

{% highlight swift %}
- (UICollectionViewLayoutAttributes*)initialLayoutAttributesForAppearingItemAtIndexPath:(NSIndexPath *)itemIndexPath
- (UICollectionViewLayoutAttributes*)finalLayoutAttributesForDisappearingItemAtIndexPath:(NSIndexPath *)itemIndexPath
{% endhighlight %}

计算新增，删除，移动的 Supplementary Element 的 layout attribute：

{% highlight swift %}
- (UICollectionViewLayoutAttributes*)initialLayoutAttributesForAppearingSupplementaryElementOfKind:(NSString *)elementKind atIndexPath:(NSIndexPath *)elementIndexPath
- (UICollectionViewLayoutAttributes*)finalLayoutAttributesForDisappearingSupplementaryElementOfKind:(NSString *)elementKind atIndexPath:(NSIndexPath *)elementIndexPath
{% endhighlight %}

确认所有的 layout attributes 变化：

{% highlight swift %}
- (void)finalizeCollectionViewUpdates
{% endhighlight %}

## Diffing

从数据变化中计算出 IndexPath 变化，如果你不能计算出 IndexPath 变化，只能通过 reloadData 来整体刷新，效率低下，也不能让用户看到感知变化的动画，不是很友好。

通过以下二篇资料来了解一下 Diffing 算法，主要是 Wagner–Fischer 算法和 IGListKit 采用的 Heckel 算法：

* [A better way to update UICollectionView data in Swift with diff framework](https://medium.com/flawless-app-stories/a-better-way-to-update-uicollectionview-data-in-swift-with-diff-framework-924db158db86)

以下是 GitHub 上的代码：

* [danjiang / CollectionUpdateExample](https://github.com/danjiang/CollectionUpdateExample)
* [onmyway133 / DeepDiff](https://github.com/onmyway133/DeepDiff)

DeepDiff 使用也比较方便，自定义的 Model 需要实现 Hashable：

{% highlight swift %}
let oldItems = items
items = DataSet.generateNewItems()
let changes = diff(old: oldItems, new: items)

collectionView.reload(changes: changes, section: 2, completion: { _ in })

tableView.reload(changes: changes, completion: { _ in })
{% endhighlight %}
