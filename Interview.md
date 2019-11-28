# 常见的面试题

### iOS中属性修饰符都有哪些，以及引申问题
- weak
- assgin
- copy
- strong
- readwrite/readonly
- atomic/nonatomic
1. weak和assign的区别
    - weak和assign都不会造成引用计数+1
    - weak只能用来修饰OC对象，assign即可以修饰OC对象，也可以修饰基础数据类型
    - runtime维护了一个weak的hash表。Key是对象的地址，Value是存着所有变量地址的weak_entry_t。
    - 当对象的引用计数为0的时候会调用dealloc方法。dealloc方法先调用object_dispose释放对象
    - 再调用objc_destructInstance释放
    - 再处理weak对象。根据内存地址将所有变量置位nil
    - 再清楚对象的记录

2. copy修饰符的使用：表示的是对其赋值是copy一份。
- NSString、NSArray、NSDictionary是因为他们有mutable的子类
    - 当A传递NSMutableString给B的时候，B当成一个String来使用。如果用的不是copy那么，当A中值更改的时候，因为B的指针指向的内存没变，会被改变。
- block也使用copy
    - 在MRC时期，需要将栈区的block copy到堆上。在ARC的时候，编译器会自动copy。所以用strong和copy没区别
3. 深copy和浅copy 
- 对非集合类对象的copy
    - 对immutable对象copy是指针复制，mutableCopy是内容复制。
    - 对mutable对象都是内容复制
- 集合对象的copy和mutablecopy
    - 对immutable对象copy是指针copy，mutableCopy是内容复制。复制是集合对象，集合里面的指针还是原来的
    - 对mutable对象都是内容复制。复制是集合对象，集合里面的指针还是原来的

### 什么对象会放入AutoreleasePool
- weak修饰的
- autorelease修饰的
- ARC环境下，非alloc出来的 

### 对象的内存销毁时间表
1. 调用release：引用计数变为0
    - 对象正在被销毁，生命周期即将结束。
    - 不能再有新的__weak弱引用，否则指向为nil
    - 调用[self dealloc]
2. 子类调用-dealloc
    - 继承关系中最底层的子类，在调用-dealloc
    - 如果是MRC代码会手动释放实例变量
    - 继承关系中每一层的父类都在调用-dealloc
3. NSObject 调-dealloc
    - 调用 Objective-C runtime 中的 object_dispose() 方法
4. 调用 object_dispose()
    - 为C++实例变量调用destructors
    - 为ARC状态下的实例变量调用-release
    - 解除所有使用 runtime Associate方法关联的对象
    - 解除所有 __weak 引用
    - 调用 free()

### 扩展阅读

[weak的生命周期：具体实现方法](http://www.cocoachina.com/articles/11990)  


