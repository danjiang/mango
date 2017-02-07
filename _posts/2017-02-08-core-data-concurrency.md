---
title: Core Data 并行
author: 但江
avatar: danjiang
location: 成都 
category: programming
tag: core-data
---

从这篇文章，你将学习到有关 Core Data 中为了提高性能而采用并行的方案，应该如何建立 Core Data Stack，[示例代码地址](https://github.com/danjiang/Shop)，让我们开始吧。

![Core Data Concurrency](/images/core-data-concurrency.png)

## 背景介绍

Core Data 中提供了一些并行的功能，要利用好这些并行功能，在于建立好适合实际需求的 Core Data Stack，前面的 Core Data 系列文章，都是建立了一个只能在主线程中操作的 Core Data Stack，所有的数据操作都会阻塞 UI，我们先来看看 Core Data 中提供的并行功能，然后列举两个 Core Data Stack 方案。

## NSManagedObjectContext

NSManagedObjectContext 是操作数据的窗口，详细了解此类是设计 Core Data Stack 的关键。

### 并行类型

NSManagedObjectContext 提供了三种不同的并行类型，在初始化时需要传入对应的类型：

{% highlight swift %}
moc = NSManagedObjectContext(concurrencyType: .mainQueueConcurrencyType)
{% endhighlight %}

* Confinement (**.confinementConcurrencyType**)：在什么线程创建，只能在什么线程使用，因为兼容目的，这是默认类型，还有一点，如果使用此类型，那么 Managed Object Context 的 Parent Store 只能是 Persistent Store Coordinator，建议使用其他两个类型；
* Private queue (**.privateQueueConcurrencyType**)：创建了一个只能由私有的队列访问的 Managed Object Context，因为私有，所以只能通过 **perform(\_:)** 和 **performAndWait(\_:)** 来调用；
* Main queue (**.mainQueueConcurrencyType**)：创建了一个只能从主线程访问的 Managed Object Context，如果从非主线程访问，可以调用 **perform(\_:)** 和 **performAndWait(\_:)**。

### NSManagedObjectContextDidSave

如果一个 Managed Object Context 中实体发生变化就会发布通知，这里主要关注  **NSManagedObjectContextDidSave**，另外一个 Managed Object Context 可以根据这个通知调用 **mergeChanges(fromContextDidSave:)** 来合并前一个 Managed Object Context 变化的内容，保持两个 Managed Object Context 的内容同步。

### Parent Store

Parent Store 代表 Managed Object Context 读取数据和提交数据变化的来源。

Parent Store 可以是 Persistent Store Coordinator、也可以是另外一个 Managed Object Context。

如果读取数据，会沿着 Parent Store 一层层查找。

如果提交数据变化，数据变化只会往上提交一层，如果 Parent Store 是 Managed Object Context，数据变化也就不会提交到数据库中。

### perform(\_:) 和 performAndWait(\_:)

这两个方法保证块中的代码只会在 Managed Object Context 对应的队列上执行，如果你不在 Managed Object Context 对应的线程中，调用这两个方法就很安全：

* **perform(\_:)**：异步调用；
* **performAndWait(\_:)**：同步调用。

## 不同 Managed Object Context 对应不同的线程

![Core Data Stack Old](/images/core-data-statck-old.png)

主线程中

{% highlight swift %}
let psc = NSPersistentStoreCoordinator(managedObjectModel: mom!)
mainMoc = NSManagedObjectContext(concurrencyType: .mainQueueConcurrencyType)
mainMoc.persistentStoreCoordinator = psc
{% endhighlight %}

非主线程中

{% highlight swift %}
workerMoc1 = NSManagedObjectContext()
workerMoc1.persistentStoreCoordinator = mainMoc.persistentStoreCoordinator
{% endhighlight %}

跨线程通信

{% highlight swift %}
NotificationCenter.default.addObserver(self, selector: #selector(contextDidSave(_:)), name: .NSManagedObjectContextDidSave, object: workerMoc1)
    
@objc func contextDidSave(_ notification: Notification) {
  if Thread.isMainThread {
    mainMoc.mergeChanges(fromContextDidSave: notification)
  } else {
    DispatchQueue.main.async {
      self.mainMoc.mergeChanges(fromContextDidSave: notification)
    }
  }
}
{% endhighlight %}

## 嵌套式的 Managed Object Context

![Core Data Stack New](/images/core-data-statck-new.png)

建立 Core Data Stack

{% highlight swift %}
let psc = NSPersistentStoreCoordinator(managedObjectModel: mom!)

persistingMoc = NSManagedObjectContext(concurrencyType: .privateQueueConcurrencyType)
persistingMoc.persistentStoreCoordinator = psc

mainMoc = NSManagedObjectContext(concurrencyType: .mainQueueConcurrencyType)
mainMoc.parent = persistingMoc
{% endhighlight %}

保存数据

{% highlight swift %}
func save() {
  guard mainMoc.hasChanges || persistingMoc.hasChanges else {
    return
  }
  
  mainMoc.performAndWait {
    do {
      try self.mainMoc.save()
      
      self.persistingMoc.perform {
        do {
          try self.persistingMoc.save()
        } catch {
          print("!!! Error: save managed object in persisting context !!!\n\(error)\n")
        }
      }
    } catch {
      print("!!! Error: save managed object in main context !!!\n\(error)\n")
    }
  }
}
{% endhighlight %}

## 了解更多

* [My Core Data Stack](http://martiancraft.com/blog/2015/03/core-data-stack/)
* [Introducing the Open-Source Big Nerd Ranch Core Data Stack](https://www.bignerdranch.com/blog/introducing-the-big-nerd-ranch-core-data-stack)
