---
title: 深入 NSFetchedResultsController
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: objective-c
---

👉 *[此文章有 Swift 的更新版本](/programming/2016/05/16/core-data-nsfetchedresultscontroller/)*

NSFetchedResultsController 可以说是让人又爱又恨，如果同常规的 UITableView 或 UICollectionView 进行结合，你会觉得非常爽，简单而又高效，但是如果同你自己写的视图或者其他内置视图结合时，你就会觉得用起来非常别扭。本文将阐述 NSFetchedResultsController 常规用法，然后深入其内部实现机制，从而知道如何根据自己情况构建类似 NSFetchedResultsController 的功能。

## 常规用法 

Controller 中有 NSFetchedResultsController 的属性。

{% highlight objc %}
@interface DTLogsViewController () <UITableViewDataSource, UITableViewDelegate, NSFetchedResultsControllerDelegate>

@property (nonatomic, weak) IBOutlet UITableView *tableView;
@property (nonatomic, strong) NSFetchedResultsController *fetchedResultsController;

@end
{% endhighlight %}

初始化 NSFetchedResultsController，创建 NSFetchRequest 来查询日志，根据日期来分组，并且按照创建时间来排序，注意一定要排序。

{% highlight objc %}
- (void)initFetchedResultsController {
  NSFetchRequest *fetchRequest = [NSFetchRequest fetchRequestWithEntityName:@"Log"];
  fetchRequest.sortDescriptors = @[[[NSSortDescriptor alloc] initWithKey:@"createTime" ascending:NO]];
  NSFetchedResultsController *fetchedResultsController = [[NSFetchedResultsController alloc] initWithFetchRequest:fetchRequest
                                                                                               managedObjectContext:self.moc
                                                                                                 sectionNameKeyPath:@"date"
                                                                                                          cacheName:nil];
  self.fetchedResultsController = fetchedResultsController
  self.fetchedResultsController.delegate = self;
  NSError *error;
  if (![self.fetchedResultsController performFetch:&error]) {
    NSLog(@"Error fetched Log %@\n%@", [error localizedDescription], [error userInfo]);
  } else {
    dispatch_async(dispatch_get_main_queue(), ^{
      [self.tableView reloadData];
    });
  }
}
{% endhighlight %}

UITableView 从 NSFetchedResultsController 中获取内容来显示。

{% highlight objc %}
- (NSInteger)numberOfSectionsInTableView:(UITableView *)tableView {
  return [self.fetchedResultsController.sections count];
}

- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
  id<NSFetchedResultsSectionInfo> sectionInfo = self.fetchedResultsController.sections[section];
  return sectionInfo.numberOfObjects;
}

- (UIView *)tableView:(UITableView *)tableView viewForHeaderInSection:(NSInteger)section {
  DTLogHeader *header = [tableView dequeueReusableHeaderFooterViewWithIdentifier:DTLogHeaderIdentifier];
  [self configueHeader:header section:section];
  return header;
}

- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
  UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:DTLogCellIdentifier];
  NSManagedObject *object = [self.fetchedResultsController objectAtIndexPath:indexPath];
  [self configueCell:cell object:object];
  return cell;
}
{% endhighlight %}

UITableView 通过 NSFetchedResultsControllerDelegate 感知数据库的变化，从而自动更新显示的内容。

{% highlight objc %}
- (void)controllerWillChangeContent:(NSFetchedResultsController *)controller {
  [self.tableView beginUpdates];
}

- (void)controller:(NSFetchedResultsController *)controller didChangeSection:(id<NSFetchedResultsSectionInfo>)sectionInfo atIndex:(NSUInteger)sectionIndex forChangeType:(NSFetchedResultsChangeType)type {
  NSIndexSet *indexSet = [NSIndexSet indexSetWithIndex:sectionIndex];
  switch(type) {
    case NSFetchedResultsChangeInsert:
      [self.tableView insertSections:indexSet withRowAnimation:UITableViewRowAnimationAutomatic];
      break;
    case NSFetchedResultsChangeDelete:
      [self.tableView deleteSections:indexSet withRowAnimation:UITableViewRowAnimationAutomatic];
      break;
  }
}

- (void)controller:(NSFetchedResultsController *)controller didChangeObject:(id)anObject atIndexPath:(NSIndexPath *)indexPath forChangeType:(NSFetchedResultsChangeType)type newIndexPath:(NSIndexPath *)newIndexPath {
   switch(type) {
     case NSFetchedResultsChangeInsert:
       [self.tableView insertRowsAtIndexPaths:@[newIndexPath] withRowAnimation:UITableViewRowAnimationAutomatic];
       break;
     case NSFetchedResultsChangeDelete:
       [self.tableView deleteRowsAtIndexPaths:@[indexPath] withRowAnimation:UITableViewRowAnimationAutomatic];
       break;
     case NSFetchedResultsChangeUpdate:
       [self.tableView reloadRowsAtIndexPaths:@[indexPath] withRowAnimation:UITableViewRowAnimationAutomatic];
       break;
     case NSFetchedResultsChangeMove:
       [self.tableView moveRowAtIndexPath:indexPath toIndexPath:newIndexPath];
       break;
   }
}

- (void)controllerDidChangeContent:(NSFetchedResultsController *)controller {
  [self.tableView endUpdates];
}
{% endhighlight %}

## 深入内部

常见的情况是我们希望自己写的视图能够感知数据库的变化来自动更新内容，但是又不想像 NSFetchedResultsController 同 UITableView 内容结构绑定的这么死，那么我们就要了解一下 NSFetchedResultsController 是如何感知数据库的变化的，NSFetchedResultsController 通过监听 NSManagedObjectContext 发出的 NSManagedObjectContextObjectsDidChangeNotification， NSManagedObjectContextWillSaveNotification 和 NSManagedObjectContextDidSaveNotification 来感知数据库的变化，并且进一步缓存数据，控制 NSIndexPath 和数据之间的对应关系等等。

我们来实现 DTContextObserver 来监听 NSManagedObjectContextDidSaveNotification，并且通过 NSPredicate 来筛选变动的数据，如果是感兴趣的数据才调用代理方法。

{% highlight objc %}
@protocol DTContextObserverDelegate <NSObject>

@optional
- (void)contextObserverDidInsertForPredicate:(NSPredicate *)predicate;
- (void)contextObserverDidDeleteForPredicate:(NSPredicate *)predicate;
- (void)contextObserverDidUpdateForPredicate:(NSPredicate *)predicate;

@end

@interface DTContextObserver : NSObject

@property (nonatomic, weak) id<DTContextObserverDelegate> delegate;
@property (nonatomic, strong) NSPredicate *predicate;

- (instancetype)initWithManagedObjectContext:(NSManagedObjectContext *)context;

@end


#import "DTContextObserver.h"

@interface DTContextObserver ()

@property (nonatomic, strong) NSManagedObjectContext *context;

@end

@implementation DTContextObserver

- (void)dealloc {
  [[NSNotificationCenter defaultCenter] removeObserver:self name:NSManagedObjectContextDidSaveNotification object:self.context];
}

- (instancetype)initWithManagedObjectContext:(NSManagedObjectContext *)context {
  self = [super init];
  if (self) {
    _context = context;
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(contextUpdated:) name:NSManagedObjectContextDidSaveNotification object:self.context];
  }
  return self;
}

- (void)contextUpdated:(NSNotification*)notification {
  NSSet *insertedObjects = [[notification userInfo] objectForKey:NSInsertedObjectsKey];
  NSSet *deletedObjects = [[notification userInfo] objectForKey:NSDeletedObjectsKey];
  NSSet *updatedObjects = [[notification userInfo] objectForKey:NSUpdatedObjectsKey];
  insertedObjects = [insertedObjects filteredSetUsingPredicate:self.predicate];
  deletedObjects = [deletedObjects filteredSetUsingPredicate:self.predicate];
  updatedObjects = [updatedObjects filteredSetUsingPredicate:self.predicate];
  if (insertedObjects.count > 0 && [self.delegate respondsToSelector:@selector(contextObserverDidInsertForPredicate:)]) {
    [self.delegate contextObserverDidInsertForPredicate:self.predicate];
  }
  if (deletedObjects.count > 0 && [self.delegate respondsToSelector:@selector(contextObserverDidDeleteForPredicate:)]) {
    [self.delegate contextObserverDidDeleteForPredicate:self.predicate];
  }
  if (updatedObjects.count > 0 && [self.delegate respondsToSelector:@selector(contextObserverDidUpdateForPredicate:)]) {
    [self.delegate contextObserverDidUpdateForPredicate:self.predicate];
  }
}

@end
{% endhighlight %}
