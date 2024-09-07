---
title: "iOS开发weak底层窥探"
date: 2023-03-07T23:45:02+08:00
draft: false
---


# Weak底层原理

## 1
`weak`关键字修饰的对象指针是弱引用，指向的对象引用技术不会+1，并在对象被释放的时候自动指向nil。
Weak修饰的对象指针，是维护一个`hash`表，`key`是所指向的对象指针，`Value`是weak指针的地址数组

> 在[https://opensource.apple.com/tarballs/](https://opensource.apple.com/tarballs/)
> 中可以下载苹果开源源码。此处下载/objc4即可。（名称中数字越大，源码版本越新）

```
_weak id weakPtr;
NSObject *o = [[NSObject alloc] init];
_weak id weakPtr = o;
```
## 2
创建weak引用的时候，会走到runtime的obj_initWeak方法，可以通过符号断点进行验证：


```
// 初始化weak指针
id objc_initWeak(id *location/* 弱指针地址 */, id newObj/* 对象指针*/) {
    if (!newObj) {
        *location = nil;
        return nil;
    }
    // C++类模板方式
    return storeWeak<DontHaveOld/*false*/, DoHaveNew/*true*/, DoCrashIfDeallocating/*true*/>
        (location, (objc_object*)newObj);
}
```
objc_initWeak初始化的时候，传入的参数为：`DontHaveOld` `DoHaveNew` `DoCrashIfDeallocating`根据传入的参数不同来区别是新增weak指针对象、初始化weak指针对象、还是删除weak指针对象。

## 3
下面来看看storeWeak<>方法，源码：


```
// Update a weak variable.
// If HaveOld is true, the variable has an existing value
//   that needs to be cleaned up. This value might be nil.
// If HaveNew is true, there is a new value that needs to be
//   assigned into the variable. This value might be nil.
// If CrashIfDeallocating is true, the process is halted if newObj is
//   deallocating or newObj's class does not support weak references.
//   If CrashIfDeallocating is false, nil is stored instead.
enum CrashIfDeallocating {
    DontCrashIfDeallocating = false, DoCrashIfDeallocating = true
};

/*
 如果HaveOld为true，则该变量存在一个需要清理的值(这个值可能为nil)
 如果HaveNew为true，则需要将一个新值赋给该变量(这个值可能为nil)
 如果crashhifdeallocate为true，则当newObj正在回收或newObj的类不支持弱引用时，进程将停止。
 如果crashifdeallocate为false，则存储nil。
 */
template <HaveOld haveOld, HaveNew haveNew, CrashIfDeallocating crashIfDeallocating> static id storeWeak(id *location, objc_object *newObj) {
    ASSERT(haveOld  ||  haveNew);//校验旧对象和新对象必须存其一
    if (!haveNew) ASSERT(newObj == nil);//校验如果haveNew=true，newObj不能为nil

    // 该过程用来更新弱引用指针的指向
    // 初始化 previouslyInitializedClass 指针
    Class previouslyInitializedClass = nil;
    id oldObj;
    
    // 声明两个 SideTable
    SideTable *oldTable;
    SideTable *newTable;

    // Acquire locks for old and new values.为旧值和新值获取锁。
    // Order by lock address to prevent lock ordering problems.根据锁地址排序
    // Retry if the old value changes underneath us.
 retry:
    if (haveOld) {
        // 如果weak 指针之前指向过其他对象，取出这个旧对象
        oldObj = *location;
        // 以旧对象为 key，从全局 SideTables()中取出旧对象对应的 SideTable
        oldTable = &SideTables()[oldObj];
    } else {
        oldTable = nil;
    }
    if (haveNew) {
        newTable = &SideTables()[newObj];
    } else {
        newTable = nil;
    }
    
    // 加锁操作，防止多线程中竞争冲突
    SideTable::lockTwo<haveOld, haveNew>(oldTable, newTable);

    //校验，如果旧值对不上 goto retry
    if (haveOld  &&  *location != oldObj) {
        SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);
        goto retry;
    }

    // Prevent a deadlock between the weak reference machinery 防止弱引用间死锁
    // and the +initialize machinery by ensuring that no
    // weakly-referenced object has an un-+initialized isa. 并且通过 +initialize 初始化构造器保证所有弱引用的isa 非空指向,防止+initialized和弱引用引起是死锁
    if (haveNew  &&  newObj) {
        // 获得新对象的 isa 指针
        Class cls = newObj->getIsa();
        if (cls != previouslyInitializedClass  &&  
            !((objc_class *)cls)->isInitialized()) { // 判断 isa 非空且已经初始化
            
            SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);
            
            // 对其 isa 指针进行初始化
            class_initialize(cls, (id)newObj);

            // If this class is finished with +initialize then we're good. 如果该类已经完成执行 +initialize 方法是最理想情况
            // If this class is still running +initialize on this thread  如果该类 +initialize 在线程中
            // (i.e. +initialize called storeWeak on an instance of itself) 例如 +initialize 正在调用 storeWeak 方法
            // then we may proceed but it will appear initializing and 
            // not yet initialized to the check above.
            // Instead set previouslyInitializedClass to recognize it on retry. 需要手动对其增加保护策略，并设置 previouslyInitializedClass 指针进行标记
            previouslyInitializedClass = cls;

            goto retry;
        }
    }

    // Clean up old value, if any. 清除旧值，
    if (haveOld) {
        // 取消注册已经注册的弱引用
        weak_unregister_no_lock(&oldTable->weak_table, oldObj, location);
    }

    // Assign new value, if any. 分配新值
    if (haveNew) {
        newObj = (objc_object *)
            weak_register_no_lock(&newTable->weak_table, (id)newObj, location, 
                                  crashIfDeallocating);
        // weak_register_no_lock returns nil if weak store should be rejected 如果弱引用被释放 weak_register_no_lock 方法返回 nil

        // Set is-weakly-referenced bit in refcount table. 在引用计数表中设置若引用标记位
        if (newObj  &&  !newObj->isTaggedPointer()) {
            newObj->setWeaklyReferenced_nolock();
        }

        // Do not set *location anywhere else. That would introduce a race.之前不要设置 location 对象，这里需要更改指针指向
        *location = (id)newObj;
    }
    else {
        // No new value. The storage is not changed. 没有新值，则无需更改
    }
    
    SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);

    return (id)newObj;
}
```
从上面storeWeak<>方法中，我们可以看出weak指针的相关逻辑：
1. 是否存在一个需要清理的值`hasOld`，还是需要新赋值给该该变量`hasNew`
2. 根据`oldObj`和`newObj`从`SideTables`取出的旧值`SideTable`表和新值`SideTable`表
3. 保护弱引用的对象在+initialize()与类对象注册之后
4. 在`oldTable`的`weak_table_t`中清空之前的weak指针
5. 在`newTable`的`weak_table_t`中注册新的weak指针
6. return`newObj` 

### 3.1
在storeWeak<>方法中，有一个全局SideTables()散列表，下面是对SideTables()的解读：

```
static objc::ExplicitInit<StripedMap<SideTable>> SideTablesMap;

static StripedMap<SideTable>& SideTables() {
    return SideTablesMap.get();
}
```

### 3.2
函数体里面调用了一个全局的静态变量`SideTablesMap`的 `get()`方法，这个静态变量保存了所有的SideTable，是objc命名空间下的一个ExplicitInit类，它里面实现了get()方法，来返回`StripedMap`如下：

```
Type &get() {
        return *reinterpret_cast<Type *>(_storage);
}
```

### 3.3
`StripedMap` 是一个以`void *p  (实际为objc_object *)`为 key，`PaddedT`为 value 的表，实现如下：

```
template<typename T>
class StripedMap {
#if TARGET_OS_IPHONE && !TARGET_OS_SIMULATOR
    enum { StripeCount = 8 }; // 真机是8
#else
    enum { StripeCount = 64 };// 模拟器是64
#endif

    struct PaddedT {
        T value alignas(CacheLineSize); //64
    };

    PaddedT array[StripeCount]; // 散列表

    // 散列函数，通过对象地址计算出对应 PaddedT在数组中的下标
    static unsigned int indexForPointer(const void *p) {
        uintptr_t addr = reinterpret_cast<uintptr_t>(p);
        return ((addr >> 4) ^ (addr >> 9)) % StripeCount;
    }

 public:
    T& operator[] (const void *p) { 
        return array[indexForPointer(p)].value; 
    }
    const T& operator[] (const void *p) const { 
        return const_cast<StripedMap<T>>(this)[p]; 
    }

    // Shortcuts for StripedMaps of locks.
    void lockAll() {
        for (unsigned int i = 0; i < StripeCount; i++) {
            array[i].value.lock();
        }
    }

    void unlockAll() {
        for (unsigned int i = 0; i < StripeCount; i++) {
            array[i].value.unlock();
        }
    }

    void forceResetAll() {
        for (unsigned int i = 0; i < StripeCount; i++) {
            array[i].value.forceReset();
        }
    }

    void defineLockOrder() {
        for (unsigned int i = 1; i < StripeCount; i++) {
            lockdebug_lock_precedes_lock(&array[i-1].value, &array[i].value);
        }
    }

    void precedeLock(const void *newlock) {
        // assumes defineLockOrder is also called
        lockdebug_lock_precedes_lock(&array[StripeCount-1].value, newlock);
    }

    void succeedLock(const void *oldlock) {
        // assumes defineLockOrder is also called
        lockdebug_lock_precedes_lock(oldlock, &array[0].value);
    }

    const void *getLock(int i) {
        if (i < StripeCount) return &array[i].value;
        else return nil;
    }
    
#if DEBUG
    StripedMap() {
        // Verify alignment expectations.
        uintptr_t base = (uintptr_t)&array[0].value;
        uintptr_t delta = (uintptr_t)&array[1].value - base;
        ASSERT(delta % CacheLineSize == 0);
        ASSERT(base % CacheLineSize == 0);
    }
#else
    constexpr StripedMap() {}
#endif
};
```
`StripedMap`维护了一个哈希表，以对象的地址作为参数通过`indexForPointer`方法计算出所在`SideTable`在哈希表中的下标

### 3.4
`StripedMap`维护的哈希表中,存储的PaddedT结构体，结构体的`value`值就是`SideTable`，`SideTable`结构体如下：


```
struct SideTable {
    spinlock_t slock; // 保证原子操作的自旋锁
    RefcountMap refcnts; // 引用计数hash表(协助isa指针的extra_rc共同引用技术的变量)
    weak_table_t weak_table; // weak 引用全局 hash 表

    SideTable() { // 构造函数
        memset(&weak_table, 0, sizeof(weak_table));
    }

    ~SideTable() { // 析构函数
        _objc_fatal("Do not delete SideTable.");
    }

    void lock() { slock.lock(); }
    void unlock() { slock.unlock(); }
    void forceReset() { slock.forceReset(); }

    // Address-ordered lock discipline for a pair of side tables.

    template<HaveOld, HaveNew>
    static void lockTwo(SideTable *lock1, SideTable *lock2);
    template<HaveOld, HaveNew>
    static void unlockTwo(SideTable *lock1, SideTable *lock2);
};
```
### 3.5
`SideTable`结构体内部有一个`weak_table_t weak_table`

```
// id对象作为key，weak_table_t结构体作为value
struct weak_table_t {
    weak_entry_t *weak_entries; // 保存了所有指向指针对象的weak指针
    size_t    num_entries; // entries的数量
    uintptr_t mask; // hash数组的长度-1
    uintptr_t max_hash_displacement; // hashkey的 最大散列偏移
};
```
### 3.6
`weak_table_t`结构体中有一个`weak_entry_t`结构体数组，就是其内部维护的哈希表，`weak_entry_t`的结构体实现如下：

```
/*
 存储在弱引用表中的内部结构。
 它维护并存储一个指向对象的弱引用散列集。
 如果out_of_line_ness != REFERRERS_OUT_OF_LINE，则该集合是一个小的内联数组。
 */
#define WEAK_INLINE_COUNT 4
#define REFERRERS_OUT_OF_LINE 2

struct weak_entry_t {
    DisguisedPtr<objc_object> referent; // 弱引用的对象
    union {
        
        struct {  (找到的资料是4，个人感觉上REFERRERS_OUT_OF_LINE代表的2，out_of_line_ness == 0b10 (0b10 = 2))
            weak_referrer_t *referrers; // 二维指针数组,存储指向该对象的弱引用
            uintptr_t        out_of_line_ness : 2;
            uintptr_t        num_refs : PTR_MINUS_2; // 引用数值
            uintptr_t        mask;
            uintptr_t        max_hash_displacement;
        };
        
        struct {
            // out_of_line_ness field is low bits of inline_referrers[1]
            weak_referrer_t  inline_referrers[WEAK_INLINE_COUNT]; // 存储指向该对象的弱引用数组
        };
    };

    bool out_of_line() {
        return (out_of_line_ness == REFERRERS_OUT_OF_LINE);
    }

    // 覆盖老数据
    weak_entry_t& operator=(const weak_entry_t& other) {
        memcpy(this, &other, sizeof(other));
        return *this;
    }

    // 构造方法
    weak_entry_t(objc_object *newReferent, objc_object **newReferrer)
        : referent(newReferent)
    {
        inline_referrers[0] = newReferrer;
        for (int i = 1; i < WEAK_INLINE_COUNT; i++) {
            inline_referrers[i] = nil;
        }
    }
};
```
在上面的代码实现中可以看出，`weak_entry_t`结构体存放的是某个对象的所有弱引用指针，存放所有弱引用指针使用的是一个联合体，如果弱引用指针的数量不超过 4 个就用`inline_referrers`存储，否则用`referrers`存储

### 3.7
`weak_entry_t`用于存放所有指向某个对象的 `weak` 指针地址，类型是`weak_entry_t`，实现如下:
```
typedef DisguisedPtr<objc_object *> weak_referrer_t;
```

## 4
我们结合上面提到的`storeWeak`方法中进行具体分析，代码如下，

```
template <HaveOld haveOld, HaveNew haveNew,
          CrashIfDeallocating crashIfDeallocating> 
storeWeak(id *location, objc_object *newObj)

{
    //校验旧对象和新对象必须存其一
    ASSERT(haveOld  ||  haveNew);
    //校验如果haveNew=true，newObj不能为nil
    if (!haveNew) ASSERT(newObj == nil);

    Class previouslyInitializedClass = nil;
    id oldObj;
    SideTable *oldTable;
    SideTable *newTable;

 retry:
    if (haveOld) {
        //如果weak 指针指向旧值，就取出旧值
        oldObj = *location;
        //以旧对象地址为 key取出旧的SideTable
        oldTable = &SideTables()[oldObj];
    } else {
        oldTable = nil;
    }
    if (haveNew) {
     // 取出对应新的SideTable
        newTable = &SideTables()[newObj];
    } else {
        newTable = nil;
    }
    
    //上锁
    SideTable::lockTwo<haveOld, haveNew>(oldTable, newTable);
    //校验，如果旧值对不上 goto retry
    if (haveOld  &&  *location != oldObj) {
        SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);
        goto retry;
    }

    //保证弱引用对象的isa非空，防止弱引用机制和+initialize 发生死锁
    if (haveNew  &&  newObj) {
        Class cls = newObj->getIsa();
        if (cls != previouslyInitializedClass  &&  
            !((objc_class *)cls)->isInitialized()) 
        {
            SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);
            //如果class没有初始化发送+initialized消息
            class_initialize(cls, (id)newObj);

            previouslyInitializedClass = cls;
            //到这里class肯定已经初始化了，在走一遍
            goto retry;
        }
    }

    if (haveOld) {
    //<<1>>如果weak 指针之前指向了其他对象，在这里清空
        weak_unregister_no_lock(&oldTable->weak_table, oldObj, location);
    }

    if (haveNew) {
    //通过newObj和location生成一个新的weak_entry_t并插入到newObj的弱引用数组中（weak_entries）
    //<<2>>
        newObj = (objc_object *)
        weak_register_no_lock(&newTable->weak_table, (id)newObj, location, 
        crashIfDeallocating);

        // Set is-weakly-referenced bit in refcount table.
        if (newObj  &&  !newObj->isTaggedPointer()) {
           //<<3>>  设置 isa 的标志位
           newObj->setWeaklyReferenced_nolock();
        }

        // Do not set *location anywhere else. That would introduce a race.
        *location = (id)newObj;
    }
    else {
        // No new value. The storage is not changed.
    }
    
    SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);

    return (id)newObj;
}
```

### 4.1
取消注册已经存在的弱引用指针

```
void weak_unregister_no_lock(weak_table_t *weak_table, id referent_id/*oldObj*/, id *referrer_id/*location*/)
```
上面的方法`weak_unregister_no_lock`主要是清除存储在`weak_entry_t`中的`weak_refrerrer_t`，如果删除后`weak_entry_t`中的数组为空，则将整个`weak_entry_t`从`weak_table_t`中移除，源码如下：

```
// 取消注册已经注册的弱引用
void weak_unregister_no_lock(weak_table_t *weak_table, id referent_id/*oldObj*/, id *referrer_id/*location*/) {
    objc_object *referent = (objc_object *)referent_id;
    objc_object **referrer = (objc_object **)referrer_id;

    weak_entry_t *entry;

    if (!referent) return;
    // 查找referent对应的weak_entry_t
    if ((entry = weak_entry_for_referent(weak_table, referent))) {
        //如果entry存在，删除entry中的referrer
        remove_referrer(entry, referrer); //
        bool empty = true;
        // 判断out_of_line的动态数组referrers中是否有值
        if (entry->out_of_line()  &&  entry->num_refs != 0) {
            empty = false;
        }
        else { //判断entry的定长数组inline_referrers中是否有值
            for (size_t i = 0; i < WEAK_INLINE_COUNT; i++) {
                if (entry->inline_referrers[i]) {
                    empty = false; 
                    break;
                }
            }
        }

        if (empty) {//如果都是空的将entry从weak_table移除
            weak_entry_remove(weak_table, entry);
        }
    }

    // Do not set *referrer = nil. objc_storeWeak() requires that the 
    // value not change.
}
```

### 4.2
查找referent对应的`weak_entry_t``weak_entry_for_referent`

```
static weak_entry_t *
weak_entry_for_referent(weak_table_t *weak_table, objc_object *referent)
{
    ASSERT(referent);

    // 获取所有的weak_entry_t数组
    weak_entry_t *weak_entries = weak_table->weak_entries;

    if (!weak_entries) return nil;

    size_t begin = hash_pointer(referent) & weak_table->mask;
    size_t index = begin;
    size_t hash_displacement = 0;
    // 通过对对象指针的哈希方法生成的值与 weak_table->mask 进行 BITMASK 操作得到一个起始值
    while (weak_table->weak_entries[index].referent != referent) {
        index = (index+1) & weak_table->mask; // hash表取下一个（index+1后&运算）
        if (index == begin) bad_weak_table(weak_table->weak_entries);
        hash_displacement++;
        if (hash_displacement > weak_table->max_hash_displacement) {
            return nil;
        }
    }
    
    return &weak_table->weak_entries[index];
}
```

- 每次遍历如果没有在`weak_entries`中找到`referent`对应`的weak_entry_t`，就对`index + 1` 再进行`BITMASK` 操作，遍历一次就记录一次，直到大于`max_hash_displacement` 最大偏移值，返回` nil`，说明当前对象在`weak_table`的`weak_entries`中没有对应的`weak_entry_t`，也就是说没有弱引用
- 如果某个`weak_entry_t`的r`eferent`和参数`referent`相等，说明找到了，返回这个`weak_entry_t`

### 4.3
删除weak_entry中的referrer
在上一步中我们找到了存储当前对象弱引用的`weak_entry_t`，现在我们就需要从`weak_entry_t`中的`referrers`或者`inline_referrers`中删除掉之前的弱引用，源码实现如下：

```
static void remove_referrer(weak_entry_t *entry, objc_object **old_referrer)
{
    if (! entry->out_of_line()) {
        for (size_t i = 0; i < WEAK_INLINE_COUNT; i++) {
            if (entry->inline_referrers[i] == old_referrer) {
                entry->inline_referrers[i] = nil;
                return;
            }
        }
        _objc_inform("Attempted to unregister unknown __weak variable "
                     "at %p. This is probably incorrect use of "
                     "objc_storeWeak() and objc_loadWeak(). "
                     "Break on objc_weak_error to debug.\n", 
                     old_referrer);
        objc_weak_error();
        return;
    }

    size_t begin = w_hash_pointer(old_referrer) & (entry->mask);
    size_t index = begin;
    size_t hash_displacement = 0;
    
    // 遍历
    while (entry->referrers[index] != old_referrer) {
        index = (index+1) & entry->mask;
        if (index == begin) bad_weak_table(entry);
        hash_displacement++;
        if (hash_displacement > entry->max_hash_displacement) {
            _objc_inform("Attempted to unregister unknown __weak variable "
                         "at %p. This is probably incorrect use of "
                         "objc_storeWeak() and objc_loadWeak(). "
                         "Break on objc_weak_error to debug.\n", 
                         old_referrer);
            objc_weak_error();
            return;
        }
    }
    
    // 将找到的对象清空
    entry->referrers[index] = nil;
    
    // 总引用数减一
    entry->num_refs--;
}
```

### 4.4
如果entry中的数组为空，则从weak_table_t中删除entry

```
static void weak_entry_remove(weak_table_t *weak_table, weak_entry_t *entry)
{
    // remove entry
    // 如果使用的是动态数组，则释放动态数组的内存
    if (entry->out_of_line()) free(entry->referrers);
     // 以 entry 为起始地址的前sizeof(*entry)个字节区域清零
    bzero(entry, sizeof(*entry));
    // 全局 weak_table_t中的弱引用对象数量-1
    weak_table->num_entries--;
    // 收缩表大小
    weak_compact_maybe(weak_table); // 指针表减少内存开销
}
```

### 4.5
weak_compact_maybe()收缩表大小

```
// Shrink the table if it is mostly empty.
static void weak_compact_maybe(weak_table_t *weak_table)
{
    size_t old_size = TABLE_SIZE(weak_table);

    // 如果 weak_table 的表大小占用超过 1024 个字节，并且 1/16比弱引用的对象的数量还多就收缩表的大小，使其不超过原来的 1/2
    // Shrink if larger than 1024 buckets and at most 1/16 full.
    if (old_size >= 1024  && old_size / 16 >= weak_table->num_entries) {
        weak_resize(weak_table, old_size / 8);
        // leaves new table no more than 1/2 full
    }
}
```

### 4.6
weak_resize() weak_table_t表重新分配内存大小

```
static void weak_resize(weak_table_t *weak_table, size_t new_size)
{
    // 获取旧的大小
    size_t old_size = TABLE_SIZE(weak_table);

    weak_entry_t *old_entries = weak_table->weak_entries;
    // 使用新的大小创建新的new_entries
    weak_entry_t *new_entries = (weak_entry_t *)
        calloc(new_size, sizeof(weak_entry_t));

    weak_table->mask = new_size - 1;
    weak_table->weak_entries = new_entries;
    weak_table->max_hash_displacement = 0;
    weak_table->num_entries = 0;  // restored by weak_entry_insert below
    
    // 如果old_entries还有值，则进行遍历重新放入 weak_table 新的weak_entries中
    if (old_entries) {
        weak_entry_t *entry;
        weak_entry_t *end = old_entries + old_size;
        for (entry = old_entries; entry < end; entry++) {
            if (entry->referent) {
                weak_entry_insert(weak_table, entry);
            }
        }
        free(old_entries);
    }
}
```
### 4.7
生成新的weak_entry_t插入到weak_table_t中的weak_entries中,即`weak_register_no_lock()`，源码如下：


```
// 注册一个新的弱引用 referrer:（上线）弱指针地址
id weak_register_no_lock(weak_table_t *weak_table, id referent_id, id *referrer_id, bool crashIfDeallocating) {
    objc_object *referent = (objc_object *)referent_id;
    objc_object **referrer = (objc_object **)referrer_id;

    // 如果为 nil，或者是TaggedPointer，则直接 return referent_id
    if (!referent  ||  referent->isTaggedPointer()) return referent_id;

    // ensure that the referenced object is viable
    // 判断当前对象是否正在释放或是否支持弱引用
    bool deallocating;
    if (!referent->ISA()->hasCustomRR()) {
        deallocating = referent->rootIsDeallocating(); // 是否正在释放
    }
    else {
        BOOL (*allowsWeakReference)(objc_object *, SEL) = 
            (BOOL(*)(objc_object *, SEL))
            object_getMethodImplementation((id)referent, 
                                           @selector(allowsWeakReference));
        if ((IMP)allowsWeakReference == _objc_msgForward) {
            return nil;
        }
        deallocating =
            ! (*allowsWeakReference)(referent, @selector(allowsWeakReference));
    }

    if (deallocating) { // 正在释放
        if (crashIfDeallocating) {
            _objc_fatal("Cannot form weak reference to instance (%p) of "
                        "class %s. It is possible that this object was "
                        "over-released, or is in the process of deallocation.",
                        (void*)referent, object_getClassName((id)referent));
        } else {
            return nil;
        }
    }

    // now remember it and where it is being stored
    // 判断如果对象已经在 weak_table 中存在弱引用记录，就在原来的 entry 上追加
    weak_entry_t *entry;
    if ((entry = weak_entry_for_referent(weak_table, referent))) { // weak_table中hash遍历查找referent
        
        // 如果存在,将弱引用指针和对象地址添加入hash表
        append_referrer(entry, referrer);
    } 
    else {
        
        // 生成新的 new_entry 弱引用指针和对象地址
        weak_entry_t new_entry(referent, referrer);
        // 根据情况hash表扩容
        weak_grow_maybe(weak_table);
        // hash表插入新生成的new_entry
        weak_entry_insert(weak_table, &new_entry);
    }

    // Do not set *referrer. objc_storeWeak() requires that the 
    // value not change.

    return referent_id;
}
```

### 4.8
在`weak_entry_t`中添加新的`weak_referrer_t`

```
static void append_referrer(weak_entry_t *entry, objc_object **new_referrer)
{
    if (! entry->out_of_line()) {
        // Try to insert inline.
        for (size_t i = 0; i < WEAK_INLINE_COUNT; i++) {
            if (entry->inline_referrers[i] == nil) {
                entry->inline_referrers[i] = new_referrer;
                return;
            }
        }

        // Couldn't insert inline. Allocate out of line.
        // 如果放不下了，就申请创建动态数组new_referrers
        weak_referrer_t *new_referrers = (weak_referrer_t *)
            calloc(WEAK_INLINE_COUNT, sizeof(weak_referrer_t));
        // This constructed table is invalid, but grow_refs_and_insert
        // will fix it and rehash it.
        // 然后将之前定长数组的弱引用赋值给new_referrers
        for (size_t i = 0; i < WEAK_INLINE_COUNT; i++) {
            new_referrers[i] = entry->inline_referrers[i];
        }
        entry->referrers = new_referrers;
        entry->num_refs = WEAK_INLINE_COUNT;
        entry->out_of_line_ness = REFERRERS_OUT_OF_LINE;
        entry->mask = WEAK_INLINE_COUNT-1;
        entry->max_hash_displacement = 0;
    }

    ASSERT(entry->out_of_line());

    // 如果引用数量超过表大小的 3/4就自动扩容
    if (entry->num_refs >= TABLE_SIZE(entry) * 3/4) {
        return grow_refs_and_insert(entry, new_referrer);
    }
    // 在 referrers 中找到一个值为 nil 的 weak_referrer_t，用新的弱引用对其赋值，并将引用数量+1
    size_t begin = w_hash_pointer(new_referrer) & (entry->mask);
    size_t index = begin;
    size_t hash_displacement = 0;
    while (entry->referrers[index] != nil) {
        hash_displacement++;
        index = (index+1) & entry->mask;
        if (index == begin) bad_weak_table(entry);
    }
    if (hash_displacement > entry->max_hash_displacement) {
        entry->max_hash_displacement = hash_displacement;
    }
    weak_referrer_t &ref = entry->referrers[index];
    ref = new_referrer;
    entry->num_refs++;
}

```

### 4.9
`grow_refs_and_insert`对某个对象的弱引用表扩容并强行插入

```
__attribute__((noinline, used))
static void grow_refs_and_insert(weak_entry_t *entry, 
                                 objc_object **new_referrer)
{
    ASSERT(entry->out_of_line());

    // 获取旧表的大小
    size_t old_size = TABLE_SIZE(entry);
    // 如果之前有旧表，则扩容为之前的 2 倍，否则为 8
    size_t new_size = old_size ? old_size * 2 : 8;

    // 获取当前对象所有弱引用指针的数量
    size_t num_refs = entry->num_refs;
    weak_referrer_t *old_refs = entry->referrers;
    entry->mask = new_size - 1;
    // mask 赋值
    entry->referrers = (weak_referrer_t *)
        calloc(TABLE_SIZE(entry), sizeof(weak_referrer_t));
    entry->num_refs = 0;
    entry->max_hash_displacement = 0;
    
    // 将老的弱引用指针地址放到新的里边
    for (size_t i = 0; i < old_size && num_refs > 0; i++) {
        if (old_refs[i] != nil) {
            append_referrer(entry, old_refs[i]);
            num_refs--;
        }
    }
    // Insert
    // 最后将新的弱引用指针地址进行插入
    append_referrer(entry, new_referrer);
    if (old_refs) free(old_refs);
}
```

### 4.10
构造函数`new_entry()`创建新的entry

```
// 构造方法
    weak_entry_t(objc_object *newReferent, objc_object **newReferrer)
        : referent(newReferent)
    {
        inline_referrers[0] = newReferrer;
        for (int i = 1; i < WEAK_INLINE_COUNT; i++) {
            inline_referrers[i] = nil;
        }
    }
```

### 4.11
`weak_grow_maybe()` ，`weak_table` 扩容


```
static void weak_grow_maybe(weak_table_t *weak_table)
{
    size_t old_size = TABLE_SIZE(weak_table);

    // Grow if at least 3/4 full.
    // 如果超过 3/4 则进行扩容，如果之前有，则为原来的 2 倍，否则为 64
    if (weak_table->num_entries >= old_size * 3 / 4) {
        weak_resize(weak_table, old_size ? old_size*2 : 64);
    }
}
```

### 4.12
在weak_table_t中插入weak_entry_t

```
static void weak_entry_insert(weak_table_t *weak_table, weak_entry_t *new_entry)
{
    weak_entry_t *weak_entries = weak_table->weak_entries;
    ASSERT(weak_entries != nil);

    // 计算出要插入的位置
    size_t begin = hash_pointer(new_entry->referent) & (weak_table->mask);
    size_t index = begin;
    size_t hash_displacement = 0;
    while (weak_entries[index].referent != nil) {
        index = (index+1) & weak_table->mask;
        if (index == begin) bad_weak_table(weak_entries);
        hash_displacement++;
    }

    // 进行赋值，数量自增
    weak_entries[index] = *new_entry;
    weak_table->num_entries++;

    // 对最大max_hash_displacement偏移值进行赋值，这也是查找时遍历的临界点
    if (hash_displacement > weak_table->max_hash_displacement) {
        weak_table->max_hash_displacement = hash_displacement;
    }
}
```

### 4.13
设置弱引用的标记位`setWeaklyReferenced_nolock`

```
inline void
objc_object::setWeaklyReferenced_nolock()
{
 retry:
    // 获取对象的 isa 指针
    isa_t oldisa = LoadExclusive(&isa.bits);
    isa_t newisa = oldisa;
    if (slowpath(!newisa.nonpointer)) {
        ClearExclusive(&isa.bits);
        sidetable_setWeaklyReferenced_nolock();
        return;
    }
    // 如果对象isa 的弱引用标志位已经是 true 了，则不进行操作
    if (newisa.weakly_referenced) {
        ClearExclusive(&isa.bits);
        return;
    }
    // 弱引用标志位设为 true
    newisa.weakly_referenced = true;
    if (!StoreExclusive(&isa.bits, oldisa.bits, newisa.bits)) goto retry;
}
```

## 5
weak修饰对象的释放


```
- (void)dealloc {
    _objc_rootDealloc(self);
}

⬇

void _objc_rootDealloc(id obj) {
    ASSERT(obj);

    obj->rootDealloc();
}

⬇

inline void objc_object::rootDealloc() {
    if (isTaggedPointer()) return;  // fixme necessary?
    
    // 没指针、没弱引用、没关联对象、没c++析构、没弱引用表retain_count，直接释放
    if (fastpath(isa.nonpointer  &&  
                 !isa.weakly_referenced  &&  
                 !isa.has_assoc  &&  
                 !isa.has_cxx_dtor  &&  
                 !isa.has_sidetable_rc))
    {
        assert(!sidetable_present());
        free(this);
    } 
    else {
        object_dispose((id)this);
    }
}

⬇

id  object_dispose(id obj) {
    if (!obj) return nil;

    objc_destructInstance(obj);    
    free(obj);

    return nil;
}

⬇

void *objc_destructInstance(id obj)  {
    if (obj) {
        // Read all of the flags at once for performance.
        bool cxx = obj->hasCxxDtor();
        bool assoc = obj->hasAssociatedObjects();

        // This order is important.
        if (cxx) object_cxxDestruct(obj);
        if (assoc) _object_remove_assocations(obj);
        obj->clearDeallocating();
    }

    return obj;
}

⬇

inline void 
objc_object::clearDeallocating() {
    if (slowpath(!isa.nonpointer)) {
        // Slow path for raw pointer isa.
        sidetable_clearDeallocating();
    }
    else if (slowpath(isa.weakly_referenced  ||  isa.has_sidetable_rc)) {
        // Slow path for non-pointer isa with weak refs and/or side table data.
        clearDeallocating_slow();
    }

    assert(!sidetable_present());
}

⬇

NEVER_INLINE void objc_object::clearDeallocating_slow()
{
    ASSERT(isa.nonpointer  &&  (isa.weakly_referenced || isa.has_sidetable_rc));

    SideTable& table = SideTables()[this];
    table.lock();
    if (isa.weakly_referenced) {
        weak_clear_no_lock(&table.weak_table, (id)this);
    }
    if (isa.has_sidetable_rc) {
        table.refcnts.erase(this);
    }
    table.unlock();
}

```

最后对象被指向的弱指针释放

```
void 
weak_clear_no_lock(weak_table_t *weak_table, id referent_id) 
{
    objc_object *referent = (objc_object *)referent_id;

    // 获取当前对象所在的weak_entry_t
    weak_entry_t *entry = weak_entry_for_referent(weak_table, referent);
    if (entry == nil) {
        /// XXX shouldn't happen, but does with mismatched CF/objc
        //printf("XXX no entry for clear deallocating %p\n", referent);
        return;
    }

    // zero out references
    weak_referrer_t *referrers;
    size_t count;
    
    if (entry->out_of_line()) {
        referrers = entry->referrers;
        count = TABLE_SIZE(entry);
    } 
    else {
        referrers = entry->inline_referrers;
        count = WEAK_INLINE_COUNT;
    }
    
    // 遍历entry的 referrers 数组，将弱引用指针全置为 nil
    for (size_t i = 0; i < count; ++i) {
        objc_object **referrer = referrers[i];
        if (referrer) {
            if (*referrer == referent) {
                *referrer = nil;
            }
            else if (*referrer) {
                _objc_inform("__weak variable at %p holds %p instead of %p. "
                             "This is probably incorrect use of "
                             "objc_storeWeak() and objc_loadWeak(). "
                             "Break on objc_weak_error to debug.\n", 
                             referrer, (void*)*referrer, (void*)referent);
                objc_weak_error();
            }
        }
    }
    
    // 从 weak_table中移除当前对象对应的 entry
    weak_entry_remove(weak_table, entry);
}
```
