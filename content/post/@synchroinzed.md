---
title: "@synchroinzed"
date: 2023-09-09T19:31:49+08:00
draft: flase
---

# @synchroinzed

# 源码
查看码源，通过clang查看编译后的cpp文件或者通过汇编短线。
xcrun --sdk iphoneos clang -arch arm64 -rewrite-objc -fobjc-arc -fobjcruntime=ios-14.2 ViewController.m

```
0x10241dd48 <+52>:  bl     0x10241e320               ; symbol stub for: objc_msgSendSuper2
0x10241dd4c <+56>:  ldur   x0, [x29, #-0x8]
0x10241dd50 <+60>:  bl     0x10241e344               ; symbol stub for: objc_retain
0x10241dd54 <+64>:  str    x0, [sp, #0x8]
0x10241dd58 <+68>:  bl     0x10241e368               ; symbol stub for: objc_sync_enter
0x10241dd5c <+72>:  adrp   x0, 3
0x10241dd60 <+76>:  add    x0, x0, #0x90             ; @""
0x10241dd64 <+80>:  bl     0x10241e2c0               ; symbol stub for: NSLog
0x10241dd68 <+84>:  b      0x10241dd6c               ; <+88> at ViewController.m
0x10241dd6c <+88>:  ldr    x0, [sp, #0x8]
0x10241dd70 <+92>:  bl     0x10241e374               ; symbol stub for: objc_sync_exit
0x10241dd74 <+96>:  ldr    x0, [sp, #0x8]
0x10241dd78 <+100>: bl     0x10241e338               ; symbol stub for: objc_release
```

在断点中添加symbol断掉objc_sync_enter，可以找到是在libobjc库中。


```
libobjc.A.dylib`objc_sync_enter:
0x180048438 <+0>:  stp    x29, x30, [sp, #-0x10]!
0x18004843c <+4>:  mov    x29, sp
0x180048440 <+8>:  cbz    x0, 0x18004845c           ; <+36>
0x180048444 <+12>: mov    w1, #0x0
0x180048448 <+16>: bl     0x180048488               ; id2data(objc_object*, usage)
0x18004844c <+20>: add    x0, x0, #0x14
0x180048450 <+24>: mov    w1, #0x0
0x180048454 <+28>: bl     0x1800526b4               ; symbol stub for: os_unfair_recursive_lock_lock_with_options
0x180048458 <+32>: b      0x18004846c               ; <+52>
0x18004845c <+36>: adrp   x8, 232525
0x180048460 <+40>: ldrb   w8, [x8, #0xf00]
0x180048464 <+44>: cbnz   w8, 0x180048478           ; <+64>
0x180048468 <+48>: bl     0x180048434               ; objc_sync_nil
0x18004846c <+52>: mov    w0, #0x0
0x180048470 <+56>: ldp    x29, x30, [sp], #0x10
0x180048474 <+60>: ret    
0x180048478 <+64>: adrp   x0, 22
0x18004847c <+68>: add    x0, x0, #0xcc             ; "NIL SYNC DEBUG: @synchronized(nil); set a breakpoint on objc_sync_nil to debug"
0x180048480 <+72>: bl     0x180050fec               ; _objc_inform
0x180048484 <+76>: b      0x180048468               ; <+48>
```

根据汇编代码执行顺序可以看到，主要是执行了objc_sync_enter和objc_sync_exit，在objc_sync_enter中主要调用了id2data(objc_object*, usage)函数

# synchroinzed数据结构

- SyncData:
- SyncList:
- sDataLists:

# SyncList: 
```

// SyncList 作为表中的⾸节点存在，存储着 SyncData 链表的头结点
struct SyncList {
    SyncData *data;// 指向的 SyncData 对象
    spinlock_t lock; // 操作 SyncList 时防⽌多线程资源竞争的锁，这⾥要和 SyncData 中的 mutex 区分开作⽤，SyncData 中的 mutex 才是实际代码块加锁使⽤的

    constexpr SyncList() : data(nil), lock(fork_unsafe_lock) { }
};

```

# SyncData: 
```

typedef struct alignas(CacheLineSize) SyncData {
    struct SyncData* nextData; // 指向下⼀个 SyncData 节点，作⽤类似链表
    DisguisedPtr<objc_object> object; // 绑定的作为 key 的对象
    int32_t threadCount;  // number of THREADS using this block 使⽤当前 obj 作为 key 的线程数
    recursive_mutex_t mutex; // 递归锁，根据源码继承链其实是 apple ⾃⼰封装了os_unfair_lock 实现的递归锁
} SyncData;

```


# sDataList:

```
static StripedMap<SyncList> sDataLists;  // 哈希表，以关联的 obj 内存地址作为 key，value是 SyncList 类 型
```



# synchroinzed中主要的方法：

```
static SyncData* id2data(id object, enum usage why) {
    spinlock_t *lockp = &LOCK_FOR_OBJ(object);
    SyncData **listp = &LIST_FOR_OBJ(object); // object作为key，通过hash map维护一个递归锁
    SyncData* result = NULL;

    // 1.快速缓存方案(TLS Cache)
#if SUPPORT_DIRECT_THREAD_KEYS
    // Check per-thread single-entry fast cache for matching object
    bool fastCacheOccupied = NO; // 快速缓存是否被占用
    SyncData *data = (SyncData *)tls_get_direct(SYNC_DATA_DIRECT_KEY); // ⾸先判断是否命中 TLS(线程局部存储-thread local storage) 快速缓存
    if (data) {
        fastCacheOccupied = YES;

        if (data->object == object) { // 命中快速缓存 且 命中的快速缓存的object是传入的object
            // Found a match in fast cache.
            uintptr_t lockCount;

            result = data;
            lockCount = (uintptr_t)tls_get_direct(SYNC_COUNT_DIRECT_KEY); //
            if (result->threadCount <= 0  ||  item->lockCount <= 0) { // threadCount当前SyncData被使用的线程数，lockCount当前线程被锁的次数
                _objc_fatal("id2data fastcache is buggy");
            }

            switch(why) { // 根据传入的usage 枚举，对lockCount做加减。
            case ACQUIRE: {
                lockCount++;
                tls_set_direct(SYNC_COUNT_DIRECT_KEY, (void*)lockCount);
                break;
            }
            case RELEASE:
                lockCount--;
                tls_set_direct(SYNC_COUNT_DIRECT_KEY, (void*)lockCount);
                if (lockCount == 0) { // 如果lockCount == 0，快速缓存中的lockCount为NULL，
                    // remove from fast cache
                    tls_set_direct(SYNC_DATA_DIRECT_KEY, NULL);
                    // atomic because may collide with concurrent ACQUIRE 原子性因为可能并发ACQUIRE
                    OSAtomicDecrement32Barrier(&result->threadCount); // 减少线程数量
                }
                break;
            case CHECK:
                // do nothing
                break;
            }

            return result;
        }
    }
#endif

    // 2.缓存方案(二级缓存 hash map)
    // Check per-thread cache of already-owned locks for matching object
    SyncCache *cache = fetch_cache(NO);
    if (cache) {
        unsigned int i;
        for (i = 0; i < cache->used; i++) {
            SyncCacheItem *item = &cache->list[i];
            if (item->data->object != object) continue;

            // Found a match.
            result = item->data;
            if (result->threadCount <= 0  ||  item->lockCount <= 0) {
                _objc_fatal("id2data cache is buggy");
            }
                
            switch(why) {
            case ACQUIRE:
                item->lockCount++;
                break;
            case RELEASE:
                item->lockCount--;
                if (item->lockCount == 0) {
                    // remove from per-thread cache
                    cache->list[i] = cache->list[--cache->used];
                    // atomic because may collide with concurrent ACQUIRE
                    OSAtomicDecrement32Barrier(&result->threadCount);
                }
                break;
            case CHECK:
                // do nothing
                break;
            }

            return result;
        }
    }

    // Thread cache didn't find anything.
    // Walk in-use list looking for matching object
    // Spinlock prevents multiple threads from creating multiple 
    // locks for the same new object.
    // We could keep the nodes in some hash table if we find that there are
    // more than 20 or so distinct locks active, but we don't do that now.
    
    // 线程缓存没有找到任何东西。
    // 遍历使用列表查找匹配对象
    // 自旋锁阻止多个线程创建多个线程
    // 锁定相同的新对象。
    // 我们可以将节点保存在某个哈希表中，如果我们发现有,超过20个不同的锁处于活动状态，但我们现在不这样做。
    
    lockp->lock();

    // 3.两个缓存都没有命中，遍历全局表SyncDataList。为了防止多线程影响，使用了SyncList结构中的lock加锁
    {
        SyncData* p;
        SyncData* firstUnused = NULL;
        for (p = *listp; p != NULL; p = p->nextData) {
            if ( p->object == object ) {
                result = p;
                // atomic because may collide with concurrent RELEASE
                OSAtomicIncrement32Barrier(&result->threadCount);
                goto done;
            }
            if ( (firstUnused == NULL) && (p->threadCount == 0) )
                firstUnused = p;
        }
    
        // no SyncData currently associated with object
        if ( (why == RELEASE) || (why == CHECK) )
            goto done;
    
        // an unused one was found, use it
        if ( firstUnused != NULL ) { // 找到SyncData，加锁，lockCount = 1
            result = firstUnused;
            result->object = (objc_object *)object;
            result->threadCount = 1;
            goto done;
        }
    }

    // Allocate a new SyncData and add to list.
    // XXX allocating memory with a global lock held is bad practice,
    // might be worth releasing the lock, allocating, and searching again.
    // But since we never free these guys we won't be stuck in allocation very often.
    
    // 没找到SyncData，则生成一个SyncData
    posix_memalign((void **)&result, alignof(SyncData), sizeof(SyncData));
    result->object = (objc_object *)object;
    result->threadCount = 1;
    new (&result->mutex) recursive_mutex_t(fork_unsafe_lock);
    result->nextData = *listp;
    *listp = result;
    
 done:
    lockp->unlock();
    if (result) {
        // Only new ACQUIRE should get here.
        // All RELEASE and CHECK and recursive ACQUIRE are 
        // handled by the per-thread caches above.
        if (why == RELEASE) {
            // Probably some thread is incorrectly exiting 
            // while the object is held by another thread.
            return nil;
        }
        if (why != ACQUIRE) _objc_fatal("id2data is buggy");
        if (result->object != object) _objc_fatal("id2data is buggy");

#if SUPPORT_DIRECT_THREAD_KEYS
        if (!fastCacheOccupied) {
            // Save in fast thread cache
            tls_set_direct(SYNC_DATA_DIRECT_KEY, result);
            tls_set_direct(SYNC_COUNT_DIRECT_KEY, (void*)1);
        } else 
#endif
        {
            // Save in thread cache
            if (!cache) cache = fetch_cache(YES);
            cache->list[cache->used].data = result;
            cache->list[cache->used].lockCount = 1;
            cache->used++;
        }
    }

    return result;
}
```


# 代码流程：
根据传入的objcect作为key，从sDataList取出对应的SyncList中存储的SyncData和lock对象
## 三个步骤查找：
1. 线程局部存储中
    使用fastCacheOccupied标记，是否已经有快速缓存
    判断是否命中 TLS 快速缓存，对应代码 SyncData *data = (SyncData*)tls_get_direct(SYNC_DATA_DIRECT_KEY);
2. 苹果实现的SyncCache中
    未命中则判断是否命中⼆级缓存 SyncCache , 对应代码 SyncCache *cache = fetch_cache(NO);
3. 遍历全局sDataList表
    如果两个缓存都没有命中，则会遍历全局表 SyncDataLists , 此时为了防⽌多线程影响查询，使⽤了 SyncList结构中的 lock加锁（注意区分和SyncData中lock的作⽤）。查找到则说明存在⼀个 SyncData 对象供其他线程在使⽤，当前线程使⽤需要设置 threadCount + 1 然后存储到上⽂的缓存中；
4. 如果以上查找都未找到，则会⽣成⼀个 SyncData 节点, 并通过 done 代码段填充到缓存中

## 命中逻辑类似，如果有result，
- 加锁，则将 lockCount ++，记录 key object 对应的 SyncData 变量lock 的加锁次数，再次存储回对应的缓存
- 解锁，同样 lockCount--，如果 ==0，表示当前线程中 object 关联的锁不再使⽤了，对应缓存中SyncData 的 threadCount 减1，当前线程中 object 作为 key 的加锁代码块完全释放


























