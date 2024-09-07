---
title: "Block的本质"
date: 2023-04-18T22:34:36+08:00
draft: false
---

#  Block

## block的本质
1. block本质是一个OC对象，它内部也有一个指针
2. block是封装了函数实现已经函数调用环境的OC对象
3. block的底层结构


```
struct __block_impl {
    void *isa;
    int Flags;
    int Reserved;
    void *FuncPtr;
    (void *(void *,...))invoke
    variables
    (struct Block_descriptor *)descriptor
};
```
        
```
Block_descriptor: {
    (unsigned long int)reserved
    (unsigned long int)size
    (void *(void *, void *))copy
    (void *(void *))dispose
}
```    

## block结构体

```
struct __ViewController__viewDidLoad_block_impl_0 {
    struct __block_impl impl;
    struct __ViewController__viewDidLoad_block_desc_0* Desc; // 描述信息
    int num;
    
    // 下面是构造函数(相当于初始化函数)
    __ViewController__viewDidLoad_block_impl_0(void *fp,
                                               struct __ViewController__viewDidLoad_block_desc_0 *desc,
                                               int _num,
                                               int flags=0) : num(_num) {
        impl.isa = &_NSConcreteStackBlock;
        impl.Flags = flags;
        impl.FuncPtr = fp;
        Desc = desc;
    }
};
```


## __block_impl结构体
```
struct __block_impl {
  void *isa;
  int Flags;
  int Reserved;
  void *FuncPtr; // 函数调用地址
};
```

##  __ViewController__viewDidLoad_block_desc_0

```
static struct __ViewController__viewDidLoad_block_desc_0 {
  size_t reserved;
  size_t Block_size; // block占用内存大小
} __ViewController__viewDidLoad_block_desc_0_DATA = { 
    0, 
    sizeof(struct __ViewController__viewDidLoad_block_impl_0)
};
```


## __ViewController__viewDidLoad_block_func_0 
```
static void __ViewController__viewDidLoad_block_func_0(struct __ViewController__viewDidLoad_block_impl_0 *__cself, int a, int b) {
      int num = __cself->num; // bound by copy
            NSLog((NSString *)&__NSConstantStringImpl__var_folders_h6_5t3f7nb176g5jy6p602g48f80000gn_T_ViewController_36b3f2_mi_0, num);
}
```
    
## 简化block声明和调用
// 定义block变量
```
void(*parameterBlock)(int, int) = (
&__ViewController__viewDidLoad_block_impl_0( __ViewController__viewDidLoad_block_func_0, // block内部执行的代码
&__ViewController__viewDidLoad_block_desc_0_DATA,  // block描述信息
num));
```

## 调用block代码
```
((parameterBlock)->FuncPtr)(parameterBlock, num, d);
```

为了保证block内部变量能够正常访问外部的变量。block有个变量捕获机制：block结构体内部新增一个变量存储值

```
变量类型                捕获到block内部              访问方式
局部变量 auto                 √                     值传递
局部变量 static               √                     指针传递
全局变量                      X                     直接访问
```

> auto:离开作用域就会自动销毁

会进行捕获self:因为self是auto局部变量 -(void)test()   ==   - (void)test(Class *self, SEL _cmd)传入block中
void(^block4)(void) = ^{
    NSLog(@"%@", self);
};
block();
    

## block类型
block类型:可以通过class方法或者isa指针查看具体类型，最终都是继承自NSBlock类型
- 1._NSGlobalBlock__(_NSConcreteGloalBlock)
- 2._NSStackBlock__(_NSConcreteStackBlock)
- 3._NSMallocBlock__(_NSConcreteMallocBlock)
    
GlobalBlock:
```
void(^globalBlock)(void) = ^{
    NSLog(@"");
};

1. __NSGlobalBlock__ : 
    NSLog(@"globalBlock:%@", [globalBlock class]);
2. NSBlock: 
    NSLog(@"globalBlock:%@", [[globalBlock class] superclass]);
3. NSObject: 
    NSLog(@"globalBlock:%@", [[[globalBlock class] superclass] superclass]);
```
        
## 应用程序的内存分配：

```
程序区域(.text)：
数据区域(.data)：                          _NSConcreteGlobalBlock
堆区(手动分配内存，需要手动管理内存，申请释放)：  _NSContreteMalllocBlock
栈区(自动释放内存)：                         _NSContreteStackBlock
```

## Block的类型原因：

```
Block类型                     环境
_NSGlobalBlock_             没有访问auto变量
_NSStackBlock_              访问了auto变量
_NSMallocBlock_             _NSStackBlock_调用类copy
```
## 每种类型的block调用copy后的结果
```
Block的类                 副本源的配置存储区           复制效果
_NSConcreteGlobalBlock_           栈                 从栈复制到堆
_NSConcreteStackBlock_          程序的数据区            什么也不做
_NSConcreteMallocBlock_           堆                  引用计数增加
```

### block的copy
1.在ARC环境下，编译器会根据情况自动将栈上的block复制到堆上(copy)：
> 1. block作为函数返回值
> 2. 将block赋值给strong指针 
> 3. block作为Cocoa API里的UsingBlock
> 4. block作为GCD API中的方法参数时
    
_NSMallocBlock 会持有引用的变量，使引用的变量引用计数器+1
_NSMallocBlock 不会持有引用的变量

### block_copy
block的复制可以使用，Block_copy()函数又或者copy实例方法。
Block_copy()的实现。在Block.h文件中看到(在Xcode中也可以找到)：

```
#define Block_copy(...) ((__typeof(__VA_ARGS__))_Block_copy((const void *)(__VA_ARGS__)))
```

Block_copy()的原型是_Block_copy()函数，而实际上最后调用的是_Block_copy_internal()函数：

```
//这里传入的参数实际上就是Block
void *_Block_copy(const void *arg) {
    return _Block_copy_internal(arg, WANTS_ONE);
}

static void *_Block_copy_internal(const void *arg, const int flags) {
    struct Block_layout *aBlock;

    //1.如果传递的参数为NULL，返回NULL。
    if (!arg) return NULL;
    
    //2.参数类型转换。转为指向Block_layout结构体的指针。Block_layout结构体请回顾文章开头，相当于clang转换后的__main_block_impl_0结构体，包括指向block的实现功能的指针和各种数据。
    aBlock = (struct Block_layout *)arg;

    //3.如果block的flags包含BLOCK_NEEDS_FREE，表明它是堆上的Block（为什么？见第7步注释）
    //增加引用计数，返回相同的block
    if (aBlock->flags & BLOCK_NEEDS_FREE) {
        // latches on high
        latching_incr_int(&aBlock->flags);
        return aBlock;
    }
    //这里删掉了与垃圾回收（GC）相关的代码，GC不做讨论

    //4.如果是全局block，什么也不做，返回相同的block
    else if (aBlock->flags & BLOCK_IS_GLOBAL) {
        return aBlock;
    }

    // Its a stack block.  Make a copy.
    if (!isGC) {
        //5.能够走到这里，表明是一个栈Block。需要复制到堆上。第一步申请内存
        struct Block_layout *result = malloc(aBlock->descriptor->size);
        if (!result) return (void *)0;
        //6.将栈数据复制到堆上
        memmove(result, aBlock, aBlock->descriptor->size); // bitcopy first
        //7.更新block的flags
        //第一句后面的注释说它不是必须的。
        result->flags &= ~(BLOCK_REFCOUNT_MASK);    // XXX not needed
        //设置flags为BLOCK_NEEDS_FREE，表明它是一个堆block。内存支持它一旦引用计数=0，
        //就进行释放。 “|1”是用来把block的引用计数设置为1。
        result->flags |= BLOCK_NEEDS_FREE | 1;
        //8.block的isa指针设置为_NSConcreteMallocBlock
        result->isa = _NSConcreteMallocBlock;
        //9.如果block有copy helper函数就调用它（和block所持有对象的内存管理有关，文章后面会讲到这部分）
        if (result->flags & BLOCK_HAS_COPY_DISPOSE) {
            //printf("calling block copy helper %p(%p, %p)...\n", aBlock->descriptor->copy, result, aBlock);
            (*aBlock->descriptor->copy)(result, aBlock); // do fixup
        }
        return result;
    }
    else {
        //GC相关
    }
}
```

### block release
同样地，block的释放可以使用Block_release()函数或者release方法。

```
#define Block_release(...) _Block_release((const void *)(__VA_ARGS__))
```
Block_release()原型是_Block_release()函数：

```
void _Block_release(void *arg) {
    //1.参数类型转换，转换为一个指向Block_layout结构体的指针。
    struct Block_layout *aBlock = (struct Block_layout *)arg;
    if (!aBlock) return;

    //2.取出flags中表示引用计数的部分，并且对它递减。
    int32_t newCount;
    newCount = latching_decr_int(&aBlock->flags) & BLOCK_REFCOUNT_MASK;
    //3.如果引用计数>0，表明仍然有对block的引用，block不需要释放
    if (newCount > 0) return;

    if (aBlock->flags & BLOCK_IS_GC) {
        //GC相关
    }
    //4.flags包含BLOCK_NEEDS_FREE（堆block），且引用计数=0
    else if (aBlock->flags & BLOCK_NEEDS_FREE) {
        //如果有copy helper函数就调用，释放block捕获的一些对象，对应_Block_copy_internal中的第9步
        if (aBlock->flags & BLOCK_HAS_COPY_DISPOSE)(*aBlock->descriptor->dispose)(aBlock);
        //释放block
        _Block_deallocator(aBlock);
    }
    //5.全局Block，什么也不做
    else if (aBlock->flags & BLOCK_IS_GLOBAL) {
        ;
    }
    //6.发生了一些奇怪的事情导致堆栈block视图被释放，打印日志警告开发者
    else {
        printf("Block_release called upon a stack Block: %p, ignored\n", (void *)aBlock);
    }
}
```

    
### 对象类型的auto变量：
#### 当block内部访问了对象类型的auto变量时
1. 如果block在栈上，讲不会对auto变量产生强引用
2. 如果block被拷贝到堆上
    会调用block内部的copy函数
    copy函数内部会调用_Block_object_assign函数
    _Block_object_assign函数会根据auto变量的修饰符（__strong , __weak, __unsafe_unretain）做出相应的操作，类似于retain（形成强引用、弱引用）
3. 如果block从堆上移除
    会调用block内部的dispose函数
    dispose函数内部会调用_Block_object_dispose函数
    _Block_object_dispose函数会自动释放引用的auto变量，类似于release
```
函数                  调用时机
copy函数              栈上的Block复制到堆时
dispose函数           堆上的block被废弃时
```

### __block修饰符（修改变量）
1.__block可以用于解决block内部无法修改auto变量值的问题
2.__block不能修饰全局变量、静态变量（static）

__block会将变量包装成一个对象


```
__block int a = 10;
struct __Block_byref_num1_0 {
    void *__isa;
    __Block_byref_num1_0 *__forwarding; // 自己的地址传给__forwarding
    int __flags;
    int __size;
    int num1;
};
```


---

###  __block的内存管理
 
1. block在栈上时，并不会对__block变量产生强引用
2. 当block被copy到堆时
    会调用block内部的copy函数
    copy函数内部会调用_Block_object_assign函数
    _Block_object_assign函数会对__block变量形成强引用（retain）
    _Block_object_assign((void*)&dst->a, (void *)src->a, 8); // BYREF
    _Block_object_assign((void*)&dst->p, (void *)src->p, 3); // OBJECT
    
1. 当block从堆中移除
    会调用block内部的dispose函数
    dispose函数内部会调用_Block_object_dispose函数
    _Block_object_dispose函数会自动释放引用的__block变量(release)
    _Block_object_dispose((void *)src->a, 8) // BYREF
    _Block_object_dispose((void *)src->p, 3) // OBJECT
    
---

### __forwarding指针

栈上的_forwarding指针指向自己本身
栈上的block复制到堆上后
    栈上_block结构体的_forwarding指针指向堆上的block结构体
    堆栈_block结构体的_forwarding指针指向自己本身的指针
    
### 被__block修饰的对象类型

当__block变量在栈上时，不会对指向的对象产生强引用

当__block变量被copy到堆时
    会调用__block结构体内部的copy函数
    copy函数内部会调用_Block_object_assign函数
    _Block_object_assign函数会根据所指向对象的修饰符(__strong, __weak, __unsafe_unretained)做出相应的操作，形成强引用(retain)或者弱引用

如果__block从堆上移除
    会调用__block变量内部的dispose函数
    dispose函数内部会调用_Block_object_dispose函数
    _Block_object_dispose函数会自动释放所指向的对象(release);

---

    

```
循环引用问题：
    object.block内部持有了 object的对象
 
ARC:    
    用__weak __unsafe_unretain解决
    __weak typeof(self) weakSelf = self;
    self.block = ^{
        weakSelf;
    };
    
    __unsafe_unretain id weakSelf = self;
    self.block = ^{
        weakSelf;
    };
    
    用__block解决 对象 = nil
    __block id weakSelf = self;
    self.block = ^{
        weakSelf = nil;
    };
    
    self.blocl();
    
MRC:
    用__unsafe_unretain解决
```


---


#### Q:block的原理是什么？本质是什么？
A:block本质是一个OC对象，内部也有一个isa指针，是封装了函数调用已经函数调用环境的OC对象

#### Q:block的作用是什么？有什么使用注意点
A:将__block修饰的变量包装成对象,解决block内部无法修改auto变量的问题。需要注意内存管理

#### Q:block的属性修饰符为什么是copy？使用block是哪些使用注意
A:block不使用copy，在栈上不会再堆上。

#### Q:block在修改NSMutableArray，需不需要添加__block
A:不需要 
