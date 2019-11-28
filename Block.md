# Block

### Block是什么
- Block是一个结构体，也可以说是OC对象(包含ISA指针)
    - Block捕获的变量会存在结构体中（构造函数的值传递）
    - Block内部的代码会被封装成一个函数，函数的IMP指针，存在struck中的IMP对象里
    - 调用block 的时候直接调用了 
    
### Block捕获变量
    - 捕获自动局部变量，值传递
    - 捕获静态局部变量，指针传递 
    - 捕获全局变量，直接访问
    - 因为局部自动变量可能消失，如果传递的是地址，那么Block访问的时候，会访问出错。所以y需要传地址
    - 因为Block内部封装的函数和局部变量不在一个区域，所以需要捕获局部变量，不需要捕获全局变量
    
### Block类型
- __NSGlobalBlock__：没有访问auto变量。 copy后什么也不做
- __NSStackBlock__ : 访问了auto变量。copy后从栈复制到堆
- __NSMallocBlock__：__NSStackBlock__调用了copy。引用计数+1
- Block作为函数返回和复制给__strong指针，usingBlock时自动copy


- 放Block内部访问了对象类型是auto变量时
    - 只要block是在栈上，就不会对外界auto变量强引用。
    - 如果block被copy到堆上。
        - 会调用block内部的copy函数
        - copy函数内部会调用block_object_assign函数
        - block_object_assign根据auto变量修饰符决定是否强引用
    - block被从堆上移除
        - 会调用内部的dispose
        - dispose函数内部会调用block_object_dispose函数
        - block_object_dispose根会自动释放引用的auto变量
        
### __block修饰符
- __block只能修饰auto变量
- __block将变量包装成一个struct
- 将结构体的指针传进block的struct
- struct有一个指向自己的forwarding指针
- 修改的时候，通过forwardding指针找到变量
- 我们访问的变量，就是struct最里面的对象
- 当__block变量从栈上copy到堆上时，栈上block的forwarding指针指向堆上的block，堆上block的forwarding指针指向自己。这样无论f使用哪个都能修改值
