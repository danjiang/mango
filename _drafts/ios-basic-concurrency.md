---
title: iOS 基础 - GCD 和 Operation
author: 但江
avatar: danjiang
location: 成都 
category: programming
tag: ios-basic
---

一些 iOS 基础知识，业务开发中经常用到，面试时也常会被问到，这里总结一下，此篇文章讲解 GCD 和 Operation。

![Apple Develop Xcode](/images/apple-develop-xcode.jpg)

## Thread

使用 GCD 和 Operation 就不用自己管理 Thread，会根据情况自动分配 Thread，避免使用 Thread 产生的很多问题。

Thread 带来的常见问题：

Race Condition：同时读写同一个变量

![Concurrency Race Condition](/images/concurrency_race_condition.png)

Priority Inversion：高优先权的线程被低优先权的线程搁置

![Concurrency Priority Inversion](/images/concurrency_priority_inversion.png)

Deadlock：互锁对方需要的资源

![Concurrency Deadlock](/images/concurrency_deadlock.png)

## GCD

### 基础

派遣任务到 Queue 的方法：

- sync() 等待任务完成，才返回
- async() 不用等待任务完成，立即返回

两种类型：

- serial queue: 按顺序一个个执行任务
- concurrent queue: 同步执行任务

系统内置的 Queue：

- main queue: UI 操作的主线程，serial queue。
- 5 global queues: 通过 DispatchQoS.QoSClass 来获取的 concurrent queue。

自己创建：

{% highlight swift %}
let mySerialQueue = DispatchQueue(label: "com.danthought.serial")

let workerQueue = DispatchQueue(label: "com.danthought.worker", attributes: .concurrent)
{% endhighlight %}

suspend 和 resume，suspend 调用时不会暂停已经在执行的 block，会暂停从 queue 里取 block 来执行，suspend 后还可以继续往 queue 里添加 block：

{% highlight swift %}
private func authorizeCamera() {
    switch AVCaptureDevice.authorizationStatus(for: .video) {
    case .authorized:
        break
    case .notDetermined:
        sessionQueue.suspend()
        AVCaptureDevice.requestAccess(for: .video) { [weak self] granted in
            guard let self = self else { return }
            if !granted {
                self.setupResult = .notAuthorized
            }
            self.sessionQueue.resume()
        }
    default:
        setupResult = .notAuthorized
    }
    
    sessionQueue.async { [weak self] in
        self?.configureSession()
    }
}
{% endhighlight %}

利用 asyncAfter 延时执行 Block：

{% highlight swift %}
DispatchQueue.main.asyncAfter(deadline: DispatchTime.now() + 0.5, execute: { [weak self] in
    self?.dismiss(animated: true, completion: nil)
})
{% endhighlight %}

### DispatchGroup

可以将派遣到不同的 Queue 的任务，分配到一个组，等待这个组里的任务都完成后，接收通知来做其他事情。

示例 1 - slowAdd 是同步的：

{% highlight swift %}
let workerQueue = DispatchQueue(label: "com.danthought.worker", attributes: .concurrent)
let numberArray = [(0,1), (2,3), (4,5), (6,7), (8,9)]

let slowAddGroup = DispatchGroup()

// 可以替换为 DispatchQueue.concurrentPerform(iterations:execute:)
for inValue in numberArray { 
  workerQueue.async(group: slowAddGroup) { // 👈 pay attention
    let result = slowAdd(inValue)
    print("Result = \(result)")
  }
}

let defaultQueue = DispatchQueue.global()
slowAddGroup.notify(queue: defaultQueue) {
  print("SLOW ADD: Completed all tasks")
}
{% endhighlight %}

示例 2 - asyncAdd 是异步的：

{% highlight swift %}
func asyncAdd(_ input: (Int, Int), runQueue: DispatchQueue, completionQueue: DispatchQueue,
              completion: @escaping (Int, Error?) -> ()) {
  runQueue.async {
    var error: Error?
    error = .none
    let result = slowAdd(input)
    completionQueue.async { completion(result, error) }
  }
}

func asyncAdd_Group(_ input: (Int, Int), runQueue: DispatchQueue, completionQueue: DispatchQueue, group: DispatchGroup, completion: @escaping (Int, Error?) -> ()) {
  group.enter() // 👈 pay attention
  asyncAdd(input, runQueue: runQueue, completionQueue: completionQueue) { result, error in
    completion(result, error)
    group.leave() // 👈 pay attention
  }
}

let wrappedGroup = DispatchGroup()

for pair in numberArray {
  asyncAdd_Group(pair, runQueue: workerQueue, completionQueue: defaultQueue, group: wrappedGroup) {
    result, error in
    print("Result = \(result)")
  }
}

wrappedGroup.notify(queue: defaultQueue) {
  print("WRAPPED ASYNC ADD: Completed all tasks")
}
{% endhighlight %}

### Dispatch Barrier

可以让添加到并行 Queue 的任务串行执行，用于解决修改共享数据的 Race Condition：

{% highlight swift %}
class ThreadSafePerson: Person {
  let isolationQueue = DispatchQueue(label: "com.danthought.isolation", attributes: .concurrent)
  
  override func changeName(firstName: String, lastName: String) {
    isolationQueue.async(flags: .barrier) {
      super.changeName(firstName: firstName, lastName: lastName)
    }
  }
  
  override var name: String {
    return isolationQueue.sync {
      return super.name
    }
  }
}
{% endhighlight %}

### DispatchSemaphore

关于信号量的三种操作：
1. 创建（create）一个信号量。这还要求调用者指定初始值，对于二值信号量来说，它通常是 1，但也可以是 0。
2. 等待（wait）一个信号量。该操作会测试会这个信号量的值，如果其值小于或等于 0，那就等待（阻塞），一旦其值变为大于 0 就将它减 1。
3. 挂出（post）一个信号量。该操作将信号量的值加 1。针对 DispatchSemaphore，这个方法是 signal()。

{% highlight swift %}
let url = URL(string: urlString)

let semaphore = DispatchSemaphore(value: 0)
let _ = DownloadPhoto(url: url!) { _, error in
  if let error = error {
    XCTFail("\(urlString) failed. \(error.localizedDescription)")
  }

  semaphore.signal()
}
let timeout = DispatchTime.now() + .seconds(defaultTimeoutLengthInSeconds)

if semaphore.wait(timeout: timeout) == .timedOut {
  XCTFail("\(urlString) timed out")
} 
{% endhighlight %}

### 取消 Dispatch Block

DispatchWorkItem 代表被派遣的 Block 对象，只能在 DispatchWorkItem 到达 Queue 的头部且开始执行前，才能被取消：

{% highlight swift %}
var storedError: NSError?
let downloadGroup = DispatchGroup()
var addresses = [PhotoURLString.overlyAttachedGirlfriend,
                 PhotoURLString.successKid,
                 PhotoURLString.lotsOfFaces]

var blocks: [DispatchWorkItem] = []

for index in 0..<addresses.count {
  downloadGroup.enter()

  let block = DispatchWorkItem(flags: .inheritQoS) {
    let address = addresses[index]
    let url = URL(string: address)
    let photo = DownloadPhoto(url: url!) { _, error in
      if error != nil {
        storedError = error
      }
      downloadGroup.leave()
    }
    PhotoManager.shared.addPhoto(photo)
  }
  blocks.append(block)

  DispatchQueue.main.async(execute: block)
}

for block in blocks[3..<blocks.count] {
  let cancel = Bool.random()
  if cancel {
    block.cancel()
    downloadGroup.leave()
  }
}

downloadGroup.notify(queue: DispatchQueue.main) {
  completion?(storedError)
}
{% endhighlight %}

### DispatchSource

DispatchSource 可以用于监听一些特定事件：Unix 信号，文件描述符，Mach ports 等。

{% highlight swift %}
#if DEBUG

  var signal: DispatchSourceSignal?

  private let setupSignalHandlerFor = { (_ object: AnyObject) in
    let queue = DispatchQueue.main

    signal =
      DispatchSource.makeSignalSource(signal: SIGSTOP, queue: queue)
        
    signal?.setEventHandler {
      print("Hi, I am: \(object.description!)")
    }

    signal?.resume()
  }
#endif
{% endhighlight %}

## Operation 和 OperationQueue

### Operation

直接使用 BlockOperation，并行执行 Operation 的封装：

{% highlight swift %}
let multiPrinter = BlockOperation()
multiPrinter.completionBlock = {
  print("Finished multiPrinting!")
}

multiPrinter.addExecutionBlock {  print("Hello"); sleep(2) }
multiPrinter.addExecutionBlock {  print("my"); sleep(2) }
multiPrinter.addExecutionBlock {  print("name"); sleep(2) }
multiPrinter.addExecutionBlock {  print("is"); sleep(2) }
multiPrinter.addExecutionBlock {  print("XXX"); sleep(2) }

duration {
  multiPrinter.start()
}
{% endhighlight %}

通过子类来创建同步的 Operation：

{% highlight swift %}
class TiltShiftOperation: Operation {
  var inputImage: UIImage?
  var outputImage: UIImage?
  
  override func main() {
    outputImage = tiltShift(image: inputImage)
  }
}

let tsOp = TiltShiftOperation()
tsOp.inputImage = inputImage
tsOp.start() // 此方法的默认实现会调用上面的 main()
tsOp.outputImage
{% endhighlight %}

Operation 的状态：

![iOS Operation States](/images/ios-operation-states.png)

### OperationQueue

管理一组 Operation 执行的 Queue，OperationQueue 可以设置为 isSuspended：

{% highlight swift %}
let filterQueue = OperationQueue()

let appendQueue = OperationQueue()
appendQueue.maxConcurrentOperationCount = 1 // serial queue

for image in images {
  let filterOp = TiltShiftOperation()
  filterOp.inputImage = image
  filterOp.completionBlock = {
    guard let output = filterOp.outputImage else { return }
    appendQueue.addOperation {
      filteredImages.append(output)
    }
  }
  filterQueue.addOperation(filterOp)
}
{% endhighlight %}

### 异步的 Operation

创建一个异步 Operation 的基类 AsyncOperation：

{% highlight swift %}
class AsyncOperation: Operation {
  enum State: String {
    case Ready, Executing, Finished // 后面会看到 isReady, isExecuting 和 isFinished 时只读，所以这里要自己定义状态
    
    fileprivate var keyPath: String {
      return "is" + rawValue
    }
  }
  
  var state = State.Ready { // 便于监听状态的变化
    willSet {
      willChangeValue(forKey: newValue.keyPath)
      willChangeValue(forKey: state.keyPath)
    }
    didSet {
      didChangeValue(forKey: oldValue.keyPath)
      didChangeValue(forKey: state.keyPath)
    }
  }
}

extension AsyncOperation {
  override var isReady: Bool {
    return super.isReady && state == .Ready
  }
  
  override var isExecuting: Bool {
    return state == .Executing
  }
  
  override var isFinished: Bool {
    return state == .Finished
  }
  
  override var isAsynchronous: Bool {
    return true
  }
  
  override func start() {
    if isCancelled {  // 先要判断是否已经 isCancelled
      state = .Finished
      return
    }
    main()
    state = .Executing
  }
  
  override func cancel() {
    state = .Finished
  }
  
}
{% endhighlight %}

通过 AsyncOperation 包装一个异步方法就三步：

1. 继承 AsyncOperation
2. 重写 main() 来调用你的异步方法
3. 当异步方法完成时，修改 state 的状态为 .Finished

{% highlight swift %}
class SumOperation: AsyncOperation {
  let lhs: Int
  let rhs: Int
  var result: Int?
  
  init(lhs: Int, rhs: Int) {
    self.lhs = lhs
    self.rhs = rhs
    super.init()
  }
  
  override func main() {
    asyncAdd_OpQ(lhs: lhs, rhs: rhs) { result in
      self.result = result
      self.state = .Finished
    }
  }
}
{% endhighlight %}

### 依赖关系

管理 Operation 之间的依赖关系，这些 Operation 可以是被添加到不同 OperationQueue，要注意依赖关系形成一个环而导致死锁：

{% highlight swift %}
func addDependency(_ op: Operation)
func removeDependency(_ op: Operation)
var dependencies: [Operation] { get }
{% endhighlight %}

示例 - 先加载图片，再修饰图片：

{% highlight swift %}
class ImageLoadOperation: AsyncOperation {
  var inputName: String?
  var outputImage: UIImage?
  
  override func main() {
    duration {
      simulateAsyncNetworkLoadImage(named: self.inputName) {
        [unowned self] (image) in
        self.outputImage = image
        self.state = .Finished
      }
    }
  }
}

protocol FilterDataProvider {
  var image: UIImage? { get }
}

extension ImageLoadOperation: FilterDataProvider {
  var image: UIImage? { return outputImage }
}

class TiltShiftOperation: Operation {
  var inputImage: UIImage?
  var outputImage: UIImage?
  
  override func main() {
    if let dependencyImageProvider = dependencies
      .filter({ $0 is FilterDataProvider})
      .first as? FilterDataProvider,
      inputImage == .none {
      inputImage = dependencyImageProvider.image
    }
    outputImage = tiltShift(image: inputImage)
  }
}

let imageLoad = ImageLoadOperation()
let filter = TiltShiftOperation()

imageLoad.inputName = "train_day.jpg"

filter.addDependency(imageLoad)
{% endhighlight %}

### 取消 Operation

cancel() 只是将 isCancelled 变为 true，start() 的默认实现会判断 isCancelled 如果是 true，则将状态变为 isFinished，AsyncOperation 中覆盖 start() 的实现也要保持这种逻辑。已经处于 isExecuting 状态的 Operation，根据情况来决定怎么取消任务的执行，或需要将状态变为 isFinished。

{% highlight swift %}
class ArraySumOperation: Operation {
  let inputArray: [(Int, Int)]
  var outputArray = [Int]()
  
  init(input: [(Int, Int)]) {
    inputArray = input
    super.init()
  }
  
  override func main() {
    for pair in inputArray {
      if isCancelled { return }  // 👈 pay attention
      outputArray.append(slowAdd(pair))
    }
  }
}
{% endhighlight %}

## 调试

Xcode Thread Sanitizer

![Xcode Thread Sanitizer](/images/xcode-tsan.png)

