# 多线程

### iOS中的几种多线程分案
- pthread
    - 一套用C语言写的，跨平台的多线程API
    - 线程的生命周期需要自己管理，使用难度大
- NSThread
    - 一套基于pthread封装的面向对象的API
    - 简单易用，可直接操作线程对象，但是线程的生命周期需要自己管理
- GCD 
    - 也是基于C语言的API，旨在替代NSThread等线程技术
    - 以队列的形式来管理线程，内部维护了一个线程池。
    - 线程的生命周期自动管理
    - 充分利用设备的多核特性
- NSOpreration
    - 基于GCD封装的一套API，更加面对对象
    - 比GCD多了一些实用功能

### GCD 

- 容易混淆的术语：队列（分为串行serial，并发concurrent）函数（分为同步sync，异步async）
    - 同步和异步主要影响能不能开启新的线程
        - 同步：在当前线程中执行任务，不具备开启线程的能力
        - 异步：在新的线程中执行任务，具备开启新线程的能力
    - 串行和并发主要影响任务的执行方式
        - 串行：一个任务执行完毕后，再执行下一个任务
        - 并发：多个任务同时执行，在异步函数下能开启新线程
- 死锁：使用sync函数往当前串行队列中添加任务，会卡住当前队列，造成死锁。


### 多线程安全  
使用线程同步技术（加锁）
- 临界区，开锁解锁之间的区域
- 互斥锁，使线程休眠
    - NSLock
    - pthread_mutex
    - @synchronized
- 自旋锁：线程反复检查变量是否可用（while循环），避免上下文调度开销
    - 目前已经不再安全，可能会出现优先级翻转现象
    - 如果等待解锁的线程优先级别较高，它会一直占用CPU资源，优先级低的线程无法完成解锁
    - OSSpinLock（
- 读写锁，共享互斥锁，读可以进入，写不可以进入。通过互斥锁，条件变量，信号变量。
    - pthread_rwlock
- 递归锁，同一个线程加锁N次不会死锁。
    - NSRecursiveLock
    - pthread_mutex(recursive)
- 条件锁，当不满足条件的时候休眠，满足条件时打开。
    - NSCondition
    - NSConditionLock
- 信号量，互斥锁就是信号量取值0/1的特例。
   - dispatch_semaphore

### 读写安全
- dispatch_barrier_async
    - 这个函数的并发队列必须是自己通过dispatch_queue_create创建的 
    - 如果传入的是一个串行队列，或者是一个全局并发队列。这个函数等于dispatch_async函数

### 多线程之间的通信
- `- (void)performSelector:(SEL)aSelector onThread:(NSThread *)thr withObject:(id)arg waitUntilDone:(BOOL)wait;`
- GCD中函数队列的调用
- 全局变量加锁

### 常见的面试题
```
class ThreadTest {
        
    static let shared = ThreadTest()
    
    private let semaphoreA = DispatchSemaphore(value: 1)
    private let semaphoreB = DispatchSemaphore(value: 0)
    private let semaphoreC = DispatchSemaphore(value: 0)
    
    private let lockA = NSLock()
    private let lockB = NSLock()
    private let lockC = NSLock()
        
    private var callbackCount = 0
    
    init() {
//        printABCSerial1()
//        printABCSerial2()
//        printAandBtoC1()
        printAandBtoC2()
        print("end")
    }
    // 三个线程顺序打印ABCABC
    // 使用三把锁，分别让三个编程休眠，然后依次解开下一个需要打印值的锁
    func printABCSerial1() {
        func printA() {
            (0..<10).forEach { _ in
                semaphoreA.wait() //
                print("A---", Thread.current)
                semaphoreB.signal()
            }
        }
        func printB() {
            (0..<10).forEach { _ in
                semaphoreB.wait()
                print("B---", Thread.current)
                semaphoreC.signal()
            }
        }
        func printC() {
            (0..<10).forEach { _ in
                semaphoreC.wait()
                print("C---", Thread.current)
                semaphoreA.signal()
            }
        }
        let queue = DispatchQueue.global()
        queue.async {
            printA()
        }
        queue.async {
            printB()
        }
        queue.async {
            printC()
        }
    }
    
    func printABCSerial2() {
        lockB.lock()
        lockC.lock()
        func printA() {
            (0..<10).forEach { _ in
                lockA.lock()
                print("A---", Thread.current)
                lockB.unlock()
                
            }
        }
        func printB() {
            (0..<10).forEach { _ in
                lockB.lock()
                print("B---", Thread.current)
                lockC.unlock()
            }
        }
        func printC() {
            (0..<10).forEach { _ in
                lockC.lock()
                print("C---", Thread.current)
                lockA.unlock()
            }
        }
        let queue = DispatchQueue.global()
        queue.async {
            printA()
        }
        queue.async {
            printB()
        }
        queue.async {
            printC()
        }
    }
    // 等A和B都打印出来后，再打印C
    func printAandBtoC1() {
        let queue = DispatchQueue.global()
        let group = DispatchGroup()
        group.enter()
        queue.async {
            sleep(2)
            print("A---", Thread.current)
            group.leave()
        }
        group.enter()
        queue.async {
            sleep(1)
            print("B---", Thread.current)
            group.leave()
        }
        group.notify(queue: queue) {
            print("C---", Thread.current)
        }
        print("func end")
    }
    
    func printAandBtoC2() {
        let queue = DispatchQueue.global()
        queue.async {
            sleep(2)
            print("A---", Thread.current)
            self.semaphoreA.signal()
        }
        queue.async {
            sleep(1)
            print("B---", Thread.current)
            self.semaphoreA.signal()
        }
        queue.async {
            self.semaphoreA.wait()
            self.semaphoreA.wait()
            self.semaphoreA.wait()
            print("C---", Thread.current)
        }
        print("func end")
    }
    
    func printAandBtoC3() {
        let queue = DispatchQueue.global()
        let printC = { [weak self] in
            guard let self = self else { return }
            defer {
                self.lockA.unlock()
            }
            self.lockA.lock()
            self.callbackCount += 1
            if self.callbackCount < 2 {
                return
            }
            print("C---", Thread.current)
            
        }
        queue.async {
            sleep(2)
            print("A---", Thread.current)
            printC()
        
        }
        queue.async {
            sleep(1)
            print("B---", Thread.current)
            printC()
        }
        print("func end")
    }
    
    // 延迟操作
    func delay(seconds: Double, execute: @escaping () -> Void) {
        DispatchQueue.main.asyncAfter(deadline: DispatchTime.now() + seconds, execute: execute)
    }
}
```
- 在子线程发送的通知，在主线程注册的观察者能收到。但是调用的方法是在发送通知的线程。

# Swift GCD
- 创建一个队列
```
DispatchQueue.main
DispatchQueue.global(qos: DispatchQoS.QoSClass.background)
/** 参数解析：
    1、label：就是给队列的一个标识，当你想要获取这个队列的时候可以根据这个属性
    2、qos：队列的优先级
    3、attributes：队列的形式，默认是串行serial，是一个optionset
    4、autoreleaseFrequency：自动释放对象的频率
    5、target：将队列添加到其它队列
*/
DispatchQueue(label: "name", qos: DispatchQoS.background, attributes: [], autoreleaseFrequency: DispatchQueue.AutoreleaseFrequency.never, target: nil)

struct Attributes : OptionSet {
    static let concurrent: DispatchQueue.Attributes // 并发队列 
    public static let initiallyInactive: DispatchQueue.Attributes // 设置后任务不会自动启动，需要手动启动 queue.active()
}

enum AutoreleaseFrequency {
    case inherit // 继承自target queue
    @available(OSX 10.12, iOS 10.0, tvOS 10.0, watchOS 3.0, *)
    case workItem // 队列在block开始的时候创建一个autoreleasepool，在block结束的时候释放对象
    @available(OSX 10.12, iOS 10.0, tvOS 10.0, watchOS 3.0, *)
    case never // 不自动释放对象
}

enum QoSClass { // 优先级从高到低
    case userInteractive // 服务于用户交互
    case userInitiated // 防止用户积极的使用app
    case `default`
    case utility // 用户未主动跟踪的任务
    case background // 自己创建和清理的
    case unspecified
}
```
- 执行函数
```
let group = DispatchGroup() // 创建一个任务分组
let block: () -> Void = {} // 添加的任务
let workItem = DispatchWorkItem(qos: DispatchQoS.default, flags: DispatchWorkItemFlags.barrier, block: block) // 将任务封装成一个工作元素，并配置优先级，和操作信息
queue.async(group: group, execute: workItem) // 异步调用
queue.async(group: group, qos: DispatchQoS.background, flags: DispatchWorkItemFlags.barrier, execute: block) // 里面应该会合成一个workitem调用上一个方法
queue.sync(flags: DispatchWorkItemFlags.inheritQoS, execute: block) // 同步函数不需要任务组

// flag 用来配置队列任务执行的行为，包括其服务质量，创建障碍，还是生成新的分离线程
struct DispatchWorkItemFlags : OptionSet, RawRepresentable {
    public static let barrier: DispatchWorkItemFlags // 当提交到并发队列中的时候，在这个workitem之前的执行完，执行这个，然后执行后面的
    public static let detached: DispatchWorkItemFlags // 不将上下文的attribute应用在该workitem上
    public static let assignCurrentContext: DispatchWorkItemFlags // 设置workitem的attitudes为负责执行任务的队列或线程继承的
    public static let noQoS: DispatchWorkItemFlags // workitem不分配QoS 该标志优先于assignCurrentContext
    public static let inheritQoS: DispatchWorkItemFlags // 只要不降低服务质量，queue的覆盖workitem的
    public static let enforceQoS: DispatchWorkItemFlags // 只要不降低服务质量，workitemqos的话覆盖queue的
}

```



[iOS 多线程：『GCD』详尽总结](https://www.jianshu.com/p/2d57c72016c6)  
