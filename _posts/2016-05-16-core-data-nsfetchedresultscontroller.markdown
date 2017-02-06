---
title: Core Data NSFetchedResultsController
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: core-data
---

从这篇文章，你将学习到有关 Core Data 中 NSFetchedResultsController 使用，[示例代码地址](https://github.com/danjiang/Shop)，NSFetchedResultsController 可以说是让人又爱又恨，如果同常规的 UITableView 或 UICollectionView 进行结合，你会觉得非常爽，简单而又高效，但是如果同你自己写的视图或者其他内置视图结合时，你就会觉得用起来非常别扭。本文将阐述 NSFetchedResultsController 常规用法，然后深入其内部实现机制，从而知道如何根据自己情况构建类似 NSFetchedResultsController 的功能。

![Core Data Helper](/images/core-data-nsfetchedresultscontroller.png)

## 常规用法 

![Core Data Shop Categories](/images/core-data-shop-categories.png)

CategoriesViewController 中有定义 NSFetchedResultsController 的属性：

{% highlight swift %}
class CategoriesViewController: UITableViewController {
  ...
  fileprivate var fetchedResultsController: NSFetchedResultsController<NSManagedObject>?
  ...
}
{% endhighlight %}

初始化 NSFetchedResultsController，指定 CategoriesViewController 为 NSFetchedResultsController 的代理，需要预先执行 performFetch 来获取初始数据：

{% highlight swift %}
class CategoriesViewController: UITableViewController {
  ...
  override func viewDidLoad() {
    super.viewDidLoad()
    
    initFetchedResultsController()
  }

  fileprivate func initFetchedResultsController() {
    fetchedResultsController = PersistenceController.sharedInstance.fetchedResultsController(enityName: "Category",
                                                                                             sortDescriptors: [NSSortDescriptor(key: "brand", ascending: false), NSSortDescriptor(key: "date", ascending: false)],
                                                                                             sectionNameKeyPath: "brand")
    fetchedResultsController?.delegate = self
    do {
      try fetchedResultsController!.performFetch()
      tableView.reloadData()
    } catch {
      print("!!! Error: perform fetch !!!\n\(error)\n")
    }
  }
  ...
}
{% endhighlight %}

指定实体名称和排序方式来创建 NSFetchRequest，注意一定要排序否则会出错，然后再生成 NSFetchedResultsController，还需要将数据是否是品牌分成不同的段，所以 sectionNameKeyPath 为 brand：

{% highlight swift %}
class PersistenceController {
  ...
  func fetchedResultsController(enityName: String, sortDescriptors: [NSSortDescriptor], sectionNameKeyPath: String?) -> NSFetchedResultsController<NSManagedObject> {
    let request = NSFetchRequest<NSManagedObject>(entityName: enityName)
    request.sortDescriptors = sortDescriptors
    return NSFetchedResultsController(fetchRequest: request, managedObjectContext: moc, sectionNameKeyPath: sectionNameKeyPath, cacheName: "Master")
  }
  ...
}
{% endhighlight %}

UITableView 从 NSFetchedResultsController 中获取内容来显示：

{% highlight swift %}
class CategoriesViewController: UITableViewController {
  ...
  override func numberOfSections(in tableView: UITableView) -> Int {
    return fetchedResultsController?.sections?.count ?? 0
  }
  
  override func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
    if let sectionInfo = fetchedResultsController?.sections?[section] {
      return sectionInfo.numberOfObjects
    } else {
      return 0
    }
  }
  
  override func tableView(_ tableView: UITableView, titleForHeaderInSection section: Int) -> String? {
    if let sectionInfo = fetchedResultsController?.sections?[section],
      let categories = sectionInfo.objects as? [NSManagedObject],
      let category = categories.first,
      let brand = category.value(forKey: "brand") as? Bool {
      return brand ? "Brands" : "Categories"
    } else {
      return nil
    }
  }
  
  override func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
    let cell = tableView.dequeueReusableCell(withIdentifier: "CategoryCell")!
    if let sectionInfo = fetchedResultsController?.sections?[indexPath.section], let categories = sectionInfo.objects as? [NSManagedObject] {
      let category = categories[indexPath.row]
      if let name = category.value(forKey: "name") as? String {
        cell.textLabel?.text = name
      }
    }
    return cell
  }
  ...
}
{% endhighlight %}

UITableView 通过 NSFetchedResultsControllerDelegate 感知数据库的变化，从而自动更新显示的内容：

{% highlight swift %}
extension CategoriesViewController: NSFetchedResultsControllerDelegate {
  
  func controllerWillChangeContent(_ controller: NSFetchedResultsController<NSFetchRequestResult>) {
    tableView.beginUpdates()
  }
  
  func controller(_ controller: NSFetchedResultsController<NSFetchRequestResult>, didChange sectionInfo: NSFetchedResultsSectionInfo, atSectionIndex sectionIndex: Int, for type: NSFetchedResultsChangeType) {
    let indexSet = IndexSet(integer: sectionIndex)
    switch type {
    case .insert:
      tableView.insertSections(indexSet, with: .automatic)
    case .delete:
      tableView.deleteSections(indexSet, with: .automatic)
    default: break
    }
  }
  
  func controller(_ controller: NSFetchedResultsController<NSFetchRequestResult>, didChange anObject: Any, at indexPath: IndexPath?, for type: NSFetchedResultsChangeType, newIndexPath: IndexPath?) {
    switch type {
    case .insert:
      if let newIndexPath = newIndexPath {
        tableView.insertRows(at: [newIndexPath], with: .automatic)
      }
    case .delete:
      if let indexPath = indexPath {
        tableView.deleteRows(at: [indexPath], with: .automatic)
      }
    case .update:
      if let indexPath = indexPath {
        tableView.reloadRows(at: [indexPath], with: .automatic)
      }
    case .move:
      if let indexPath = indexPath, let newIndexPath = newIndexPath {
        tableView.moveRow(at: indexPath, to: newIndexPath)
      }
    }
  }
  
  func controllerDidChangeContent(_ controller: NSFetchedResultsController<NSFetchRequestResult>) {
    tableView.endUpdates()
  }
  
}
{% endhighlight %}

## 深入内部

常见的情况是我们希望自己写的视图能够感知数据库的变化来自动更新内容，但是又不想像 NSFetchedResultsController 同 UITableView 内容结构绑定的这么死，那么我们就要了解一下 NSFetchedResultsController 是如何感知数据库的变化的。

NSFetchedResultsController 通过监听 NSManagedObjectContext 发出的： 

* NSNotification.Name.NSManagedObjectContextWillSave
* NSNotification.Name.NSManagedObjectContextDidSave
* NSNotification.Name.NSManagedObjectContextObjectsDidChange

来感知数据库的变化，并且进一步缓存数据，控制 NSIndexPath 和数据之间的对应关系等等。

我们来实现 ContextObserver 来监听 NSManagedObjectContextObjectsDidChange，并且通过 NSPredicate 来筛选变动的数据，如果是感兴趣的数据才调用代理方法：

{% highlight swift %}
protocol ContextObserverDelegate {
  func didInsert(for predicate: NSPredicate)
  func didDelete(for predicate: NSPredicate)
  func didUpdate(for predicate: NSPredicate)
}

class ContextObserver: NSObject {
  
  var delegate: ContextObserverDelegate?
  
  fileprivate var moc: NSManagedObjectContext?
  fileprivate var predicate: NSPredicate?
  
  deinit {
    if let moc = moc {
      NotificationCenter.default.removeObserver(self, name: .NSManagedObjectContextDidSave, object: moc)
    }
  }
  
  init(moc: NSManagedObjectContext?, predicate: NSPredicate?) {
    super.init()
    
    self.moc = moc
    self.predicate = predicate
    
    if let moc = moc {
      NotificationCenter.default.addObserver(self, selector: #selector(contextUpdated(_:)), name: .NSManagedObjectContextDidSave, object: moc)
    }
  }
  
  func contextUpdated(_ notification: Notification) {
    if let userInfo = notification.userInfo {
      let insertedObjects = userInfo[NSInsertedObjectsKey] as? NSSet
      let deletedObjects = userInfo[NSDeletedObjectsKey] as? NSSet
      let updatedObjects = userInfo[NSUpdatedObjectsKey] as? NSSet
      if let predicate = predicate {
        if insertedObjects?.filtered(using: predicate).count > 0 {
          delegate?.didInsert(for: predicate)
        }
        if deletedObjects?.filtered(using: predicate).count > 0 {
          delegate?.didDelete(for: predicate)
        }
        if updatedObjects?.filtered(using: predicate).count > 0 {
          delegate?.didUpdate(for: predicate)
        }
      }
    }
  }
  
}
{% endhighlight %}
