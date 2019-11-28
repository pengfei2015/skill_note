# 优化

## 虚拟内存
- 开发者开发过程中接触到的内存为虚拟内存，虚拟内存让APP认为其拥有连续的内存空间。实际上是多个物理内存碎片。
- 虚拟内存映射到物理内存上。
- 因为ASLR存在，可执行文件和动态链接库每次启动的虚拟内存地址不一样。所以需要修复到正确地址，也就是rebase，binding
### 分页
- 为了方便管理，虚拟内存会对一段连续的内存进行分页，对应的物理内存叫做物理页
- 虚拟页和物理页的映射关系称为页表，每个条目叫做页表条目
- 操作系统为每个进程提供一个页表放在主存中，CPU 在使用虚拟地址时交给 MMU 去翻译地址，MMU 去查询在主存中的页表来翻译。

### 缺页处理
- 每个页表条目都包含着映射信息。未分配，未缓存，已缓存
- 当访问一块未缓存的地址的时候，会触发缺页中断。
- 将需要使用的内存加载到物理内存中

## 启动优化

### 启动过程
1. SpringBoard阶段
    1. 点击启动，系统桌面SpringBoard先创建LaunchScreen
        - 虽然LaunchScreen的加载不占用APP进程的启动的启动时间。但是由于占用CPU资源，也有影响
    2. fork()函数新建一个进程
        - 父进程与子进程共享代码段（TEXT）但数据空间（DATA）是分开的。父进程会把自己的数据空间copy到子进程（写时拷贝）
    3. 进程执行exec调用
2. premain
    1. Load Dylibs
        - 链接系统的动态库
            - 减少非系统库的依赖
            - 使用静态库而不是静态库
            - 合并非系统动态库为一个动态库
    2. Rebase 
        - 将镜像读入内存，以Page为单位加密验证。
        - 修复的是指向当前镜像内部的资源指针
    3. Bind
        - 查询符号表，指向的是镜像外部的资源指针。来指向跨镜像的资源。
            - rebase/bind的优化，主要是减少__DATA segment中的指针数量
            - 减少Objc类数量，减少selector数量，
            - 减少C++虚函数数量
            - 使用swift struct 
            - 二进制重排，将启动时使用的方法，提前写入到虚拟内存。避免缺页中断。
    4. Objc
        - 读取二进制文件的 DATA 段内容，找到与 objc 相关的信息
        - 注册 Objc 类，ObjC Runtime 需要维护一张映射类名与类的全局表。当加载一个 dylib 时，其定义的所有的类都需要被注册到这个全局表中；
        - 读取 protocol 以及 category 的信息，把category的定义插入方法列表 (category registration)
        - 确保 selector 的唯一性
    5. Initializers
        - 以上都是静态调整
        - Objc的+load()函数
        - C++的构造函数属性函数 形如attribute((constructor)) void DoSomeInitializationWork()
        - 非基本类型的C++静态全局变量的创建(通常是类或结构体)(non-trivial initializer) 比如一个全局静态结构体的构建，如果在构造函数中有繁重的工作，那么会拖慢启动速度
            - 使用 +initialize 来替代 +load
            - 不要使用 atribute((constructor)) 将方法显式标记为初始化器，而是让初始化方法调用时才执行。
            - 尽量不要用到C++的静态对象。
3. main后面
1. 不使用xib，直接视用代码加载首页视图
2. NSUserDefaults实际上是在Library文件夹下会生产一个plist文件，如果文件太大的话一次能读取到内存中可能很耗时，这个影响需要评估，如果耗时很大的话需要拆分(需考虑老版本覆盖安装兼容问题)  
3. 每次用NSLog方式打印会隐式的创建一个Calendar，因此需要删减启动时各业务方打的log，或者仅仅针对内测版输出log  
4. 梳理应用启动时发送的所有网络请求，是否可以统一在异步线程请求
5. 减少启动初始化的流程，能懒加载的就懒加载，能放后台初始化的就放后台，能够延时初始化的就延时，不要卡主线程的启动时间，已经下线的业务直接删掉；

## 网络优化
### DNS优化  
[貌似很深奥的样子，然而我看不太明白1](https://mp.weixin.qq.com/s/ZObLVTOaPgJE4bSMBVSsyw)
### 连接优化
[貌似很深奥的样子，然而我看不太明白2](https://mp.weixin.qq.com/s/ZObLVTOaPgJE4bSMBVSsyw)
### 弱网优化
[貌似很深奥的样子，然而我看不太明白3](https://segmentfault.com/a/1190000020806491?utm_source=tag-newest)







[初步探索LaunchScreen](https://everettjf.github.io/2018/09/18/launch-screen-async-with-process-creation)  

[iOS启动时间优化](https://mp.weixin.qq.com/s/zeWfmAi0YnoQowcPpFhHUA)


[iOS Memory Deep Dive](https://mp.weixin.qq.com/s/WQ7rrTJm-cn3Cb6e_zZ4cA)
