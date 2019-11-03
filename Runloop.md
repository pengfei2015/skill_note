# Runloop

### runloop是什么
- runloop是运行循环。如果没有runloop程序运行完就会立即退出。
- runloop可以在需要的时候自己跑起来，在没有操作的时候休眠。节省CPU资源，提高程序性能。

### Runloop的作用
- 保持程序运行，程序启动后会开启一个线程，主动开启runloop。Runloop不销毁线程就不会销毁。
- 处理APP中的各种事件，比如定时器事件，触摸事件，Selector事件
- 节省CPU资源，提高程序性能。

### Runloop和线程的关系
- 每个线程都有唯一一个与之对应的Runloop对象。关系保存在全局的字典中，(key: pthread value:runLoop)
- Thread默认是没有对应的RunLoop的，仅当主动调用Get方法时，才会创建。
- Runloop会在线程结束时销毁
- 主线程的Runloop已经自动获取，子线程没有开启Runloop

### Runloop相关的类
- CFRunloopRef
```
struct __CFRunLoop {
    CFRuntimeBase _base;
    pthread_mutex_t _lock;            /* locked for accessing mode list */
    __CFPort _wakeUpPort;            // used for CFRunLoopWakeUp 
    Boolean _unused;
    volatile _per_run_data *_perRunData;              // reset for runs of the run loop
    pthread_t _pthread;
    uint32_t _winthread;
    CFMutableSetRef _commonModes; 
    CFMutableSetRef _commonModeItems; 
    CFRunLoopModeRef _currentMode; // 当前mode只有一个
    CFMutableSetRef _modes; // 一个RunLoop有很多Mode
    struct _block_item *_blocks_head;
    struct _block_item *_blocks_tail;
    CFAbsoluteTime _runTime;
    CFAbsoluteTime _sleepTime;
    CFTypeRef _counterpart;
};
```
- CFRunloopModeRef
    - kCFRunLoopDefaultMode：默认的Mode，通常主线程在这个mode下运行
    - UITrackingRunLoopMode：界面跟踪Mode，用于ScrollView的滑动，保证滑动时不受其它Mode影响
    - UIInitializationRunLoopMode：在刚启动 App 时第进入的第一个 Mode，启动完成后就不再使用，会切换到kCFRunLoopDefaultMode
    - GSEventReceiveRunLoopMode：接受系统事件的内部 Mode，通常用不到
    - kCFRunLoopCommonModes：这是一个占位用的Mode，作为标记kCFRunLoopDefaultMode和UITrackingRunLoopMode用，并不是一种真正的Mode 
- CFRunloopSourceRef
    - source0 
        - 触摸事件处理
        - performselector
    - source1
        - 基于port的线程间通信
        - 系统事件捕捉(点击事件source1捕捉后，包装后交给source0处理)
- CFRunloopTimerRef
    - NSTimer
    - performSelector: withObject:afterDelay
- CFRunloopObserverRef
    - 监听Runloop的状态
    - UI刷新（beforeWaiting）
    - AutoreleasePool（beforeWaiting）
    ```
    // 自己创建一个Observer监听Runloop
    // 创建observer
    CFRunLoopObserverRef observer = CFRunLoopObserverCreateWithHandler(kCFAllocatorDefault, kCFRunLoopAllActivities, YES, 0, ^(CFRunLoopObserverRef observer, CFRunLoopActivity activity) {
        switch (activity) {
            case kCFRunLoopEntry:
                NSLog(@"");
                break;
            default:
                break;
        }
    });
    // 添加到runloop中
    CFRunLoopAddObserver(CFRunLoopGetMain(), observer, kCFRunLoopCommonModes);
    // 释放
    CFRelease(observer);
    ```
### Runloop的状态
```
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
    kCFRunLoopEntry = (1UL << 0),  // 即将进入mode
    kCFRunLoopBeforeTimers = (1UL << 1), // 即将处理Timer
    kCFRunLoopBeforeSources = (1UL << 2), // 即将处理Source 
    kCFRunLoopBeforeWaiting = (1UL << 5), // 即将睡眠
    kCFRunLoopAfterWaiting = (1UL << 6), // 刚从睡眠中唤醒
    kCFRunLoopExit = (1UL << 7), // 即将退出mode
    kCFRunLoopAllActivities = 0x0FFFFFFFU
};
```
### Runloop运行逻辑
1. 通知Observer：进入Loop
2. 通知Observer：即将处理Timers
3. 通知Observer：即将处理Sources
4. 处理添加进Runloop的Blocks
5. 处理Source0（可能再次处理Blocks）
6. 如果有Source1，跳转第8步
7. 通知Observer：开始休眠
8. 通知Observer：结束休眠（被某个消息唤醒）
    1. 处理Timer
    2. 处理GCD Async To Main Queue
    3. 处理Source1
9. 处理Blocks
10. 根据签名的执行结果，决定如何操作
    1. 回到第二步
    2. 退出Loop
11. 通知Observer：退出Loop

### Runloop总结
- CFRunloopModeRef代表Runloop的运行模式
- 一个Runloop有d若干个Mode，每个Mode又包含若干个source0、source1，timer，observer
- Runloop启动的时候只能选择一个Mode，作为currentMode
- 如果切换Mode，需要退出当前Runloop，选择一个Mode进入
- runloop没有source0、source1，timer，observer事件时会退出

[Runloop源码](https://opensource.apple.com/tarballs/CF/)   
[iOS底层原理总结 - RunLoop](https://www.jianshu.com/p/de752066d0ad)

