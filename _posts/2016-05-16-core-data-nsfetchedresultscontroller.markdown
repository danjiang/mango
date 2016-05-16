---
title: Core Data NSFetchedResultsController
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: core-data
---

从这篇文章，你将学习到有关 Core Data 中 **NSFetchedResultsController** 使用，[示例代码地址](https://github.com/danjiang/blog-core-data)，**NSFetchedResultsController** 可以说是让人又爱又恨，如果同常规的 **UITableView** 或 **UICollectionView** 进行结合，你会觉得非常爽，简单而又高效，但是如果同你自己写的视图或者其他内置视图结合时，你就会觉得用起来非常别扭。本文将阐述 **NSFetchedResultsController** 常规用法，然后深入其内部实现机制，从而知道如何根据自己情况构建类似 **NSFetchedResultsController**的功能。

![Core Data Data Model Graph](/images/core-data-data-model-graph.jpg)

## 常规用法 

**CategoriesViewController** 中有定义 **NSFetchedResultsController** 的属性。

{% highlight swift %}
class CategoriesViewController: UITableViewController {
  ...
  private var fetchedResultsController: NSFetchedResultsController?
  ...
}
{% endhighlight %}

初始化 **NSFetchedResultsController**，指定 **CategoriesViewController** 为 **NSFetchedResultsController** 的代理，需要预先执行 **performFetch** 来获取初始数据。

{% highlight swift %}
class CategoriesViewController: UITableViewController {
  ...
  override func viewDidLoad() {
    super.viewDidLoad()
    
    initFetchedResultsController()
  }

  private func initFetchedResultsController() {
    fetchedResultsController = DataManager.sharedInstance.fetchedResultsControllerForCategories()
    fetchedResultsController?.delegate = self
    do {
      try fetchedResultsController!.performFetch()
      self.tableView.reloadData()
    } catch {
      print("!!! Error: perform fetch !!!\n\(error)\n")
    }
  }
  ...
}
{% endhighlight %}

指定实体名称和排序方式来创建 **NSFetchRequest**，注意一定要排序否则会出错，然后再生成 **NSFetchedResultsController**，不需要将数据分成不同的部分，所以 **sectionNameKeyPath** 为 **nil**。

{% highlight swift %}
class DataManager {
  ...
  func fetchedResultsControllerForCategories() -> NSFetchedResultsController {
    let request = NSFetchRequest(entityName: "Category")
    request.sortDescriptors = [NSSortDescriptor(key: "createTime", ascending: false)]
    return NSFetchedResultsController(fetchRequest: request, managedObjectContext: moc!, sectionNameKeyPath: nil, cacheName: "Master")
  }
  ...
}
{% endhighlight %}

**UITableView** 从 **NSFetchedResultsController** 中获取内容来显示。

{% highlight swift %}
class CategoriesViewController: UITableViewController {
  ...
  override func tableView(tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
    if let sectionInfo = fetchedResultsController?.sections?[section] {
      return sectionInfo.numberOfObjects
    } else {
      return 0
    }
  }
  
  override func tableView(tableView: UITableView, cellForRowAtIndexPath indexPath: NSIndexPath) -> UITableViewCell {
    let cell = tableView.dequeueReusableCellWithIdentifier("CategoryCell")!
    
    if let sectionInfo = self.fetchedResultsController?.sections?[indexPath.section], categories = sectionInfo.objects {
      let category = categories[indexPath.row] as! NSManagedObject
      if let name = category.valueForKey("name") as? String {
        cell.textLabel?.text = name
      }
    }
    
    return cell
  }
  ...
}
{% endhighlight %}

**UITableView** 通过 **NSFetchedResultsControllerDelegate** 感知数据库的变化，从而自动更新显示的内容。

{% highlight swift %}
extension CategoriesViewController: NSFetchedResultsControllerDelegate {

  func controllerWillChangeContent(controller: NSFetchedResultsController) {
    tableView.beginUpdates()
  }
  
  func controller(controller: NSFetchedResultsController, didChangeSection sectionInfo: NSFetchedResultsSectionInfo, atIndex sectionIndex: Int, forChangeType type: NSFetchedResultsChangeType) {
    let indexSet = NSIndexSet(index: sectionIndex)
    switch type {
    case .Insert:
      tableView.insertSections(indexSet, withRowAnimation: .Automatic)
    case .Delete:
      tableView.deleteSections(indexSet, withRowAnimation: .Automatic)
    default:
      break
    }
  }
  
  func controller(controller: NSFetchedResultsController, didChangeObject anObject: AnyObject, atIndexPath indexPath: NSIndexPath?, forChangeType type: NSFetchedResultsChangeType, newIndexPath: NSIndexPath?) {
    switch type {
    case .Insert:
      if let newIndexPath = newIndexPath {
        tableView.insertRowsAtIndexPaths([newIndexPath], withRowAnimation: .Automatic)
      }
    case .Delete:
      if let indexPath = indexPath {
        tableView.deleteRowsAtIndexPaths([indexPath], withRowAnimation: .Automatic)
      }
    case .Update:
      if let indexPath = indexPath {
        tableView.reloadRowsAtIndexPaths([indexPath], withRowAnimation: .Automatic)
      }
    case .Move:
      if let indexPath = indexPath, newIndexPath = newIndexPath {
        tableView.moveRowAtIndexPath(indexPath, toIndexPath: newIndexPath)
      }
    }
  }
  
  func controllerDidChangeContent(controller: NSFetchedResultsController) {
    tableView.endUpdates()
  }
    
}
{% endhighlight %}

## 深入内部

常见的情况是我们希望自己写的视图能够感知数据库的变化来自动更新内容，但是又不想像 **NSFetchedResultsController** 同 **UITableView** 内容结构绑定的这么死，那么我们就要了解一下 **NSFetchedResultsController** 是如何感知数据库的变化的，**NSFetchedResultsController** 通过监听 **NSManagedObjectContext** 发出的 **NSManagedObjectContextObjectsDidChangeNotification**、**NSManagedObjectContextWillSaveNotification** 和 **NSManagedObjectContextDidSaveNotification** 来感知数据库的变化，并且进一步缓存数据，控制 **NSIndexPath** 和数据之间的对应关系等等。

我们来实现 **ContextObserver** 来监听 **NSManagedObjectContextDidSaveNotification**，并且通过 **NSPredicate** 来筛选变动的数据，如果是感兴趣的数据才调用代理方法。

{% highlight swift %}
protocol ContextObserverDelegate {
  func contextObserverDidInsertForPredicate(predicate: NSPredicate?)
  func contextObserverDidDeleteForPredicate(predicate: NSPredicate?)
  func contextObserverDidUpdateForPredicate(predicate: NSPredicate?)
}

class ContextObserver: NSObject {
  
  var delegate: ContextObserverDelegate?
  
  private var moc: NSManagedObjectContext?
  private var predicate: NSPredicate?
  
  deinit {
    if let moc = self.moc {
      NSNotificationCenter.defaultCenter().removeObserver(self, name: NSManagedObjectContextDidSaveNotification, object: moc)
    }
  }
  
  init(moc: NSManagedObjectContext?, predicate: NSPredicate?) {
    super.init()
    
    self.moc = moc
    self.predicate = predicate
    
    if let moc = self.moc {
      NSNotificationCenter.defaultCenter().addObserver(self, selector: #selector(contextUpdated(_:)), name: NSManagedObjectContextDidSaveNotification, object: moc)
    }
  }
  
  func contextUpdated(notification: NSNotification) {
    if let userInfo = notification.userInfo {
      var insertedObjects = userInfo[NSInsertedObjectsKey] as! NSSet?
      var deletedObjects = userInfo[NSDeletedObjectsKey] as! NSSet?
      var updatedObjects = userInfo[NSUpdatedObjectsKey] as! NSSet?
      if let predicate = self.predicate {
        insertedObjects = insertedObjects?.filteredSetUsingPredicate(predicate)
        deletedObjects = deletedObjects?.filteredSetUsingPredicate(predicate)
        updatedObjects = updatedObjects?.filteredSetUsingPredicate(predicate)
        if insertedObjects?.count > 0 {
          self.delegate?.contextObserverDidInsertForPredicate(predicate)
        }
        if deletedObjects?.count > 0 {
          self.delegate?.contextObserverDidDeleteForPredicate(predicate)
        }
        if updatedObjects?.count > 0 {
          self.delegate?.contextObserverDidUpdateForPredicate(predicate)
        }
      }
    }
  }
  
}
{% endhighlight %}
