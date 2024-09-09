---
title: "转生之KVO&KVO再了解一次"
date: 2023-09-29T23:58:27+08:00
draft: false
---

# KVO

KVO全称Key-Value-Observing,俗称"键值监听"，可以用于监听某个对象属性值的改变

`使用KVO监听后,会生成NSKVONotifying_类名的一个类,是一个继承自原先类的子类，由runtime在运行过程中动态生成的一个类`
重写了`class`,`dealloc`, `_isKVO`三个方法

- `NSKVONotifying_Person`是`Person`的一个子类 
- `[person setAge:]` 调用了`[NSKVONotifying_Person setage:]`,调用了`_NSSetIntValueAndNotify`

> - p (IMP)+内存地址 可以打印调用的方法名

`_NSSet*ValueAndNotify`的内部实现:
- [self willChangeValueForKey:@""];
- [super setValue:];
- [self didChangeValueForKey:@""];

Q: KVO的本质是什么
A: 利用Runtime 
- 1.API动态生成一个子类，并且让instance对象的isa指向这个全新的子类，子类NSKVONotifying_类名
- 2.当修改instance对象的属性时，会调用Foundation的_NSSet*ValueaAndNofity函数
- _NSSet*ValueAndNotify:{ 1.willChangeValueForKey: 2.[super setValue:] 3.didChangeValueForKey:} 
- 3.内部调用observeValueForKeyPath:ofObject:change:context:
    
Q: 如何手动触发KVO
A: 
- [self willChangeValueForKey:@""] 
- [self didChangeValueForKey:@""]

# KVC

KVC全称Key-Value Coding，俗称"键值编码"，可以通过一个key来访问某个属性
`setValue:forKey:`
`setValue:forKeyPath:`
`valueForKey:`
`valueForKeyPath:`

[self.person setValue:value forKeyPath:@"cat.age"]

Q: 通过KVC修改属性会触发KVO吗
A: 会触发(通过KVC修改成员变量也会触发)

## setValue顺序
setValue:forKey:顺序
1.setKey: _setKey:顺序查找方法
2.accessInstanceVariablesDirectly(是否允许直接访问成员变量)  
->第一种没找的情况
    2.1 设置No后，会调用setValue:forUndefinedKey:
    2.2 设置YES后，按照_key,_isKey,key,isKey的顺序查找成员变量，失败后会调用setValue:forUndefinedKey:
> {
> `setKey:`
> `_setKey:`
> `_key`
> `_isKey`
> `key`
> `isKey`
> }

## getValue顺序
valueForKey:顺序
1.getKey,key,isKey,_key顺序查找方法
2.accessInstanceVariablesDirectly(是否允许直接访问成员变量) -> 第一种没找到的情况
    2.1 设置NO后，会调用valueForUndefinedKey:
    2.2 设置YES后，按照_key, _isKey, key, isKey的顺序查找成员变量
> 
> {
> `getKey:`
> `key:`
> `isKey:`
> `_key:`
> `_key`
> `_isKey`
> `key`
> `isKey`
> }

## Q: KVC赋值的过程是什么？原理是什么？
A: 上面的顺序
getter流程： getKey:  key:  isKey:  _key: _key  _isKey  key  isKey 顺序查找
setter流程：setKey:  _setKey:  _key  _isKey  key  isKey 顺序查找






## Q：哪些情况下使用KVO会崩溃，怎么保护崩溃

1.dealloc没有移除KVO观察者。解决方案：创建一个中间对象，将其最为某个属性的观察者，然后dealloc的时候去除观察者。调用者是持有中间对象，调用者释放，中间对象也就释放，dealloc也就移除观察者
2.多次重复移除同一属性的观察者，或者移除了未添加过的观察者
3.被观察者提前被释放，被观察者在dealloc时仍然注册着KVO，导致崩溃。例如：被观察者是局部变量，weak
4.添加观察者，但是未实现+observeValueForKeyPath:ofObject:change:context:方法，导致崩溃
5.添加或者移除时，keyPath:nil ，导致崩溃



## 5.KVO的优缺点

优点：
1.运动了设计模式：观察者模式
2.支持多个观察者观察同一属性，或者同一观察者监听不同属性
3.不需要实现属性变化的通知发送
4.对创建的对象的状态改变做出响应，不要改变对象的实现（比如SDK对象）
5.能够提供观察的属性新值和旧值
6.可以用key path来观察属性，所以可以观察嵌套对象
7.完成了对观察对象的抽象，因为不需要额外的代码来允许观察值能够被观察

缺点：
1.观察的属性键值硬编码（字符串），编译器无法发出警告
2.允许一对多观察属性，回调方法中可能有很多分支情况
