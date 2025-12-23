---
layout:     post
title:      iOS RunLoop原理探究
subtitle:   iOS RunLoop原理探究
date:       2020-07-05
author:     LXY
header-img: img/home1.jpeg
catalog: true
tags:
    - iOS
---



本文篇幅比较长，创作的目的为了自己日后温习知识所用，希望这篇文章能对你有所帮助。如发现任何有误之处，恳请留言纠正，谢谢。


# 前言
**RunLoop作为iOS中一个基础组件和线程有着千丝万缕的关系，同时也是很多常见技术的幕后功臣。尽管在平时多数开发者很少直接使用RunLoop，但是理解RunLoop可以帮助开发者更好的利用多线程编程模型，同时也可以帮助开发者解答日常开发中的一些疑惑。**

**这篇文章将从 CFRunLoop 的源码入手，介绍 RunLoop 的概念以及底层实现原理。之后会介绍一下在 iOS 中，苹果是如何利用 RunLoop 实现自动释放池、延迟回调、触摸事件、屏幕刷新等功能的**

# 一、什么是 RunLoop？
可以理解为字面意思：Run 表示运行，Loop 表示循环。结合在一起就是运行的循环的意思。NSRunloop是CFRunloop的封装，CFRunloop是一套C接口。

RunLoop 这个对象，在 iOS 里由 CFRunLoop 实现。简单来说，RunLoop 是用来监听输入源，进行调度处理的。这里的输入源可以是输入设备、网络、周期性或者延迟时间、异步回调。RunLoop 会接收两种类型的输入源：一种是来自另一个线程或者来自不同应用的异步消息；另一种是来自预订时间或者重复间隔的同步事件。


# 二、Runloop 内部概念介绍

在 CoreFoundation 里面关于 RunLoop 有5个类:

- CFRunLoopRef
- CFRunLoopModeRef
- CFRunLoopSourceRef
- CFRunLoopTimerRef
- CFRunLoopObserverRef


### 1、Runloop Mode

**CFRunLoopModeRef** 类并没有对外暴露，只是通过 CFRunLoopRef 的接口进行了封装。他们的关系如下:


![](https://youke2.picui.cn/s1/2025/12/23/694a7af3245e6.png)
![](https://youke2.picui.cn/s1/2025/12/17/6942ade7686f5.png)
![](https://youke2.picui.cn/s1/2025/12/17/6942adef98a38.png)

一个 RunLoop 包含若干个 Mode，每个 Mode 又包含若干个 Source/Timer/Observer。每次调用 RunLoop 的主函数时，只能指定其中一个 Mode，这个Mode被称作 CurrentMode。如果需要切换 Mode，只能退出 Loop，再重新指定一个 Mode 进入。这样做主要是为了分隔开不同组的 Source/Timer/Observer，让其互不影响。


```
	struct __CFRunLoop {
	    CFRuntimeBase _base;
	    pthread_mutex_t _lock;			/* locked for accessing mode list */
	    __CFPort _wakeUpPort;			// used for CFRunLoopWakeUp 
	    Boolean _unused;
	    volatile _per_run_data *_perRunData;              // reset for runs of the run loop
	    pthread_t _pthread;    //线程
	    uint32_t _winthread;
	    CFMutableSetRef _commonModes;   // commonModes下的两个mode（kCFRunloopDefaultMode和UITrackingMode）
	    CFMutableSetRef _commonModeItems;  // 在commonModes状态下运行的对象（例如Timer）
	    CFRunLoopModeRef _currentMode;  //在当前loop下运行的mode
	    CFMutableSetRef _modes;  // 运行的所有模式（CFRunloopModeRef类）
	    struct _block_item *_blocks_head;
	    struct _block_item *_blocks_tail;
	    CFAbsoluteTime _runTime;
	    CFAbsoluteTime _sleepTime;
	    CFTypeRef _counterpart;
	};

	struct __CFRunLoopMode {
	    CFRuntimeBase _base;
	    pthread_mutex_t _lock;	/* must have the run loop locked before locking this */
	    CFStringRef _name;
	    Boolean _stopped;
	    char _padding[3];
	    CFMutableSetRef _sources0;
	    CFMutableSetRef _sources1;
	    CFMutableArrayRef _observers;
	    CFMutableArrayRef _timers;
	    CFMutableDictionaryRef _portToV1SourceMap;
	    __CFPortSet _portSet;
	    CFIndex _observerMask;
	#if USE_DISPATCH_SOURCE_FOR_TIMERS
	    dispatch_source_t _timerSource;
	    dispatch_queue_t _queue;
	    Boolean _timerFired; // set to true by the source when a timer has fired
	    Boolean _dispatchTimerArmed;
	#endif
	#if USE_MK_TIMER_TOO
	    mach_port_t _timerPort;
	    Boolean _mkTimerArmed;
	#endif
	#if DEPLOYMENT_TARGET_WINDOWS
	    DWORD _msgQMask;
	    void (*_msgPump)(void);
	#endif
	    uint64_t _timerSoftDeadline; /* TSR */
	    uint64_t _timerHardDeadline; /* TSR */
	};

```

系统默认提供的Run Loop Modes有**kCFRunLoopDefaultMode(NSDefaultRunLoopMode)**和**UITrackingRunLoopMode**，需要切换到对应的Mode时只需要传入对应的名称即可。前者是系统默认的Runloop Mode，例如进入iOS程序默认不做任何操作就处于这种Mode中，此时滑动UIScrollView，主线程就切换Runloop到到UITrackingRunLoopMode，不再接受其他事件操作（除非你将其他Source/Timer设置到UITrackingRunLoopMode下）。


对于开发者而言经常用到**kCFRunLoopCommonModes（NSRunLoopCommonModes）**：

通过上面源码可以看到，有个概念叫 “CommonModes”：一个 Mode 可以将自己标记为”Common”属性（通过将其 ModeName 添加到 RunLoop 的 “commonModes” 中）。每当 RunLoop 的内容发生变化时，RunLoop 都会自动将 _commonModeItems 里的 Source/Observer/Timer 同步到具有 “Common” 标记的所有Mode里，可以通过调用CFRunLoopAddCommonMode()方法将自定义Mode放到 **kCFRunLoopCommonModes**组合）。

**NSRunLoopCommonModes**默认包含了**NSDefaultRunLoopMode**和 **UITrackingRunLoopMode**（注意：当使用NSRunLoopCommonModes时并不是说Runloop会运行在**kCFRunLoopCommonModes**这种模式下，而是相当于runloop分别注册了commonModes下的所有model。)



### 2、RunLoop Source

**CFRunLoopSourceRef** 是事件产生的地方。Source有两个版本：Source0 和 Source1。

- Source0 只包含了一个回调（函数指针），它并不能主动触发事件。使用时，你需要先调用 CFRunLoopSourceSignal(source)，将这个 Source 标记为待处理，然后手动调用 CFRunLoopWakeUp(runloop) 来唤醒 RunLoop，让其处理这个事件。

- Source1 包含了一个 mach_port 和一个回调（函数指针），被用于通过内核和其他线程相互发送消息。这种 Source 能主动唤醒 RunLoop 的线程，其原理在下面会讲到。

苹果文档将RunLoop能够处理的事件分为Input sources和timer事件。下面这张图取自苹果官网:
![](https://youke2.picui.cn/s1/2025/12/17/6942af060bd2a.png)
根据CF的源码，Input source在RunLoop中被分类成source0和source1两大类。source0和source1均有结构体__CFRunLoopSource表示：

```
struct __CFRunLoopSource {
    CFRuntimeBase _base;
    uint32_t _bits;
    pthread_mutex_t _lock;
    CFIndex _order;         /* 优先级，越小，优先级越高。可以是负数。immutable */
    CFMutableBagRef _runLoops;
    union {  // 联合，用于保存source的信息，同时可以区分source是0还是1类型
        CFRunLoopSourceContext version0;    /* immutable, except invalidation */
        CFRunLoopSourceContext1 version1;   /* immutable, except invalidation */
    } _context;
};

typedef struct {
    CFIndex version;      // 类型:source0
    void *  info;
    const void *(*retain)(const void *info);
    void    (*release)(const void *info);
    CFStringRef (*copyDescription)(const void *info);
    Boolean (*equal)(const void *info1, const void *info2);
    CFHashCode  (*hash)(const void *info);
    void    (*schedule)(void *info, CFRunLoopRef rl, CFRunLoopMode mode);
    void    (*cancel)(void *info, CFRunLoopRef rl, CFRunLoopMode mode);
    void    (*perform)(void *info);  // call out 
} CFRunLoopSourceContext;

typedef struct {
    CFIndex version;  // 类型:source1
    void *  info;
    const void *(*retain)(const void *info);
    void    (*release)(const void *info);
    CFStringRef (*copyDescription)(const void *info);
    Boolean (*equal)(const void *info1, const void *info2);
    CFHashCode  (*hash)(const void *info);
#if (TARGET_OS_MAC && !(TARGET_OS_EMBEDDED || TARGET_OS_IPHONE)) || (TARGET_OS_EMBEDDED || TARGET_OS_IPHONE)
    mach_port_t (*getPort)(void *info);
    void *  (*perform)(void *msg, CFIndex size, CFAllocatorRef allocator, void *info);
#else
    void *  (*getPort)(void *info);
    void    (*perform)(void *info); // call out
#endif
} CFRunLoopSourceContext1;
```

source0和source1由联合_context来做代码区分：

- 相同
   1. 均是__CFRunLoopSource类型，这就像一个协议，我们甚至可以自己拓展__CFRunLoopSource，定义自己的source。
   2. 均是需要被Signaled后，才能够被处理。
   3. 处理时，均是调用__CFRunLoopSource._context.version(0?1).perform,其实这就是调用一个函数指针。

- 不同
    1. source0需要手动signaled，source1系统会自动signaled
    2. source0需要手动唤醒RunLoop，才能够被处理: CFRunLoopWakeUp(CFRunLoopRef rl)。而source1 会自动唤醒(通过mach port)RunLoop来处理。

- 总结：
   1. Source1 :基于mach_Port的,来自系统内核或者其他进程或线程的事件，可以主动唤醒休眠中的RunLoop（iOS里进程间通信开发过程中我们一般不主动使用）。mach_port大家就理解成进程间相互发送消息的一种机制就好, 比如屏幕点击, 网络数据的传输都会触发sourse1。
  2. Source0 ：非基于Port的 处理事件，什么叫非基于Port的呢？就是说你这个消息不是其他进程或者内核直接发送给你的。一般是APP内部的事件, 比如hitTest:withEvent的处理, performSelectors的事件。

### 3、RunLoop Timer

**CFRunLoopTimerRef** 是基于时间的触发器，它和 NSTimer 是toll-free bridged 的，可以混用。其包含一个时间长度和一个回调（函数指针）。当其加入到 RunLoop 时，RunLoop会注册对应的时间点，当时间点到时，RunLoop会被唤醒以执行那个回调。

**我们经常使用的timer有几种？**

- NSTimer & PerformSelector:afterDelay:(由RunLoop处理，内部结构为CFRunLoopTimerRef)
- GCD Timer（由GCD自己实现，不通过RunLoop）
- CADisplayLink(通过向RunLoop投递source1 实现回调)

NSObject perform系列函数中的dealy类型, 其实也是一种Timer事件，可能不那么明显：

```
- (void)performSelector:(SEL)aSelector withObject:(nullable id)anArgument afterDelay:(NSTimeInterval)delay inModes:(NSArray<NSRunLoopMode> *)modes;
- (void)performSelector:(SEL)aSelector withObject:(nullable id)anArgument afterDelay:(NSTimeInterval)delay;
```

这种Perform delay的函数底层的实现是和NSTimer一样的，根据苹果官方文档所述：

```
This method sets up a timer to perform the aSelector message on the current thread’s run loop. The timer is configured to run in the default mode (NSDefaultRunLoopMode). When the timer fires, the thread attempts to dequeue the message from the run loop and perform the selector. It succeeds if the run loop is running and in the default mode; otherwise, the timer waits until the run loop is in the default mode.
If you want the message to be dequeued when the run loop is in a mode other than the default mode, use the performSelector:withObject:afterDelay:inModes: method instead. 
```

翻译：

```
此方法设置一个计时器，以便在当前线程的run loop上执行aSelector消息。timer配置为在默认模式（NSDefaultRunLoopMode）下运行。当timer触发时，线程尝试从运行循环中退出消息队列并执行selector。如果run loop正在运行且处于default mode，则会成功；否则，计时器将等待运行循环处于default mode。

如果希望在run loop处于default mode以外的模式时将消息退出队列，请改用performSelector:withObject:afterDelay:inModes:方法。
```


**NSTimer & PerformSelector:afterDelay:**
NSTimer在CF源码中的结构是这样的：

```
struct __CFRunLoopTimer {
    CFRuntimeBase _base;
    uint16_t _bits;
    pthread_mutex_t _lock;
    CFRunLoopRef _runLoop;
    CFMutableSetRef _rlModes;
    CFAbsoluteTime _nextFireDate;
    CFTimeInterval _interval;       /* immutable */
    CFTimeInterval _tolerance;          /* mutable */
    uint64_t _fireTSR;          /* 触发时间，TSR units */
    CFIndex _order;         /* immutable */
    CFRunLoopTimerCallBack _callout;    /* immutable */ // timer 回调
    CFRunLoopTimerContext _context; /* immutable, except invalidation */
};
```

Timer的触发流程大致是这样的：
- 用户添加timer到runloop的某个或几个mode下
- 根据timer是否设置了tolerance，如果没有设置，则调用底层xnu内核的mk_timer注册一个mach-port事件，如果设置了tolerance，则注册一个GCD timer
- 当由XNU内核或GCD管理的timer的 fire time到了，通过对应的mach port唤醒RunLoop(mk_timer对应rlm的_timerPort, GCD timer对应GCD queue port)
- RunLoop执行__CFRunLoopDoTimers，其中会调用__CFRunLoopDoTimer， DoTimer方法里面会根据当前mach time和Timer的fireTSR属性，判断fireTSR 是否< 当前的mach time，如果小于，则触发timer，同时更新下一次fire时间fireTSR。


关于Timer的计时，是通过内核的mach time或GCD time来实现的。

在RunLoop中，NSTimer在激活时，会将休眠中的RunLoop通过_timerPort唤醒,(如果是通过GCD实现的NSTimer，则会通过另一个CGD queue专用mach port)。


### 4、RunLoop Observer


**CFRunLoopObserverRef** 是观察者，每个 Observer 都包含了一个回调（函数指针），当 RunLoop 的状态发生变化时，观察者就能通过回调接受到这个变化。

Observer在CF中的结构如下：

```
struct __CFRunLoopObserver {
    CFRuntimeBase _base;
    pthread_mutex_t _lock;
    CFRunLoopRef _runLoop;
    CFIndex _rlCount;
    CFOptionFlags _activities;      /*所监听的事件，通过位异或，可以监听多种事件 immutable */
    CFIndex _order;         /* 优先级 immutable */
    CFRunLoopObserverCallBack _callout; /* observer 回调 immutable */
    CFRunLoopObserverContext _context;  /* immutable, except invalidation */
};
```

Observer的作用是可以让外部监听RunLoop的运行状态，从而根据不同的时机，做一些操作。
系统会在APP启动时，向main RunLoop里注册了两个 Observer，其回调都是 _wrapRunLoopWithAutoreleasePoolHandler()。

- 第一个 Observer 监视的事件是 Entry(即将进入Loop)，其回调内会调用
_objc_autoreleasePoolPush() 创建自动释放池。其 order 是-2147483647，优先级最高，保证创建释放池发生在其他所有回调之前。

- 第二个 Observer 监视了两个事件： BeforeWaiting(准备进入休眠)
时调用_objc_autoreleasePoolPop() 和 _objc_autoreleasePoolPush()
释放旧的池并创建新池；Exit(即将退出Loop) 时调用 _objc_autoreleasePoolPop() 来释放自动释放池。这个Observer 的 order 是 2147483647，优先级最低，保证其释放池子发生在其他所有回调之后。在主线程执行的代码，通常是写在诸如事件回调、Timer回调内的。这些回调会被 RunLoop 创建好的 AutoreleasePool环绕着，所以不会出现内存泄漏，开发者也不必显示创建 Pool 了。

**Observer可以监听的事件在CF中以位异或表示：**

```
/* Run Loop Observer Activities */
	typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
	    kCFRunLoopEntry = (1UL << 0), // 进入RunLoop 
	    kCFRunLoopBeforeTimers = (1UL << 1), // 即将开始Timer处理
	    kCFRunLoopBeforeSources = (1UL << 2), // 即将开始Source处理
	    kCFRunLoopBeforeWaiting = (1UL << 5), // 即将进入休眠
	    kCFRunLoopAfterWaiting = (1UL << 6), //从休眠状态唤醒
	    kCFRunLoopExit = (1UL << 7), //退出RunLoop
	    kCFRunLoopAllActivities = 0x0FFFFFFFU
	};
```

- kCFRunLoopEntry、kCFRunLoopExit 在每次RunLoop循环中仅调用一次，用于表示即将进入循环和退出循环。
- kCFRunLoopBeforeTimers、kCFRunLoopBeforeSources、kCFRunLoopBeforeWaiting、kCFRunLoopAfterWaiting这些通知会在循环内部发出，可能会调用多次。

相对来说CFRunloopObserverRef理解起来并不复杂，它相当于消息循环中的一个监听器，随时通知外部当前RunLoop的运行状态（它包含一个函数指针_callout_将当前状态及时告诉观察者）。

### 5、mode item

Source/Timer/Observer 被统称为 mode item，一个 item 可以被同时加入多个 mode。但一个 item 被重复加入同一个 mode 时是不会有效果的。如果一个 mode 中一个 item 都没有，则 RunLoop 会直接退出，不进入循环。


# 三、源码解析runloop流程

[苹果runloop源码](https://github.com/opensource-apple/CF/blob/master/CFRunLoop.c)

## 1、入口方法CFRunLoopRun
```
void CFRunLoopRun(void) {	/* DOES CALLOUT */
    int32_t result;
    do {
        result = CFRunLoopRunSpecific(CFRunLoopGetCurrent(), kCFRunLoopDefaultMode, 1.0e10, false);
        CHECK_FOR_FORK();
    } while (kCFRunLoopRunStopped != result && kCFRunLoopRunFinished != result);
}
```

通过代码可以看出来，如果kCFRunLoopRunStopped != result && kCFRunLoopRunFinished != result，则一直会循环执行CFRunLoopRunSpecific函数

## 2、循环执行的函数 

**CFRunLoopRun -> CFRunLoopRunSpecific -> CFRunLoopRun**

### 2.1 CFRunLoopRunSpecific源码

```
SInt32 CFRunLoopRunSpecific(CFRunLoopRef rl, CFStringRef modeName, CFTimeInterval seconds, Boolean returnAfterSourceHandled) {     /* DOES CALLOUT */
    CHECK_FOR_FORK();
    if (__CFRunLoopIsDeallocating(rl)) return kCFRunLoopRunFinished;
    __CFRunLoopLock(rl);
    /// 首先根据modeName找到对应mode
    CFRunLoopModeRef currentMode = __CFRunLoopFindMode(rl, modeName, false);
    /// 如果mode里没有source/timer/observer, 直接返回。
    if (NULL == currentMode || __CFRunLoopModeIsEmpty(rl, currentMode, rl->_currentMode)) {
        Boolean did = false;
        if (currentMode) __CFRunLoopModeUnlock(currentMode);
        __CFRunLoopUnlock(rl);
        return did ? kCFRunLoopRunHandledSource : kCFRunLoopRunFinished;
    }
    volatile _per_run_data *previousPerRun = __CFRunLoopPushPerRunData(rl);
    CFRunLoopModeRef previousMode = rl->_currentMode;
    rl->_currentMode = currentMode;
    int32_t result = kCFRunLoopRunFinished;
    /// 通知 Observers: RunLoop 即将进入 loop。
    if (currentMode->_observerMask & kCFRunLoopEntry ) __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopEntry);
    
    /// 内部函数，进入loop
    result = __CFRunLoopRun(rl, currentMode, seconds, returnAfterSourceHandled, previousMode);
    
    /// 通知 Observers: RunLoop 即将退出。
    if (currentMode->_observerMask & kCFRunLoopExit ) __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopExit);
    
    __CFRunLoopModeUnlock(currentMode);
    __CFRunLoopPopPerRunData(rl, previousPerRun);
    rl->_currentMode = previousMode;
    __CFRunLoopUnlock(rl);
    return result;
}

```

主要逻辑代码：

```
    /// 首先根据modeName找到对应mode
    CFRunLoopModeRef currentMode = __CFRunLoopFindMode(rl, modeName, false);
    
    /// 通知 Observers: RunLoop 即将进入 loop。
	if (currentMode->_observerMask & kCFRunLoopEntry ) __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopEntry);
	
    //进入 loop
	result = __CFRunLoopRun(rl, currentMode, seconds, returnAfterSourceHandled, previousMode);
	
	/// 通知 Observers: RunLoop 即将退出。
    if (currentMode->_observerMask & kCFRunLoopExit ) __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopExit);
```

找到 **RunLoopMode** -> 通知 observers：RunLoop 要开始 -> 进入 RunLoop -> 通知 observers：RunLoop 即将退出。


### 2.2 CFRunLoopRun源码（由于源码量比较大，这里就不全部贴出来了，只贴出来核心步骤）

#### 第一步

开启一个 do while 来保活线程。通知 Observers：RunLoop 会触发 Timer 回调、Source0 回调，接着执行加入的 block

```
// 通知 Observers RunLoop 会触发 Timer 回调
if (rlm->_observerMask & kCFRunLoopBeforeTimers)
 	__CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeTimers);

// 通知 Observers RunLoop 会触发 Source0 回调
if (rlm->_observerMask & kCFRunLoopBeforeSources)
 	__CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeSources);

// 执行 block
__CFRunLoopDoBlocks(rl, rlm);
```

接下来，触发 Source0 回调，如果有 Source1 是 ready 状态的话，就会跳转到 handle_msg 去处理消息。

```
Boolean sourceHandledThisLoop = __CFRunLoopDoSources0(rl, rlm, stopAfterHandle);
if (sourceHandledThisLoop) {
	/// 执行被加入的block
	__CFRunLoopDoBlocks(rl, rlm);
}


if (MACH_PORT_NULL != dispatchPort ) {
    Boolean hasMsg = __CFRunLoopServiceMachPort(dispatchPort, &msg)
    if (hasMsg) goto handle_msg;
}
```



**source0和source1的区别，第二节已经详细说明！！**


#### 第二步

回调触发后，通知 Observers：RunLoop 的线程将进入休眠（sleep）状态。

```

Boolean poll = sourceHandledThisLoop || (0ULL == timeout_context->termTSR);
if (!poll && (currentMode->_observerMask & kCFRunLoopBeforeWaiting)) {
    __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopBeforeWaiting);
}
```


#### 第三步

进入休眠后，会等待 mach_port 的消息，以再次唤醒。只有在下面四个事件出现时才会被再次唤醒：
- 基于 port 的 Source 事件；
- Timer 时间到；
- RunLoop 超时；
- 被调用者唤醒。

```
do {
    __CFRunLoopServiceMachPort(waitSet, &msg, sizeof(msg_buffer), &livePort) {
        // 基于 port 的 Source 事件、调用者唤醒
        if (modeQueuePort != MACH_PORT_NULL && livePort == modeQueuePort) {
            break;
        }
        // Timer 时间到、RunLoop 超时
        if (currentMode->_timerFired) {
            break;
        }
} while (1);
```

#### 第四步

唤醒时通知 Observer：RunLoop 的线程刚刚被唤醒了。

```
if (!poll && (currentMode->_observerMask & kCFRunLoopAfterWaiting))
    __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopAfterWaiting);
```

#### 第五步

RunLoop 被唤醒后就要开始处理消息了：

- 如果是 Timer 时间到的话，就触发 Timer 的回调；
- 如果是 dispatch 的话，就执行 block；
- 如果是 source1 事件的话，就处理这个事件。

消息执行完后，就执行加到 loop 里的 block。

```

handle_msg:
// 如果 Timer 时间到，就触发 Timer 回调
if (msg-is-timer) {
    __CFRunLoopDoTimers(runloop, currentMode, mach_absolute_time())
} 
// 如果 dispatch 就执行 block
else if (msg_is_dispatch) {
    __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__(msg);
} 

// Source1 事件的话，就处理这个事件
else {
    CFRunLoopSourceRef source1 = __CFRunLoopModeFindSourceForMachPort(runloop, currentMode, livePort);
    sourceHandledThisLoop = __CFRunLoopDoSource1(runloop, currentMode, source1, msg);
    if (sourceHandledThisLoop) {
        mach_msg(reply, MACH_SEND_MSG, reply);
    }
}
```

#### 第六步

根据当前 RunLoop 的状态来判断是否需要走下一个 loop。当被外部强制停止或 loop 超时时，就不继续下一个 loop 了，否则继续走下一个 loop 。

```

if (sourceHandledThisLoop && stopAfterHandle) {
     // 事件已处理完
    retVal = kCFRunLoopRunHandledSource;
} else if (timeout) {
    // 超时
    retVal = kCFRunLoopRunTimedOut;
} else if (__CFRunLoopIsStopped(runloop)) {
    // 外部调用者强制停止
    retVal = kCFRunLoopRunStopped;
} else if (__CFRunLoopModeIsEmpty(runloop, currentMode)) {
    // mode 为空，RunLoop 结束
    retVal = kCFRunLoopRunFinished;
}
```

整个 RunLoop 过程，我们可以总结为如下所示的一张图片。
![](https://youke2.picui.cn/s1/2025/12/17/6942aa6c9c9f0.png)

#### 总结

将整个流程总结成伪代码如下：

```

/// RunLoop的实现
int CFRunLoopRunSpecific(runloop, modeName, seconds, stopAfterHandle) {
 
    /// 首先根据modeName找到对应mode
    CFRunLoopModeRef currentMode = __CFRunLoopFindMode(runloop, modeName, false);
    /// 如果mode里没有source/timer/observer, 直接返回。
    if (__CFRunLoopModeIsEmpty(currentMode)) return;
 
    /// 1. 通知 Observers: RunLoop 即将进入 loop。
    __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopEntry);
 
    /// 内部函数，进入loop
    __CFRunLoopRun(runloop, currentMode, seconds, returnAfterSourceHandled) {
 
        Boolean sourceHandledThisLoop = NO;
        int retVal = 0;
        do {
 
            /// 2. 通知 Observers: RunLoop 即将触发 Timer 回调。
            __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopBeforeTimers);
            /// 3. 通知 Observers: RunLoop 即将触发 Source0 (非port) 回调。
            __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopBeforeSources);
            /// 执行被加入的block
            __CFRunLoopDoBlocks(runloop, currentMode);
 
            /// 4. RunLoop 触发 Source0 (非port) 回调。
            sourceHandledThisLoop = __CFRunLoopDoSources0(runloop, currentMode, stopAfterHandle);
            /// 执行被加入的block
            __CFRunLoopDoBlocks(runloop, currentMode);
 
            /// 5. 如果有 Source1 (基于port) 处于 ready 状态，直接处理这个 Source1 然后跳转去处理消息。
            if (__Source0DidDispatchPortLastTime) {
                Boolean hasMsg = __CFRunLoopServiceMachPort(dispatchPort, &msg)
                if (hasMsg) goto handle_msg;
            }
 
            /// 通知 Observers: RunLoop 的线程即将进入休眠(sleep)。
            if (!sourceHandledThisLoop) {
                __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopBeforeWaiting);
            }
 
            /// 7. 调用 mach_msg 等待接受 mach_port 的消息。线程将进入休眠, 直到被下面某一个事件唤醒。
            /// 一个基于 port 的Source 的事件。
            /// 一个 Timer 到时间了
            /// RunLoop 自身的超时时间到了
            /// 被其他什么调用者手动唤醒
            __CFRunLoopServiceMachPort(waitSet, &msg, sizeof(msg_buffer), &livePort) {
                mach_msg(msg, MACH_RCV_MSG, port); // thread wait for receive msg
            }
 
            /// 8. 通知 Observers: RunLoop 的线程刚刚被唤醒了。
            __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopAfterWaiting);
 
            /// 收到消息，处理消息。
            handle_msg:
 
            /// 9.1 如果一个 Timer 到时间了，触发这个Timer的回调。
            if (msg_is_timer) {
                __CFRunLoopDoTimers(runloop, currentMode, mach_absolute_time())
            } 
 
            /// 9.2 如果有dispatch到main_queue的block，执行block。
            else if (msg_is_dispatch) {
                __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__(msg);
            } 
 
            /// 9.3 如果一个 Source1 (基于port) 发出事件了，处理这个事件
            else {
                CFRunLoopSourceRef source1 = __CFRunLoopModeFindSourceForMachPort(runloop, currentMode, livePort);
                sourceHandledThisLoop = __CFRunLoopDoSource1(runloop, currentMode, source1, msg);
                if (sourceHandledThisLoop) {
                    mach_msg(reply, MACH_SEND_MSG, reply);
                }
            }
 
            /// 执行加入到Loop的block
            __CFRunLoopDoBlocks(runloop, currentMode);
 
            if (sourceHandledThisLoop && stopAfterHandle) {
                /// 进入loop时参数说处理完事件就返回。
                retVal = kCFRunLoopRunHandledSource;
            } else if (timeout) {
                /// 超出传入参数标记的超时时间了
                retVal = kCFRunLoopRunTimedOut;
            } else if (__CFRunLoopIsStopped(runloop)) {
                /// 被外部调用者强制停止了
                retVal = kCFRunLoopRunStopped;
            } else if (__CFRunLoopModeIsEmpty(runloop, currentMode)) {
                /// source/timer/observer一个都没有了
                retVal = kCFRunLoopRunFinished;
            }
 
            /// 如果没超时，mode里没空，loop也没被停止，那继续loop。
        } while (retVal == 0);
    }
 
    /// 10. 通知 Observers: RunLoop 即将退出。
    __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopExit);
}

```
**下图描述了Runloop运行流程**

![](https://youke2.picui.cn/s1/2025/12/17/6942ac62e81fd.png)


注意的是尽管CFRunLoopPerformBlock在上图中作为唤醒机制有所体现，但事实上执行CFRunLoopPerformBlock只是入队，下次RunLoop运行才会执行，而如果需要立即执行则必须调用CFRunLoopWakeUp。




# 四、Runloop和线程的关系
RunLoop和线程是息息相关的,我们知道线程的作用是用来执行特定的一个或多个任务,但是在默认情况下,线程执行完之后就会退出,就不能再执行任务了。这时我们就需要采用一种方式来让线程能够处理任务,并不退出。所以,我们就有了RunLoop。

iOS开发中能遇到两个线程对象: pthread_t和NSThread，pthread_t和NSThread 是一一对应的。比如，你可以通过 pthread_main_thread_np()或 [NSThread mainThread]来获取主线程；也可以通过pthread_self()或[NSThread currentThread]来获取当前线程。CFRunLoop 是基于 pthread 来管理的。

线程与RunLoop是一一对应的关系（对应关系保存在一个全局的Dictionary里），线程创建之后是没有RunLoop的（主线程除外），RunLoop的创建是发生在第一次获取时,销毁则是在线程结束的时候。只能在当前线程中操作当前线程的RunLoop,而不能去操作其他线程的RunLoop。

### 一"码"当先


```

/// 全局的Dictionary，key 是 pthread_t， value 是 CFRunLoopRef
static CFMutableDictionaryRef loopsDic;
/// 访问 loopsDic 时的锁
static CFSpinLock_t loopsLock;
 
/// 获取一个 pthread 对应的 RunLoop。
CFRunLoopRef _CFRunLoopGet(pthread_t thread) {
    OSSpinLockLock(&loopsLock);
    
    if (!loopsDic) {
        // 第一次进入时，初始化全局Dic，并先为主线程创建一个 RunLoop。
        loopsDic = CFDictionaryCreateMutable();
        CFRunLoopRef mainLoop = _CFRunLoopCreate();
        CFDictionarySetValue(loopsDic, pthread_main_thread_np(), mainLoop);
    }
    
    /// 直接从 Dictionary 里获取。
    CFRunLoopRef loop = CFDictionaryGetValue(loopsDic, thread));
    
    if (!loop) {
        /// 取不到时，创建一个
        loop = _CFRunLoopCreate();
        CFDictionarySetValue(loopsDic, thread, loop);
        /// 注册一个回调，当线程销毁时，顺便也销毁其对应的 RunLoop。
        _CFSetTSD(..., thread, loop, __CFFinalizeRunLoop);
    }
    
    OSSpinLockUnLock(&loopsLock);
    return loop;
}
 
CFRunLoopRef CFRunLoopGetMain() {
    return _CFRunLoopGet(pthread_main_thread_np());
}
 
CFRunLoopRef CFRunLoopGetCurrent() {
    return _CFRunLoopGet(pthread_self());
}
```

苹果开发的接口中并没有直接创建Runloop的接口，如果需要使用Runloop通常CFRunLoopGetMain()和CFRunLoopGetCurrent()两个方法来获取（通过上面的源代码也可以看到，核心逻辑在_CFRunLoopGet_当中）,通过代码并不难发现其实只有当我们使用线程的方法主动get Runloop时才会在第一次创建该线程的Runloop，同时将它保存在全局的Dictionary中（线程和Runloop二者一一对应），默认情况下线程并不会创建Runloop（主线程的Runloop比较特殊，任何线程创建之前都会保证主线程已经存在Runloop），同时在线程结束的时候也会销毁对应的Runloop。


iOS开发过程中对于开发者而言更多的使用的是NSRunloop,它默认提供了三个常用的run方法：

```
- (void)run; 
- (void)runUntilDate:(NSDate *)limitDate;
- (BOOL)runMode:(NSRunLoopMode)mode beforeDate:(NSDate *)limitDate;

```

- 1.使用第一种方式runLoop会一直运行下去，在此期间会处理来自输入源的数据，并且会在NSDefaultRunLoopMode模式下重复调用 - (BOOL)runMode:(NSRunLoopMode)mode beforeDate:(NSDate *)limitDate方法 

- 2.使用第二种启动方式，可以设置超时时间，在超时时间到达之前，runLoop会一直运行，在此期间runLoop会处理来自输入源的数据，并且也会在NSDefaultRunLoopMode模式下重复调用 - (BOOL)runMode:(NSRunLoopMode)mode beforeDate:(NSDate *)limitDate;方法 

- 3.使用第三种方法runLoop会运行一次，超时时间到达或者一个输入源被处理，则runLoop就会自动退出 


 
至此runloop的底层实现原理已经大致做了介绍，之后会更新runloop的应用篇，详细介绍iOS中苹果对runloop的应用及一些知名三方SDK对runloop的实战应用！！



# 五、扩展知识 - Call out

当 RunLoop 进行回调时，一般都是通过一个很长的函数调用出去 (call out), 当你在你的代码中下断点调试时，通常能在调用栈上看到这些函数。下面是这几个函数的整理版本:

```
	static void __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__();
	static void __CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__();
	static void __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__();
	static void __CFRUNLOOP_IS_CALLING_OUT_TO_A_TIMER_CALLBACK_FUNCTION__();
	static void __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__();
	static void __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE1_PERFORM_FUNCTION__();
```


如果你在调用栈中看到这些长函数名，在这里查找一下就能定位到具体的调用地点了：

```
{
    /// 1. 通知Observers，即将进入RunLoop
    /// 此处有Observer会创建AutoreleasePool: _objc_autoreleasePoolPush();
    __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopEntry);
    do {
 
        /// 2. 通知 Observers: 即将触发 Timer 回调。
        __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopBeforeTimers);
        /// 3. 通知 Observers: 即将触发 Source (非基于port的,Source0) 回调。
        __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopBeforeSources);
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__(block);
 
        /// 4. 触发 Source0 (非基于port的) 回调。
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__(source0);
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__(block);
 
        /// 6. 通知Observers，即将进入休眠
        /// 此处有Observer释放并新建AutoreleasePool: _objc_autoreleasePoolPop(); _objc_autoreleasePoolPush();
        __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopBeforeWaiting);
 
        /// 7. sleep to wait msg.
        mach_msg() -> mach_msg_trap();
        
 
        /// 8. 通知Observers，线程被唤醒
        __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopAfterWaiting);
 
        /// 9. 如果是被Timer唤醒的，回调Timer
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_TIMER_CALLBACK_FUNCTION__(timer);
 
        /// 9. 如果是被dispatch唤醒的，执行所有调用 dispatch_async 等方法放入main queue 的 block
        __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__(dispatched_block);
 
        /// 9. 如果如果Runloop是被 Source1 (基于port的) 的事件唤醒了，处理这个事件
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE1_PERFORM_FUNCTION__(source1);
 
 
    } while (...);
 
    /// 10. 通知Observers，即将退出RunLoop
    /// 此处有Observer释放AutoreleasePool: _objc_autoreleasePoolPop();
    __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopExit);
}
```

**代码测试：**

在控制器的touchBegin中打入断点查看堆栈（由于UIEvent是Source0，所以可以看到一个Source0的Call out函数CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION调用）：

![](https://youke2.picui.cn/s1/2025/12/17/6942af061a1e2.png)



参考资料：
[iOS刨根问底-深入理解RunLoop](https://www.cnblogs.com/kenshincui/p/6823841.html)
[如何利用 RunLoop 原理去监控卡顿？](https://time.geekbang.org/column/article/89494)
