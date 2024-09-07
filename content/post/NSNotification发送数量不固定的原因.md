---
title: "NSNotification发送数量不固定的原因"
date: 2023-05-02T11:49:58+08:00
draft: false
---

# 一、了解NSNotification
## NSNotification用法：
```
// 发送通知
[[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(noti:) name:@"NSNotificationCenterPostName" object:nil];

// 接收通知
[[NSNotificationCenter defaultCenter] postNotificationName:@"NSNotificationCenterPostName" object:nil];

- (void)noti:(NSNotification *)notifica {
    NSLog(@"%@", notifica);
}

```

## NSNotification类

```

@interface NSNotification : NSObject <NSCopying, NSCoding>

@property (readonly, copy) NSNotificationName name; // 通知名称
@property (nullable, readonly, retain) id object;   // 携带的对象，可以用于传参
@property (nullable, readonly, copy) NSDictionary *userInfo; // 配置信息

@end
```

## NSNotificationCenter类
### 主要功能：
1. 添加通知观察者
2. 发送通知
3. 移除通知

### 核心API：
```
- (void)addObserver:(id)observer selector:(SEL)aSelector name:(nullable NSNotificationName)aName object:(nullable id)anObject;
- (void)postNotification:(NSNotification *)notification;
- (void)postNotificationName:(NSNotificationName)aName object:(nullable id)anObject;
- (void)postNotificationName:(NSNotificationName)aName object:(nullable id)anObject userInfo:(nullable NSDictionary *)aUserInfo;
- (void)removeObserver:(id)observer;
- (void)removeObserver:(id)observer name:(nullable NSNotificationName)aName object:(nullable id)anObject;
- (id <NSObject>)addObserverForName:(nullable NSNotificationName)name object:(nullable id)obj queue:(nullable NSOperationQueue *)queue usingBlock:(void (NS_SWIFT_SENDABLE ^)(NSNotification *note))block API_AVAILABLE(macos(10.6), ios(4.0), watchos(2.0), tvos(9.0));
```

## NSNotificationQueue类
通知队列，用于异步发送消息，这个异步并不是开启线程，而是把通知存到双向链表实现的队列里面，等到某个时机出发时调用NSNotificationCenter的发送接口进行发送通知。NSNotificationQueue最终还是调用了NSNotificationCenter进行消息发送。
NSNotificationQueue是依赖runloop的，所以如果线程的runloop未开启则无效。

### 主要功能：

1. 添加到通知队列
2. 删除通知

### 核心API：
```

// 把通知添加到队列中，NSPostingStyle是个枚举，下面会介绍
- (void)enqueueNotification:(NSNotification *)notification postingStyle:(NSPostingStyle)postingStyle;
- (void)enqueueNotification:(NSNotification *)notification postingStyle:(NSPostingStyle)postingStyle coalesceMask:(NSNotificationCoalescing)coalesceMask forModes:(nullable NSArray<NSRunLoopMode> *)modes;
// 删除通知，把满足合并条件的通知从队列中删除
- (void)dequeueNotificationsMatching:(NSNotification *)notification coalesceMask:(NSUInteger)coalesceMask;
```

##    队列的合并策略和发送时机

把通知添加到队列等待发送，同时提供一些附加条件提供开发者选择，如什么时候发送、如何合并通知发送

```
typedef NS_ENUM(NSUInteger, NSPostingStyle) {
    NSPostWhenIdle = 1, // runloop空闲时发送
    NSPostASAP = 2,     // as soon as possible 尽快发送
    NSPostNow = 3       // 立刻发送或者合并通知完成后发送
};// 表示通知的发送时机


typedef NS_OPTIONS(NSUInteger, NSNotificationCoalescing) {
    NSNotificationNoCoalescing = 0,        // 默认不合并
    NSNotificationCoalescingOnName = 1,    // 只要name相同，就认为是相同通知
    NSNotificationCoalescingOnSender = 2   // object相同
};// 通知合并的策略，有些时候同名通知只想存在一个，这时候就可以用到它了
```

上面介绍的是NSNotification对外暴露的API和类，具体通知发送的机制，我们可以通过开源的GNUStep源码中查看具体的逻辑。

### 注册通知：
```
- (id <NSObject>)addObserverForName:(nullable NSNotificationName)name object:(nullable id)obj queue:(nullable NSOperationQueue *)queue usingBlock:(void (NS_SWIFT_SENDABLE ^)(NSNotification *note))block API_AVAILABLE(macos(10.6), ios(4.0), watchos(2.0), tvos(9.0));
上面这个函数，最终调用的是
- (void)addObserver:(id)observer selector:(SEL)aSelector name:(nullable NSNotificationName)aName object:(nullable id)anObject;
不同点是，addObserverForName:object:queue:usingBlock初始化了一个GSNotificationObserver对象作为观察者来触发响应的方法，并且在响应的方法中把block抛到指定的queue中。所以我们先来看addObserver:selector:name:object的源代码和用到的NCTbl数据结构
```

### 添加通知底层代码GNU实现
```
- (void)addObserver:(id)observer
       selector: (SEL)selector
           name: (NSString*)name
         object: (id)object {
    Observation    *list; // 保存既没有name也没有object的通知
    Observation    *o;
    GSIMapTable    m;    // Map 存储带有name的通知，不管有没有object
    GSIMapNode    n;    // Node
    
    // 容错判断
    if (observer == nil)
        [NSException raise: NSInvalidArgumentException
                    format: @"Nil observer passed to addObserver ..."];
    
    if (selector == 0)
        [NSException raise: NSInvalidArgumentException
                    format: @"Null selector passed to addObserver ..."];
    
    if ([observer respondsToSelector: selector] == NO)
    {
        [NSException raise: NSInvalidArgumentException
                    format: @"[%@-%@] Observer '%@' does not respond to selector '%@'",
         NSStringFromClass([self class]), NSStringFromSelector(_cmd),
         observer, NSStringFromSelector(selector)];
    }
    
    lockNCTable(TABLE);
    
    // 创建Observation对象，持有观察者和SEL，下面进行的逻辑就是为了存储它
    o = obsNew(TABLE, selector, observer);
    
    /*
     * Record the Observation in one of the linked lists.
     *
     * NB. It is possible to register an observer for a notification more than
     * once - in which case, the observer will receive multiple messages when
     * the notification is posted... odd, but the MacOS-X docs specify this.
     */
    
    if (name) // name存在
    {
        
        /* --- 存的过程(注册通知的过程) ---*/
        //1.以name为key，从named字典中取出n（这个n被MapNode包装了一层），这个n还是个字典
        //2.以object为key，从字典n中取出对应的值，这个值就是Observation类型的链表，然后把刚创建的obs对象o存储进去
        
        /*
         named表（maptable） |----------key（name）
         |----------value（maptable） |----------key（object）
         |----------value（Observation对象）
         */
        
        // 1.找到NCTable中的named表，这个表中存储了还有name的通知
        // 2.以name作为key，找到value，这个value依然是个map
        // 3.map的结构是以object作为key，obs对象作为value，这个obs主要存储了observer && sel
        
        /*
         * Locate the map table for this name - create it if not present.
         */
        n = GSIMapNodeForKey(NAMED, (GSIMapKey)(id)name); // 以name为key，从named表中获取对应的mapTable
        if (n == 0) // 没有node,创建node
        {
            m = mapNew(TABLE); // 先取缓存，如果缓存没有则新建一个map
            /*
             * As this is the first observation for the given name, we take a
             * copy of the name so it cannot be mutated while in the map.
             */
            name = [name copyWithZone: NSDefaultMallocZone()];
            GSIMapAddPair(NAMED, (GSIMapKey)(id)name, (GSIMapVal)(void*)m); // 将key和value对存储在maptable中
            GS_CONSUMED(name)
        }
        else
        {
            m = (GSIMapTable)n->value.ptr; // 存在则把值取出来 赋值给m
        }
        
        /*
         * Add the observation to the list for the correct object.
         */
        //以object为key，从字典m中取出对应的value，其实value被MapNode的结构包装了一层
        n = GSIMapNodeForSimpleKey(m, (GSIMapKey)object);
        if (n == 0) // 不存在，则创建
        {
            o->next = ENDOBS;
            GSIMapAddPair(m, (GSIMapKey)object, (GSIMapVal)o); // key:object和value:o存储在maptable中
        }
        else
        {
            list = (Observation*)n->value.ptr;
            o->next = list->next;
            list->next = o;
        }
    }
    else if (object) // 无name有object
    {
        /*
         1.以object作为key，从nameless字典中取出value，此value是个obs类型的链表
         2.把创建的obs类型对象o存储到链表中
         
         nameless表（maptable）   |----------key（object）
         |----------value（Observation对象-链表）
         
         只存在object时存储只有一层，那就是object和obs对象之间的映射
         */
        //以object为key，从nameless字典中取出对应的value，value是个链表结构
        n = GSIMapNodeForSimpleKey(NAMELESS, (GSIMapKey)object);
        if (n == 0) // 不存在则新建链表，并存到map中
        {
            o->next = ENDOBS;
            GSIMapAddPair(NAMELESS, (GSIMapKey)object, (GSIMapVal)o);
        }
        else
        {
            list = (Observation*)n->value.ptr;
            o->next = list->next;
            list->next = o;
        }
    }
    else // name 和 object 都为空 则存储到wildcard链表中
    {
        o->next = WILDCARD;
        WILDCARD = o;
    }
    
    unlockNCTable(TABLE);
}

```

### NCTbl结构

```
typedef struct NCTbl {
  Observation        *wildcard;    /* Get ALL messages.链表结构，保存既没有name也没有object的通知         */
  GSIMapTable        nameless;    /* Get messages for any name. 存储没有name但是有object的通知    */
  GSIMapTable        named;        /* Getting named messages only. 存储带有name的通知，不管有没有object    */
  unsigned        lockCount;    /* Count recursive operations.    */
  NSRecursiveLock    *_lock;        /* Lock out other threads.    */
  Observation        *freeList;
  Observation        **chunks;
  unsigned        numChunks;
  GSIMapTable        cache[CACHESIZE];
  unsigned short    chunkIndex;
  unsigned short    cacheIndex;
} NCTable;

```


可以看到NSNotificationCenter流程先是做了一个前置判空的容错，通过obsNew()初始化了NCTbl数据结构。
    NCTbl主要有wildcard、named、nameless三种不同的变量。分别代表传入参数的三种情况。
    
### 1.named表（存在name，无论是否存在object）
```

    named表（maptable） |----------key（name）
                       |----------value（maptable） |----------key（object）
                                                       |----------value（Observation对象）
```

`GSIMapNodeForKey(NAMED, (GSIMapKey)(id)name)`以name为key查找链表中的_GSIMapNode类型的node。不存在node，就创建map和node。将key:name和value:map对存储在maptable:named中，`GSIMapAddPair(NAMED, (GSIMapKey)(id)name, (GSIMapVal)(void*)m)`; 存在node，通过指针取出maptable:named以object为key，从字典m中取出对应的value，其实value被MapNode的结构包装了一层。不存在object，则创建一个node，`GSIMapAddPair(m, (GSIMapKey)object, (GSIMapVal)o); // key:object和value:o存储在maptable中`
    存在object，则遍历maptable:named链表找到对应的obs

### 2.nameless表（无name有object）
```
nameless表（maptable）   |----------key（object）
 |----------value（Observation对象-链表）
```
`n = GSIMapNodeForSimpleKey(NAMELESS, (GSIMapKey)object);`以object作为key，从nameless字典中取出value，此value是个obs类型的链表。不存在，则新建链表并存储到map中`GSIMapAddPair(NAMELESS,(GSIMapKey)object,(GSIMapVal)o);`。存在，则找到对应的obs

### 3.wildcard表（name 和 object 都为空）

```
- (id) addObserverForName: (NSString *)name  object: (id)object queue: (NSOperationQueue *)queue usingBlock: (GSNotificationBlock)block {
    // 创建一个GSNotificationObserver类型的对象observer，并把queue和block保存下来
    // 调用接口1进行通知的注册
    // 接收到通知时会响应observer的didReceiveNotification:方法，然后在didReceiveNotification:中把block抛给指定的queue去执行
    GSNotificationObserver *observer = 
        [[GSNotificationObserver alloc] initWithQueue: queue block: block];

    [self addObserver: observer  selector: @selector(didReceiveNotification:)  name: name  object: object];

    return observer;
}

```

## 发送通知：
```
- (void) postNotification: (NSNotification*)notification {
  if (notification == nil) {
      [NSException raise: NSInvalidArgumentException format: @"Tried to post a nil notification."];
    }
  [self _postAndRelease: RETAIN(notification)];
}

- (void) postNotificationName: (NSString*)name object: (id)object {
  [self postNotificationName: name object: object userInfo: nil];
}

- (void) postNotificationName: (NSString*)name object: (id)object userInfo: (NSDictionary*)info {
    // 构造一个GSNotification对象， GSNotification继承了NSNotification
  GSNotification    *notification;

  notification = (id)NSAllocateObject(concrete, 0, NSDefaultMallocZone());
  notification->_name = [name copyWithZone: [self zone]];
  notification->_object = [object retain];
  notification->_info = [info retain];
    
    // 进行发送操作
  [self _postAndRelease: notification];
}
```

核心可见都是通过调用_postAndRelease:方法

```
- (void) _postAndRelease: (NSNotification*)notification
{
  Observation    *o;
  unsigned    count;
  NSString    *name = [notification name];
  id        object;
  GSIMapNode    n;
  GSIMapTable    m;
  GSIArrayItem    i[64];
  GSIArray_t    b;
  GSIArray    a = &b;

    //step1: 从named、nameless、wildcard表中查找对应的通知
  if (name == nil)
    {
      RELEASE(notification);
      [NSException raise: NSInvalidArgumentException
          format: @"Tried to post a notification with no name."];
    }
  object = [notification object];

  /*
   * Lock the table of observations while we traverse it.
   *
   * The table of observations contains weak pointers which are zeroed when
   * the observers get garbage collected.  So to avoid consistency problems
   * we disable gc while we copy all the observations we are interested in.
   * We use scanned memory in the array in the case where there are more
   * than the 64 observers we allowed room for on the stack.
   */
  GSIArrayInitWithZoneAndStaticCapacity(a, _zone, 64, i);
  lockNCTable(TABLE);
    
    // 1.通过name && object 查找到所有的obs对象(保存了observer和sel)，放到数组中
    // 2.通过performSelector：逐一调用sel，这是个同步操作
    // 3.释放notification对象

    // 查找顺序1.先找没有name和没有object（WILDCARD类型）
  /*
   * Find all the observers that specified neither NAME nor OBJECT.
   */
  for (o = WILDCARD = purgeCollected(WILDCARD); o != ENDOBS; o = o->next)
    {
      GSIArrayAddItem(a, (GSIArrayItem)o);
    }
    
    
    // 查找顺序2.找只有object的（NAMELESS类型）
  /*
   * Find the observers that specified OBJECT, but didn't specify NAME.
   */
  if (object) //
    {
      n = GSIMapNodeForSimpleKey(NAMELESS, (GSIMapKey)object);
      if (n != 0)
    {
      o = purgeCollectedFromMapNode(NAMELESS, n);
      while (o != ENDOBS)
        {
          GSIArrayAddItem(a, (GSIArrayItem)o);
          o = o->next;
        }
    }
    }

    // 查找顺序3.找name和object都有的maptable（NAMED类型）
  /*
   * Find the observers of NAME, except those observers with a non-nil OBJECT
   * that doesn't match the notification's OBJECT).
   */
  if (name)
    {
      n = GSIMapNodeForKey(NAMED, (GSIMapKey)((id)name));
      if (n)
    {
      m = (GSIMapTable)n->value.ptr;
    }
      else
    {
      m = 0;
    }
      if (m != 0)
    {
      /*
       * First, observers with a matching object.
       */
      n = GSIMapNodeForSimpleKey(m, (GSIMapKey)object);
      if (n != 0)
        {
          o = purgeCollectedFromMapNode(m, n);
          while (o != ENDOBS)
        {
          GSIArrayAddItem(a, (GSIArrayItem)o);
          o = o->next;
        }
        }

      if (object != nil)
        {
          /*
           * Now observers with a nil object.
           */
            n = GSIMapNodeForSimpleKey(m, (GSIMapKey)(id)nil);
          if (n != 0)
        {
              o = purgeCollectedFromMapNode(m, n);
          while (o != ENDOBS)
            {
              GSIArrayAddItem(a, (GSIArrayItem)o);
              o = o->next;
            }
        }
        }
    }
    }

  /* Finished with the table ... we can unlock it,
   */
  unlockNCTable(TABLE);

  /*
   * Now send all the notifications.
   */
  count = GSIArrayCount(a);
  while (count-- > 0)
    {
      o = GSIArrayItemAtIndex(a, count).ext;
      if (o->next != 0)
    {
          NS_DURING
            {
                //step2：执行发送，即调用performSelector执行响应方法，从这里可以看出是同步的
              [o->observer performSelector: o->selector
                                withObject: notification];
            }
          NS_HANDLER
            {
          BOOL    logged;

          /* Try to report the notification along with the exception,
           * but if there's a problem with the notification itself,
           * we just log the exception.
           */
          NS_DURING
        NSLog(@"Problem posting %@: %@", notification, localException);
        logged = YES;
          NS_HANDLER
        logged = NO;
          NS_ENDHANDLER
            if (NO == logged)
        { 
          NSLog(@"Problem posting notification: %@", localException);
        }  
            }
          NS_ENDHANDLER
    }
    }
  lockNCTable(TABLE);
  GSIArrayEmpty(a);
  unlockNCTable(TABLE);

    //step3: 释放资源
  RELEASE(notification);
}
```

-(void)_postAndRelease: 主要流程
1. 遍历按照顺序分别wildcard，nameless，named,，找到所有的obs
2. 遍历所有的obs（Observation），obs->observer执行对应的obs->selector
3. 释放所有的obs和传入的notification对象（所以传入的参数是被强引用的）

wildcard表中，依次根据链表通过`GSIArrayAddItem(a,(GSIArrayItem)o);`加入到二维数组`GSIArray_t`中`nameless`表中，以object作为key，查找`n = GSIMapNodeForSimpleKey(NAMELESS,(GSIMapKey)object);`然后遍历通过`GSIArrayAddItem(a,(GSIArrayItem)o);`加入到二维数组`GSIArray_t`中named表中，通过name先找到value，value是一个maptable。然后通过object作为key，通过`n = GSIMapNodeForSimpleKey(m,(GSIMapKey)(id)nil);`查找，然后遍历通过`GSIArrayAddItem(a,(GSIArrayItem)o);`加入到二维数组`GSIArray_t`中三个表查完后，遍历二维数组，执行`[o->observer performSelector: o->selector withObject: notification]`;


## 删除通知：
删除通知的方式，通过observer，name，object依次删除三个表wildcard、nameless、named中的observer，并且将找到的重复的通知清空后的表重新赋值给自己。查找的逻辑与上面发送通知差不多。


## NSNotificationQueue 异步通知：
```
@interface NSNotificationQueue : NSObject {
#if    GS_EXPOSE(NSNotificationQueue)
@public
  NSNotificationCenter            *_center;
  struct _NSNotificationQueueList    *_asapQueue; // 尽快发送
  struct _NSNotificationQueueList    *_idleQueue; // runloop空闲时候发
  NSZone                *_zone;
#endif
}
```

主要是根据两个参数postingStyle、coalesceMask来判断发送的时机和是否合并。以及三种模式下的发送队列_asapQueue（尽快发送）、_idleQueue（runloop空闲时候发）_center（同步线程立即发送）

### 核心点：
- 两个发送队列
- 队列是双向链表实现
- 当postingStyle值是立即发送时，调用的是NSNotificationCenter进行发送的，所以NSNotificationQueue还是依赖NSNotificationCenter进行发送

```
- (void) enqueueNotification: (NSNotification*)notification
        postingStyle: (NSPostingStyle)postingStyle
        coalesceMask: (NSUInteger)coalesceMask
            forModes: (NSArray*)modes
{
  if (modes == nil)
    {
      modes = defaultMode;
    }
    // 判断是否需要合并通知
  if (coalesceMask != NSNotificationNoCoalescing) // 合并发送->删除_asapQueue（尽快发送队列）和_idleQueue（runloop空闲时发送队列）里面的notification
    {
      [self dequeueNotificationsMatching: notification
                coalesceMask: coalesceMask];
    }
  switch (postingStyle)
    {
      case NSPostNow:
    {
      NSString    *mode;

      mode = [[NSRunLoop currentRunLoop] currentMode];
      if (mode == nil || [modes indexOfObject: mode] != NSNotFound)
        {
            // 如果是立马发送，则调用NSNotificationCenter进行发送
          [_center postNotification: notification];
        }
    }
    break;

      case NSPostASAP:
            // 添加到_asapQueue队列，等待发送
    add_to_queue(_asapQueue, notification, modes, _zone);
    break;

      case NSPostWhenIdle:
            // 添加到_idleQueue队列，等待发送
    add_to_queue(_idleQueue, notification, modes, _zone);
    break;
    }
}
```

### 两个维度来进行notification的发送:
- 一个是是否合并、
- 一个是发送时机。在发送队列中添加通知

#### 是否合并：
1. 合并name和object，根据name和object判断相同，分别删除_asapQueue、_idleQueue里面的NSNotificationQueueRegistration、从后往前删是因为这样不会漏删。
1. 合并name，根据name判断相同，分别删除_asapQueue、_idleQueue里面的NSNotificationQueueRegistration
1. 合并object，根据objecte判断相同，分别删除_asapQueue、_idleQueue里面的NSNotificationQueueRegistration

#### 发送时机：
1. NSPostNow-立即发送，执行postNotification:
2. NSPostASAP-尽快发送，添加到_asapQueue队列，等待发送
3. NSPostWhenIdle-闲时发送，添加到_idleQueue，等待发送

### 最终发送通知：
这里会涉及到NSRunLoop的相关代码，不赘述。
1. runloop触发某个时机，调用GSPrivateNotifyASAP()和GSPrivateNotifyIdle()方法，这两个方法最终都调用了notify()方法
2. notify()所做的事情就是调用NSNotificationCenter的postNotification:进行发送通知

### 主线程响应通知
异步线程发送通知则响应函数也是在异步线程，如果执行UI刷新相关的话就会出问题，那么如何保证在主线程响应通知呢？
其实也是比较常见的问题了，基本上解决方式如下几种：
- 使用addObserverForName: object: queue: usingBlock方法注册通知，指定在mainqueue上响应block
- 在主线程注册一个machPort，它是用来做线程通信的，当在异步线程收到通知，然后给machPort发送消息，这样肯定是在主线程处理的，具体用法去网上资料很多，苹果官网也有


