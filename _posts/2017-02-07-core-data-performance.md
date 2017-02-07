---
title: Core Data 性能
author: 但江
avatar: danjiang
location: 成都 
category: programming
tag: core-data
---

从这篇文章，你将学习到有关 Core Data 的性能方面问题，应该如何处理，[示例代码地址](https://github.com/danjiang/Shop)，让我们开始吧。

![Core Data Performance](/images/core-data-performance.png)

## 优化数据模型

### 二进制数据放在哪里？

图片这种二进制数据应该放在哪里：

* 小于 100 KB：在相关表格直接存储为一个属性；
* 大于 100 KB，小于 1 M：单独建立一个表格来存储，并且和主表建立关系；
* 大于 1M：不放在数据库中，放在磁盘文件中。

开启 Store in External Record File，开启过后，Core Data 会自行决定这个图片是否足够小，可以保存在 SQLite 文件中，或者是太大了，要保存在外部磁盘文件中。

![Core Data Store In External Record File](/images/core-data-store-in-external-record-file.jpg)

### 避免实体继承

Core Data 虽然有处理实体继承的功能，但是内部的实现方式很不优雅，将所有可能的字段放在一个表格中，导致字段空值很多，最好别用这个功能。

### 反范式化数据来提高性能

范式化的目的就是为了消除冗余，但是很多时候，设置一些冗余字段可以大大提高性能。

* 只用于查询的字段：如果表格中有一个大文本字段，存储的文字非常多，可以单独在创建一个字段只存储文本字段中的摘要信息，在实现检索功能时，可以提高性能。
* 耗时的计算：如果要展示的信息可以从表格中已有的字段计算而来，但是非常耗时，可以直接将计算结果作为一个字段存储起来，再次读取就不用在计算了。
* 重复字段：如果一个字段在多个实体中重复，也不一定要将其抽取出来放在一个单独的表格中，遍历关系比直接从实体中获取要耗费性能的多。
* 区分常用和不常用的数据：这就要根据需求来了，如果只是在详细界面才显示的数据，很多列表中不显示的数据，可以将其从主表中抽取出来放在单独的表格中，减少主表中的字段数量。

## 查询

NSFetchRequest 查询方式不同也会影响获取的效率。

### 只获取 NSManagedObjectID

NSManagedObjectID 是每个 NSManagedObject 的唯一标示，只获取标示性能自然很高，还会将 NSManagedObject 加载到 Persistent Store Coordinator 中作为缓存：

{% highlight swift %}
let request = NSFetchRequest<NSFetchRequestResult>(entityName: "Item")
request.resultType = .managedObjectIDResultType
{% endhighlight %}

### 加载为一个 Fault

查询返回的也是一个 NSManagedObject 数组，但是每一个 NSManagedObject 只包含 NSManagedObjectID，NSManagedObject 的属性值处于一个 Faulted 状态，当一个属性值被获取时，才会加载 NSManagedObject 的所有属性值：

{% highlight swift %}
let request = NSFetchRequest<NSFetchRequestResult>(entityName: "Item")
request.includesPropertyValues = false
{% endhighlight %}

### 加载属性值

加载 NSManagedObject 的所有属性值，但是不会加载关系，这是 NSFetchRequest 的默认表现：

{% highlight swift %}
let request = NSFetchRequest<NSFetchRequestResult>(entityName: "Item")
{% endhighlight %}

### 加载关系

加载 NSManagedObject 的所有属性值，同时加载关系为 Fault：

{% highlight swift %}
let request = NSFetchRequest<NSFetchRequestResult>(entityName: "Category")
request.relationshipKeyPathsForPrefetching = ["items"]
{% endhighlight %}

### 磁盘访问

> NSFetchRequest 总是会访问磁盘

### Faulting

> Faulting 过程，从缓存到磁盘

1. 如果要获取的对象已经在 Managed Object Context 中，就直接从 Managed Object Context 中返回值；
2. 如果要获取的对象不在 Managed Object Context 中，而在 Persistent Store Coordinator 中有缓存值，因为之前获取过，就直接从 Persistent Store Coordinator 中返回值；
3. 如果要获取的对象在 Managed Object Context 和 Persistent Store Coordinator 都是第一次，那就只能从数据库文件中获取。


