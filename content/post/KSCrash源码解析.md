---
title: "KSCrash源码解析"
date: 2024-01-18T14:18:57+08:00
draft: false
---

[TOC]

# KSCrash
KSCrash 是 iOS 上一个知名的crash收集框架。大部分第三方crash库，都是基于KSCrash修改的。

# KSCrash的基本功能
KSCrash的功能是选择好需要收集的信息和上报方式后，当项目中崩溃发生的时候，KSCrash会收集所有崩溃产生时的crash信息记录在crashContext上下文中，然后将crashContext信息解析并且通过对crashContext上下文的解析降崩溃发生的崩溃信息解析并记录到磁盘文件中，在下次打开App上报内容。

## crash时收集信息
Mach层异常
BSD层POSIX signal异常
App层NSException异常
C++异常
主线程死锁、
主动上报的异常

## app二进制文件解析的信息
mach-O 二进制镜像
mach-O header
mach-O segment
crash时所有线程信息
crash对应thread线程信息
symbolicate符号化信息
register寄存器信息
调用的stack堆栈信息
寄存器或者堆栈对应的对象信息（野指针信息）

# 总体结构

> ├── Installations（启动器）
> ├── Recording （记录crash）
> │   ├── Monitors
> │   └── Tools
> ├── Reporting （上报crash）
> │   ├── Filters
> │   │   └── Tools
> │   ├── Sinks
> │   └── Tools
> ├── llvm
> │   ├── ADT
> │   ├── Config
> │   └── Support
> └── swift
>     └── Basic
    
可以看到总共分为三大部分：
- Crash Recording
- Crash Reporting
- Installation

其中 Installation 用来启动 KSCrash，并且指定了 Crash 收集的方式。Crash收集方式可以包括：邮件发送，向指定服务器发送，向专门的 Crash 收集服务器发送等方式。每种方式对应于一个实现类：
> ├── KSCrashInstallation+Alert.h
> ├── KSCrashInstallation+Alert.m
> ├── KSCrashInstallation+Private.h
> ├── KSCrashInstallation.h
> ├── KSCrashInstallation.m
> ├── KSCrashInstallationConsole.h
> ├── KSCrashInstallationConsole.m
> ├── KSCrashInstallationEmail.h
> ├── KSCrashInstallationEmail.m
> ├── KSCrashInstallationQuincyHockey.h
> ├── KSCrashInstallationQuincyHockey.m
> ├── KSCrashInstallationStandard.h
> ├── KSCrashInstallationStandard.m
> ├── KSCrashInstallationVictory.h
> └── KSCrashInstallationVictory.m

Crash Reporting 包含了三个子文件夹，分别是 Filter，Sink 和 Tools，主要用来上报 Crash。

> ├── Filters
> │   ├── KSCrashReportFilter.h
> │   ├── KSCrashReportFilterAlert.h
> │   ├── KSCrashReportFilterAlert.m
> │   ├── KSCrashReportFilterAppleFmt.h
> │   ├── KSCrashReportFilterAppleFmt.m
> │   ├── KSCrashReportFilterBasic.h
> │   ├── KSCrashReportFilterBasic.m
> │   ├── KSCrashReportFilterGZip.h
> │   ├── KSCrashReportFilterGZip.m
> │   ├── KSCrashReportFilterJSON.h
> │   ├── KSCrashReportFilterJSON.m
> │   ├── KSCrashReportFilterSets.h
> │   ├── KSCrashReportFilterSets.m
> │   ├── KSCrashReportFilterStringify.h
> │   ├── KSCrashReportFilterStringify.m
> │   └── Tools
> ├── Sinks
> │   ├── KSCrashReportSinkConsole.h
> │   ├── KSCrashReportSinkConsole.m
> │   ├── KSCrashReportSinkEMail.h
> │   ├── KSCrashReportSinkEMail.m
> │   ├── KSCrashReportSinkQuincyHockey.h
> │   ├── KSCrashReportSinkQuincyHockey.m
> │   ├── KSCrashReportSinkStandard.h
> │   ├── KSCrashReportSinkStandard.m
> │   ├── KSCrashReportSinkVictory.h
> │   └── KSCrashReportSinkVictory.m
> └── Tools

其中 Filter 包含了将存储在设备上的 Crash 信息的 NSData 转为 NSString，JSON，GZip 等几种方式的实现类。
Sink 则是不同发送方式的真正的处理者，和 Installation 中的几种收集方式一一对应。
Tools 是一些工具类，以及发送请求相关的类的合集。
最关键的是 Crash Recording，它包含了捕捉各种类型 crash 的方式。


# 基本使用
以官方 Demo 为例
```
@implementation AppDelegate

- (BOOL) application:(__unused UIApplication *) application
didFinishLaunchingWithOptions:(__unused NSDictionary *) launchOptions {
    [self installCrashHandler];    
    return YES;
}

- (void) installCrashHandler {
    /// 👇是各种 crash 收集方式，选择一种使用
//    KSCrashInstallation* installation = [self makeStandardInstallation];
    KSCrashInstallation* installation = [self makeEmailInstallation];
//    KSCrashInstallation* installation = [self makeHockeyInstallation];
//    KSCrashInstallation* installation = [self makeQuincyInstallation];
//    KSCrashInstallation *installation = [self makeVictoryInstallation];

    /// 注册 crash handler
    [installation install];

    /// 发送所有的 crash 日志
    [installation sendAllReportsWithCompletion:^(NSArray* reports, BOOL completed, NSError* error) {
         if(completed) {
             NSLog(@"Sent %d reports", (int)[reports count]);
         } else {
             NSLog(@"Failed to send reports: %@", error);
         }
     }];
}

- (KSCrashInstallation*) makeEmailInstallation {
    NSString* emailAddress = @"your@email.here";
    
    KSCrashInstallationEmail* email = [KSCrashInstallationEmail sharedInstance];
    email.recipients = @[emailAddress];
    email.subject = @"Crash Report"
    email.message = @"This is a crash report";
    email.filenameFmt = @"crash-report-%d.txt.gz";
    
    [email addConditionalAlertWithTitle:@"Crash Detected"
                                message:@"The app crashed last time it was launched. Send a crash report?"
                              yesAnswer:@"Sure!"
                               noAnswer:@"No thanks"];
    
    // Uncomment to send Apple style reports instead of JSON.
    [email setReportStyle:KSCrashEmailReportStyleApple useDefaultFilenameFormat:YES];

    return email;
}

@end
```

如上面代码所示，在 AppDelegate 中注册 KSCrash。在注册 handler 期间，通过工厂模式选择一个上传方式的 installation 进行实例化。不同的 installation 会有不同的处理，比如上面的 email 上传方式，那就需要提供邮箱地址。

不同的 installation有不同的上传日志的方式，但是它们注册监听的方式都是一样的。它们都继承于基类KSCrashInstallation，在基类中有统一的 install 方法。

# Crash监测
在 install 的过程中，installation创建了一个单例对象KSCrash，并且调用它的 install

```
// KSCrash.m
/// 开启监控
- (BOOL) install {
    _monitoring = kscrash_install(self.bundleName.UTF8String,
                                          self.basePath.UTF8String);
    if(self.monitoring == 0) {
        return false;
    }
    return true;
}
```

它调用了c方法kscrash_install，这个方法在 KSCrashC.c 中：

```
/// 真正开始的方法, 创建一个 monitor
KSCrashMonitorType kscrash_install(const char* appName, const char* const installPath) {
    KSLOG_DEBUG("Installing crash reporter.");

    if(g_installed) { // 已经初始化过，直接返回监视器类型
        KSLOG_DEBUG("Crash reporter already installed.");
        return g_monitoring;
    }
    g_installed = 1;

    // 设置路径
    {
        char path[KSFU_MAX_PATH_LENGTH];
        snprintf(path, sizeof(path), "%s/Reports", installPath);
        ksfu_makePath(path);
        kscrs_initialize(appName, path);
        
        snprintf(path, sizeof(path), "%s/Data", installPath);
        ksfu_makePath(path);
        snprintf(path, sizeof(path), "%s/Data/CrashState.json", installPath);
        kscrashstate_initialize(path);
        
        snprintf(g_consoleLogPath, sizeof(g_consoleLogPath), "%s/Data/ConsoleLog.txt", installPath);
        if(g_shouldPrintPreviousLog)
        {
            printPreviousLog(g_consoleLogPath);
        }
        kslog_setLogFilename(g_consoleLogPath, true);
        
    }
    
    // 初始化线程
    ksccd_init(60);

    // monitor设置回调
    kscm_setEventCallback(onCrash);
    // monitor类型
    KSCrashMonitorType monitors = kscrash_setMonitoring(g_monitoring);

    KSLOG_DEBUG("Installation complete.");

    // 初始化完成前通知
    notifyOfBeforeInstallationState();

    return monitors;
}
```

oncrash是设置的回调部分。monitor主要设置检测的部分
```
/// 设置监测
KSCrashMonitorType kscrash_setMonitoring(KSCrashMonitorType monitors)
{
    g_monitoring = monitors;
    
    if(g_installed)
    {
        // 激活Monitor并且返回监视器类型
        kscm_setActiveMonitors(monitors);
        return kscm_getActiveMonitors();
    }
    // Return what we will be monitoring in future.
    return g_monitoring;
}
```

由于前面设置了 g_installed ，因此，这里就会通过 kscm_setActiveMonitors() 这个方法激活 monitor。这是整个激活步骤的核心方法：

```
/// 设置监测状态激活
void kscm_setActiveMonitors(KSCrashMonitorType monitorTypes)
{
    // 调试中的应用只记录 Mach Signal C++ OC 的 crash
    if(ksdebug_isBeingTraced() && (monitorTypes & KSCrashMonitorTypeDebuggerUnsafe))
    {
        static bool hasWarned = false;
        if(!hasWarned)
        {
            hasWarned = true;
            KSLOGBASIC_WARN("    ************************ Crash Handler Notice ************************");
            KSLOGBASIC_WARN("    *     App is running in a debugger. Masking out unsafe monitors.     *");
            KSLOGBASIC_WARN("    * This means that most crashes WILL NOT BE RECORDED while debugging! *");
            KSLOGBASIC_WARN("    **********************************************************************");
        }
        monitorTypes &= KSCrashMonitorTypeDebuggerSafe;
    }
    
    /// 是否需要线程安全，只有 Mach 和 Signal 异常是线程安全的
    if(g_requiresAsyncSafety && (monitorTypes & KSCrashMonitorTypeAsyncUnsafe))
    {
        KSLOG_DEBUG("Async-safe environment detected. Masking out unsafe monitors.");
        monitorTypes &= KSCrashMonitorTypeAsyncSafe;
    }

    KSLOG_DEBUG("Changing active monitors from 0x%x tp 0x%x.", g_activeMonitors, monitorTypes);

    KSCrashMonitorType activeMonitors = KSCrashMonitorTypeNone;
    for(int i = 0; i < g_monitorsCount; i++)
    {
        Monitor* monitor = &g_monitors[i];
        bool isEnabled = monitor->monitorType & monitorTypes;
        // 开启Monitor
        setMonitorEnabled(monitor, isEnabled);
        if(isMonitorEnabled(monitor)) // 已经启动的，按位或
        {
            activeMonitors |= monitor->monitorType;
        } /// 还未启动的，按位与再取返 启动
        else
        {
            activeMonitors &= ~monitor->monitorType;
        }
    }

    KSLOG_DEBUG("Active monitors are now 0x%x.", activeMonitors);
    g_activeMonitors = activeMonitors;
}
```

外部传入的 g_monitoring 默认值为 KSCrashMonitorTypeProductionSafeMinimal，它的定义如下：
```
#define KSCrashMonitorTypeProductionSafe (KSCrashMonitorTypeAll & (~KSCrashMonitorTypeExperimental))
#define KSCrashMonitorTypeProductionSafeMinimal (KSCrashMonitorTypeProductionSafe & (~KSCrashMonitorTypeOptional))
```

主要的检测类型KSCrashMonitorType：
```
typedef enum
{
    /* 捕获并报告 Mach 异常。*/
    KSCrashMonitorTypeMachException      = 0x01,
    
    /* 捕获并报告 POSIX 信号。*/
    KSCrashMonitorTypeSignal             = 0x02,
    
    /* 捕获并报告 C++ 异常。
     * 注意：这会稍微减慢异常处理速度。
     */
    KSCrashMonitorTypeCPPException       = 0x04,
    
    /* 捕获并报告 NSException 异常。*/
    KSCrashMonitorTypeNSException        = 0x08,
    
    /* 检测并报告主线程的死锁。*/
    KSCrashMonitorTypeMainThreadDeadlock = 0x10,
    
    /* 接受并报告用户生成的异常。*/
    KSCrashMonitorTypeUserReported       = 0x20,
    
    /* 跟踪并注入系统信息。*/
    KSCrashMonitorTypeSystem             = 0x40,
    
    /* 跟踪并注入应用程序状态。*/
    KSCrashMonitorTypeApplicationState   = 0x80,
    
    /* 跟踪僵尸对象，并注入最后一个僵尸 NSException 异常。*/
    KSCrashMonitorTypeZombie             = 0x100,
} KSCrashMonitorType;
```

可以看到，all 代表的就是所有异常方式：

1. Mach 异常
2. Signal 异常
3. C++ 异常
4. OC 异常
5. 死锁
6. 用户抛出的异常


设置 monitor 的时候，会判断是否在被调试，这是通过 sysctl 获取进程的信息的方式来进行判断的：
```
/// 是否在被调试
bool ksdebug_isBeingTraced(void) {
    /// 查询进程信息结果的结构体
    struct kinfo_proc procInfo;
    size_t structSize = sizeof(procInfo);
    int mib[] = {CTL_KERN, KERN_PROC, KERN_PROC_PID, getpid()};
    
    /// 通过 sysctl 获取进程信息
    if(sysctl(mib, sizeof(mib)/sizeof(*mib), &procInfo, &structSize, NULL, 0) != 0)
    {
        KSLOG_ERROR("sysctl: %s", strerror(errno));
        return false;
    }
    
    /// 通过进程信息判断是否被调试
    return (procInfo.kp_proc.p_flag & P_TRACED) != 0;
}
```

最终通过 for 循环，遍历启动每一个 monitor。启动某个 monitor 的方法如下：
```
/// 启动某个 monitor
static inline void setMonitorEnabled(Monitor* monitor, bool isEnabled)
{
    KSCrashMonitorAPI* api = getAPI(monitor);
    /// 调用各个 monitor 的 enable 方法，启动 monitor
    if(api != NULL && api->setEnabled != NULL)
    {
        api->setEnabled(isEnabled);
    }
}
```

Monitor 是一个结构体，它保存了 Monitor 的类型以及启动 monitor 的方法：

```
/// monitor 的数据结构
typedef struct
{
    KSCrashMonitorType monitorType; // 类型
    KSCrashMonitorAPI* (*getAPI)(void); // monitor的api，是否启动，设置启动，设置上下文
} Monitor;
```

## Mach层异常

### 开启Mach层异常捕获
Mach 异常在 KSCrashMonitor_MachException.c 中被处理外部调用 setEnabled() 开启 Mach 异常的监听：
```
static void setEnabled(bool isEnabled)
{
    if(isEnabled != g_isEnabled)
    {
        g_isEnabled = isEnabled;
        if(isEnabled) // 变成开启状态->设置一级事件ID和二级事件ID（EventID）
        {
            ksid_generate(g_primaryEventID);
            ksid_generate(g_secondaryEventID);
            if(!installExceptionHandler())
            {
                return;
            }
        }
        else // 不需要开启
        {
            uninstallExceptionHandler();
        }
    }
}
```

其中包含两个 c 函数，installExceptionHandler() 和 uninstallExceptionHandler() 分别用于开启和关闭监听。
这里设置了两个事件ID，一级事件记录的是crash发生时候的信息，二级事件记录的是crash发生后又有crash发生的信息


> 在说如何捕捉 Mach 异常前，先补充什么是Mach，Mach是一种操作系统微内核，是许多新操作系统的设计基础。iOS操作系统框架基于Drawin，其核心部分是Mach微内核和BSD（Berkeley Software Distribution）子系统。Mach主要负责进程和线程管理，内存管理，进程间通信，属于硬件层面。BSD主要负责文件系统，网络协议栈、进程管理属于系统层面。
> 
> 概括来说一个Mach微内核有多个Task进行进程管理，task可以拥有多个thread，thread之间共享task的上下文资源，task之间以消息队列的方式通过port进程间通信（IPC），通信内容为message单元
> 
> Mach微内核中的基础概念：
> Tasks，拥有一组系统资源的对象，允许”thread”在其中执行。每一个BSD 进程都在底层关联了一个Mach 任务对象(BSD 层在 Mach 之上，提供了更高层次的功能，比如 UNIX 进程模型，POSIX 线程模型等)
> Threads，执行的基本单位，拥有task的上下文，并共享其资源。
> Ports，task之间通讯的一组受保护的消息队列；task可对任何port发送/接收数据。
> Message，有类型的数据对象集合，只可以发送到port。
> 关于什么是微内核？微内核把系统服务，比如文件管理、虚拟内存、设备I/O，单独包装为一个个模块。微内核作为底层使用进程间通信收发消息。这一来大量的内核代码可以转移到用户空间，使内核变得更小。这种方式拓展性强，但是通信有效率损耗。宏内核正相反，把所有系统服务放在一起。
> 以上概括来说，
> 
> UNIX 下，触发到内核态需要通过系统调用触发陷阱。Mach可以通过申请port，然后利用IPC机制向这个port发送消息。
> 
> 陷阱是软中断，是主动触发的，立刻同步处理。异常是当前指令执行出现问题，是被动的，立刻同步处理。中断是外部硬件触发的，是异步的。


**当异常发生时，内核会向当前 task 的某个专门处理异常的 port 发消息，该消息会依次被转为 signal，NSException 抛出。如果我们要在 Mach 层捕获异常，就需要注册自己的 port，来接收这个异常：**

```
/// 创建 mach 捕捉者
static bool installExceptionHandler(void) {
    KSLOG_DEBUG("Installing mach exception handler.");
    
    bool attributes_created = false;
    pthread_attr_t attr;
    
    kern_return_t kr;
    int error;
    
    /// 获取当前进程的 task
    const task_t thisTask = mach_task_self();
    exception_mask_t mask = EXC_MASK_BAD_ACCESS |
    EXC_MASK_BAD_INSTRUCTION |
    EXC_MASK_ARITHMETIC |
    EXC_MASK_SOFTWARE |
    EXC_MASK_BREAKPOINT;
    
    /// 保存之前的异常处理端口到 g_previousExceptionPorts 中
    KSLOG_DEBUG("Backing up original exception ports.");
    kr = task_get_exception_ports(thisTask,
                                  mask,
                                  g_previousExceptionPorts.masks,
                                  &g_previousExceptionPorts.count,
                                  g_previousExceptionPorts.ports,
                                  g_previousExceptionPorts.behaviors,
                                  g_previousExceptionPorts.flavors);
    if(kr != KERN_SUCCESS)
    {
        KSLOG_ERROR("task_get_exception_ports: %s", mach_error_string(kr));
        goto failed;
    }
    
    /// 如果自己的异常处理端口 g_exceptionPort 是空的，那么创建
    if(g_exceptionPort == MACH_PORT_NULL)
    {
        KSLOG_DEBUG("Allocating new port with receive rights.");
        /// 创建新的异常处理端口
        kr = mach_port_allocate(thisTask,
                                MACH_PORT_RIGHT_RECEIVE,
                                &g_exceptionPort);
        if(kr != KERN_SUCCESS)
        {
            KSLOG_ERROR("mach_port_allocate: %s", mach_error_string(kr));
            goto failed;
        }
        
        KSLOG_DEBUG("Adding send rights to port.");
        /// 申请端口权限
        kr = mach_port_insert_right(thisTask,
                                    g_exceptionPort,
                                    g_exceptionPort,
                                    MACH_MSG_TYPE_MAKE_SEND);
        if(kr != KERN_SUCCESS)
        {
            KSLOG_ERROR("mach_port_insert_right: %s", mach_error_string(kr));
            goto failed;
        }
    }
    
    KSLOG_DEBUG("Installing port as exception handler.");
    /// 把异常设置为自己的 port
    kr = task_set_exception_ports(thisTask,
                                  mask,
                                  g_exceptionPort,
                                  (int)(EXCEPTION_DEFAULT | MACH_EXCEPTION_CODES),
                                  THREAD_STATE_NONE);
    if(kr != KERN_SUCCESS)
    {
        KSLOG_ERROR("task_set_exception_ports: %s", mach_error_string(kr));
        goto failed;
    }
    
    KSLOG_DEBUG("Creating secondary exception thread (suspended).");
    /// 以下整个部分用来创建读取异常端口数据的线程,设置异常端口的处理函数
    pthread_attr_init(&attr);
    attributes_created = true;
    pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);
    /// 创建第二条处理crash的线程，以防止主处理crash的线程crash了
    error = pthread_create(&g_secondaryPThread,
                           &attr,
                           &handleExceptions,
                           (void*)kThreadSecondary);
    if(error != 0)
    {
        KSLOG_ERROR("pthread_create_suspended_np: %s", strerror(error));
        goto failed;
    }
    /// pthread与mach内核层的thread绑定，并且返回mach内存层的线程
    g_secondaryMachThread = pthread_mach_thread_np(g_secondaryPThread);
    /// 保存线程
    ksmc_addReservedThread(g_secondaryMachThread);
    
    KSLOG_DEBUG("Creating primary exception thread.");
    /// 创建主处理crash的线程
    error = pthread_create(&g_primaryPThread,
                           &attr,
                           &handleExceptions,
                           (void*)kThreadPrimary);
    if(error != 0)
    {
        KSLOG_ERROR("pthread_create: %s", strerror(error));
        goto failed;
    }
    pthread_attr_destroy(&attr);
    /// pthread与mach内核层的thread绑定，并且返回mach内存层的线程
    g_primaryMachThread = pthread_mach_thread_np(g_primaryPThread);
    /// 保存线程
    ksmc_addReservedThread(g_primaryMachThread);
    
    KSLOG_DEBUG("Mach exception handler installed.");
    return true;
    
    
failed:
    KSLOG_DEBUG("Failed to install mach exception handler.");
    if(attributes_created)
    {
        pthread_attr_destroy(&attr);
    }
    uninstallExceptionHandler();
    return false;
}
```

代码很长，关键地方做了注释：

1. 通过 mach_task_self() 获取当前进程对应的 task。
2. 通过 task_get_exception_ports() 获取原本处理异常的 port，并保存在 g_previousExceptionPorts 中。
3. 通过 mach_port_allocate() 创建新的异常处理端口。
4. 通过 mach_port_insert_right() 给这个新创建的端口申请权限
5. 通过 task_set_exception_ports() 把异常接收的 port 设置为自己新创建的 port
6. 创建好 port 之后，就可以创建自己的线程去一直读取 port 上的消息了。通过 pthread_create() 创建自己的线程，以及设置好执行的方法为 handleExceptions。

这里会有个疑问，原先系统层面监测异常的端口被新创建的端口替换了，会引起系统无法获取到闪退反馈到应用层面。在后面自己创建的端口收集到异常后会把原始crash的信息，全部重新通过mach_msg发送给。

### 监测到Mach层异常并收集

```
// 等待异常消息，卸载异常端口，记录异常信息，并生成报告。
static void* handleExceptions(void* const userData) {
    MachExceptionMessage exceptionMessage = {{0}};
    MachReplyMessage replyMessage = {{0}};
    char* eventID = g_primaryEventID;
    
    const char* threadName = (const char*) userData;
    pthread_setname_np(threadName);
    if(threadName == kThreadSecondary) // 第二线程
    {
        KSLOG_DEBUG("This is the secondary thread. Suspending."); // 挂起
        thread_suspend((thread_t)ksthread_self());
        eventID = g_secondaryEventID;
    }
    
    /// 等待异常
    for(;;)
    {
        KSLOG_DEBUG("Waiting for mach exception");
        
        // Wait for a message.
        /// 不断调用 mach_msg 接收消息，从异常端口中读取信息到 exceptionMessage 中
        kern_return_t kr = mach_msg(&exceptionMessage.header, // msg头部
                                    MACH_RCV_MSG,            // 接收
                                    0,
                                    sizeof(exceptionMessage),
                                    g_exceptionPort,         // 端口
                                    MACH_MSG_TIMEOUT_NONE,   // 超时时间=0
                                    MACH_PORT_NULL);         // 端口号
        
        /// 上面一直循环读取，直到读取成功了，进入后面的处理函数中
        if(kr == KERN_SUCCESS)
        {
            break;
        }
        
        // Loop and try again on failure.
        KSLOG_ERROR("mach_msg: %s", mach_error_string(kr));
    }
    
    KSLOG_DEBUG("Trapped mach exception code 0x%llx, subcode 0x%llx",
                exceptionMessage.code[0], exceptionMessage.code[1]);
    /// 捕获异常code，异常subcode
    if(g_isEnabled)
    {
        thread_act_array_t threads = NULL;
        mach_msg_type_number_t numThreads = 0;
        /// 暂停所有非当前线程以及白名单线程的线程
        ksmc_suspendEnvironment(&threads, &numThreads);
        g_isHandlingCrash = true; // 记录开始处理异常
        
        /// 捕捉到异常之后清除所有的 monitor
        kscm_notifyFatalExceptionCaptured(true);
        
        KSLOG_DEBUG("Exception handler is installed. Continuing exception handling.");
        
        
        // Switch to the secondary thread if necessary, or uninstall the handler
        // to avoid a death loop.
        /// 捕捉到 exception 后，恢复原来的 port
        if(ksthread_self() == g_primaryMachThread)
        {
            KSLOG_DEBUG("This is the primary exception thread. Activating secondary thread.");
            // TODO: This was put here to avoid a freeze. Does secondary thread ever fire?
            /// 重新存储异常端口
            restoreExceptionPorts();
            if(thread_resume(g_secondaryMachThread) != KERN_SUCCESS)
            {
                KSLOG_DEBUG("Could not activate secondary thread. Restoring original exception ports.");
            }
        }
        else
        {
            KSLOG_DEBUG("This is the secondary exception thread.");// Restoring original exception ports.");
            //            restoreExceptionPorts();
        }
        
        // Fill out crash information
        KSLOG_DEBUG("Fetching machine state.");
        
        /// 设置 crash 信息的 context上下文
        
        /// 创建一个 machineContext 用来保存异常信息
        KSMC_NEW_CONTEXT(machineContext);
        KSCrash_MonitorContext* crashContext = &g_monitorContext;
        crashContext->offendingMachineContext = machineContext;
        /// 创建一个遍历调用栈的 cursor
        kssc_initCursor(&g_stackCursor, NULL, NULL);
        
        /// 把线程信息附加到 machineContext 上
        if(ksmc_getContextForThread(exceptionMessage.thread.name, machineContext, true))
        {
            kssc_initWithMachineContext(&g_stackCursor, KSSC_MAX_STACK_DEPTH, machineContext);
            KSLOG_TRACE("Fault address %p, instruction address %p",
                        kscpu_faultAddress(machineContext), kscpu_instructionAddress(machineContext));
            if(exceptionMessage.exception == EXC_BAD_ACCESS)
            {
                crashContext->faultAddress = kscpu_faultAddress(machineContext);
            }
            else
            {
                crashContext->faultAddress = kscpu_instructionAddress(machineContext);
            }
        }
        
        KSLOG_DEBUG("Filling out context.");
        
        /// 填充上下文
        crashContext->crashType = KSCrashMonitorTypeMachException;
        crashContext->eventID = eventID;
        crashContext->registersAreValid = true;
        crashContext->mach.type = exceptionMessage.exception;
        crashContext->mach.code = exceptionMessage.code[0] & (int64_t)MACH_ERROR_CODE_MASK;
        crashContext->mach.subcode = exceptionMessage.code[1] & (int64_t)MACH_ERROR_CODE_MASK;
        if(crashContext->mach.code == KERN_PROTECTION_FAILURE && crashContext->isStackOverflow)
        {
            // A stack overflow should return KERN_INVALID_ADDRESS, but
            // when a stack blasts through the guard pages at the top of the stack,
            // it generates KERN_PROTECTION_FAILURE. Correct for this.
            crashContext->mach.code = KERN_INVALID_ADDRESS;
        }
        
        /// 将 mach 异常转为对应的 signal
        crashContext->signal.signum = signalForMachException(crashContext->mach.type, crashContext->mach.code);
        crashContext->stackCursor = &g_stackCursor;
        
        /// 回调异常
        /// context 交给 kscrashmonitor 处理
        kscm_handleException(crashContext);
        
        KSLOG_DEBUG("Crash handling complete. Restoring original handlers.");
        /// 结束了捕获恢复所有线程, 存储原始句柄
        g_isHandlingCrash = false;
        ksmc_resumeEnvironment(threads, numThreads);
    }
    
    KSLOG_DEBUG("Replying to mach exception message.");
    // Send a reply saying "I didn't handle this exception".
    /// 发送msg给系统异常处理
    replyMessage.header = exceptionMessage.header;
    replyMessage.NDR = exceptionMessage.NDR;
    replyMessage.returnCode = KERN_FAILURE;
    
    mach_msg(&replyMessage.header,
             MACH_SEND_MSG,
             sizeof(replyMessage),
             0,
             MACH_PORT_NULL,
             MACH_MSG_TIMEOUT_NONE,
             MACH_PORT_NULL);
    
    return NULL;
}

```

整体的监听crash并收集流程：

1. 不停循环通过 mach_msg() 读取 port 中传来的消息。
2. 读取成功后挂起所有线程。
3. 清除所有的 monitor，恢复原来的 port
4. 抓取所有线程的信息保存到 KSMachineContext 结构体中
5. 将各种信息交给 crashContext
6. 把 crashContext 抛出给外部处理方法
7. 恢复所有的线程
8. 通过 mach_msg() 再发出一个消息告知没有处理这个异常


通过方法 task_threads() 获取当前 task 的所有线程，通过 thread_suspend() 方法挂起某个线程：
```
/// 暂停所有线程
void ksmc_suspendEnvironment(__unused thread_act_array_t *suspendedThreads, __unused mach_msg_type_number_t *numSuspendedThreads)
{
#if KSCRASH_HAS_THREADS_API
    KSLOG_DEBUG("Suspending environment.");
    kern_return_t kr;
    const task_t thisTask = mach_task_self();
    const thread_t thisThread = (thread_t)ksthread_self();
    
    /// task_threads获取当前task线程
    if((kr = task_threads(thisTask, suspendedThreads, numSuspendedThreads)) != KERN_SUCCESS)
    {
        KSLOG_ERROR("task_threads: %s", mach_error_string(kr));
        return;
    }
    
    for(mach_msg_type_number_t i = 0; i < *numSuspendedThreads; i++)
    {
        thread_t thread = (*suspendedThreads)[i];
        if(thread != thisThread && !isThreadInList(thread, g_reservedThreads, g_reservedThreadsCount))
        {
            if((kr = thread_suspend(thread)) != KERN_SUCCESS)
            {
                // Record the error and keep going.
                KSLOG_ERROR("thread_suspend (%08x): %s", thread, mach_error_string(kr));
            }
        }
    }
    
    KSLOG_DEBUG("Suspend complete.");
#endif
}
```

通过 task_threads() 方法获取所有的 thread：

```
/// 获取当前 task 所有的 thread，把它保存到 context 中
static inline bool getThreadList(KSMachineContext* context)
{
    const task_t thisTask = mach_task_self();
    KSLOG_DEBUG("Getting thread list");
    kern_return_t kr;
    thread_act_array_t threads;
    mach_msg_type_number_t actualThreadCount;

    if((kr = task_threads(thisTask, &threads, &actualThreadCount)) != KERN_SUCCESS)
    {
        KSLOG_ERROR("task_threads: %s", mach_error_string(kr));
        return false;
    }
    KSLOG_TRACE("Got %d threads", context->threadCount);
    int threadCount = (int)actualThreadCount;
    int maxThreadCount = sizeof(context->allThreads) / sizeof(context->allThreads[0]);
    if(threadCount > maxThreadCount)
    {
        KSLOG_ERROR("Thread count %d is higher than maximum of %d", threadCount, maxThreadCount);
        threadCount = maxThreadCount;
    }
    for(int i = 0; i < threadCount; i++)
    {
        context->allThreads[i] = threads[i];
    }
    context->threadCount = threadCount;

    for(mach_msg_type_number_t i = 0; i < actualThreadCount; i++)
    {
        mach_port_deallocate(thisTask, threads[i]);
    }
    vm_deallocate(thisTask, (vm_address_t)threads, sizeof(thread_t) * actualThreadCount);

    return true;
}
```

恢复线程的时候，使用相应的 thread_resume() 方法：
```
/// 恢复所有线程
void ksmc_resumeEnvironment(__unused thread_act_array_t threads, __unused mach_msg_type_number_t numThreads)
{
#if KSCRASH_HAS_THREADS_API
    KSLOG_DEBUG("Resuming environment.");
    kern_return_t kr;
    const task_t thisTask = mach_task_self();
    const thread_t thisThread = (thread_t)ksthread_self();
    
    if(threads == NULL || numThreads == 0)
    {
        KSLOG_ERROR("we should call ksmc_suspendEnvironment() first");
        return;
    }
    
    for(mach_msg_type_number_t i = 0; i < numThreads; i++)
    {
        thread_t thread = threads[i];
        if(thread != thisThread && !isThreadInList(thread, g_reservedThreads, g_reservedThreadsCount))
        {
            if((kr = thread_resume(thread)) != KERN_SUCCESS)
            {
                // Record the error and keep going.
                KSLOG_ERROR("thread_resume (%08x): %s", thread, mach_error_string(kr));
            }
        }
    }
    
    for(mach_msg_type_number_t i = 0; i < numThreads; i++)
    {
        mach_port_deallocate(thisTask, threads[i]);
    }
    vm_deallocate(thisTask, (vm_address_t)threads, sizeof(thread_t) * numThreads);
    
    KSLOG_DEBUG("Resume complete.");
#endif
}
```

**所有 Mach 异常都在 BSD 层被 ux_exception 转换为相应的 Unix 信号，并通过 threadsignal 将信号投递到出错的线程。在 Mach 层，我们可以直接通过 Mach 异常推测出 Signal 类型：**
```
// 将内核层错误码 mach.type 和mach.code 转为 sytem层 signal
static int signalForMachException(exception_type_t exception, mach_exception_code_t code)
{
    switch(exception) {
        case EXC_ARITHMETIC: return SIGFPE;
        case EXC_BAD_ACCESS: return code == KERN_INVALID_ADDRESS ? SIGSEGV/*虚拟内存坏地址错误*/ : SIGBUS/*硬件内存访问错误*/;
        case EXC_BAD_INSTRUCTION: return SIGILL/*无效或者不支持的机器命令*/;
        case EXC_BREAKPOINT: return SIGTRAP;/* 调试断点*/
        case EXC_EMULATION: return SIGEMT;/*模拟器错误*/
        case EXC_SOFTWARE: {
            switch (code) {
                case EXC_UNIX_BAD_SYSCALL:
                    return SIGSYS;/* 被系统层禁止的调用*/
                case EXC_UNIX_BAD_PIPE:
                    return SIGPIPE;/* 系统内核通信某一方已经关闭错误*/
                case EXC_UNIX_ABORT:
                    return SIGABRT; /*abort,多为assert断言*/
                case EXC_SOFT_SIGNAL:
                    return SIGKILL;/*强制关闭，比如资源不足,用户强关*/
            }
            break;
        }
    }
    return 0;
}
```

### 取消Mach层检测

在捕获到异常后就要取消原本的监听，主要分为两步：
1. 将原本用作监测异常的 port 恢复
2. 结束自己创建的用于处理异常的线程
```
// 取消异常监听
static void uninstallExceptionHandler(void)
{
    KSLOG_DEBUG("Uninstalling mach exception handler.");
    
    // NOTE: Do not deallocate the exception port. If a secondary crash occurs
    // it will hang the process.
    /// 恢复原本的mach处理端口
    restoreExceptionPorts();
    
    thread_t thread_self = (thread_t)ksthread_self();
    
    /// 当前不是 primary 处理 crash 的线程，那么终止线程
    if(g_primaryPThread != 0 && g_primaryMachThread != thread_self)
    {
        KSLOG_DEBUG("Canceling primary exception thread.");
        if(g_isHandlingCrash)
        {
            thread_terminate(g_primaryMachThread);
        }
        else
        {
            pthread_cancel(g_primaryPThread);
        }
        g_primaryMachThread = 0;
        g_primaryPThread = 0;
    }
    
    /// 当前不是备用处理 crash 的线程，那么终止线程
    if(g_secondaryPThread != 0 && g_secondaryMachThread != thread_self)
    {
        KSLOG_DEBUG("Canceling secondary exception thread.");
        if(g_isHandlingCrash)
        {
            thread_terminate(g_secondaryMachThread);
        }
        else
        {
            pthread_cancel(g_secondaryPThread);
        }
        g_secondaryMachThread = 0;
        g_secondaryPThread = 0;
    }
    
    /// 释放检测端口port
    g_exceptionPort = MACH_PORT_NULL;
    KSLOG_DEBUG("Mach exception handlers uninstalled.");
}
```

## signal异常

Mach 异常会在 BSD 层转化为相应的 UNIX 信号，投递到相应的线程中。我们同样可以捕捉相应的 Signal。

### 开启signal监测
同样是调用 setEnabled() 方法，它会执行到 installSignalHandler() 方法中。相关的参数设置比较多，坦白来说确实不太好理解它们的作用，不过其实粗略的看下来也不影响对于主流程的理解。总的来说就是通过 sigaction() 方法记录下某个 siganl 对应的处理方法，并且保存先前的处理方法：
```
/// 创建 signal 的捕捉者
static bool installSignalHandler(void)
{
    KSLOG_DEBUG("Installing signal handler.");

#if KSCRASH_HAS_SIGNAL_STACK

    if(g_signalStack.ss_size == 0)
    {
        KSLOG_DEBUG("Allocating signal stack area.");
        /// 初始化分配signal栈内存
        g_signalStack.ss_size = SIGSTKSZ;
        g_signalStack.ss_sp = malloc(g_signalStack.ss_size);
    }

    KSLOG_DEBUG("Setting signal stack area.");
    if(sigaltstack(&g_signalStack, NULL) != 0)
    {
        KSLOG_ERROR("signalstack: %s", strerror(errno));
        goto failed;
    }
#endif

    /// 需要监听的 signal 数组
    const int* fatalSignals = kssignal_fatalSignals();
    /// 需要监听的 signal 数组大小
    int fatalSignalsCount = kssignal_numFatalSignals();

    if(g_previousSignalHandlers == NULL)
    {
        KSLOG_DEBUG("Allocating memory to store previous signal handlers.");
        /// 设置该 signal 对应的处理方法，并且保存原始的处理方法
        g_previousSignalHandlers = malloc(sizeof(*g_previousSignalHandlers)
                                          * (unsigned)fatalSignalsCount);
    }

    struct sigaction action = {{0}};
    action.sa_flags = SA_SIGINFO | SA_ONSTACK;
#if KSCRASH_HOST_APPLE && defined(__LP64__)
    action.sa_flags |= SA_64REGSET;
#endif
    sigemptyset(&action.sa_mask);
    action.sa_sigaction = &handleSignal;

    for(int i = 0; i < fatalSignalsCount; i++)
    {
        KSLOG_DEBUG("Assigning handler for signal %d", fatalSignals[i]);
        if(sigaction(fatalSignals[i], &action, &g_previousSignalHandlers[i]) != 0)
        {
            char sigNameBuff[30];
            const char* sigName = kssignal_signalName(fatalSignals[i]);
            if(sigName == NULL)
            {
                snprintf(sigNameBuff, sizeof(sigNameBuff), "%d", fatalSignals[i]);
                sigName = sigNameBuff;
            }
            KSLOG_ERROR("sigaction (%s): %s", sigName, strerror(errno));
            // Try to reverse the damage
            for(i--;i >= 0; i--)
            {
                sigaction(fatalSignals[i], &g_previousSignalHandlers[i], NULL);
            }
            goto failed;
        }
    }
    KSLOG_DEBUG("Signal handlers installed.");
    return true;

failed:
    KSLOG_DEBUG("Failed to install signal handlers.");
    return false;
}
```

fatal_signal 包括如下：
```
static const int g_fatalSignals[] =
{
    SIGABRT,
    SIGBUS,
    SIGFPE,
    SIGILL,
    SIGPIPE,
    SIGSEGV,
    SIGSYS,
    SIGTRAP,
};
```

### 监测到signal异常并收集
处理异常的回调方法中会返回 signal 信息，以及一个 context：
```
/// 处理signal异常
static void handleSignal(int sigNum, siginfo_t* signalInfo, void* userContext)
{
    KSLOG_DEBUG("Trapped signal %d", sigNum);
    if(g_isEnabled)
    {
        thread_act_array_t threads = NULL;
        mach_msg_type_number_t numThreads = 0;
        /// 暂停所有线程
        ksmc_suspendEnvironment(&threads, &numThreads);
        
        /// 通知异常已经捕获（非异步的）
        kscm_notifyFatalExceptionCaptured(false);

        KSLOG_DEBUG("Filling out context.");
        /// 初始化异常上下文
        KSMC_NEW_CONTEXT(machineContext);
        /// 保存 context 到 machineContext 中，并且获取 thread 信息
        ksmc_getContextForSignal(userContext, machineContext);
        /// 把 machineContext 放到  g_stackCursor 中
        kssc_initWithMachineContext(&g_stackCursor, KSSC_MAX_STACK_DEPTH, machineContext);

        
        /// 生成真正的 context
        KSCrash_MonitorContext* crashContext = &g_monitorContext;
        memset(crashContext, 0, sizeof(*crashContext));
        crashContext->crashType = KSCrashMonitorTypeSignal;
        crashContext->eventID = g_eventID;
        crashContext->offendingMachineContext = machineContext;
        crashContext->registersAreValid = true;
        crashContext->faultAddress = (uintptr_t)signalInfo->si_addr;
        crashContext->signal.userContext = userContext;
        crashContext->signal.signum = signalInfo->si_signo;
        crashContext->signal.sigcode = signalInfo->si_code;
        crashContext->stackCursor = &g_stackCursor;

        /// 把 context 传给外部处理函数
        kscm_handleException(crashContext);
        
        /// 恢复原来的环境
        ksmc_resumeEnvironment(threads, numThreads);
    }

    KSLOG_DEBUG("Re-raising signal for regular handlers to catch.");
    
    /// 重新抛出 signal
    raise(sigNum);
}
```

整个流程和 Mach 异常还是非常类似的，先暂停线程，然后读取线程信息，再把 signal 信息线程信息保存到 context 中，传递给外部的处理函数。最后恢复原来的环境。


### 取消signal监测

```
/// 取消捕捉 signal
static void uninstallSignalHandler(void)
{
    KSLOG_DEBUG("Uninstalling signal handlers.");

    const int* fatalSignals = kssignal_fatalSignals();
    int fatalSignalsCount = kssignal_numFatalSignals();

    for(int i = 0; i < fatalSignalsCount; i++)
    {
        KSLOG_DEBUG("Restoring original handler for signal %d", fatalSignals[i]);
        sigaction(fatalSignals[i], &g_previousSignalHandlers[i], NULL);
    }
    
#if KSCRASH_HAS_SIGNAL_STACK
    g_signalStack = (stack_t){0};
#endif
    KSLOG_DEBUG("Signal handlers uninstalled.");
}
```

取消捕捉的方式和启动捕捉类似，都是通过sigaction()方法，不同的是，现在将原本的处理方法传回。

## C++异常

### 开启C++异常监测
通过 set_terminate() 方法设置自己的捕获函数：
```
static void setEnabled(bool isEnabled)
{
    if(isEnabled != g_isEnabled)
    {
        g_isEnabled = isEnabled;
        if(isEnabled)
        {
            initialize();

            ksid_generate(g_eventID);
            /// 保存原始的 c++ 处理 handler，设置自己的处理 handler
            g_originalTerminateHandler = std::set_terminate(CPPExceptionTerminate);
        }
        else
        {
            /// 恢复原始的 c++ 处理 handler
            std::set_terminate(g_originalTerminateHandler);
        }
        g_captureNextStackTrace = isEnabled;
    }
}
```



### 监测到C++异常并收集

```
/// c++ 处理函数
static void CPPExceptionTerminate(void)
{
    thread_act_array_t threads = NULL;
    mach_msg_type_number_t numThreads = 0;
    
    /// 挂起非处理现场和白名单线程的其他所有线程
    ksmc_suspendEnvironment(&threads, &numThreads);
    KSLOG_DEBUG("Trapped c++ exception");
    const char* name = NULL;
    std::type_info* tinfo = __cxxabiv1::__cxa_current_exception_type();
    if(tinfo != NULL)
    {
        name = tinfo->name();
    }
    
    if(name == NULL || strcmp(name, "NSException") != 0)
    {
        /// 捕捉到 crash 后，清空 KSCrash 的所有 monitor
        kscm_notifyFatalExceptionCaptured(false);
        KSCrash_MonitorContext* crashContext = &g_monitorContext;
        memset(crashContext, 0, sizeof(*crashContext));

        char descriptionBuff[DESCRIPTION_BUFFER_LENGTH];
        const char* description = descriptionBuff;
        descriptionBuff[0] = 0;

        KSLOG_DEBUG("Discovering what kind of exception was thrown.");
        g_captureNextStackTrace = false;
        try
        {
            throw;
        }
        catch(std::exception& exc)
        {
            strncpy(descriptionBuff, exc.what(), sizeof(descriptionBuff));
        }
#define CATCH_VALUE(TYPE, PRINTFTYPE) \
catch(TYPE value)\
{ \
    snprintf(descriptionBuff, sizeof(descriptionBuff), "%" #PRINTFTYPE, value); \
}
        CATCH_VALUE(char,                 d)
        CATCH_VALUE(short,                d)
        CATCH_VALUE(int,                  d)
        CATCH_VALUE(long,                ld)
        CATCH_VALUE(long long,          lld)
        CATCH_VALUE(unsigned char,        u)
        CATCH_VALUE(unsigned short,       u)
        CATCH_VALUE(unsigned int,         u)
        CATCH_VALUE(unsigned long,       lu)
        CATCH_VALUE(unsigned long long, llu)
        CATCH_VALUE(float,                f)
        CATCH_VALUE(double,               f)
        CATCH_VALUE(long double,         Lf)
        CATCH_VALUE(char*,                s)
        catch(...)
        {
            description = NULL;
        }
        g_captureNextStackTrace = g_isEnabled;

        // TODO: Should this be done here? Maybe better in the exception handler?
        KSMC_NEW_CONTEXT(machineContext);
        ksmc_getContextForThread(ksthread_self(), machineContext, true);

        KSLOG_DEBUG("Filling out context.");
        crashContext->crashType = KSCrashMonitorTypeCPPException;
        crashContext->eventID = g_eventID;
        crashContext->registersAreValid = false;
        crashContext->stackCursor = &g_stackCursor;
        crashContext->CPPException.name = name;
        crashContext->exceptionName = name;
        crashContext->crashReason = description;
        crashContext->offendingMachineContext = machineContext;

        /// 处理异常
        kscm_handleException(crashContext);
    }
    else
    {
        KSLOG_DEBUG("Detected NSException. Letting the current NSException handler deal with it.");
    }
    
    /// 恢复线程
    ksmc_resumeEnvironment(threads, numThreads);

    KSLOG_DEBUG("Calling original terminate handler.");
    
    /// 触发原本的  handler
    g_originalTerminateHandler();
}
```

### 取消C++异常监测
C++异常检测无需取消，是通过记录并重写原先命名空间下的CPPExceptionTerminate方法，然后最后重新调用原先记录的方法。

## NSException异常

### 开启NSException异常监测
先通过 NSGetUncaughtExceptionHandler() 获取原先的异常处理函数，然后再通过 NSSetUncaughtExceptionHandler() 方法设置自己的处理函数。
```
static void setEnabled(bool isEnabled)
{
    if(isEnabled != g_isEnabled)
    {
        g_isEnabled = isEnabled;
        if(isEnabled)
        {
            KSLOG_DEBUG(@"Backing up original handler.");
            /// 拿到原来的 handler
            g_previousUncaughtExceptionHandler = NSGetUncaughtExceptionHandler();
            
            /// 设置新的 handler
            KSLOG_DEBUG(@"Setting new handler.");
            NSSetUncaughtExceptionHandler(&handleUncaughtException);
            KSCrash.sharedInstance.uncaughtExceptionHandler = &handleUncaughtException;
            KSCrash.sharedInstance.currentSnapshotUserReportedExceptionHandler = &handleCurrentSnapshotUserReportedException;
        }
        else
        {
            KSLOG_DEBUG(@"Restoring original handler.");
            NSSetUncaughtExceptionHandler(g_previousUncaughtExceptionHandler);
        }
    }
}
```
### 监测到NSException异常并收集
关于调用堆栈的获取都不需要通过task，直接从 NSException 就可以获取到：
```
static void handleException(NSException* exception, BOOL currentSnapshotUserReported) {
    KSLOG_DEBUG(@"Trapped exception %@", exception);
    if(g_isEnabled)
    {
        thread_act_array_t threads = NULL;
        mach_msg_type_number_t numThreads = 0;
        ksmc_suspendEnvironment(&threads, &numThreads);
        kscm_notifyFatalExceptionCaptured(false);

        KSLOG_DEBUG(@"Filling out context.");
        
        /// 调用堆栈的地址
        NSArray* addresses = [exception callStackReturnAddresses];
        NSUInteger numFrames = addresses.count;
        uintptr_t* callstack = malloc(numFrames * sizeof(*callstack));
        
        /// 转为堆栈
        for(NSUInteger i = 0; i < numFrames; i++)
        {
            callstack[i] = (uintptr_t)[addresses[i] unsignedLongLongValue];
        }

        char eventID[37];
        ksid_generate(eventID);
        KSMC_NEW_CONTEXT(machineContext);
        ksmc_getContextForThread(ksthread_self(), machineContext, true);
        KSStackCursor cursor;
        kssc_initWithBacktrace(&cursor, callstack, (int)numFrames, 0);

        KSCrash_MonitorContext* crashContext = &g_monitorContext;
        memset(crashContext, 0, sizeof(*crashContext));
        crashContext->crashType = KSCrashMonitorTypeNSException;
        crashContext->eventID = eventID;
        crashContext->offendingMachineContext = machineContext;
        crashContext->registersAreValid = false;
        crashContext->NSException.name = [[exception name] UTF8String];
        crashContext->NSException.userInfo = [[NSString stringWithFormat:@"%@", exception.userInfo] UTF8String];
        crashContext->exceptionName = crashContext->NSException.name;
        crashContext->crashReason = [[exception reason] UTF8String];
        crashContext->stackCursor = &cursor;
        crashContext->currentSnapshotUserReported = currentSnapshotUserReported;

        KSLOG_DEBUG(@"Calling main crash handler.");
        
        /// 回调异常， context 交给 kscrashmonitor 处理
        kscm_handleException(crashContext);

        free(callstack);
        if (currentSnapshotUserReported) {
            ksmc_resumeEnvironment(threads, numThreads);
        }
        if (g_previousUncaughtExceptionHandler != NULL)
        {
            KSLOG_DEBUG(@"Calling original exception handler.");
            g_previousUncaughtExceptionHandler(exception);
        }
    }
}
```
### 取消NSException异常监测
NSException跟C++类似，通过记录并替换原先的异常捕获函数地址，等捕获完成后重新调用原先的函数地址

## DealLock

### 开启DeadLock监测
```
- (void) runMonitor
{
    BOOL cancelled = NO;
    do
    {
        // Only do a watchdog check if the watchdog interval is > 0.
        // If the interval is <= 0, just idle until the user changes it.
        @autoreleasepool {
            NSTimeInterval sleepInterval = g_watchdogInterval;
            BOOL runWatchdogCheck = sleepInterval > 0;
            if(!runWatchdogCheck)
            {
                sleepInterval = kIdleInterval;
            }
            [NSThread sleepForTimeInterval:sleepInterval];
            cancelled = self.monitorThread.isCancelled;
            if(!cancelled && runWatchdogCheck)
            {
                if(self.awaitingResponse) // 已经在等待响应
                {
                    [self handleDeadlock];
                }
                else
                {
                    [self watchdogPulse];  // 响应
                }
            }
        }
    } while (!cancelled);
}
```

处理死锁的检测通过一个符号变量控制开启一只看门狗，通过看门狗响应的方式监测。如果开启看门狗5秒内会等待狗响应，如果未响应两次，则出现死锁，开启收集DeadLock信息

### 处理DeadLock

```
/// 处理死锁
- (void) handleDeadlock
{
    thread_act_array_t threads = NULL;
    mach_msg_type_number_t numThreads = 0;
    
    /// 暂停所有线程
    ksmc_suspendEnvironment(&threads, &numThreads);
    kscm_notifyFatalExceptionCaptured(false);

    KSMC_NEW_CONTEXT(machineContext);
    ksmc_getContextForThread(g_mainQueueThread, machineContext, false);
    KSStackCursor stackCursor;
    kssc_initWithMachineContext(&stackCursor, KSSC_MAX_STACK_DEPTH, machineContext);
    char eventID[37];
    ksid_generate(eventID);

    KSLOG_DEBUG(@"Filling out context.");
    KSCrash_MonitorContext* crashContext = &g_monitorContext;
    memset(crashContext, 0, sizeof(*crashContext));
    crashContext->crashType = KSCrashMonitorTypeMainThreadDeadlock;
    crashContext->eventID = eventID;
    crashContext->registersAreValid = false;
    crashContext->offendingMachineContext = machineContext;
    crashContext->stackCursor = &stackCursor;
    
    /// 收集异常，
    kscm_handleException(crashContext);
    /// 恢复所有线程
    ksmc_resumeEnvironment(threads, numThreads);

    KSLOG_DEBUG(@"Calling abort()");
    abort();
}
```

死锁异常挂起所有线程，处理结束后恢复所有线程会手动调用abort()关闭进程。


## 异常收集总结
替换原来的捕获处理，捕获到异常后保存context信息，暂停所有线程获取所有线程信息，恢复原本的捕获处理方法，调用统一的异常处理函数。


# Crash回调

在接收到各种类型的Crash之后，会统一回调处理函数，处理函数会把Crash发生时候记录的信息全部写入磁盘：
1. image信息
2. 线程信息:
    2.1 符号化调用堆栈
    2.2 寄存器
    2.3 调用堆栈信息
    2.4 内存地址对应的对象
3. 野指针信息

crash回调的入口函数，它就是 KSCrashC.c 中的 onCrash() 方法：

```
/// 接收到 crash 信息的处理函数
static void onCrash(struct KSCrash_MonitorContext* monitorContext) {
    if (monitorContext->currentSnapshotUserReported == false) {
        KSLOG_DEBUG("Updating application state to note crash.");
        kscrashstate_notifyAppCrash();
    }
    monitorContext->consoleLogPath = g_shouldAddConsoleLogToReport ? g_consoleLogPath : NULL;

    /// 如果在crash的时候又crash了
    if(monitorContext->crashedDuringCrashHandling)
    {
        /// 写入recrash信息
        kscrashreport_writeRecrashReport(monitorContext, g_lastCrashReportFilePath);
    }
    else
    {
        char crashReportFilePath[KSFU_MAX_PATH_LENGTH];
        int64_t reportID = kscrs_getNextCrashReport(crashReportFilePath);
        strncpy(g_lastCrashReportFilePath, crashReportFilePath, sizeof(g_lastCrashReportFilePath));
        
        /// 将 context 转为 report 写入路径crashReportFilePath中
        kscrashreport_writeStandardReport(monitorContext, crashReportFilePath);

        if(g_reportWrittenCallback)
        {
            g_reportWrittenCallback(reportID);
        }
    }
}
```

从上面可以可以看到crash的时候monitorContext->crashedDuringCrashHandling又有oncrash()的上下文，将会通过kscrashreport_writeRecrashReport()进入重复闪退的逻辑

其中重要的就是将 context 写入文件的 kscrashreport_writeStandardReport() 方法：
```
/// 将 context 转为 report 写入 path
void kscrashreport_writeStandardReport(const KSCrash_MonitorContext* const monitorContext, const char* const path)
{
    KSLOG_INFO("Writing crash report to %s", path);
    /// 每个buffer长度
    char writeBuffer[1024];
    KSBufferedWriter bufferedWriter;

    if(!ksfu_openBufferedWriter(&bufferedWriter, path, writeBuffer, sizeof(writeBuffer)))
    {
        return;
    }

    ksccd_freeze();
    
    /// jsonContext 的 userData 是 writer  writer 的 context 是 jsonContext
    KSJSONEncodeContext jsonContext;
    jsonContext.userData = &bufferedWriter;
    /// 写入操作的执行者
    KSCrashReportWriter concreteWriter;
    KSCrashReportWriter* writer = &concreteWriter;
    /// 初始化 writer书写器
    prepareReportWriter(writer, &jsonContext);

    ksjson_beginEncode(getJsonContext(writer), true, addJSONData, &bufferedWriter);

    /// 将 “{” 写入 buffer
    writer->beginObject(writer, KSCrashField_Report);
    {
        /// 写入基本信息
        writeReportInfo(writer,
                        KSCrashField_Report,//"report"
                        KSCrashReportType_Standard,//"standard"
                        monitorContext->eventID,
                        monitorContext->System.processName);
        
        /// 将 buffer 保存到磁盘中
        ksfu_flushBufferedWriter(&bufferedWriter);

        /// 把所有 image 信息写入磁盘
        writeBinaryImages(writer, KSCrashField_BinaryImages);
        ksfu_flushBufferedWriter(&bufferedWriter);

        /// 写入 ZombieException 出错的信息
        writeProcessState(writer, KSCrashField_ProcessState, monitorContext);
        ksfu_flushBufferedWriter(&bufferedWriter);

        /// 写入系统信息
        writeSystemInfo(writer, KSCrashField_System, monitorContext);
        ksfu_flushBufferedWriter(&bufferedWriter);

        // "crash"
        writer->beginObject(writer, KSCrashField_Crash);
        {
            // 写入error信息
            writeError(writer, KSCrashField_Error, monitorContext);
            ksfu_flushBufferedWriter(&bufferedWriter);
            /// 写入所有的线程信息
            writeAllThreads(writer,
                            KSCrashField_Threads,
                            monitorContext,
                            g_introspectionRules.enabled);
            ksfu_flushBufferedWriter(&bufferedWriter);
        }
        writer->endContainer(writer);

        if(g_userInfoJSON != NULL)
        {
            addJSONElement(writer, KSCrashField_User, g_userInfoJSON, false);
            ksfu_flushBufferedWriter(&bufferedWriter);
        }
        else
        {
            writer->beginObject(writer, KSCrashField_User);
        }
        if(g_userSectionWriteCallback != NULL)
        {
            ksfu_flushBufferedWriter(&bufferedWriter);
            if (monitorContext->currentSnapshotUserReported == false) {
                g_userSectionWriteCallback(writer);
            }
        }
        writer->endContainer(writer);
        ksfu_flushBufferedWriter(&bufferedWriter);

        /// 写入console_log里的debug信息
        writeDebugInfo(writer, KSCrashField_Debug, monitorContext);
    }
    writer->endContainer(writer);
    
    ksjson_endEncode(getJsonContext(writer));
    ksfu_closeBufferedWriter(&bufferedWriter);
    ksccd_unfreeze();
}
```

## 写入数据方法
不管哪些初始化方法，直接看 beginObject()，它在兜兜转转后调用了如下方法：
```
/// 将 data 保存到 buffer 或者文件系统中
static int addJSONData(const char* restrict const data, const int length, void* restrict userData)
{
    KSBufferedWriter* writer = (KSBufferedWriter*)userData;
    const bool success = ksfu_writeBufferedWriter(writer, data, length);
    return success ? KSJSON_OK : KSJSON_ERROR_CANNOT_ADD_DATA;
}

/// 将 data 写入 buffer 或者文件系统中
bool ksfu_writeBufferedWriter(KSBufferedWriter* writer, const char* restrict const data, const int length)
{
    /// 如果 buffer 中的字符数量超出最大值了，那么 flush buffer 到文件系统中
    
    /// 如果buffer中的字符数量超出上一个在写入的buffer，直接写入
    if(length > writer->bufferLength - writer->position)
    {
        ksfu_flushBufferedWriter(writer);
    }
    /// 如果buffer中的字符数量超每个buffer最大长度，直接写入
    if(length > writer->bufferLength)
    {
        return ksfu_writeBytesToFD(writer->fd, data, length);
    }
    
    /// 否则写入 buffer 中
    memcpy(writer->buffer + writer->position, data, length);
    writer->position += length;
    return true;
}

/// 判断是否要将 buffer 写入文件
bool ksfu_flushBufferedWriter(KSBufferedWriter* writer)
{
    if(writer->fd > 0 && writer->position > 0)
    {
        
        if(!ksfu_writeBytesToFD(writer->fd, writer->buffer, writer->position))
        {
            return false;
        }
        writer->position = 0;
    }
    return true;
}


/// 判断buffer能否写入文件中
bool ksfu_writeBytesToFD(const int fd, const char* const bytes, int length)
{
    const char* pos = bytes;
    while(length > 0)
    {
        int bytesWritten = (int)write(fd, pos, (unsigned)length);
        if(bytesWritten == -1)
        {
            KSLOG_ERROR("Could not write to fd %d: %s", fd, strerror(errno));
            return false;
        }
        length -= bytesWritten;
        pos += bytesWritten;
    }
    return true;
}
```

KSBufferedWriter 打开了一个写入流，同时维护了一个 1024 大小的 buffer。当 buffer 中的数据超出1024字节之后，就会将 buffer 写入文件中。

### 写入二进制文件binaryImage信息

```
static void writeBinaryImages(const KSCrashReportWriter* const writer, const char* const key)
{
    /// 通过 _dyld_image_count 获取 image 的数量
    const int imageCount = ksdl_imageCount();

    writer->beginArray(writer, key);
    {
        for(int iImg = 0; iImg < imageCount; iImg++)
        {
            /// 通过 _dyld_get_image_header 获取 image 信息
            writeBinaryImage(writer, NULL, iImg);
        }
    }
    writer->endContainer(writer);
}

/// 获取二进制文件镜像总数量
int ksdl_imageCount(void)
{
    return (int)_dyld_image_count();
}

```

通过 dyld 提供的 _dyld_image_count() 方法获取加载的 image 数量。具体的 image 中的信息在 writeBinaryImage() 中获取：
```
static void writeBinaryImage(const KSCrashReportWriter* const writer,
                             const char* const key,
                             const int index)
{
    // 初始化mach-o信息
    KSBinaryImage image = {0};
    /// 通过 _dyld_get_image_header 获取 image 信息
    if(!ksdl_getBinaryImage(index, &image))
    {
        return;
    }

    writer->beginObject(writer, key);
    {
        writer->addUIntegerElement(writer, KSCrashField_ImageAddress, image.address);
        writer->addUIntegerElement(writer, KSCrashField_ImageVmAddress, image.vmAddress);
        writer->addUIntegerElement(writer, KSCrashField_ImageSize, image.size);
        writer->addStringElement(writer, KSCrashField_Name, image.name);
        writer->addUUIDElement(writer, KSCrashField_UUID, image.uuid);
        writer->addIntegerElement(writer, KSCrashField_CPUType, image.cpuType);
        writer->addIntegerElement(writer, KSCrashField_CPUSubType, image.cpuSubType);
        writer->addUIntegerElement(writer, KSCrashField_ImageMajorVersion, image.majorVersion);
        writer->addUIntegerElement(writer, KSCrashField_ImageMinorVersion, image.minorVersion);
        writer->addUIntegerElement(writer, KSCrashField_ImageRevisionVersion, image.revisionVersion);
        if(image.crashInfoMessage != NULL)
        {
            writer->addStringElement(writer, KSCrashField_ImageCrashInfoMessage, image.crashInfoMessage);
        }
        if(image.crashInfoMessage2 != NULL)
        {
            writer->addStringElement(writer, KSCrashField_ImageCrashInfoMessage2, image.crashInfoMessage2);
        }
        if(image.crashInfoBacktrace != NULL)
        {
            writer->addStringElement(writer, KSCrashField_ImageCrashInfoBacktrace, image.crashInfoBacktrace);
        }
        if(image.crashInfoSignature != NULL)
        {
            writer->addStringElement(writer, KSCrashField_ImageCrashInfoSignature, image.crashInfoSignature);
        }
    }
    writer->endContainer(writer);
}

/// 获取二进制（mach-o）header部分信息
bool ksdl_getBinaryImage(int index, KSBinaryImage* buffer)
{
    /// 获取mach-o 镜像的头文件
    const struct mach_header* header = _dyld_get_image_header((unsigned)index);
    if(header == NULL)
    {
        return false;
    }
    
    return ksdl_getBinaryImageForHeader((const void*)header, _dyld_get_image_name((unsigned)index), buffer);
}

/// 获取mach-o二进制文件头文件镜像
bool ksdl_getBinaryImageForHeader(const void* const header_ptr, const char* const image_name, KSBinaryImage* buffer)
{
    const struct mach_header* header = (const struct mach_header*)header_ptr;
    uintptr_t cmdPtr = firstCmdAfterHeader(header);
    if(cmdPtr == 0)
    {
        return false;
    }
    
    // Look for the TEXT segment to get the image size.
    // Also look for a UUID command.
    uint64_t imageSize = 0;
    uint64_t imageVmAddr = 0;
    uint64_t version = 0;
    uint8_t* uuid = NULL;
    
    for(uint32_t iCmd = 0; iCmd < header->ncmds; iCmd++)
    {
        struct load_command* loadCmd = (struct load_command*)cmdPtr;
        switch(loadCmd->cmd)
        {
            case LC_SEGMENT: // 32位段区索引
            {
                struct segment_command* segCmd = (struct segment_command*)cmdPtr;
                if(strcmp(segCmd->segname, SEG_TEXT) == 0)
                {
                    imageSize = segCmd->vmsize;
                    imageVmAddr = segCmd->vmaddr;
                }
                break;
            }
            case LC_SEGMENT_64: // 64位段区索引
            {
                struct segment_command_64* segCmd = (struct segment_command_64*)cmdPtr;
                if(strcmp(segCmd->segname, SEG_TEXT) == 0)
                {
                    imageSize = segCmd->vmsize;
                    imageVmAddr = segCmd->vmaddr;
                }
                break;
            }
            case LC_UUID: // UUID
            {
                struct uuid_command* uuidCmd = (struct uuid_command*)cmdPtr;
                uuid = uuidCmd->uuid;
                break;
            }
            case LC_ID_DYLIB: // 动态链接库
            {
                
                struct dylib_command* dc = (struct dylib_command*)cmdPtr;
                version = dc->dylib.current_version;
                break;
            }
        }
        cmdPtr += loadCmd->cmdsize;
    }

    buffer->address = (uintptr_t)header;
    buffer->vmAddress = imageVmAddr;
    buffer->size = imageSize;
    buffer->name = image_name;
    buffer->uuid = uuid;
    buffer->cpuType = header->cputype;
    buffer->cpuSubType = header->cpusubtype;
    buffer->majorVersion = version >> 16;
    buffer->minorVersion = (version >> 8) & 0xff;
    buffer->revisionVersion = version & 0xff;
    getCrashInfo(header, buffer);
    
    return true;
}


```

总的来说，通过 _dyld_get_image_header 获取 image 的 header 部分。然后通过 header 位置定位到 load command，遍历 load command 的信息

## 写入crash时线程信息

在 crash 的时候，会suspend所有的线程，并且获取它们的基本信息。现在就要对获取到的线程进行解析：
```
/// 写入crash时候所有的线程信息
static void writeAllThreads(const KSCrashReportWriter* const writer,
                            const char* const key,
                            const KSCrash_MonitorContext* const crash,
                            bool writeNotableAddresses)
{
    const struct KSMachineContext* const context = crash->offendingMachineContext;
    KSThread offendingThread = ksmc_getThreadFromContext(context);
    int threadCount = ksmc_getThreadCount(context);
    KSMC_NEW_CONTEXT(machineContext);

    // Fetch info for all threads.
    writer->beginArray(writer, key);
    {
        KSLOG_DEBUG("Writing %d threads.", threadCount);
        for(int i = 0; i < threadCount; i++)
        {
            KSThread thread = ksmc_getThreadAtIndex(context, i);
            if(thread == offendingThread)
            {
                writeThread(writer, NULL, crash, context, i, writeNotableAddresses);
            }
            else
            {
                ksmc_getContextForThread(thread, machineContext, false);
                writeThread(writer, NULL, crash, machineContext, i, writeNotableAddresses);
            }
        }
    }
    writer->endContainer(writer);
}


static void writeThread(const KSCrashReportWriter* const writer,
                        const char* const key,
                        const KSCrash_MonitorContext* const crash,
                        const struct KSMachineContext* const machineContext,
                        const int threadIndex,
                        const bool shouldWriteNotableAddresses)
{
    bool isCrashedThread = ksmc_isCrashedContext(machineContext);
    KSThread thread = ksmc_getThreadFromContext(machineContext);
    KSLOG_DEBUG("Writing thread %x (index %d). is crashed: %d", thread, threadIndex, isCrashedThread);

    KSStackCursor stackCursor;
    bool hasBacktrace = getStackCursor(crash, machineContext, &stackCursor);

    writer->beginObject(writer, key);
    {
        if(hasBacktrace)
        {
            /// 写入调用栈
            writeBacktrace(writer, KSCrashField_Backtrace, &stackCursor);
        }
        if(ksmc_canHaveCPUState(machineContext))
        {
            /// 写入寄存器
            writeRegisters(writer, KSCrashField_Registers, machineContext);
        }
        
        /// 写入线程index
        writer->addIntegerElement(writer, KSCrashField_Index, threadIndex);
        const char* name = ksccd_getThreadName(thread);
        if(name != NULL)
        {
            /// 线程有名称就写入线程名称
            writer->addStringElement(writer, KSCrashField_Name, name);
        }
        name = ksccd_getQueueName(thread);
        if(name != NULL)
        {
            /// 写入 dispatch_queue 的名称
            writer->addStringElement(writer, KSCrashField_DispatchQueue, name);
        }
        
        /// 是否是崩溃线程
        writer->addBooleanElement(writer, KSCrashField_Crashed, isCrashedThread);
        writer->addBooleanElement(writer, KSCrashField_CurrentThread, thread == ksthread_self());
        
        /// 如果是崩溃的线程
        if(isCrashedThread)
        {
            /// 将 stack （堆栈信息） 上的部分数据拷贝出来
            writeStackContents(writer, KSCrashField_Stack, machineContext, stackCursor.state.hasGivenUp);
            if(shouldWriteNotableAddresses)
            {
                /// 将通过 zombie 记录的地址拷贝出来
                writeNotableAddresses(writer, KSCrashField_NotableAddresses, machineContext);
            }
        }
    }
    writer->endContainer(writer);
}
```

其中比较重要的是符号化调用堆栈，写入寄存器的值，以及写入 zombie 记录的信息。

### 符号化调用堆栈

符号化的过程是通过实际的地址找到符号表中相应的符号，再到字符串表中找到对应的字符串：
```
/// 符号化调用堆栈
static void writeBacktrace(const KSCrashReportWriter* const writer,
                           const char* const key,
                           KSStackCursor* stackCursor)
{
    writer->beginObject(writer, key);
    {
        writer->beginArray(writer, KSCrashField_Contents);
        {
            /// 循环开始,对每一层调用栈进行符号化
            while(stackCursor->advanceCursor(stackCursor))
            {
                writer->beginObject(writer, NULL);
                {
                    /// 对调用栈符号化
                    if(stackCursor->symbolicate(stackCursor))
                    {
                        /// 把符号化后的 image 名，地址，symbol 名，地址写入
                        if(stackCursor->stackEntry.imageName != NULL)
                        {
                            /// 
                            writer->addStringElement(writer, KSCrashField_ObjectName, ksfu_lastPathEntry(stackCursor->stackEntry.imageName));
                        }
                        /// 写入symbol object_addr 地址
                        writer->addUIntegerElement(writer, KSCrashField_ObjectAddr, stackCursor->stackEntry.imageAddress);
                        if(stackCursor->stackEntry.symbolName != NULL)
                        {
                            writer->addStringElement(writer, KSCrashField_SymbolName, stackCursor->stackEntry.symbolName);
                        }
                        writer->addUIntegerElement(writer, KSCrashField_SymbolAddr, stackCursor->stackEntry.symbolAddress);
                    }
                    writer->addUIntegerElement(writer, KSCrashField_InstructionAddr, stackCursor->stackEntry.address);
                }
                writer->endContainer(writer);
            }
        }
        writer->endContainer(writer);
        writer->addIntegerElement(writer, KSCrashField_Skipped, 0);
    }
    writer->endContainer(writer);
}
```

核心逻辑根据address获取符号名
```
/// 根据 address 获取符号名
bool ksdl_dladdr(const uintptr_t address, Dl_info* const info)
{
    info->dli_fname = NULL;
    info->dli_fbase = NULL;
    info->dli_sname = NULL;
    info->dli_saddr = NULL;

    /// 判断 address 是在第几个 image 内
    const uint32_t idx = imageIndexContainingAddress(address);
    if(idx == UINT_MAX)
    {
        return false;
    }
    
    /// 获得该 image 的 header
    const struct mach_header* header = _dyld_get_image_header(idx);
    
    /// 获得该 image 的基地址
    const uintptr_t imageVMAddrSlide = (uintptr_t)_dyld_get_image_vmaddr_slide(idx);
    
    /// 获取 address 相对于 image 的偏移
    const uintptr_t addressWithSlide = address - imageVMAddrSlide;
    
    /// 获得 segment 在虚拟内存中的基地址
    const uintptr_t segmentBase = segmentBaseOfImageIndex(idx) + imageVMAddrSlide;
    if(segmentBase == 0)
    {
        return false;
    }

    /// 获取 image 的名字
    info->dli_fname = _dyld_get_image_name(idx);
    
    /// 获取 header 地址
    info->dli_fbase = (void*)header;

    // Find symbol tables and get whichever symbol is closest to the address.
    const nlist_t* bestMatch = NULL;
    uintptr_t bestDistance = ULONG_MAX;
    uintptr_t cmdPtr = firstCmdAfterHeader(header);
    if(cmdPtr == 0)
    {
        return false;
    }
    for(uint32_t iCmd = 0; iCmd < header->ncmds; iCmd++)
    {
        const struct load_command* loadCmd = (struct load_command*)cmdPtr;
        if(loadCmd->cmd == LC_SYMTAB)
        {
            
            /// 找到 symbol table 的 load command
            const struct symtab_command* symtabCmd = (struct symtab_command*)cmdPtr;
            
            /// 通过 load command 中的 offset + segment 的基地址，得到 symbol table 的实际地址
            const nlist_t* symbolTable = (nlist_t*)(segmentBase + symtabCmd->symoff);
            /// 找到 string table 的位置
            const uintptr_t stringTable = segmentBase + symtabCmd->stroff;
            
            /// 找到最佳的符号
            for(uint32_t iSym = 0; iSym < symtabCmd->nsyms; iSym++)
            {
                // Skip all debug N_STAB symbols
                if ((symbolTable[iSym].n_type & N_STAB) != 0) 
                {
                    continue;
                }

                // If n_value is 0, the symbol refers to an external object.
                if(symbolTable[iSym].n_value != 0)
                {
                    uintptr_t symbolBase = symbolTable[iSym].n_value;
                    uintptr_t currentDistance = addressWithSlide - symbolBase;
                    if((addressWithSlide >= symbolBase) &&
                       (currentDistance <= bestDistance))
                    {
                        bestMatch = symbolTable + iSym;
                        bestDistance = currentDistance;
                    }
                }
            }
            
            /// 根据符号的位置找到其在 string table 中表示的字符串
            if(bestMatch != NULL)
            {
                info->dli_saddr = (void*)(bestMatch->n_value + imageVMAddrSlide);
                if(bestMatch->n_desc == 16)
                {
                    // This image has been stripped. The name is meaningless, and
                    // almost certainly resolves to "_mh_execute_header"
                    info->dli_sname = NULL;
                }
                else
                {
                    info->dli_sname = (char*)((intptr_t)stringTable + (intptr_t)bestMatch->n_un.n_strx);
                    if(*info->dli_sname == '_')
                    {
                        info->dli_sname++;
                    }
                }
                break;
            }
        }
        cmdPtr += loadCmd->cmdsize;
    }
    
    return true;
}
```

这个过程中就完成了从地址到 image 中符号的转化


### 写入寄存器信息

在通过 crash 获取的 context 信息中，我们可以拿到寄存器相关的信息：
```
static void writeBasicRegisters(const KSCrashReportWriter* const writer,
                                const char* const key,
                                const struct KSMachineContext* const machineContext)
{
    char registerNameBuff[30];
    const char* registerName;
    writer->beginObject(writer, key);
    {
        const int numRegisters = kscpu_numRegisters();
        for(int reg = 0; reg < numRegisters; reg++)
        {
            /// 寄存器名字
            registerName = kscpu_registerName(reg);
            if(registerName == NULL)
            {
                snprintf(registerNameBuff, sizeof(registerNameBuff), "r%d", reg);
                registerName = registerNameBuff;
            }
            
            /// 寄存器的值
            writer->addUIntegerElement(writer, registerName,
                                       kscpu_registerValue(machineContext, reg));
        }
    }
    writer->endContainer(writer);
}

/// 获取到的寄存器
static const char* g_registerNames[] =
{
    "rax", "rbx", "rcx", "rdx",
    "rdi", "rsi",
    "rbp", "rsp",
    "r8", "r9", "r10", "r11", "r12", "r13", "r14", "r15",
    "rip", "rflags",
    "cs", "fs", "gs"
};

/// 根据寄存器的序号通过如下方法在 context 中获取寄存器中存储的值：
uint64_t kscpu_registerValue(const KSMachineContext* const context, const int regNumber)
{
    if(regNumber <= 29)
    {
        return context->machineContext.__ss.__x[regNumber];
    }

    switch(regNumber)
    {
        case 30: return context->machineContext.__ss.__fp;
        case 31: return context->machineContext.__ss.__lr;
        case 32: return context->machineContext.__ss.__sp;
        case 33: return context->machineContext.__ss.__pc;
        case 34: return context->machineContext.__ss.__cpsr;
    }

    KSLOG_ERROR("Invalid register number: %d", regNumber);
    return 0;
}
```

寄存器的序号会根据架构arm，arm64，x86等不同返回不同的值

### 记录调用堆栈的部分信息
分析crash信息需要用到堆栈的内容，KSCrash提供了方法将出错线程的堆栈的部分信息拷贝出来：
```
static void writeStackContents(const KSCrashReportWriter* const writer,
                               const char* const key,
                               const struct KSMachineContext* const machineContext,
                               const bool isStackOverflow)
{
    /// 拿到 stack pointer
    uintptr_t sp = kscpu_stackPointer(machineContext);
    if((void*)sp == NULL)
    {
        return;
    }

    // 10 - 20
    uintptr_t lowAddress = sp + (uintptr_t)(kStackContentsPushedDistance * (int)sizeof(sp) * kscpu_stackGrowDirection() * -1);
    uintptr_t highAddress = sp + (uintptr_t)(kStackContentsPoppedDistance * (int)sizeof(sp) * kscpu_stackGrowDirection());
    if(highAddress < lowAddress)
    {
        uintptr_t tmp = lowAddress;
        lowAddress = highAddress;
        highAddress = tmp;
    }
    writer->beginObject(writer, key);
    {
        writer->addStringElement(writer, KSCrashField_GrowDirection, kscpu_stackGrowDirection() > 0 ? "+" : "-");
        writer->addUIntegerElement(writer, KSCrashField_DumpStart, lowAddress);
        writer->addUIntegerElement(writer, KSCrashField_DumpEnd, highAddress);
        writer->addUIntegerElement(writer, KSCrashField_StackPtr, sp);
        writer->addBooleanElement(writer, KSCrashField_Overflow, isStackOverflow);
        uint8_t stackBuffer[kStackContentsTotalDistance * sizeof(sp)];
        int copyLength = (int)(highAddress - lowAddress);
        /// 拷贝 lowAddress 上的数据到 buffer 中
        if(ksmem_copySafely((void*)lowAddress, stackBuffer, copyLength))
        {
            writer->addDataElement(writer, KSCrashField_Contents, (void*)stackBuffer, copyLength);
        }
        else
        {
            writer->addStringElement(writer, KSCrashField_Error, "Stack contents not accessible");
        }
    }
    writer->endContainer(writer);
}
```
通过 sp 寄存器拿到stack pointer：
```
uintptr_t kscpu_stackPointer(const KSMachineContext* const context)
{
    return context->machineContext.__ss.__rsp;
}
```

```
#define kStackContentsPushedDistance 20
#define kStackContentsPoppedDistance 10
#define kStackContentsTotalDistance (kStackContentsPushedDistance + kStackContentsPoppedDistance)
```

作者设置的，也就说是，拷贝栈内 20 个对象，以及刚刚出栈的 10 个对象的地址。在拿到 sp 和范围之后，就可以通过 c 的方法获取：
```
static inline int copySafely(const void* restrict const src, void* restrict const dst, const int byteCount)
{
    vm_size_t bytesCopied = 0;
    kern_return_t result = vm_read_overwrite(mach_task_self(),
                                             (vm_address_t)src,
                                             (vm_size_t)byteCount,
                                             (vm_address_t)dst,
                                             &bytesCopied);
    if(result != KERN_SUCCESS)
    {
        return 0;
    }
    return (int)bytesCopied;
}
```

### 取出地址上的内存对象

无论是寄存器还是堆栈，取出的都是地址。但是我们其实更需要的是对象的信息。因此，我们还需要到地址上去解析对象信息：

```
static void writeNotableAddresses(const KSCrashReportWriter* const writer,
                                  const char* const key,
                                  const struct KSMachineContext* const machineContext)
{
    writer->beginObject(writer, key);
    {
        /// 获取 register 上的对象
        writeNotableRegisters(writer, machineContext);
        /// 获取 stack 上的对象
        writeNotableStackContents(writer,
                                  machineContext,
                                  kStackNotableSearchBackDistance,
                                  kStackNotableSearchForwardDistance);
    }
    writer->endContainer(writer);
}
```

以解析寄存器上的对象为例：
```
static void writeMemoryContents(const KSCrashReportWriter* const writer,
                                const char* const key,
                                const uintptr_t address,
                                int* limit)
{
    (*limit)--;
    const void* object = (const void*)address;
    writer->beginObject(writer, key);
    {
        writer->addUIntegerElement(writer, KSCrashField_Address, address);
        writeZombieIfPresent(writer, KSCrashField_LastDeallocObject, address);
        if(!writeObjCObject(writer, address, limit))
        {
            if(object == NULL)
            {
                writer->addStringElement(writer, KSCrashField_Type, KSCrashMemType_NullPointer);
            }
            else if(isValidString(object))
            {
                writer->addStringElement(writer, KSCrashField_Type, KSCrashMemType_String);
                writer->addStringElement(writer, KSCrashField_Value, (const char*)object);
            }
            else
            {
                writer->addStringElement(writer, KSCrashField_Type, KSCrashMemType_Unknown);
            }
        }
    }
    writer->endContainer(writer);
}
```

writeObjCObject() 将地址转化为对象：

```
static bool writeObjCObject(const KSCrashReportWriter* const writer,
                            const uintptr_t address,
                            int* limit)
{
#if KSCRASH_HAS_OBJC
    const void* object = (const void*)address;
    switch(ksobjc_objectType(object))
    {
        case KSObjCTypeClass:
            writer->addStringElement(writer, KSCrashField_Type, KSCrashMemType_Class);
            writer->addStringElement(writer, KSCrashField_Class, ksobjc_className(object));
            return true;
        case KSObjCTypeObject:
        {
            writer->addStringElement(writer, KSCrashField_Type, KSCrashMemType_Object);
            const char* className = ksobjc_objectClassName(object);
            writer->addStringElement(writer, KSCrashField_Class, className);
            if(!isRestrictedClass(className))
            {
                switch(ksobjc_objectClassType(object))
                {
                    case KSObjCClassTypeString:
                        writeNSStringContents(writer, KSCrashField_Value, address, limit);
                        return true;
                    case KSObjCClassTypeURL:
                        writeURLContents(writer, KSCrashField_Value, address, limit);
                        return true;
                    case KSObjCClassTypeDate:
                        writeDateContents(writer, KSCrashField_Value, address, limit);
                        return true;
                    case KSObjCClassTypeArray:
                        if(*limit > 0)
                        {
                            writeArrayContents(writer, KSCrashField_FirstObject, address, limit);
                        }
                        return true;
                    case KSObjCClassTypeNumber:
                        writeNumberContents(writer, KSCrashField_Value, address, limit);
                        return true;
                    case KSObjCClassTypeDictionary:
                    case KSObjCClassTypeException:
                        // TODO: Implement these.
                        if(*limit > 0)
                        {
                            writeUnknownObjectContents(writer, KSCrashField_Ivars, address, limit);
                        }
                        return true;
                    case KSObjCClassTypeUnknown:
                        if(*limit > 0)
                        {
                            writeUnknownObjectContents(writer, KSCrashField_Ivars, address, limit);
                        }
                        return true;
                }
            }
            break;
        }
        case KSObjCTypeBlock:
            writer->addStringElement(writer, KSCrashField_Type, KSCrashMemType_Block);
            const char* className = ksobjc_objectClassName(object);
            writer->addStringElement(writer, KSCrashField_Class, className);
            return true;
        case KSObjCTypeUnknown:
            break;
    }
#endif

    return false;
}
```

把 address 强转为一个 object，然后判断 object 到底是 tagged pointer 还是 block 还是 OC 类型，还是自建的 Class。根据类型写入相应的信息。

## 野指针的监控

获取地址对象方法中，我们要注意到一个方法 writeZombieIfPresent()。这是 KSCrash 提供的 Zombie，用于监控野指针。我们来看一下它的实现：
```
static void writeZombieIfPresent(const KSCrashReportWriter* const writer,
                                 const char* const key,
                                 const uintptr_t address)
{
#if KSCRASH_HAS_OBJC
    const void* object = (const void*)address;
    const char* zombieClassName = kszombie_className(object);
    if(zombieClassName != NULL)
    {
        writer->addStringElement(writer, key, zombieClassName);
    }
#endif
}

const char* kszombie_className(const void* object)
{
    volatile Zombie* cache = g_zombieCache;
    if(cache == NULL || object == NULL)
    {
        return NULL;
    }

    Zombie* zombie = (Zombie*)cache + hashIndex(object);
    if(zombie->object == object)
    {
        return zombie->className;
    }
    return NULL;
}
```

它的方法非常简洁。就是判断当前address上的object是否是记录过的对象。这是怎么做到的呢？g_zombieCache 又是什么呢？这就需要回到KSCrashMonitor_Zombie.c 中查看。

每一个 monitor 会被调用其 setEnabled() 方法启动。Zombie 也不例外。它在 setEnabled() 中调用了 install 方法：
```
static void install(void)
{
    unsigned cacheSize = CACHE_SIZE;
    g_zombieHashMask = cacheSize - 1;
    g_zombieCache = calloc(cacheSize, sizeof(*g_zombieCache));
    if(g_zombieCache == NULL)
    {
        KSLOG_ERROR("Error: Could not allocate %u bytes of memory. KSZombie NOT installed!",
              cacheSize * sizeof(*g_zombieCache));
        return;
    }

    g_lastDeallocedException.class = objc_getClass("NSException");
    g_lastDeallocedException.address = NULL;
    g_lastDeallocedException.name[0] = 0;
    g_lastDeallocedException.reason[0] = 0;

    installDealloc_NSObject();
    installDealloc_NSProxy();
}
```

根据代码，我们可以知道，它 hook 了 NSObject 以及 NSProxy 的 dealloc 方法。先调用自己的处理方法 handleDealloc() 然后再调用原来的 dealloc 方法。注意这里调用原来 dealloc 方法的实现:
```
typedef void (*fn)(id,SEL); \
fn f = (fn)g_originalDealloc_ ## CLASS; \
f(self, _cmd); \
```

handleDealloc() 方法：
```
static inline void handleDealloc(const void* self)
{
    volatile Zombie* cache = g_zombieCache;
    likely_if(cache != NULL)
    {
        Zombie* zombie = (Zombie*)cache + hashIndex(self);
        zombie->object = self;
        Class class = object_getClass((id)self);
        zombie->className = class_getName(class);
        for(; class != nil; class = class_getSuperclass(class))
        {
            unlikely_if(class == g_lastDeallocedException.class)
            {
                storeException(self);
            }
        }
    }
}
```

在 install 的时候的时候创建了一个空的 Cache：
```
g_zombieCache = calloc(cacheSize, sizeof(*g_zombieCache));
```

在处理 dealloc 的时候，会把对象的地址放到这个 Zombie 的 cache 中。这样的作用就是对于已经销毁的对象，我们记录了一份它们的地址信息，这样以后出现野指针 crash 的时候，如果发现是 zombie 中指向的对象，那么就可以说明它被提前释放了。当然，这并不是非常准确的，因为 hash 获取 index 的方式总是会产生一定的碰撞导致对象被覆盖。当然不可否认这是一种经济有效的测试野指针的方式。作者自己在注释中说明这是一种 Poor man’s Zombie tracking XD

```
/* Poor man's zombie tracking.
 *
 * Benefits:
 * - Very low CPU overhead.
 * - Low memory overhead.
 *
 * Limitations:
 * - Not guaranteed to catch all zombies.
 * - Can generate false positives or incorrect class names.
 * - KSZombie itself must be compiled with ARC disabled. You can enable ARC in
 *   your app, but KSZombie must be compiled in a separate library if you do.
 */
```
