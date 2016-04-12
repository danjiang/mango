---
title: Core Data 基础
author: 但江
avatar: danjiang
location: 成都 
category: programming
tag: core-data
---

从这篇文章，你将学习到有关 Core Data 的基础知识，如何创建 Core Data Stack，如何增删改查，让我们开始吧。

![Swift Functions](/images/swift-functions.jpg)

## Core Data Stack

创建了 Core Data Stack，才能正常地使用 Core Data 存取数据，之所以称呼为 Stack，就如同下图所示上一层依赖于下一层。

![Core Data Stack](/images/core-data-stack.png)

### NSManagedObjectModel

NSManagedObjectModel 描述了存储数据的结构，如果你有关系型数据库的使用经验，你应该明白这就是 Database Schema，和其它所有关系型数据库一样，结构就是实体和关系，如下图所示。

![Core Data Data Model Graph](/images/core-data-data-model-graph.jpg)

通过 Xcode 创建 Data Model 文件。

![Core Data Data Model Create](/images/core-data-data-model-create.jpg)

Data Model 文件以 .xcdatamodeld 做为文件名后缀，Xcode 会将 .xcdatamodeld 文件编译成 .momd，后面会用到。

![Core Data Data Model Filename](/images/core-data-data-model-filename.jpg)

在 Data Model 文件中设计存储数据的结构，创建实体，定义实体中的属性名和属性类型，还有实体之间的关系。

![Core Data Data Model Table](/images/core-data-data-model-table.jpg)

### NSPersistentStore

代表数据存储之地，iOS 支持的格式有二进制之类，OS X 还支持 XML 格式，这里只说 SQLite 格式。

### NSPersistentStoreCoordinator

负责存储，加载和缓存数据，一旦初始化过后，很少需要再和它打交道，初始化的时候需要 NSManagedObjectModel，这样才知道存储数据的结构嘛，还需要 NSPersistentStore，这样才知道以什么样的格式存储在哪里。

### NSManagedObjectContext

- Core Data`s "scratch pad" of your objects
- Tracks changes to model properties
- Handles actions on your data: CRUD
- not thread-safe, NSManagedObjectContext should be accessed only on the thread that created it

### 来上代码吧

{% highlight swift %}
import Foundation
import CoreData

class DataManager {
  func createStack() {
    let modelURL = NSBundle.mainBundle().URLForResource("Diary", withExtension: "momd")
    let mom = NSManagedObjectModel(contentsOfURL: modelURL!)
    let psc = NSPersistentStoreCoordinator(managedObjectModel: mom!)
    
    let moc = NSManagedObjectContext(concurrencyType: NSManagedObjectContextConcurrencyType.MainQueueConcurrencyType)
    moc.persistentStoreCoordinator = psc
    
    let fileManager = NSFileManager.defaultManager()
    
    var options = [String:AnyObject]()
    options[NSMigratePersistentStoresAutomaticallyOption] = true
    options[NSInferMappingModelAutomaticallyOption] = true
    
    let localStoreURL = fileManager.URLsForDirectory(NSSearchPathDirectory.DocumentDirectory, inDomains: NSSearchPathDomainMask.UserDomainMask).last!.URLByAppendingPathComponent("Diary.sqlite")
    
    do {
      try psc.addPersistentStoreWithType(NSSQLiteStoreType, configuration: nil, URL: localStoreURL, options: options)
    } catch {
      print("!!! Error: adding local persistent store to coordinator !!!\n\(error)\n")
    }
  }
}
{% endhighlight %}

## 增删改查

### NSManagedObject

- Each instance represents a "row" of data
- Encapsulates your data and relationships
- Provides validation

### 使用 NSFetchRequest 查询

- retrieve all instances of an entity from the object hierarchy
- create stored fetch requests through Xcode
- NSEntityDescription : setting the entity to be retrieved
- NSPredicate : narrow the search or filter the results
- NSSortDescriptor

### 新增，修改和删除
