---
title: "iOS开发Runtime"
date: 2023-03-11T00:08:26+08:00
draft: false
---

[TOC]

# 一、Runtime

## 1.什么是Runtime
iOS开发过程，将源代码转换为可执行的程序，通常需要经过三个步骤：`编译`,`链接`,`运行`。不同的编译语言，在这三个步骤中所进行的操作有不同。
Ojbective-C是一门动态语言，在编译阶段并不知道变量的具体数据类型，也不知道所真正调用的哪个函数。只有在运行时间才检查变量的数据类型，同时在运行时才能根据函数名查找要调用的具体函数。

## 2.消息机制的基本原理
OC中，对象的方法调用都是类似`[receiver selector]`,其本质就是让对象在运行时发送消息的过程。
### 2.1 编译阶段
`[receiver selector]`方法被编译器转换为：
- objc_msgSend(receiver, selector)  -- 不带参数
- objc_msgSend(receiver, selector, vlg1, vg2) -- 带参数
### 2.2运行阶段
1. 通过`receiver`的`isa`指针找到`receiver`的`Class`
2. 在`Class`的`cache`（方法缓存）的散列表中寻找对应的`IMP`（方法实现）
3. 如果在`cache`（方法缓存）中没有找到对应的`IMP`（方法实现）的话，就继续在`Class`的`objc_method_list`中找到对应的`selector`，如果找到，填充到`cache`（方法缓存）中，并返回`selector`
4. 如果在`Class中`没有找到这个`selector`，就继续在他的`superClass`中寻找
5. 一旦找到对应的`selector`，直接执行`receiver`对应`selector`方法实现的`IMP`（方法实现）
6. 若找不到对应的`selector`，消息被转发或者临时向`receiver`添加这个`selector`对应的实现方法，否则就会发生崩溃。

## 3.Runtime中的概念解析

### 3.1 objc_msgSend
所有的Objective-C方法调用在编译时都会转化为对C函数`objc_msgSend`的调用，`objc_msgSend(receiver，selector)`是` [receiver selector]`对应的 C 函数。

### 3.2 Class（类）typedef struct objc_class *Class;

在obc/rumtime.h中，Class被定义为指向objc_class
```
struct objc_class {
    Class _Nonnull isa  OBJC_ISA_AVAILABILITY;                   // 保存所属类的结构体的实例的指针
#if !__OBJC2__
    Class _Nullable super_class                                  // 指向父类的指针
       const char * _Nonnull name                                   // 类的名称
    long version                                                 // 类的版本号，默认为0
    long info                                                    // 类的信息，供运行时期使用的一些标识
    long instance_size                                           // 该类的实例变量大小
    struct objc_ivar_list * _Nullable ivars                      // 该类的实力变量列表
    struct objc_method_list * _Nullable * _Nullable methodLists  // 方法定义的列表
    struct objc_cache * _Nonnull cache                           // 方法缓存
    struct objc_protocol_list * _Nullable protocols              // 遵守的协议列表
#endif

} OBJC2_UNAVAILABLE;
```

### 3.3 Object - objc/objc.h
```
struct objc_object {
    Class _Nonnull isa  OBJC_ISA_AVAILABILITY;
};

typedef struct objc_object *id;
```


```
uintptr_t nonpointer        : 1; //(0：普通指针，存储Class、Meta-Class对象 1：优化过使用位域存储更多信息)
uintptr_t has_assoc         : 1; //(是否有设置过关联对象,如果没有，释放会更快）
uintptr_t has_cxx_dtor      : 1; //(是否有C++析构函数，如果没有，释放更快）
uintptr_t shiftcls          : 33; //MACH_VM_MAX_ADDRESS 0x1000000000(存储Class、Meta-Classs对象的内存地址)
uintptr_t magic             : 6; // (用于在调试的时候分辨对象是否有初始化)
uintptr_t weakly_referenced : 1; // (是否有被弱引用指向，如果没有，释放会更快)
uintptr_t deallocating      : 1; // (对象是否正在释放)
uintptr_t has_sidetable_rc  : 1; // (引用计数器是否很大无法存储在isa中、如果为1，那么引用计数器会存储在一个叫SideTable的类的属性中)
uintptr_t extra_rc          : 19 // (里面存储的值是引用计数器减一)
```


`id`定义为一个指向`objc_object`结构提的指针。可以看出`objc_object`只包含一个`Class`类型的`isa`指针。换句话说，一个`object`唯一保存的就是它所属`Class`的地址。当我们对一个对象进行方法调用时，比如[`receiver selector]`，它会通过`obj_object`结构体的`isa`指针去找对应的`objc_class`结构体，然后在`objc_class`结构体的`objc_method_list`（方法列表）中找到调用对应的方法，然后执行。

### 3.4 Meta class

`objc_object`的`isa`指针指向对应的`objc_class`（类对象）
`objc_class`的`isa`指针指向什么？
`objc_class`结构体的`isa`指针实际上指向的是类对象自身的`meta_class`。`Meta class`（元类）就是一个类对象所属的类，一个对象所属的类叫做`类对象`，而一个类对象所属的类叫做`元类`

> Runtime 中把类对象所属类型就叫做 `Meta Class`（元类），用于描述类对象本身所具有的特征，而在元类的`methodLists` 中，保存了类的方法链表，即所谓的`「类方法」`。并且类对象中的 isa 指针 指向的就是元类。每个类对象有且仅有一个与之相关的元类。

类方法的调用过程和对象方法调用差不多：
1.通过类对象`isa`指针找到所属的`meta_class`
2.在`meta_class`的`objc_method_list`（方法列表）中找到对应的`selector`
3.执行对应的`selector`

eg:
> NSString *string = [NSString stringWithFormat:@"%@ %s", @"1", 3];
> stringWithFormat被发送给了NSString类，NSString类通过isa指针找到NSString元类，然后该元类的方法列表中找到对应的stringWithFormat方法，然后执行该方法

### 3.5 实例对象，类，元类对象的关系
⬆ 父类   -----> isa指针

根类   object实例对象   ---->  object类对象 ---->     NSObject元类
                                ⬆                        ⬆
父类   Person实例对象   ----> Person类对象  ---->    Person元类
                                ⬆                        ⬆
子类   Man实例对象        ---->  man类对象    ---->      Man元类

#### isa指针:
- 1.水平方向:每一级中的实力对象的isa指针指向对应的类对象,而类对象的isa指针指向了对应的元类.而所有的元类isa指针最终指向了NSObject元类,因此NSObject也被成为根元类
- 2.垂直方向:元类的isa指针和父类元类的isa指针都指向了根元类.而根元类的isa指针又指向了自己.

#### 父类指针:
- 1.类对象的父类指针指向了父类的类对象,父类的类对象又指向了根类的类对象,根类的类对象最终指向了nil
- 2.元类的父类指针指向父类对象的元类.父类对象的元类的父类指针指向了根类对象的元类,也就是根元类.根元类的父类指针指向了根类对象,最终指向了nil

### 3.6 Method(方法)
`objc_class`结构体的`objc_method_list`(方法列表)中存放的元素就是`objc_method`

objc/runtime.h中`objc_method`的`objc_method`的数据结构

```
typedef struct objc_method *Method;

struct objc_method {
    SEL _Nonnull method_name       // 方法名
    char * _Nullable method_types  // 方法类型
    IMP _Nonnull method_imp        // 方法实现
}
```

可以看到`objc_method`结构体中包含了`method_name`(方法名), `method_types`(方法类型),`method_imp`(方法实现)

#### 3.6.1.SEL method_name 
```
typedef struct objc_selector *SEL;
```
SEL是一个指向objc_selector结构体的指针,但是在runtime相关头文件理没有找到明确的定义.经过测试可以得出:SEL只是一个保存方法名的字符串.

#### 3.6.2.IMP method_imp
```
#if !OBJC_OLD_DISPATCH_PROTOTYPES
    typedef void (*IMP)(void /* id, SEL, ... */ ); 
#else
    typedef id _Nullable (*IMP)(id _Nonnull, SEL _Nonnull, ...); 
#endif
```
IMP的实质是一个函数指针,所指向的就是方法的实现.IMP用来找函数地址,然后执行函数

#### 3.6.3.char * _Nullable method_types 
方法类型method_types是个字符串,用来存储方法的参数类型和返回值
Method将SEL和IMP关联起来,当对一个对象发送消息时,会通过给出的SEL去找到IMP,然后执行

#### 3.6.4方法缓存


```
struct cache_t {
    explicit_atomic<struct bucket_t *> _buckets; // 散列表
    explicit_atomic<mask_t> _mask; // 散列表长度-1
    mask_t _occupied;// 已经缓存的方法数量
}
```


```
struct bucket_t { 
    explicit_atomic<uintptr_t> _imp;  //函数的内存地址
    explicit_atomic<SEL> _sel; // sel作为key
}
```

#### 3.6.5 class_rw_t结构体

```
struct class_rw_t {
    // Be warned that Symbolication knows the layout of this structure.
    uint32_t flags;
    uint16_t witness;
#if SUPPORT_INDEXED_ISA
    uint16_t index;
#endif

    explicit_atomic<uintptr_t> ro_or_rw_ext;

    Class firstSubclass;
    Class nextSiblingClass;

private:
    using ro_or_rw_ext_t = objc::PointerUnion<const class_ro_t *, class_rw_ext_t *>;

    const ro_or_rw_ext_t get_ro_or_rwe() const {
        return ro_or_rw_ext_t{ro_or_rw_ext};
    }

    void set_ro_or_rwe(const class_ro_t *ro) {
        ro_or_rw_ext_t{ro}.storeAt(ro_or_rw_ext, memory_order_relaxed);
    }

    void set_ro_or_rwe(class_rw_ext_t *rwe, const class_ro_t *ro) {
        // the release barrier is so that the class_rw_ext_t::ro initialization
        // is visible to lockless readers
        rwe->ro = ro;
        ro_or_rw_ext_t{rwe}.storeAt(ro_or_rw_ext, memory_order_release);
    }

    class_rw_ext_t *extAlloc(const class_ro_t *ro, bool deep = false);

public:
    void setFlags(uint32_t set) {
        __c11_atomic_fetch_or((_Atomic(uint32_t) *)&flags, set, __ATOMIC_RELAXED);
    }

    void clearFlags(uint32_t clear)  {
        __c11_atomic_fetch_and((_Atomic(uint32_t) *)&flags, ~clear, __ATOMIC_RELAXED);
    }

    // set and clear must not overlap
    void changeFlags(uint32_t set, uint32_t clear)  {
        ASSERT((set & clear) == 0);

        uint32_t oldf, newf;
        do {
            oldf = flags;
            newf = (oldf | set) & ~clear;
        } while (!OSAtomicCompareAndSwap32Barrier(oldf, newf, (volatile int32_t *)&flags));
    }

    class_rw_ext_t *ext() const {
        return get_ro_or_rwe().dyn_cast<class_rw_ext_t *>();
    }

    class_rw_ext_t *extAllocIfNeeded() {
        auto v = get_ro_or_rwe();
        if (fastpath(v.is<class_rw_ext_t *>())) {
            return v.get<class_rw_ext_t *>();
        } else {
            return extAlloc(v.get<const class_ro_t *>());
        }
    }

    class_rw_ext_t *deepCopy(const class_ro_t *ro) {
        return extAlloc(ro, true);
    }

    const class_ro_t *ro() const {
        auto v = get_ro_or_rwe();
        if (slowpath(v.is<class_rw_ext_t *>())) {
            return v.get<class_rw_ext_t *>()->ro;
        }
        return v.get<const class_ro_t *>();
    }

    void set_ro(const class_ro_t *ro) {
        auto v = get_ro_or_rwe();
        if (v.is<class_rw_ext_t *>()) {
            v.get<class_rw_ext_t *>()->ro = ro;
        } else {
            set_ro_or_rwe(ro);
        }
    }

    // 方法列表
    const method_array_t methods() const {
        auto v = get_ro_or_rwe();
        if (v.is<class_rw_ext_t *>()) { // 优先从class_rw_t查找方法
            return v.get<class_rw_ext_t *>()->methods;
        } else { // 再从class初始化的时候class_ro_t查找方法
            return method_array_t{v.get<const class_ro_t *>()->baseMethods()};
        }
    }

    // 属性列表
    const property_array_t properties() const {
        auto v = get_ro_or_rwe();
        if (v.is<class_rw_ext_t *>()) {
            return v.get<class_rw_ext_t *>()->properties;
        } else {
            return property_array_t{v.get<const class_ro_t *>()->baseProperties};
        }
    }

    // 协议列表
    const protocol_array_t protocols() const {
        auto v = get_ro_or_rwe();
        if (v.is<class_rw_ext_t *>()) {
            return v.get<class_rw_ext_t *>()->protocols;
        } else {
            return protocol_array_t{v.get<const class_ro_t *>()->baseProtocols};
        }
    }
}
```

#### 3.6.6 class_ro_t


```
struct class_ro_t {
    uint32_t flags;
    uint32_t instanceStart;
    uint32_t instanceSize;
#ifdef __LP64__
    uint32_t reserved;
#endif

    const uint8_t * ivarLayout;
    
    const char * name;
    method_list_t * baseMethodList;
    protocol_list_t * baseProtocols;
    const ivar_list_t * ivars;

    const uint8_t * weakIvarLayout;
    property_list_t *baseProperties;

    // This field exists only when RO_HAS_SWIFT_INITIALIZER is set.
    _objc_swiftMetadataInitializer __ptrauth_objc_method_list_imp _swiftMetadataInitializer_NEVER_USE[0];

    _objc_swiftMetadataInitializer swiftMetadataInitializer() const {
        if (flags & RO_HAS_SWIFT_INITIALIZER) {
            return _swiftMetadataInitializer_NEVER_USE[0];
        } else {
            return nil;
        }
    }

    method_list_t *baseMethods() const {
        return baseMethodList;
    }

    class_ro_t *duplicate() const {
        if (flags & RO_HAS_SWIFT_INITIALIZER) {
            size_t size = sizeof(*this) + sizeof(_swiftMetadataInitializer_NEVER_USE[0]);
            class_ro_t *ro = (class_ro_t *)memdup(this, size);
            ro->_swiftMetadataInitializer_NEVER_USE[0] = this->_swiftMetadataInitializer_NEVER_USE[0];
            return ro;
        } else {
            size_t size = sizeof(*this);
            class_ro_t *ro = (class_ro_t *)memdup(this, size);
            return ro;
        }
    }
}
```


## 4.RunTime消息流程msg_Send
1.消息发送
2.动态方法解析
3.消息转发

### 4.0
**方法**：与一个类相关的一段实际代码，并给出一个特定的名字。例：- (int)meaning { return 42; }
**消息**：发送给对象的名称和一组参数。示例：向0x12345678对象发送meaning并且没有参数。
**选择器**：表示消息或方法名称的一种特殊方式，表示为类型SEL。选择器本质上就是不透明的字符串，它们被管理，因此可以使用简单的指针相等来比较它们，从而提高速度。（实现可能会有所不同，但这基本上是他们在外部看起来的样子。）例如：@selector(meaning)。
**消息发**送：接收信息并查找和执行适当方法的过程。

## 4.1 动态特性：方法解析和消息转发
若找不到对应的selector,消息被转发或者临时向receiver添加这个selector对应的实现方法,否则就会发生崩溃.
    当一个方法找不到的时候,Runtime提供了消息动态解析,消息接受者重定向,消息重定向等三步处理,具体流程如下:

> +resolveInstanceMethod/+resolveClassMethod   

> +forwardingTargetForSelector/-forwardingTargetForSelector  

> +methodSignatureForSelector/-methodSignatureForSelector

### 4.1.2 动态方法解析: Method Resolution

Objective-C运行时会调用`+resolveInstanceMethod`或者`+resolveClassMethod`,前者在对象方法未找到时,后者在类方法未找到时调用,可以通过重写这两个方法,添加其他函数实现,并返回YES,运行时系统会重新启动一次消息发送的过程.
```
+ (BOOL)resolveClassMethod:(SEL)sel;
+ (BOOL)resolveInstanceMethod:(SEL)sel;
BOOL class_addMethod(Class _Nullable cls, SEL _Nonnull name, IMP _Nonnull imp, const char * _Nullable types)
```

```
+ (BOOL)resolveInstanceMethod:(SEL)sel {
    return YES; // 为了进行下一步 消息接受者重定向，返回YES，代表有动态添加方法
}

```

## 4.2 快速转发: Fast Rorwarding
如果上一步中`+resolveInstanceMethod`或者`+resolveClassMethod`没有添加其他函数实现,运行时就会进行下一步:`消息接受者重定向`

如果当前对象实现了`forwardingTargetForSelector`或者`forwardingTargetForSelector`方法,`Runtime`就会调用这个方法,允许我们将消息的接受者转发给其他对象.
```
- (id)forwardingTargetForSelector:(SEL)aSelector;
- (void)forwardInvocation:(NSInvocation *)anInvocation;
```
- 1.类方法和对象方法消息转发的第二步调用的方法不一样,前者是`+forwardingTargetForSelector`方法,后者是`-forwardingTargetForSelector`
- 2.这里`resolveInstanceMethod`或者`resolveClassMethod`无论是返回`YES`还是返回`NO`,只要其中没返回其他函数实现,运行时都会进行下一步.

### 4.2.1 消息重定向
如果经过的消息动态解析,消息接受者重定向,Runtime系统还是找不到相应的方法实现而无法响应消息,Runtime系统会利用`-methodSignatureForSelector:`或者`+methodSignatureForSelector`方法获取函数的参数和返回值类型.
- 如果`methodSignatureForSelector`返回一个`NSMethodSignature`对象(函数签名),Runtime系统就会创建一个`NSInvocation`对象,并通过`forwardInvocation:`消息通知当前对象,给予此次消息发送最后一次寻找IMP机会
- 如果`methodSignatureForSelector:`返回`nil`,则Runtime系统会发出`doesNotRecognizeSelector:`消息,程序也就崩溃.

所以我们可以在`forwardInvocation:`方法中对消息进行转发
#### 类方法
```
+methodSignatureForSelector
+forwardInvocation:
+doesNotRecognizeSelector:
```
#### 对象方法
```
-methodSignatureForSelector:
-forwardInvocation:
-doesNotRecognizeSelector:
```
`forwardingTargetForSelector:`和`forwarInvocation:`都可以将消息转发给其他对象处理,`forwardingTargetForSelector:`只能将消息转发给一个对象,而`forwardInvocation:`可以将消息发给对个对象  


```
- (id)forwardingTargetForSelector:(SEL)aSelector {
    return nil; // 为了进行下一步 消息重定向
}

// 获取函数的参数和返回值类型，返回签名
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
    if ([NSStringFromSelector(aSelector) isEqualToString:@"fun"]) {
        return [NSMethodSignature signatureWithObjCTypes:"v@:"];
    }
    return [super methodSignatureForSelector:aSelector];
}

- (void)forwardInvocation:(NSInvocation *)anInvocation {
    SEL seletor = anInvocation.selector; // 从 anInvocation 中获取消息
    
    Person *p = [[Person alloc] init];
    if ([p respondsToSelector:seletor]) {  // 判断 Person 对象方法是否可以响应 sel
        [anInvocation invokeWithTarget:p];  // 若可以响应，则将消息转发给其他对象处理
    } else {
        [self doesNotRecognizeSelector:seletor]; // 若仍然无法响应，则报错：找不到响应方法
    }
}
```


# 二、Runtime  Method Swizzling(动态方法交换)

## 1.Method Swizzling简介
`Method Swizzling`用于改变一个已经存在的`selector`实现.我们在程序运行时,通过改变`selector`所在Class(类)的`objc_method_list`(方法列表)的映射从而改变方法的调用.其实质就是交换两个方法的IMP(方法实现).

Method(方法)就是objc_method结构体
```
// objc_method 结构体
typedef struct objc_method *Method;

struct objc_method {
    SEL _Nonnull method_name;                    // 方法名
    char * _Nullable method_types;               // 方法类型
    IMP _Nonnull method_imp;                     // 方法实现
};
```

在运行时,`Class`(类)维护了一个`methodLists`(方法列表)来确定消息的正确发送.`methodLists`(方法列表)存放的元素就是`method`(方法),而`method`中映射了一对键值对:`SEL`(方法名), `IMP`(方法实现).
`Method Swizzling`修改了`methodLists`(方法列表),使得不同的`method`中不同的键值对发生了交换.比如交换前两个键值对分别为`SEL A`, `IMP A` | `SEL B`, `IMP B`,交换之后就变成了`SEL A`, `IMP B` | `SEL B`, `IMP B`.

## 2.Method Swizzling 使用方法
假如当前类中有两个方法:-(void)originalFunction;和-(void)swizzlingFunction; 如果我们想要交换两个方法的实现,从而实现调用-(void)originalFunction;的时候实际上调用的是-(void)swizzlingFunction; ,而调用-(void)swizzlingFunction; 实际上是-(void)originalFunction;.

### 2.1 Method Swizzling 简单使用
    在当前类+ (void)load;方法中添加Method Swizzling

Method Swizzling 方法A -- 添加Method Swizzling 交换方法,用普通方式
1. Method Swizzling 方法B -- 使用函数指针的方式。
2. AFNetworking交换方法
3. JRSwizzle 和 RSSwizzle 

### 3.Method Swizzle 使用注意
Method Swizzle 之所以被大家称为黑魔法,就是因为使用Method Swizzling进行方法交换是一个危险动作.

### 1.应该只在+ (void)load中执行Method swizzling
程序在启动的时候,会先加载所有的类,这时会调用每个类的`+(void)load`方法,而且在整个程序运行周期只会调整一次.所以在`+(void)load`方法进行`Method Swizzling`再好不过

为什么不用`+ (void)initialize`方法?
因为`+ (void)initialize`方法的调用时机是在第一次向该类发送第一个消息的时候才会被调用.如果该类只是引用,没有调用,则不会执行`+ (void)initialize`.
MethodSwizzling影响的是全局状态,`+ (void)load`方法能保证在加载类的时候就进行交换,保证交换结果.而使用`+ (void)initialize`方法则不能保证这一点,有可能在使用的时候起不到交换方法的作用

### 2.Method Swizzling 在+ (void)load中执行,不要调用[super load]
程序在启动的时候,会加载所有的类,如果在`+ (void)load`方法中调换`[super load]`方法,,就会导致父类 的`Method Swizzling` 被重复两次,而方法交换也被执行了两次,相当于互换了一次方法之后,两次又叫换回去了

### 3.Method Swizzling应该总是在dispatch_once中执行
`Method Swizzling`不是原子操作,`dispatch_once`可以保证在不同的线程中也能确保代码只执行一次.

### 4.使用Method Swizzling后要记得调用原声方法的实现
在交换方法实现后记得要调用原生方法的实现（除非你非常确定可以不用调用原生方法的实现）：APIs 提供了输入输出的规则，而在输入输出中间的方法实现就是一个看不见的黑盒。交换了方法实现并且一些回调方法不会调用原生方法的实现这可能会造成底层实现的崩溃。

### 5.避免命名冲突和参数`_cmd`被篡改
避免命名冲突一个比较好的做法是为替换的方法加个前缀以区别原生方法。一定要确保调用了原生方法的所有地方不会因为自己交换了方法的实现而出现意料不到的结果。
在使用 Method Swizzling 交换方法后记得要在交换方法中调用原生方法的实现。在交换了方法后并且不调用原生方法的实现可能会造成底层实现的崩溃。

避免方法命名冲突另一个更好的做法是使用函数指针，也就是上边提到的 方案 B，这种方案能有效避免方法命名冲突和参数 _cmd 被篡改.

### 6.谨慎对待 Method Swizzling。
使用`Method Swizzling`，会改变非自己拥有的代码。我们使用 `Method Swizzling` 通常会更改一些系统框架的对象方法，或是类方法。我们改变的不只是一个对象实例，而是改变了项目中所有的该类的对象实例，以及所有子类的对象实例。所以，在使用 Method Swizzling 的时候，应该保持足够的谨慎。

例如，你在一个类中重写一个方法，并且不调用 `super`方法，则可能会出现问题。在大多数情况下，`super` 方法是期望被调用的（除非有特殊说明）。如果你是用同样的思想来进行`Method Swizzling` ，可能就会引起很多问题。如果你不调用原始的方法实现，那么你 Method Swizzling 改变的越多代码就越不安全。

### 7.对于 Method Swizzling 来说，调用顺序 很重要。
####  load 方法的调用规则为：
- 1.先调用主类，按照编译顺序，顺序地根据继承关系由父类向子类调用；
- 2.再调用分类，按照编译顺序，依次调用；
- 3.+ load 方法除非主动调用，否则只会调用一次。
这样的调用规则导致了 + load 方法调用顺序并不一定确定。一个顺序可能是：父类 -> 子类 -> 父类类别 -> 子类类别，也可能是 父类 -> 子类 -> 子类类别 -> 父类类别。所以 Method Swizzling 的顺序不能保证，那么就不能保证 Method Swizzling 后方法的调用顺序是正确的。

所以被用于 Method Swizzling 的方法必须是当前类自身的方法，如果把继承父类来的 IMP 复制到自身上面可能会存在问题。如果 + load 方法调用顺序为：父类 -> 子类 -> 父类类别 -> 子类类别，那么造成的影响就是调用子类的替换方法并不能正确调起父类分类的替换方法

## 4.Method Swizzling 应用场景
    4.1 全局页面统计功能
    
#     三、Runtime Category

## 1.Category(分类)简介
### 1.1 什么是Category
Category主要作用是为已经存在的类添加方法.Category可以做到在既不子类化,也不侵入一个类的源码情况下,为原有的类添加新的方法,从而实现拓展一个类或者分离一个类的目的.

虽然继承也能为已有的类添加新的方法,而且还能直接添加属性,但继承关系增加了不必要的代码复杂度,在运行时也无法与父类的原始方法进行区分.所以我们可以优先考虑使用自定义Category.
- 1.把类的不同实现方法分开到不同的文件里
- 2.声明私有方法.
- 3.模拟多继承
- 4.将framework私有方法公开化

### 1.2 Category(分类)和Extension(扩展)
`Extension`在编译阶段与该类同时编译,是`类`的一部分.而且`Extension`(扩展)中声明的方法只能在该类的`@impelementation`中实现.也就意味着无法对系统的类进行扩展.`Extension`不但可以声明方法,也可以声明成员变量.
`Category`的特性:可以在运行时阶段动态的为已有类添加新行为.是在运行时决定的,而成员变量的内存布局已经在编译阶段就确定好了,如果在运行时阶段添加成员变量的话,就会破坏原来类的内存布局,所以`Category`无法添加成员

## 2.Category的实质
### 2.1 Category 结构体简介

```
typedef struct category_t *Category;

struct category_t {
    const char *name;                                // 类名
    classref_t cls;                                  // 类，在运行时阶段通过 class_name（类名）对应到类对象
    struct method_list_t *instanceMethods;           // Category 中所有添加的对象方法列表
    struct method_list_t *classMethods;              // Category 中所有添加的类方法列表
    struct protocol_list_t *protocols;               // Category 中实现的所有协议列表
    struct property_list_t *instanceProperties;      // Category 中添加的所有属性
};
```

> ·在项目中添加 Person 类文件 Person.h 和 Person.m，Person 类继承自 NSObject 。
> ·在项目中添加 Person 类的 Category 文件 Person+Addition.h 和 Person+Addition.m，并在 Category 中添加的相关对象方法，类方法，属性，以及代理。
> ·打开『终端』，执行 cd XXX/XXX 命令，其中 XXX/XXX 为 Category 文件 所在的目录。
> ·继续在终端执行 clang -rewrite-objc Person+Addition.m
> ·执行完命令之后，Person+Addition.m 所在目录下就会生成一个 Person+Addition.cpp 文件，这就是我们需要的Category（分类） 相关的 C++ 源码。

#### 2.2.1

Person 类的 Category 结构体
```
struct _category_t {
    const char *name;
    struct _class_t *cls;
    const struct _method_list_t *instance_methods;
    const struct _method_list_t *class_methods;
    const struct _protocol_list_t *protocols;
    const struct _prop_list_t *properties;
};
```

Person 类的 Category 结构体赋值
```
static struct _category_t _OBJC_$_CATEGORY_Person_$_Addition __attribute__ ((used, section ("__DATA,__objc_const"))) = 
{
    "Person",
    0, // &OBJC_CLASS_$_Person,
    (const struct _method_list_t *)&_OBJC_$_CATEGORY_INSTANCE_METHODS_Person_$_Addition,
    (const struct _method_list_t *)&_OBJC_$_CATEGORY_CLASS_METHODS_Person_$_Addition,
    (const struct _protocol_list_t *)&_OBJC_CATEGORY_PROTOCOLS_$_Person_$_Addition,
    (const struct _prop_list_t *)&_OBJC_$_PROP_LIST_Person_$_Addition,
};
```

Category 数组，如果 Person 有多个分类，则 Category 数组中对应多个 Category 
```
static struct _category_t *L_OBJC_LABEL_CATEGORY_$ [1] __attribute__((used, section ("__DATA, __objc_catlist,regular,no_dead_strip")))= {
    &_OBJC_$_CATEGORY_Person_$_Addition, 
};
```

> 从『Category结构体』源码中我们可以看到:
> 1.Category结构体
> 2.Category结构体的赋值语句
> 3.Category结构体数组

#### 2.2.2 Category中对象方法列表结构体
`- (void)printName;` 对象方法的实现
```
static void _I_Person_Addition_printName(Person * self, SEL _cmd) {
    NSLog((NSString *)&__NSConstantStringImpl__var_folders_ct_0dyw1pvj6k16t5z8t0j0_ghw0000gn_T_Person_Addition_405207_mi_1);
}
```

`- (void)personProtocolMethod; `方法的实现
```
static void _I_Person_Addition_forwardingTargetForSelectorpersonProtocolMethod(Person * self, SEL _cmd) {
    NSLog((NSString *)&__NSConstantStringImpl__var_folders_ct_0dyw1pvj6k16t5z8t0j0_ghw0000gn_T_Person_Addition_f09f6a_mi_2);
}
```

`Person`分类中添加的『对象方法列表结构体』
```
static struct /*_method_list_t*/ {
    unsigned int entsize;  // sizeof(struct _objc_method)
    unsigned int method_count;
    struct _objc_method method_list[2];
} _OBJC_$_CATEGORY_INSTANCE_METHODS_Person_$_Addition __attribute__ ((used, section ("__DATA,__objc_const"))) = {
    sizeof(_objc_method),
    2,
    {{(struct objc_selector *)"printName", "v16@0:8", (void *)_I_Person_Addition_printName},
    {(struct objc_selector *)"personProtocolMethod", "v16@0:8", (void *)_I_Person_Addition_personProtocolMethod}}
};
```

> 从 『对象方法列表结构体』源码中我们可以看到:
> 1.- (void)printName; 对象方法的实现。
> 2.- (void)personProtocolMethod; 方法的实现。
> 3.对象方法列表结构体.
> 只要是Category中实现了对象方法.都会添加到对象方法_OBJC_$_CATEGORY_INSTANCE_METHODS_Person_$_Addition 中来.如果只是在Persona.h中定义,而没有实现,则不会添加

#### 2.2.3 Category中『类方法列表结构体』
`+ (void)printClassName; `类方法的实现
```
static void _C_Person_Addition_printClassName(Class self, SEL _cmd) {
    NSLog((NSString *)&__NSConstantStringImpl__var_folders_ct_0dyw1pvj6k16t5z8t0j0_ghw0000gn_T_Person_Addition_c2e684_mi_0);
}
```

`+ (void)personProtocolClassMethod;` 方法的实现

```
static void _C_Person_Addition_personProtocolClassMethod(Class self, SEL _cmd) {
    NSLog((NSString *)&__NSConstantStringImpl__var_folders_ct_0dyw1pvj6k16t5z8t0j0_ghw0000gn_T_Person_Addition_c2e684_mi_3);
}
```

Person 分类中添加的『类方法列表结构体』
```
static struct /*_method_list_t*/ {
    unsigned int entsize;  // sizeof(struct _objc_method)
    unsigned int method_count;
    struct _objc_method method_list[2];
} _OBJC_$_CATEGORY_CLASS_METHODS_Person_$_Addition __attribute__ ((used, section ("__DATA,__objc_const"))) = {
    sizeof(_objc_method),
    2,
    {{(struct objc_selector *)"printClassName", "v16@0:8", (void *)_C_Person_Addition_printClassName},
    {(struct objc_selector *)"personProtocolClassMethod", "v16@0:8", (void *)_C_Person_Addition_personProtocolClassMethod}}
};
```

> ·从『类方法列表结构体』源码中我们可以看到：
> ·+ (void)printClassName; 类方法的实现。
> ·+ (void)personProtocolClassMethod; 类方法的实现。
> ·类方法列表结构体。
只要是Category中实现的类方法，都会添加到 _OBJC_$_CATEGORY_CLASS_METHODS_Person_$_Addition中来。如果只是在Person.h中定义为没有实现则不会添加。

#### 2.2.4 Category中『协议列表结构体』

Person 分类中添加的『协议列表结构体』
```
static struct /*_protocol_list_t*/ {
long protocol_count;  // Note, this is 32/64 bit
struct _protocol_t *super_protocols[1];
} _OBJC_CATEGORY_PROTOCOLS_$_Person_$_Addition __attribute__ ((used, section ("__DATA,__objc_const"))) = {
    1,
    &_OBJC_PROTOCOL_PersonProtocol
};
```

协议列表 对象方法列表结构体
```
static struct /*_method_list_t*/ {
    unsigned int entsize;  // sizeof(struct _objc_method)
    unsigned int method_count;
    struct _objc_method method_list[1];
} _OBJC_PROTOCOL_INSTANCE_METHODS_PersonProtocol __attribute__ ((used, section ("__DATA,__objc_const"))) = {
    sizeof(_objc_method),
    1,
    {{(struct objc_selector *)"personProtocolMethod", "v16@0:8", 0}}
};
```

协议列表 类方法列表结构体
```
static struct /*_method_list_t*/ {
    unsigned int entsize;  // sizeof(struct _objc_method)
    unsigned int method_count;
    struct _objc_method method_list[1];
} _OBJC_PROTOCOL_CLASS_METHODS_PersonProtocol __attribute__ ((used, section ("__DATA,__objc_const"))) = {
    sizeof(_objc_method),
    1,
    {{(struct objc_selector *)"personProtocolClassMethod", "v16@0:8", 0}}
};
```

PersonProtocol 结构体赋值
```
struct _protocol_t _OBJC_PROTOCOL_PersonProtocol __attribute__ ((used)) = {
    0,
    "PersonProtocol",
    (const struct _protocol_list_t *)&_OBJC_PROTOCOL_REFS_PersonProtocol,
    (const struct method_list_t *)&_OBJC_PROTOCOL_INSTANCE_METHODS_PersonProtocol,
    (const struct method_list_t *)&_OBJC_PROTOCOL_CLASS_METHODS_PersonProtocol,
    0,
    0,
    0,
    sizeof(_protocol_t),
    0,
    (const char **)&_OBJC_PROTOCOL_METHOD_TYPES_PersonProtocol
};
struct _protocol_t *_OBJC_LABEL_PROTOCOL_$_PersonProtocol = &_OBJC_PROTOCOL_PersonProtocol;
```

> 从『协议列表结构』源码中我们可以看到：
> 1.协议列表结构体
> 2.协议列表对象方法列表结构体
> 3.协议列表 类方法列表结构体
> 4.Personprotocol 协议结构体赋值语句

#### 2.2.5 Category中    『属性列表结构体』
Person 分类中添加的属性列表
```
static struct /*_prop_list_t*/ {
    unsigned int entsize;  // sizeof(struct _prop_t)
    unsigned int count_of_properties;
    struct _prop_t prop_list[1];
} _OBJC_$_PROP_LIST_Person_$_Addition __attribute__ ((used, section ("__DATA,__objc_const"))) = {
    sizeof(_prop_t),
    1,
    {{"personName","T@\"NSString\",C,N"}}
};
```
>  从『属性』列表结构体 中可以看到
>  只有Person分类中添加的属性列表架构题_OBJC_$_PROP_LIST_Person_$_Addition，没有成员变量结构体_ivar_list_t 结构体，没有set/get方法

## 3 Category 的加载过程

### 3.1 dyld加载大致流程
`Category`是在运行阶段动态加载的,而`Runtime`加载的过程,离不开一个叫dyld的动态链接器
在MacOS和iOS上,动态链接器dyld用来加载所有的库和可执行文件.而加载Runtime的过程,就在dyld加载的时候方法.
dyld的大致流程:
- 1.配置环境变量
- 2.加载共享缓存
- 3.初始化主app
- 4.插入动态缓存库
- 5.链接主程序
- 6.链接插入的动态库
- 7.初始化主程序,OC,C++全局变量初始化
- 8.返回主程序入口函数

#### Category在第7步执行

> `dyldbootstrap::start` ---> `dyld::_main` ---> `initializeMainExecutable` ---> `runInitializers` ---> `recursiveInitialization`---> `doInitialization` ---> `doModInitFunctions` ---> `_objc_init`
> 在`_objc_init`这一步中,`Runtime`向`dyly`绑定了回调,当`image`加载到内存后,`dyld`会通知`Runtime`进行处理,Runtime接手后调用`map_images`做解析和处理,调用`_read_images`方法把`Category`的对象方法,协议,属性添加到类上,把`Category`的类方法,协议添加到类的`metaclass`上,接下来`load_images`中调用`call_load_methods`方法,遍历所有加载进来的class,按继承层级和编译顺序一次调用class的load方法和Category的load方法
>     加载Category的调用栈如下:
>     `_objc_init` ---> `map_images` ---> `map_images_nolock` ---> `_read_images`（加载分类） ---> `load_images`。

### 3.2 Category 加载过程

#### 3.2.1 _read_images方法
// 获取镜像中的分类数组
```
category_t **catlist = 
_getObjc2CategoryList(hi, &count);
bool hasClassProperties = hi->info()->hasCategoryClassProperties();
```

```
// 遍历分类数组
for (i = 0; i < count; i++) {
    category_t *cat = catlist[i];
    Class cls = remapClass(cat->cls);
    // 处理这个分类
    // 首先，使用目标类注册当前分类
    // 然后，如果实现了这个类，重建类的方法列表
    bool classExists = NO;
    if (cat->instanceMethods ||  cat->protocols   ||  cat->instanceProperties) {
        addUnattachedCategoryForClass(cat, cls, hi);  
        if (cls->isRealized()) {
            remethodizeClass(cls);
            classExists = YES;
        }
    }   
}
```

```
if (cat->classMethods  ||  cat->protocols   ||  (hasClassProperties && cat->_classProperties))  {
    addUnattachedCategoryForClass(cat, cls->ISA(), hi);
    if (cls->ISA()->isRealized()) {
        remethodizeClass(cls->ISA());
    }
}
```

主要用到两个方法:
·addUnattachedCategoryForClass(cat, cls, hi); 为类添加未依附的分类
·remethodizeClass(cls); 重建类的方法列表
把Category的对象方法、协议、属性添加到类上。
把Category的类方法、协议、添加到类的metaclass上

#### 3.2.2 addUnattachedCategoryForClass(cat, cls, hi); 方法
```
static void addUnattachedCategoryForClass(category_t *cat, Class cls,  header_info *catHeader)
{
    runtimeLock.assertLocked();

    // 取得存储所有未依附分类的列表：cats
    NXMapTable *cats = unattachedCategories();
    category_list *list;
    // 从 cats 列表中找到 cls 对应的未依附分类的列表：list
    list = (category_list *)NXMapGet(cats, cls);
    if (!list) {
        list = (category_list *)
        calloc(sizeof(*list) + sizeof(list->list[0]), 1);
    } else {
        list = (category_list *)
        realloc(list, sizeof(*list) + sizeof(list->list[0]) * (list->count + 1));
    }
    // 将新增的分类 cat 添加 list 中
    list->list[list->count++] = (locstamped_category_t){cat, catHeader};
    // 将新生成的 list 添加重新插入 cats 中，会覆盖旧的 list
    NXMapInsert(cats, cls, list);
}
```
addUnattachedCategoryForClass(cat,cls,hi);的执行过程可以参考注释代码.执行完这个方法之后,系统将当前分类cat放到该类clas对应的未衣服分类的列表list中.把类和分类做了一个关联映射.

#### 3.2.3 remethodizeClas(cls)
```
static void remethodizeClass(Class cls) {
    category_list *cats;
    bool isMeta;

    runtimeLock.assertLocked();

    isMeta = cls->isMetaClass();

    // 取得 cls 类的未依附分类的列表：cats
    if ((cats = unattachedCategoriesForClass(cls, false/*not realizing*/))) {
        // 将未依附分类的列表 cats 附加到 cls 类上
        attachCategories(cls, cats, true /*flush caches*/);        
        free(cats);
    }
}
```
remethodizeClas(cls);方法主要就做一件事,调用addachCategories(cls,cats,true)方法将未依附分类的列表cats附加到cls类上

#### 3.2.4 attachCategories(cls, cats, true)
```
static void  attachCategories(Class cls, category_list *cats, bool flush_caches) {
    if (!cats) return;
    if (PrintReplacedMethods) printReplacements(cls, cats);

    bool isMeta = cls->isMetaClass();

    // 创建方法列表、属性列表、协议列表，用来存储分类的方法、属性、协议
    method_list_t **mlists = (method_list_t **)
        malloc(cats->count * sizeof(*mlists));
    property_list_t **proplists = (property_list_t **)
        malloc(cats->count * sizeof(*proplists));
    protocol_list_t **protolists = (protocol_list_t **)
        malloc(cats->count * sizeof(*protolists));

    // Count backwards through cats to get newest categories first
    int mcount = 0;           // 记录方法的数量
    int propcount = 0;        // 记录属性的数量
    int protocount = 0;       // 记录协议的数量
    int i = cats->count;      // 从分类数组最后开始遍历，保证先取的是最新的分类
    bool fromBundle = NO;     // 记录是否是从 bundle 中取的
    while (i--) { // 从后往前依次遍历
        auto& entry = cats->list[i];  // 取出当前分类
    
        // 取出分类中的方法列表。如果是元类，取得的是类方法列表；否则取得的是对象方法列表
        method_list_t *mlist = entry.cat->methodsForMeta(isMeta);
        if (mlist) {
            mlists[mcount++] = mlist;            // 将方法列表放入 mlists 方法列表数组中
            fromBundle |= entry.hi->isBundle();  // 分类的头部信息中存储了是否是 bundle，将其记住
        }

        // 取出分类中的属性列表，如果是元类，取得的是 nil
        property_list_t *proplist = 
            entry.cat->propertiesForMeta(isMeta, entry.hi);
        if (proplist) {
            proplists[propcount++] = proplist;
        }

        // 取出分类中遵循的协议列表
        protocol_list_t *protolist = entry.cat->protocols;
        if (protolist) {
            protolists[protocount++] = protolist;
        }
    }

    // 取出当前类 cls 的 class_rw_t 数据
    auto rw = cls->data();

    // 存储方法、属性、协议数组到 rw 中
    // 准备方法列表 mlists 中的方法
    prepareMethodLists(cls, mlists, mcount, NO, fromBundle);
    // 将新方法列表添加到 rw 中的方法列表中
    rw->methods.attachLists(mlists, mcount);
    // 释放方法列表 mlists
    free(mlists);
    // 清除 cls 的缓存列表
    if (flush_caches  &&  mcount > 0) flushCaches(cls);

    // 将新属性列表添加到 rw 中的属性列表中
    rw->properties.attachLists(proplists, propcount);
    // 释放属性列表
    free(proplists);

    // 将新协议列表添加到 rw 中的协议列表中
    rw->protocols.attachLists(protolists, protocount);
    // 释放协议列表
    free(protolists);
}
```
从 attachCategories(cls, cats, true); 方法的注释中可以看出这个方法就是存储分类的方法、属性、协议的核心代码。

1.Category的方法,属性,协议只是添加到原有类上,并没有将原有类的方法、属性、协议进行完全替换
    举个例子：假设原来类有MethodA方法，分类也拥有MethodA方法，那么加载完分类后，类的方法列表中会拥有两个MethodA方法。
2.Category的方法、属性、协议会被添加到原有类的方法列表、属性列表、协议列表的最前面，而原有类的方法，属性、协议则被移动到列表后面
    因为在运行时查找的方法的时候是顺着方法列表的顺序依次查找的，所以Category的方法会被搜索到，然后直接执行，而原有类的方法则不被执行，

## 4. Category（分类）和Class（类）的+load方法
Category中的方法、属性、协议附加到类上的操作，是在load方法执行之前进行的,也就是说,在+load方法之前,类中就已经加载了category的方法、属性、协议

Category（分类）和Class（类）的+load方法的调用顺序规则如下所示：
- 1.先调用主类、按照编译顺序、想混序的根据继承关系由父类向子类调用
- 2.调用完主类、再调用分类，按照编译顺序、依次调用
- 3.+load方法除非主动调用、否则只会调用一次

通过这样的调用规则，我们可以知道：主类的+load方法调用一定在分类+load方法调用之前。但是+load方法调用顺序并不是按照继承关系调用，而是依照编译顺序确定的，这也导致了+load方法的调用顺序不一样确定，可能是：父类 -> 子类 -> 父类类别 -> 子类类别，也可能是 父类 -> 子类 -> 子类类别 -> 父类类别。

## 5.Category与关联对象 
在Category中虽然可以添加属性,但是不会生产对应的成员变量,也不能生产getter,setter,因此在调用Category中声明的属性会报错.
可以使用关联对象(Objective-C Associated Objects)来实现getter, setter:

- 1.通过 key : value 的形式给对象 object 设置关联属性
- void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy);
- 2.通过 key 获取关联的属性 object
- id objc_getAssociatedObject(id object, const void *key);
- 3.移除对象所关联的属性
- void objc_removeAssociatedObjects(id object);
    
# 四、Runtime 获取类详细属性、方法

## 1.获取类详细属性、方法
    Runtime中为我们提供了一系列API来获取Class(类)的成员变量(iVar)、属性（Property）、方法（Method）、协议（Protocol ）等。我们可以通过这些方法来遍历一个类中的成员变量列表、属性列表、方法列表、协议列表。从而查找我们需要的变量和方法。

## 2.获取属性，成员变量，方法，协议

## 3.应用场景
### 3.1 修改私有方法 （UITextFiled 占位文字颜色和字号）
    1.通过获取类的属性和成员变量列表的方法打印UITextField所有属性和成员变量
    2.找到私有的成员变量_placeHolderLabel:
    3.利用KVC对_placeholderLabel:进行修改

### 3.2 实现字典转模型
            
