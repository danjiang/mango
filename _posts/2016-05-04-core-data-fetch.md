---
title: Core Data 查询
author: 但江
avatar: danjiang
location: 成都 
category: programming
tag: core-data
---

从这篇文章，你将学习到有关 Core Data 的查询更详细的解读，[示例代码地址](https://github.com/danjiang/blog-core-data)，让我们开始吧。

![Core Data Data Model Graph](/images/core-data-data-model-graph.jpg)

## 基本查询

通过 **NSFetchRequest** 传入 **entityName** 参数指定要查询的实体 **Item** 来构造查询，然后通过 **NSManagedObjectContext** 的 **executeFetchRequest** 方法执行查询，查询过程有可能会出错，所以需要捕获异常。

{% highlight swift %}
func findItemsBasic() -> [NSManagedObject]? {
  let request = NSFetchRequest(entityName: "Item")
  do {
    return try moc!.executeFetchRequest(request) as? [NSManagedObject]
  } catch {
    print("!!! Error: find managed object in context !!!\n\(error)\n")
  }
  return nil
}
{% endhighlight %}

## 排序结果

在构造 **NSFetchRequest** 查询时，可以进一步通过 **NSSortDescriptor** 来指定排序方式，**sortDescriptors** 可以传入多个排序方式，结果将按照从左到右的排序方式依次排序。

{% highlight swift %}
func findItemsOrder() -> [NSManagedObject]? {
  let request = NSFetchRequest(entityName: "Item")
  request.sortDescriptors = [NSSortDescriptor(key: "price", ascending: true)]
  do {
    return try moc!.executeFetchRequest(request) as? [NSManagedObject]
  } catch {
    print("!!! Error: find managed object in context !!!\n\(error)\n")
  }
  return nil
}
{% endhighlight %}

## 筛选结果

在构造 **NSFetchRequest** 查询时，还可以进一步通过 **NSPredicate** 来指定过滤条件，下面是通过 [Predicate Format String Syntax](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/Predicates/Articles/pSyntax.html) 来创建过滤条件：指定类别和条目名称开头。

{% highlight swift %}
func findItemsPredicateFormatString(category: NSManagedObject, name: String) -> [NSManagedObject]? {
  let request = NSFetchRequest(entityName: "Item")
  request.sortDescriptors = [NSSortDescriptor(key: "price", ascending: true)]
  request.predicate = NSPredicate(format: "category = %@ AND name BEGINSWITH %@", category, name)
  do {
    return try moc!.executeFetchRequest(request) as? [NSManagedObject]
  } catch {
    print("!!! Error: find managed object in context !!!\n\(error)\n")
  }
  return nil
}
{% endhighlight %}

还可以通过 [Predicate Classes](https://developer.apple.com/library/mac/documentation/Cocoa/Reference/Foundation/Classes/NSCompoundPredicate_Class/index.html#//apple_ref/occ/cl/NSCompoundPredicate) 来构建过滤条件，这种创建类的方式相比上面的字符串方式，在需要判断很多条件来组装过滤条件时更加方便。

{% highlight swift %}
func findItemsCompoundPredicate(category: NSManagedObject, name: String) -> [NSManagedObject]? {
  let request = NSFetchRequest(entityName: "Item")
  request.sortDescriptors = [NSSortDescriptor(key: "price", ascending: true)]
  let categoryPredicate = NSComparisonPredicate(
    leftExpression: NSExpression(forKeyPath: "category"),
    rightExpression: NSExpression(forConstantValue: category),
    modifier: NSComparisonPredicateModifier.DirectPredicateModifier,
    type: NSPredicateOperatorType.EqualToPredicateOperatorType,
    options: [])
  let namePredicate = NSComparisonPredicate(
    leftExpression: NSExpression(forKeyPath: "name"),
    rightExpression: NSExpression(forConstantValue: name),
    modifier: NSComparisonPredicateModifier.DirectPredicateModifier,
    type: NSPredicateOperatorType.BeginsWithPredicateOperatorType,
    options: [])
  request.predicate = NSCompoundPredicate(andPredicateWithSubpredicates: [categoryPredicate, namePredicate])
  do {
    return try moc!.executeFetchRequest(request) as? [NSManagedObject]
  } catch {
    print("!!! Error: find managed object in context !!!\n\(error)\n")
  }
  return nil
}
{% endhighlight %}