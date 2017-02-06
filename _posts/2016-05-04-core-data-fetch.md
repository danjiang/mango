---
title: Core Data 查询
author: 但江
avatar: danjiang
location: 成都 
category: programming
tag: core-data
---

从这篇文章，你将学习到有关 Core Data 的新增、修改、删除和查询的详细解读，[示例代码地址](https://github.com/danjiang/Shop)，让我们开始吧。

![Core Data Fetch](/images/core-data-fetch.png)

## 新增，修改和删除

{% highlight swift %}
let object = NSEntityDescription.insertNewObject(forEntityName: "Category", into: moc)
object.setValue("Running", forKey: "name")
object.setValue(false, forKey: "brand")
object.setValue(Date(), forKey: "date")

do {
  try moc.save()
} catch {
  print("!!! Error: save managed object in context !!!\n\(error)\n")
}
{% endhighlight %}

{% highlight swift %}
let object = moc.object(with: objectId)
object.setValue("Nike", forKey: "name")
object.setValue(true, forKey: "brand")
object.setValue(Date(), forKey: "date")

do {
  try moc.save()
} catch {
  print("!!! Error: save managed object in context !!!\n\(error)\n")
}
{% endhighlight %}

{% highlight swift %}
let object = moc.object(with: objectId)
moc.delete(object)

do {
  try moc.save()
} catch {
  print("!!! Error: save managed object in context !!!\n\(error)\n")
}
{% endhighlight %}

## 使用 NSFetchRequest 查询

通过创建 NSFetchRequest 来进行数据库信息查询，NSEntityDescription 来指定实体类型，NSPredicate 来过滤数据结果，NSSortDescriptor 描述数据排序方式。

### 基本查询

通过 NSFetchRequest 传入 entityName 参数指定要查询的实体 Item 来构造查询，然后通过 NSManagedObjectContext 的 executeFetchRequest 方法执行查询，查询过程有可能会出错，所以需要捕获异常：

{% highlight swift %}
let request = NSFetchRequest<NSFetchRequestResult>(entityName: "Item")

do {
  return try moc.fetch(request) as? [NSManagedObject]
} catch {
  print("!!! Error: find managed object in context !!!\n\(error)\n")
  return nil
}
{% endhighlight %}

## 排序结果

在构造 NSFetchRequest 查询时，可以进一步通过 NSSortDescriptor 来指定排序方式，sortDescriptors 可以传入多个排序方式，结果将按照从左到右的排序方式依次排序：

{% highlight swift %}
let request = NSFetchRequest<NSFetchRequestResult>(entityName: "Category")
request.sortDescriptors = [NSSortDescriptor(key: "brand", ascending: false),
                           NSSortDescriptor(key: "date", ascending: false)]

do {
  return try moc.fetch(request) as? [NSManagedObject]
} catch {
  print("!!! Error: find managed object in context !!!\n\(error)\n")
  return nil
}
{% endhighlight %}

## 筛选结果

在构造 NSFetchRequest 查询时，还可以进一步通过 NSPredicate 来指定过滤条件，下面是通过 [Predicate Format String Syntax](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/Predicates/Articles/pSyntax.html) 来创建过滤条件：

{% highlight swift %}
let request = NSFetchRequest<NSFetchRequestResult>(entityName: "Item")
request.sortDescriptors = [NSSortDescriptor(key: "date", ascending: false)]
request.predicate = NSPredicate(format: "category = %@ AND name BEGINSWITH %@", category, name)

do {
  return try moc.fetch(request) as? [NSManagedObject]
} catch {
  print("!!! Error: find managed object in context !!!\n\(error)\n")
  return nil
}
{% endhighlight %}

还可以通过 [Predicate Classes](https://developer.apple.com/library/mac/documentation/Cocoa/Reference/Foundation/Classes/NSCompoundPredicate_Class/index.html#//apple_ref/occ/cl/NSCompoundPredicate) 来构建过滤条件，这种创建类的方式相比上面的字符串方式，在需要判断很多条件来组装过滤条件时更加方便：

{% highlight swift %}
let request = NSFetchRequest<NSFetchRequestResult>(entityName: "Item")
request.sortDescriptors = [NSSortDescriptor(key: "date", ascending: false)]

let categoryPredicate = NSComparisonPredicate(
  leftExpression: NSExpression(forKeyPath: "category"),
  rightExpression: NSExpression(forConstantValue: category),
  modifier: .direct,
  type: .equalTo,
  options: [])

let namePredicate = NSComparisonPredicate(
  leftExpression: NSExpression(forKeyPath: "name"),
  rightExpression: NSExpression(forConstantValue: name),
  modifier: .direct,
  type: .equalTo,
  options: [])

request.predicate = NSCompoundPredicate(andPredicateWithSubpredicates: [categoryPredicate, namePredicate])

do {
  return try moc.fetch(request) as? [NSManagedObject]
} catch {
  print("!!! Error: find managed object in context !!!\n\(error)\n")
  return nil
}
{% endhighlight %}