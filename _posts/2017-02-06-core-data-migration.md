---
title: Core Data 数据迁移
author: 但江
avatar: danjiang
location: 成都 
category: programming
tag: core-data
---

从这篇文章，你将学习到有关 Core Data 的数据库结构发生变化时，应该如何处理，[示例代码地址](https://github.com/danjiang/Shop)，让我们开始吧。

![Core Data Migration](/images/core-data-migration.png)

## 背景介绍

随着用户使用你的应用，你发现了一些新的需求，准备开发新的功能，需要改变 Core Data 的数据库结构，如果你直接修改 Model.xcdatamodeld 中的数据结构，当用户升级到新版本时，应用会直接挂掉，因为用户手机上已经存在的数据库和你现在修改过的数据库结构不兼容，但是应用不知道怎么从旧的数据库结构迁移到新的数据库结构，Core Data 给我们提供了数据迁移的办法，分为轻量迁移和繁重迁移，下面分别介绍。

## 轻量迁移

> 只是数据库结构发生变化，不需要修改数据库中的数据，也就不需要将数据加载到内存中

在菜单 Xcode > Editor > Add Model Version，在出现的对话框中输入 Version name：

![Core Data Add Model Version](/images/core-data-add-model-version.jpg)

在 Model.xcdatamodeld 上 Show in Finder，将新旧 .xcdatamodel 文件分别命名为 V1.xcdatamodel 和 V2.xcdatamodel：

![Core Data Data Model Version Filename](/images/core-data-data-model-version-filename.jpg)

查看 Model.xcdatamodeld 的 File Inspector，修改 Model Version Current 为 V2，对应的 .xcdatamodel 文件会有绿色勾勾：

![Core Data Data Model File Inspector](/images/core-data-data-model-file-inspector.jpg)

旧有的数据库结构：

![Core Data Data Model Graph Old](/images/core-data-data-model-graph-old.jpg)

新的数据库结构，在 Category 上新增了 Boolean 型 brand 属性来表明这个分类是不是品牌分类：

![Core Data Data Model Graph](/images/core-data-data-model-graph.jpg)

还需要在代码中告诉 Core Data 发现数据库结构不同时，自动进行数据迁移：

{% highlight swift %}
var options = [String: Any]()
options[NSMigratePersistentStoresAutomaticallyOption] = true
options[NSInferMappingModelAutomaticallyOption] = true

psc.addPersistentStore(ofType: NSSQLiteStoreType, configurationName: nil, at: localStoreURL, options: options)
{% endhighlight %}

这样就 OK 了，用户升级到新版本时，数据库会自动迁移为新结构，不会再崩溃。

## 繁重迁移

> 数据库结构发生了变化，还需要修改数据库中的数据，就会将数据加载到内存中，所以要尽量避免这种情况发生

前面说到，在 Category 上新增了 Boolean 型 brand 属性来表明这个分类是不是品牌分类，在进行数据迁移时，如果根据 Category 的 name 属性来自动判断这个分类是不是品牌分类就会比较便利，我们设定 name 为 Nike、Adidas、UA 就自动判断为品牌分类。

在菜单 Xcode > File > New File，新增一个 Mapping Model 文件：

![Core Data Mapping Model Create](/images/core-data-mapping-model-create.jpg)

工程中就多了一个 Model.xcmappingmodel 文件，在进行数据迁移的时候，根据此文件中的规则，将旧数据转换为新数据：

![Core Data Mapping Model Filename](/images/core-data-mapping-model-filename.jpg)

Model.xcmappingmodel 文件中自动生成了两个 Entity Mapping 文件：ItemToItem 和 CategoryToCategory，ItemToItem 因为没有结构变化不需要修改，CategoryToCategory 修改为 CategoryIsBrand，在新增一个名为 CategoryIsNotBrand 的 Entity Mapping。

每一个 Entity Mapping 都分为 Attribute Mapping 和 Relationship Mapping，一个属性映射规则，一个关系映射规则。

在属性映射规则的左侧可以看到都是新结构中的属性，在右侧可以引用一些变量来将旧值计算为新值，这些变量包括下面这些：

* $manager 对应 NSMigrationManagerKey* $source 对应 NSMigrationSourceObjectKey* $destination 对应 NSMigrationDestinationObjectKey* $entityMapping 对应 NSMigrationEntityMappingKey* $propertyMapping 对应 NSMigrationPropertyMappingKey
* $entityPolicy 对应 NSMigrationEntityPolicyKey

注意右侧边栏中 Filter Predicate 写的表达式，可以过滤查询出来的旧数据。

CategoryIsBrand 中，Filter Predicate 过滤出是品牌的分类，brand 值则为 1：

![Core Data Mapping Model Category Is Brand](/images/core-data-mapping-model-category-is-brand.jpg)

CategoryIsNotBrand 中，Filter Predicate 过滤出不是品牌的分类，brand 值则为 0，Relationship Mapping 复制上面的：

![Core Data Mapping Model Category Is Not Brand](/images/core-data-mapping-model-category-is-not-brand.jpg)

就这样通过一些配置，我们达到目的，下面说一下另外一种方法，通过设置 Custom Policy。

CategoryIsBrand 修改为 CategoryToCategory，删除 CategoryIsNotBrand，在 CategoryToCategory 中删除掉 Attribute Mapping 中所有映射策略，Relationship Mapping 保持不变，右侧边栏中 Filter Predicate 表达式为空，这样就会查询所有旧数据，Custom Policy 为 DetectBrandMigrationPolicy，这是我们指定自定义映射规则的类。

![Core Data Mapping Model Custom Policy](/images/core-data-mapping-model-custom-policy.jpg)

DetectBrandMigrationPolicy 继承自 NSEntityMigrationPolicy，NSEntityMigrationPolicy 中定义了一系列数据迁移过程中可以插入自有逻辑的方法，覆盖这些方法，在合适的时机就会被调用：

{% highlight swift %}
open class NSEntityMigrationPolicy : NSObject {
  open func begin(_ mapping: NSEntityMapping, with manager: NSMigrationManager) throws
  open func createDestinationInstances(forSource sInstance: NSManagedObject, in mapping: NSEntityMapping, manager: NSMigrationManager) throws
  open func endInstanceCreation(forMapping mapping: NSEntityMapping, manager: NSMigrationManager) throws
  open func createRelationships(forDestination dInstance: NSManagedObject, in mapping: NSEntityMapping, manager: NSMigrationManager) throws
  open func endRelationshipCreation(forMapping mapping: NSEntityMapping, manager: NSMigrationManager) throws
  open func performCustomValidation(forMapping mapping: NSEntityMapping, manager: NSMigrationManager) throws
  open func end(_ mapping: NSEntityMapping, manager: NSMigrationManager) throws
}
{% endhighlight %}

DetectBrandMigrationPolicy 中只需要覆盖 createDestinationInstances 方法：

{% highlight swift %}
class DetectBrandMigrationPolicy: NSEntityMigrationPolicy {

  override func createDestinationInstances(forSource sInstance: NSManagedObject, in mapping: NSEntityMapping, manager: NSMigrationManager) throws {
    guard let name = sInstance.value(forKey: "name") as? String,
      let date = sInstance.value(forKey: "date") as? Date,
      let destinationEntityName = mapping.destinationEntityName else {
      return
    }
    let destinationContext = manager.destinationContext
    
    let dInstance = NSEntityDescription.insertNewObject(forEntityName: destinationEntityName, into: destinationContext)
    dInstance.setValue(name, forKey: "name")
    dInstance.setValue(date, forKey: "date")
    if ["Nike", "Adidas", "UA"].contains(name) {
      dInstance.setValue(true, forKey: "brand")
    } else {
      dInstance.setValue(false, forKey: "brand")
    }
    
    manager.associate(sourceInstance: sInstance, withDestinationInstance: dInstance, for: mapping)
  }

}
{% endhighlight %}