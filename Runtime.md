# Runtime

### Runtime是什么？
- 一般的程序都是：编写代码-> 编译链接 -> 运行。方法调用在编译链接的时候已经确定了。
- Objective-C是一门动态语言，把一些编译链接的工作推迟到了运行时。所以需要一个运行时系统去执行编译后的代码。这就是Runtime库。
- Runtime是用C和汇编写的，Runtime的特性是消息传递机制。也就是在运行的过程中可以添加方法，更改方法的调用等等。

### 底层的数据结构
#### isa指针
在arm64结构前，实例isa指针就指向了类对象的地址。类对象的isa指针就指向了元类对象地址。  
在arm64结构后，对isa指针进行了优化，通过位域存储更多的信息。通过&isa_mask，取出特定的位存放的指针。类和元类对象地址的最后三位肯定是0
- isa结构
```
union isa_t  // 联合体，两个变量共用一块内存。
{
    Class cls;
    uintptr_t bits;
    # if __arm64__ // arm64架构
#   define ISA_MASK        0x0000000ffffffff8ULL //用来取出33位内存地址使用（&）操作
#   define ISA_MAGIC_MASK  0x000003f000000001ULL
#   define ISA_MAGIC_VALUE 0x000001a000000001ULL
    struct {
        uintptr_t nonpointer        : 1; //0：代表普通指针，1：表示优化过的，可以存储更多信息。
        uintptr_t has_assoc         : 1; //是否设置过关联对象。如果没设置过，释放会更快
        uintptr_t has_cxx_dtor      : 1; //是否有C++的析构函数
        uintptr_t shiftcls          : 33; // MACH_VM_MAX_ADDRESS 0x1000000000 内存地址值
        uintptr_t magic             : 6; //用于在调试时分辨对象是否未完成初始化
        uintptr_t weakly_referenced : 1; //是否有被弱引用指向过
        uintptr_t deallocating      : 1; //是否正在释放
        uintptr_t has_sidetable_rc  : 1; //引用计数器是否过大无法存储在ISA中。如果为1，那么引用计数会存储在一个叫做SideTable的类的属性中
        uintptr_t extra_rc          : 19; //里面存储的值是引用计数器减1

#       define RC_ONE   (1ULL<<45)
#       define RC_HALF  (1ULL<<18)
    };
```

### class对象的内部结构  和 元类对象的内部结构是相同的
```
typedef struct objc_class *Class;  // 类
typedef struct objc_object *id;  // id 
typedef struct objc_selector *SEL;  // id 


struct objc_class: objc_object {
    // Class ISA; 继承自objc_object的
    Class superclass;
    cache_t cache;
    class_data_bits_t bits: // class的数据，包括class_rw_t, 由bits&fask_data_mask得到
    class_rw_t *data() {
    return bits&fast_data_mask
    }
}

struct cache_t {
    struct bucket_t buckets; // 散列表结构，存放缓存的方法
    mask_t _mask;  // 总共多少槽位-1
    mask_t _occupied;  // 已经用了多少槽位
}

struct bucket_t {
    uintptr_t imp; // 方法的imp指针， unsigned long 
    SEL _sel; // 方法的名字d
}

struct class_rw_t {
    uint32_t flags;
    uint32_t version;
    const class_ro_t *ro; // 不能修改的原始类信息
    method_array_t methods; // 方法列表，二维数组里面是method_t
    property_array_t properties; // 属性列表
    protocol_array_t protocols; // 协议列表
    Class firstSubclass; 
    Class nextSiblingClass;
    char *demangledName;
}

struct class_ro_t {
    uint32_t flags;
    uint32_t instanceStart;
    uint32_t instanceSize;
#ifdef __LP64__
    uint32_t reserved;
#endif

    const uint8_t * ivarLayout;
    
    const char * name; // 名称
    method_list_t * baseMethodList; // 原始类方法列表
    protocol_list_t * baseProtocols; // 原始类协议列表
    const ivar_list_t * ivars; // 原始类变量列表

    const uint8_t * weakIvarLayout; // 
    property_list_t *baseProperties; // 原始类属性列表
}

struct method_t {
    SEL name; // 方法选择器，字符串的名字，不同类中相同名字的方法的方法选择器是相同的
    const char *types; // 方法类型，包括返回值类型，参数类型与个数
    IMP imp; // 方法实现的指针，根据它调用方法
}

```

### super   
当使用super调用方法的时候，汇编代码会调用objc_msgSendSuper(objc_super2,  SEL)。
C++源码会转换成objc_msgSendSuper2(objc_super, SEL)
本质是一样的，都是标识在父类的方法中查找。方法的调用者还是self
```
struct objc_super2 {
    id receiver;
    Class current_class; // 传递的是receiver的类，根据当前类的superclass找到父类，从父类的方法中查找
};

struct objc_super {
    __unsafe_unretained _Nonnull id receiver;
    __unsafe_unretained _Nonnull Class super_class; // 传的是父类，直接在方法中查找
};
```

### cache散列表
- 根据方法的selector&mask获得散列表的index
- 如果index已经被使用，将index-1
- 调用的时候，先根据index取出方法，比较key是不是selector，不是的话讲index-1再取
- 散列表扩容（旧的长度乘以2）的时候清空缓存

### 分类Category
- 结构对象
```
struct category_t {
    const char *name; // 分类的名字
    classref_t cls; // 类
    struct method_list_t *instanceMethods; // 实例的方法列表
    struct method_list_t *classMethods; // 类方法列表
    struct protocol_list_t *protocols; // 协议方法列表
    struct property_list_t *instanceProperties; // 实例属性列表
    struct property_list_t *_classProperties; // 类属性列表
}
```
- 实现原理  
    - 在运行的时候，将分类中的方法列表，属性列表等合并到类，元类的list中
    - 插入的时候是头插法。便利寻找方法的时候是顺序便利。所以，如果和类原始方法相同的时候，会覆盖掉原始方法。
        - 如何解决不让原始方法被覆盖。做到先调用原始方法，再调用添加方法。
        - 在添加方法的时候，先找到该方法的IMP，将IMP转换成函数调用
        ```
        // 替换ViewController的ViewDidLoad方法
        #import "ViewController+add.h"
        #import <objc/runtime.h>

        @implementation ViewController (add)

        - (void)viewDidLoad {
            unsigned int count;
            Method *methodlist = class_copyMethodList([ViewController class], &count);
            IMP lastMethodIMP = NULL;
            SEL lastSEL = NULL;
            for (NSInteger i = 0; i < count; i ++) {
                Method method = methodlist[i];
                NSString *methodname = [NSString stringWithCString:sel_getName(method_getName(method)) encoding: NSUTF8StringEncoding];
                if ([methodname isEqualToString:@"viewDidLoad"]) {
                    lastMethodIMP = method_getImplementation(method);
                    lastSEL = method_getName(method);
                }
            }
            typedef void(* function) (id, SEL);
            if (lastMethodIMP != NULL) {
                function f = (function)lastMethodIMP;
                f(self, lastSEL);
            }
            NSLog(@"category viewDidLoad");
        }
        @end
      ```
        - `+ (void)load`
            - load方法会在runtime加载完类，分类信息后调用，可以使用分类的方法
            - 每个类，分类的+load，在程序运行的过程中只调用一次
            - 分类中的+load方法，不会覆盖原类方法，因为程序启动的时候会直接找到所有load方法的地址调用
            - 先调用类的
                - 按照编译先后顺序调用（先编译，先调用）
                - 调用子类的+load方法之前，会调用父类的+load方法
            - 再调用分类的方法
                - 按照编译先后顺序调用（先编译，先调用）
        - `+ initilize`
            - 会在类第一次接收到消息的时候调用。
            - 先调用父类的，再调用子类的。
            - 走消息转发机制
- category和extension的区别
    - extension在编译的时候决议，在编译的时候和原始类一起组成完整的类。
    - category在运行期决议的。所以extension能添加实例变量，category不能。

### 关联对象
- 关联对象存储在全局统一的一个AssociationHashMap中，关联对象objc为key，value也是一个hashmap。key为设置的key，value包括值和策略
- 如果设置关联对象的value为nil，相当于移除关联对象
- objc被销毁的时候，会移除所有的关联对象。

### 消息传递机制
Objective-C的方法调用，会转成objc_msgSend(id, SEL)函数的调用。
- 消息发送
    - 先判断消息接收者是否为nil,如果为nil，就return
    - 根据对象的isa&isa_mask找到类对象，从类对象中的方法缓存cache_t中查找方法的IMP指针(方法的地址)
        - 将SEL & cache_t.mask 得到buckets的索引index。根据index取bucket。
        - 如果取到的值不为nil，判断bucket.key是否和SEL相等。如果不相等，就将index-1，重新取值。当index==0的时候，index=mask。
    - 如果从cache中没有取到值，就从class_rw_t的methods查找（方法列表是个二元数组，排序后会根据二分法优化查找），找到对应的方法，会将方法添加到cache中。
    - 如果没有找到，就根据superclass找到父类，继续查找。找到对应的方法，会将方法添加到自己class的cache中。
    - 如果找到基类没有找到。走动态方法解析。（类方法找到最后会找到实例方法，因为元类的superclass是类对象）
- 动态方法解析
    - 先判断该方法是否曾经有动态解析
    - 如果没有，调用`+resolveInstanceMethod:`，（返回值没什么用）解析完后标记已经动态解析了。
    - 动态解析后会重新走消息发送。
- 消息转发
    - 动态方法解析失败，说明类本身没有能力去处理这个方法，所以需要交给其它类去处理。
    - 首先会调用`-forwardingTargetForSelector:`返回一个能够处理消息的对象。
    - 如果返回一个新对象，会使用新对象重新发送该消息。如果返回nil，就启动重定向。
    - 先调用`methodSignatureForSelector`,方法签名包括返回值，参数的类型。就是和method_t的types类似
    - 将返回的方法签名封装到Invocation中包括方法调用者，方法名称，方法参数，调用 `forwardInvocation`方法。如果返回nil，报错！
    - 在forwardInvocation方法中随便处理这次消息，包括修改参数，方法，调用者~
- 如果以上流程都找不到消息的处理者，就报错。


### Runtime使用
- 利用关联对象给分类添加属性
- 消息转发解决bug
- 实现NSCoding的自动归档和i解档
- 实现字典转模型
- 交换方法


[iOS Runtime详解](https://www.jianshu.com/p/6ebda3cd8052)
[源码下载地址](https://opensource.apple.com/tarballs/objc4/)
