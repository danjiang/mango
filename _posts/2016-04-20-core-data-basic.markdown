---
title: Core Data 基础
author: 但江
avatar: danjiang
location: 成都 
category: programming
tag: core-data
---

从这篇文章，你将学习到有关 Core Data 的基础知识，如何创建 Core Data Stack，了解其中每一个构成部分，[示例代码地址](https://github.com/danjiang/Shop)，让我们开始吧。

![Core Data Basic](/images/core-data-basic.png)

## Core Data Stack 简介

创建了 Core Data Stack，才能正常地使用 Core Data 存取数据，之所以称呼为 Stack，就如同下图所示上一层依赖于下一层。

![Core Data Stack](/images/core-data-stack.png)

## NSManagedObjectModel

NSManagedObjectModel 描述了存储数据的结构，如果你有关系型数据库的使用经验，你应该明白这就是 Database Schema，和其它所有关系型数据库一样，结构就是实体和关系，如下图所示：

![Core Data Data Model Graph](/images/core-data-data-model-graph.jpg)

通过 Xcode 创建 Data Model 文件：

![Core Data Data Model Create](/images/core-data-data-model-create.jpg)

Data Model 文件以 .xcdatamodeld 做为文件名后缀，Xcode 会将 .xcdatamodeld 文件编译成 .momd，后面会用到这个编译后的文件名：

![Core Data Data Model Filename](/images/core-data-data-model-filename.jpg)

在 Data Model 文件中设计存储数据的结构，创建实体，定义实体中的属性名和属性类型，还有实体之间的关系。

![Core Data Data Model Table 1](/images/core-data-data-model-table1.jpg)

![Core Data Data Model Table 2](/images/core-data-data-model-table2.jpg)

## NSPersistentStore

代表数据存储之地，iOS 支持的格式有二进制等，OS X 还支持 XML 格式，这里只说 SQLite 格式。

## NSPersistentStoreCoordinator

负责存储，加载和缓存数据，一旦初始化过后，很少需要再和它打交道，初始化的时候需要 NSManagedObjectModel，这样才知道存储数据的结构嘛，还需要 NSPersistentStore，这样才知道以什么样的格式存储在哪里。

### NSManagedObjectContext

这是你主要打交道的类，通过此类来查询，新增，修改和删除数据，所有你操作过的数据对象，都会在这里暂存，也是为了提高频繁访问的效率，值得注意一点是此类不是线程安全的，简单来说，如果你是在主线程创建的此类的对象，那么只能在主线程通过此对象操作数据，否则就会出问题。

## 代码创建 Core Data Stack

{% highlight swift %}
import Foundation
import CoreData

class PersistenceController {
  
  static let sharedInstance = PersistenceController()
  
  fileprivate let model = "Model"
  fileprivate let db = "db.sqlite"
  fileprivate var moc: NSManagedObjectContext
  
  fileprivate init() {
    // 找到通过 Xcode 创建的 Data Model 文件
    let modelURL = Bundle.main.url(forResource: model, withExtension: "momd")
    // 创建 NSManagedObjectModel
    let mom = NSManagedObjectModel(contentsOf: modelURL!)
    // 创建 NSPersistentStoreCoordinator
    let psc = NSPersistentStoreCoordinator(managedObjectModel: mom!)
    
    // 创建 NSManagedObjectContext，并指定是主线程类型
    moc = NSManagedObjectContext(concurrencyType: .mainQueueConcurrencyType)
    
    // NSManagedObjectContext 关联 NSPersistentStoreCoordinator
    moc.persistentStoreCoordinator = psc
    
    let fileManager = FileManager.default
    
    // 设置 NSPersistentStore 需要的一些参数
    var options = [String: Any]()
    options[NSMigratePersistentStoresAutomaticallyOption] = true
    options[NSInferMappingModelAutomaticallyOption] = true
    
    // 指定 SQLite 数据库文件存储位置
    let localStoreURL = fileManager.urls(for: .documentDirectory, in: .userDomainMask).last!.appendingPathComponent(db)
    
    do {
      try psc.addPersistentStore(ofType: NSSQLiteStoreType, configurationName: nil, at: localStoreURL, options: options)
    } catch {
      print("!!! Error: adding local persistent store to coordinator !!!\n\(error)\n")
    }
  }
  
}
{% endhighlight %}

## NSManagedObject

每一个 NSManagedObject 对象代表数据库表中每一行数据，封装了你定义的数据类型和关系。

访问属性，会触发 KVO 通知：

{% highlight swift %}
let object = NSEntityDescription.insertNewObject(forEntityName: "Category", into: moc)
object.setValue("Running", forKey: "name")
object.value(forKey: "name")
{% endhighlight %}

原始访问属性，不会触发 KVO 通知：

{% highlight swift %}
let object = NSEntityDescription.insertNewObject(forEntityName: "Category", into: moc)
object.setPrimitiveValue("Running", forKey: "name")
object.primitiveValue(forKey: "name")
{% endhighlight %}

