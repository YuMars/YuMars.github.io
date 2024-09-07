---
title: "不谈Mach O都项目中实实在在的快速瘦身"
date: 2023-04-04T14:22:20+08:00
draft: false
---


# IPA瘦身

## 包体瘦身的前提，缘由

苹果近几年销售的手机存储空间配置越来越大，同时也在2017年9月和最近2019年6月分别修改了在移动流量下载上线为150M和200M。但是包体瘦身对于用户来说，可以为用户节省手机存储空间，同时也减少了下载app的时间。如果包体太大上架到app store，用户没办法在非Wifi环境下下载app。是在WWDC中有专门的App Thinning内容。

随着app的版本迭代，包体大小从80M上升到了超过150M，导致了用户无法在非Wifi环境下下载app（当时app store移动流量下ipa下载上限是150M），影响了用户的使用，所以专门调研了ipa瘦身的相关问题，实践了很多办法，将ipa的大小降低到了差不多100M上下。

> ipa瘦身是一个持续的过程

## 急于求成之外的情况下最好能了解ipa包的构成-Mach-O结构

下面我们快速解决ipa包体积大小问题。项目中Product安装包的解压后的目录结构有资源图片和Mach-O代码。只要删除任何一项，包体大小都是实打实的能变小的，下面简单粗暴速成来提供解决思路。

### 1.资源（图片）  
    1.优化压缩png（展开说动画资源），删除重复资源，删除无用资源
    图片做压缩（放在resource 和 image.asset的区别）
    2.本地资源放占位图，然后从服务端获取
    3.序列帧动画制作成svga效果，通过下载

### 2.可执行文件瘦身
    1.删除无用的类（多人维护的情况下，appcode）
    2.整理第三方库瘦身（第三方类库，小功能把整个库拿进来）
    3.静态库瘦身（去掉多余的armv7）
    4.删除无用的方法，属性，不用的函数，重复代码
    5.对linkmap做脚本监控
    
### 3.去掉无用的armv7框架

### 4.业务上多变的功能做成H5

### 5.了解了每个设置的意思，个人觉得对于一个普通的app来说可以这样配置这些设置：  

### 6.Build Setting配置项的修改
1. Generate Debug Symbols：DEBUG和RELEASE下均设为YES（和Xcode默认一致）；  
2. Debug Information Level：DEBUG和RELEASE下均设为Compiler default（和Xcode默认一致）；
3. Deployment Postprocessing：DEBUG下设为NO，RELEASE下设为YES，这样RELEASE模式下就可以去除符号缩减app的大小（但是似乎设置为YES后，会牵涉一些和bitcode有关的设置，对于bitcode暂时还不太了解(´･_･`)）；
4. Strip Linked Product：DEBUG下设为NO，RELEASE下设为YES，用于RELEASE模式下缩减app的大小；
5. Strip Style：DEBUG和RELEASE下均设为All Symbols（和Xcode默认一致）；
6. Strip Debug Symbols During Copy：DEBUG下设为NO，RELEASE下设为YES；
7. Debug Information Format：DEBUG下设为DWARF，RELEASE下设为DWARF with dSYM File，dSYM文件需要用于符号化crash log（和Xcode默认一致）；
