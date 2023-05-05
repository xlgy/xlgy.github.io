---
layout:     post
title:      iOS RunLoop应用分析——原来这些都在使用RunLoop
subtitle:   iOS RunLoop应用分析——原来这些都在使用RunLoop
date:       2023-05-05
author:     LXY
header-img: img/home1.jpeg
catalog: true
tags:
    - iOS
---

```
之前已经介绍过RunLoop原理，感兴趣的同学可以阅读[iOS——解密RunLoop原理](https://xiaozhuanlan.com/topic/9503712864)；本文将从RunLoop的应用场景出发，介绍RunLoop的实际应用。

本文篇幅比较长，创作的目的为了自己日后温习知识所用，希望这篇文章能对你有所帮助。如发现任何有误之处，恳请留言纠正，谢谢。
```

# 1、事件响应
iOS设备的事件响应，是有RunLoop参与的。
提起iOS设备的事件响应，相信大家都会有一个大概的了解:

(1)用户触发事件->(2)系统将事件转交到对应APP的事件队列->(3)APP从消息队列头取出事件->(4)交由Main Window进行消息分发->(5)找到合适的Responder进行处理，如果没找到，则会沿着Responder chain返回到APP层，丢弃不响应该事件

这里涉及到两个问题，(3)到(5)步是由进程内处理的，而(1)到(2)步则涉及到设备硬件，iOS操作系统，以及目标APP之间的通信，通信的大致步骤是什么样的呢？
当我们的APP在接收到任何事件请求之前，main RunLoop都是处于mach_msg_trap休眠状态中的，那么，又是谁唤醒它的呢？

借助lldb，执行：
```
po [NSRunLoop mainRunLoop]
```

打印信息如下：
```
<CFRunLoop 0x60000307c900 [0x1dc655ae0]>{wakeup port = 0x2203, stopped = false, ignoreWakeUps = false, 
current mode = kCFRunLoopDefaultMode,
common modes = <CFBasicHash 0x600000252b20 [0x1dc655ae0]>{type = mutable set, count = 2,
entries =>
	0 : <CFString 0x1d008c078 [0x1dc655ae0]>{contents = "UITrackingRunLoopMode"}
	2 : <CFString 0x1dc88d768 [0x1dc655ae0]>{contents = "kCFRunLoopDefaultMode"}
}
,
common mode items = <CFBasicHash 0x60000023cb70 [0x1dc655ae0]>{type = mutable set, count = 9,
entries =>
	0 : <CFRunLoopSource 0x600003964300 [0x1dc655ae0]>{signalled = No, valid = Yes, order = -1, context = <CFRunLoopSource context>{version = 0, info = 0x0, callout = PurpleEventSignalCallback (0x18c03d28c)}}
	2 : <CFRunLoopObserver 0x600003d7c280 [0x1dc655ae0]>{valid = Yes, activities = 0x20, repeats = Yes, order = 0, callout = _UIGestureRecognizerUpdateObserver (0x1848adbd0), context = <CFRunLoopObserver context 0x60000277c620>}
	3 : <CFRunLoopObserver 0x600003d606e0 [0x1dc655ae0]>{valid = Yes, activities = 0xa0, repeats = Yes, order = 1999000, callout = _beforeCACommitHandler (0x184dc4888), context = <CFRunLoopObserver context 0x151004ae0>}
	4 : <CFRunLoopObserver 0x600003d60500 [0x1dc655ae0]>{valid = Yes, activities = 0xa0, repeats = Yes, order = 2001000, callout = _afterCACommitHandler (0x184dc488c), context = <CFRunLoopObserver context 0x151004ae0>}
	5 : <CFRunLoopSource 0x600003968000 [0x1dc655ae0]>{signalled = No, valid = Yes, order = 0, context = <CFRunLoopSource context>{version = 0, info = 0x6000019568c0, callout = FBSSerialQueueRunLoopSourceHandler (0x18619d338)}}
	6 : <CFRunLoopSource 0x600003960300 [0x1dc655ae0]>{signalled = No, valid = Yes, order = -2, context = <CFRunLoopSource context>{version = 0, info = 0x600000252e80, callout = __eventFetcherSourceCallback (0x184e38034)}}
	9 : <CFRunLoopObserver 0x600003d7c5a0 [0x1dc655ae0]>{valid = Yes, activities = 0xa0, repeats = Yes, order = 2000000, callout = _ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv (0x188468b28), context = <CFRunLoopObserver context 0x0>}
	10 : <CFRunLoopSource 0x600003960180 [0x1dc655ae0]>{signalled = No, valid = Yes, order = -1, context = <CFRunLoopSource context>{version = 0, info = 0x60000377c0d0, callout = __eventQueueSourceCallback (0x184e37f7c)}}
	12 : <CFRunLoopSource 0x6000039606c0 [0x1dc655ae0]>{signalled = No, valid = Yes, order = 0, context = <CFRunLoopSource MIG Server> {port = 24075, subsystem = 0x1d0773ff8, context = 0x600002879140}}
}
,
modes = <CFBasicHash 0x600000252c70 [0x1dc655ae0]>{type = mutable set, count = 4,
entries =>
	2 : <CFRunLoopMode 0x6000037681a0 [0x1dc655ae0]>{name = UITrackingRunLoopMode, port set = 0x4703, queue = 0x600002260400, source = 0x600002261280 (not fired), timer port = 0x4603, 
	sources0 = <CFBasicHash 0x60000023cc00 [0x1dc655ae0]>{type = mutable set, count = 4,
entries =>
	0 : <CFRunLoopSource 0x600003964300 [0x1dc655ae0]>{signalled = No, valid = Yes, order = -1, context = <CFRunLoopSource context>{version = 0, info = 0x0, callout = PurpleEventSignalCallback (0x18c03d28c)}}
	1 : <CFRunLoopSource 0x600003968000 [0x1dc655ae0]>{signalled = No, valid = Yes, order = 0, context = <CFRunLoopSource context>{version = 0, info = 0x6000019568c0, callout = FBSSerialQueueRunLoopSourceHandler (0x18619d338)}}
	2 : <CFRunLoopSource 0x600003960180 [0x1dc655ae0]>{signalled = No, valid = Yes, order = -1, context = <CFRunLoopSource context>{version = 0, info = 0x60000377c0d0, callout = __eventQueueSourceCallback (0x184e37f7c)}}
	4 : <CFRunLoopSource 0x600003960300 [0x1dc655ae0]>{signalled = No, valid = Yes, order = -2, context = <CFRunLoopSource context>{version = 0, info = 0x600000252e80, callout = __eventFetcherSourceCallback (0x184e38034)}}
}
,
	sources1 = <CFBasicHash 0x60000023c6c0 [0x1dc655ae0]>{type = mutable set, count = 1,
entries =>
	0 : <CFRunLoopSource 0x6000039606c0 [0x1dc655ae0]>{signalled = No, valid = Yes, order = 0, context = <CFRunLoopSource MIG Server> {port = 24075, subsystem = 0x1d0773ff8, context = 0x600002879140}}
}
,
	observers = (
    "<CFRunLoopObserver 0x600003d7c280 [0x1dc655ae0]>{valid = Yes, activities = 0x20, repeats = Yes, order = 0, callout = _UIGestureRecognizerUpdateObserver (0x1848adbd0), context = <CFRunLoopObserver context 0x60000277c620>}",
    "<CFRunLoopObserver 0x600003d606e0 [0x1dc655ae0]>{valid = Yes, activities = 0xa0, repeats = Yes, order = 1999000, callout = _beforeCACommitHandler (0x184dc4888), context = <CFRunLoopObserver context 0x151004ae0>}",
    "<CFRunLoopObserver 0x600003d7c5a0 [0x1dc655ae0]>{valid = Yes, activities = 0xa0, repeats = Yes, order = 2000000, callout = _ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv (0x188468b28), context = <CFRunLoopObserver context 0x0>}",
    "<CFRunLoopObserver 0x600003d60500 [0x1dc655ae0]>{valid = Yes, activities = 0xa0, repeats = Yes, order = 2001000, callout = _afterCACommitHandler (0x184dc488c), context = <CFRunLoopObserver context 0x151004ae0>}"
),
	timers = (null),
	currently 663695489 (6974909751689) / soft deadline in: 7.68614046e+11 sec (@ -1) / hard deadline in: 7.68614046e+11 sec (@ -1)
},

	3 : <CFRunLoopMode 0x600003768270 [0x1dc655ae0]>{name = GSEventReceiveRunLoopMode, port set = 0x3803, queue = 0x600002261300, source = 0x600002261400 (not fired), timer port = 0x3903, 
	sources0 = <CFBasicHash 0x60000023ce10 [0x1dc655ae0]>{type = mutable set, count = 1,
entries =>
	0 : <CFRunLoopSource 0x600003964300 [0x1dc655ae0]>{signalled = No, valid = Yes, order = -1, context = <CFRunLoopSource context>{version = 0, info = 0x0, callout = PurpleEventSignalCallback (0x18c03d28c)}}
}
,
	sources1 = <CFBasicHash 0x60000023ca20 [0x1dc655ae0]>{type = mutable set, count = 0,
entries =>
}
,
	observers = (null),
	timers = (null),
	currently 663695489 (6974909768440) / soft deadline in: 7.68614046e+11 sec (@ -1) / hard deadline in: 7.68614046e+11 sec (@ -1)
},

	4 : <CFRunLoopMode 0x6000037680d0 [0x1dc655ae0]>{name = kCFRunLoopDefaultMode, port set = 0x3103, queue = 0x600002260e00, source = 0x600002261080 (not fired), timer port = 0x5103, 
	sources0 = <CFBasicHash 0x60000023cc60 [0x1dc655ae0]>{type = mutable set, count = 4,
entries =>
	0 : <CFRunLoopSource 0x600003964300 [0x1dc655ae0]>{signalled = No, valid = Yes, order = -1, context = <CFRunLoopSource context>{version = 0, info = 0x0, callout = PurpleEventSignalCallback (0x18c03d28c)}}
	1 : <CFRunLoopSource 0x600003968000 [0x1dc655ae0]>{signalled = No, valid = Yes, order = 0, context = <CFRunLoopSource context>{version = 0, info = 0x6000019568c0, callout = FBSSerialQueueRunLoopSourceHandler (0x18619d338)}}
	2 : <CFRunLoopSource 0x600003960180 [0x1dc655ae0]>{signalled = No, valid = Yes, order = -1, context = <CFRunLoopSource context>{version = 0, info = 0x60000377c0d0, callout = __eventQueueSourceCallback (0x184e37f7c)}}
	4 : <CFRunLoopSource 0x600003960300 [0x1dc655ae0]>{signalled = No, valid = Yes, order = -2, context = <CFRunLoopSource context>{version = 0, info = 0x600000252e80, callout = __eventFetcherSourceCallback (0x184e38034)}}
}
,
	sources1 = <CFBasicHash 0x60000023cc30 [0x1dc655ae0]>{type = mutable set, count = 1,
entries =>
	0 : <CFRunLoopSource 0x6000039606c0 [0x1dc655ae0]>{signalled = No, valid = Yes, order = 0, context = <CFRunLoopSource MIG Server> {port = 24075, subsystem = 0x1d0773ff8, context = 0x600002879140}}
}
,
	observers = (
    "<CFRunLoopObserver 0x600003d7c280 [0x1dc655ae0]>{valid = Yes, activities = 0x20, repeats = Yes, order = 0, callout = _UIGestureRecognizerUpdateObserver (0x1848adbd0), context = <CFRunLoopObserver context 0x60000277c620>}",
    "<CFRunLoopObserver 0x600003d606e0 [0x1dc655ae0]>{valid = Yes, activities = 0xa0, repeats = Yes, order = 1999000, callout = _beforeCACommitHandler (0x184dc4888), context = <CFRunLoopObserver context 0x151004ae0>}",
    "<CFRunLoopObserver 0x600003d7c5a0 [0x1dc655ae0]>{valid = Yes, activities = 0xa0, repeats = Yes, order = 2000000, callout = _ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv (0x188468b28), context = <CFRunLoopObserver context 0x0>}",
    "<CFRunLoopObserver 0x600003d60500 [0x1dc655ae0]>{valid = Yes, activities = 0xa0, repeats = Yes, order = 2001000, callout = _afterCACommitHandler (0x184dc488c), context = <CFRunLoopObserver context 0x151004ae0>}"
),
	timers = <CFArray 0x60000287d920 [0x1dc655ae0]>{type = mutable-small, count = 0, values = ()},
	currently 663695489 (6974909768895) / soft deadline in: 7.68614046e+11 sec (@ -1) / hard deadline in: 7.68614046e+11 sec (@ -1)
},

	5 : <CFRunLoopMode 0x600003768b60 [0x1dc655ae0]>{name = kCFRunLoopCommonModes, port set = 0x5c07, queue = 0x600002262b80, source = 0x600002262c80 (not fired), timer port = 0x5d03, 
	sources0 = (null),
	sources1 = (null),
	observers = (null),
	timers = (null),
	currently 663695489 (6974909785136) / soft deadline in: 7.68614046e+11 sec (@ -1) / hard deadline in: 7.68614046e+11 sec (@ -1)
},

}
}

```
这里有一坨，我们从中间找出三个关键的代码段！！

**a、kCFRunLoopDefaultMode**
```
<CFRunLoopMode 0x6000037680d0 [0x1dc655ae0]>{name = kCFRunLoopDefaultMode, port set = 0x3103, queue = 0x600002260e00, source = 0x600002261080 (not fired), timer port = 0x5103, 
	sources0 = <CFBasicHash 0x60000023cc60 [0x1dc655ae0]>{type = mutable set, count = 4,
entries =>
	0 : <CFRunLoopSource 0x600003964300 [0x1dc655ae0]>{signalled = No, valid = Yes, order = -1, context = <CFRunLoopSource context>{version = 0, info = 0x0, callout = PurpleEventSignalCallback (0x18c03d28c)}}
	1 : <CFRunLoopSource 0x600003968000 [0x1dc655ae0]>{signalled = No, valid = Yes, order = 0, context = <CFRunLoopSource context>{version = 0, info = 0x6000019568c0, callout = FBSSerialQueueRunLoopSourceHandler (0x18619d338)}}
	2 : <CFRunLoopSource 0x600003960180 [0x1dc655ae0]>{signalled = No, valid = Yes, order = -1, context = <CFRunLoopSource context>{version = 0, info = 0x60000377c0d0, callout = __eventQueueSourceCallback (0x184e37f7c)}}
	4 : <CFRunLoopSource 0x600003960300 [0x1dc655ae0]>{signalled = No, valid = Yes, order = -2, context = <CFRunLoopSource context>{version = 0, info = 0x600000252e80, callout = __eventFetcherSourceCallback (0x184e38034)}}
}
,
	sources1 = <CFBasicHash 0x60000023cc30 [0x1dc655ae0]>{type = mutable set, count = 1,
entries =>
	0 : <CFRunLoopSource 0x6000039606c0 [0x1dc655ae0]>{signalled = No, valid = Yes, order = 0, context = <CFRunLoopSource MIG Server> {port = 24075, subsystem = 0x1d0773ff8, context = 0x600002879140}}
}
,
	observers = (
    "<CFRunLoopObserver 0x600003d7c280 [0x1dc655ae0]>{valid = Yes, activities = 0x20, repeats = Yes, order = 0, callout = _UIGestureRecognizerUpdateObserver (0x1848adbd0), context = <CFRunLoopObserver context 0x60000277c620>}",
    "<CFRunLoopObserver 0x600003d606e0 [0x1dc655ae0]>{valid = Yes, activities = 0xa0, repeats = Yes, order = 1999000, callout = _beforeCACommitHandler (0x184dc4888), context = <CFRunLoopObserver context 0x151004ae0>}",
    "<CFRunLoopObserver 0x600003d7c5a0 [0x1dc655ae0]>{valid = Yes, activities = 0xa0, repeats = Yes, order = 2000000, callout = _ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv (0x188468b28), context = <CFRunLoopObserver context 0x0>}",
    "<CFRunLoopObserver 0x600003d60500 [0x1dc655ae0]>{valid = Yes, activities = 0xa0, repeats = Yes, order = 2001000, callout = _afterCACommitHandler (0x184dc488c), context = <CFRunLoopObserver context 0x151004ae0>}"
),
	timers = <CFArray 0x60000287d920 [0x1dc655ae0]>{type = mutable-small, count = 0, values = ()},
	currently 663695489 (6974909768895) / soft deadline in: 7.68614046e+11 sec (@ -1) / hard deadline in: 7.68614046e+11 sec (@ -1)
},
```
**b、UITrackingRunLoopMode**
```
x<CFRunLoopMode 0x6000037681a0 [0x1dc655ae0]>{name = UITrackingRunLoopMode, port set = 0x4703, queue = 0x600002260400, source = 0x600002261280 (not fired), timer port = 0x4603, 
	sources0 = <CFBasicHash 0x60000023cc00 [0x1dc655ae0]>{type = mutable set, count = 4,
entries =>
	0 : <CFRunLoopSource 0x600003964300 [0x1dc655ae0]>{signalled = No, valid = Yes, order = -1, context = <CFRunLoopSource context>{version = 0, info = 0x0, callout = PurpleEventSignalCallback (0x18c03d28c)}}
	1 : <CFRunLoopSource 0x600003968000 [0x1dc655ae0]>{signalled = No, valid = Yes, order = 0, context = <CFRunLoopSource context>{version = 0, info = 0x6000019568c0, callout = FBSSerialQueueRunLoopSourceHandler (0x18619d338)}}
	2 : <CFRunLoopSource 0x600003960180 [0x1dc655ae0]>{signalled = No, valid = Yes, order = -1, context = <CFRunLoopSource context>{version = 0, info = 0x60000377c0d0, callout = __eventQueueSourceCallback (0x184e37f7c)}}
	4 : <CFRunLoopSource 0x600003960300 [0x1dc655ae0]>{signalled = No, valid = Yes, order = -2, context = <CFRunLoopSource context>{version = 0, info = 0x600000252e80, callout = __eventFetcherSourceCallback (0x184e38034)}}
}
,
	sources1 = <CFBasicHash 0x60000023c6c0 [0x1dc655ae0]>{type = mutable set, count = 1,
entries =>
	0 : <CFRunLoopSource 0x6000039606c0 [0x1dc655ae0]>{signalled = No, valid = Yes, order = 0, context = <CFRunLoopSource MIG Server> {port = 24075, subsystem = 0x1d0773ff8, context = 0x600002879140}}
}
,
	observers = (
    "<CFRunLoopObserver 0x600003d7c280 [0x1dc655ae0]>{valid = Yes, activities = 0x20, repeats = Yes, order = 0, callout = _UIGestureRecognizerUpdateObserver (0x1848adbd0), context = <CFRunLoopObserver context 0x60000277c620>}",
    "<CFRunLoopObserver 0x600003d606e0 [0x1dc655ae0]>{valid = Yes, activities = 0xa0, repeats = Yes, order = 1999000, callout = _beforeCACommitHandler (0x184dc4888), context = <CFRunLoopObserver context 0x151004ae0>}",
    "<CFRunLoopObserver 0x600003d7c5a0 [0x1dc655ae0]>{valid = Yes, activities = 0xa0, repeats = Yes, order = 2000000, callout = _ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv (0x188468b28), context = <CFRunLoopObserver context 0x0>}",
    "<CFRunLoopObserver 0x600003d60500 [0x1dc655ae0]>{valid = Yes, activities = 0xa0, repeats = Yes, order = 2001000, callout = _afterCACommitHandler (0x184dc488c), context = <CFRunLoopObserver context 0x151004ae0>}"
),
	timers = (null),
	currently 663695489 (6974909751689) / soft deadline in: 7.68614046e+11 sec (@ -1) / hard deadline in: 7.68614046e+11 sec (@ -1)
},
```


在kCFRunLoopDefaultMode，UITrackingRunLoopMode 下面均注册了的source0事件源：
```
<CFRunLoopSource 0x600003960180 [0x1dc655ae0]>{signalled = No, valid = Yes, order = -1, context = <CFRunLoopSource context>{version = 0, info = 0x60000377c0d0, callout = __eventQueueSourceCallback (0x184e37f7c)}}
```
他的回调函数是__eventQueueSourceCallback，没错，APP就是通过这个回调函数来处理事件队列的。

但是，我们注意到了，__eventQueueSourceCallback所对应的source类型是0，也就是说它本身不会唤醒休眠的main RunLoop, main线程自身在休眠状态中也不可能自己去唤醒自己~

所以：**系统肯定还有一个子线程，用来接收事件并唤醒main thread，并将事件传递到main thread上！！**


![查看xcode线程信息](https://images.xiaozhuanlan.com/photo/2022/4ddfe66c6290961a93f00f0c599bd1ba.png)

他的名字叫做:
```
com.apple.uikit.eventfetch-thread
```

看线程的名字就知道，它是UIKit所创建的用于接收event的线程(以下我们简称为event fetch thread)。
我们打印出com.apple.uikit.eventfetch-thread的RunLoop 的 kCFRunLoopDefaultMode。

```
<CFRunLoopMode 0x604000190740 [0x107f54bb0]>{name = kCFRunLoopDefaultMode, port set = 0x1d03, queue = 0x6040001569a0, source = 0x604000190810 (not fired), timer port = 0x2b03, 
    sources0 = <CFBasicHash 0x60800005f8f0 [0x107f54bb0]>{type = mutable set, count = 1,
entries =>
    0 : <CFRunLoopSource 0x608000167440 [0x107f54bb0]>{signalled = No, valid = Yes, order = -1, context = <CFRunLoopSource context>{version = 0, info = 0x600000131940, callout = _UIEventFetcherTriggerHandOff (0x108d71599)}}
}
,
    sources1 = <CFBasicHash 0x608000240b10 [0x107f54bb0]>{type = mutable set, count = 3,
entries =>
    0 : <CFRunLoopSource 0x604000167a40 [0x107f54bb0]>{signalled = No, valid = Yes, order = 0, context = <CFMachPort 0x608000156dc0 [0x107f54bb0]>{valid = Yes, port = 300f, source = 0x604000167a40, callout = __IOHIDEventSystemClientQueueCallback (0x10b17904d), context = <CFMachPort context 0x7f8b28d02ae0>}}
    1 : <CFRunLoopSource 0x6040001678c0 [0x107f54bb0]>{signalled = No, valid = Yes, order = 1, context = <CFMachPort 0x608000157080 [0x107f54bb0]>{valid = Yes, port = 4e0f, source = 0x6040001678c0, callout = __IOMIGMachPortPortCallback (0x10b185692), context = <CFMachPort context 0x6080000d3780>}}
    2 : <CFRunLoopSource 0x604000167980 [0x107f54bb0]>{signalled = No, valid = Yes, order = 0, context = <CFMachPort 0x608000156f20 [0x107f54bb0]>{valid = Yes, port = 4c07, source = 0x604000167980, callout = __IOHIDEventSystemClientAvailabilityCallback (0x10b179281), context = <CFMachPort context 0x7f8b28d02ae0>}}
}
,
    observers = (null),
    timers = (null),
    currently 547998693 (173591812879054) / soft deadline in: 1.84465705e+10 sec (@ -1) / hard deadline in: 1.84465705e+10 sec (@ -1)
},

}
```

里面有sources1，callout: __IOHIDEventSystemClientQueueCallback

**既然这个是source1类型，则是可以被系统通过mach port唤醒event fetch thread的RunLoop， 来执行__IOHIDEventSystemClientQueueCallback回调的。**

**到此破案了！！结论如下：**
- - - - 
用户触发事件， IOKit.framework 生成一个 IOHIDEvent 事件并由 SpringBoard 接收，SpringBoard会利用mach port，产生source1，来唤醒目标APP的com.apple.uikit.eventfetch-thread的RunLoop。Eventfetch thread会将main runloop 中__handleEventQueue所对应的source0设置为signaled == Yes状态，同时唤醒main RunLoop。mainRunLoop则调用__eventQueueSourceCallback进行事件队列处理。
- - - - 

# 2、手势识别

iOS的手势识别也依赖于RunLoop。

当系统识别出一个手势时，会打断touch系列的回调，同时更新标记对应手势的UIGestureRecognizer状态(Needing update?)

而UIKit会向main RunLoop注册一个observer（其实是分别在tracking mode，default mode和common modes下分别注册）:
```
<CFRunLoopObserver 0x600003d7c280 [0x1dc655ae0]>{valid = Yes, activities = 0x20, repeats = Yes, order = 0, callout = _UIGestureRecognizerUpdateObserver (0x1848adbd0), context = <CFRunLoopObserver context 0x60000277c620>}
```
这里查看一下它的activities属性0x20，结合CF中的位标记定义:
```
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
    kCFRunLoopEntry = (1UL << 0),
    kCFRunLoopBeforeTimers = (1UL << 1),
    kCFRunLoopBeforeSources = (1UL << 2),
    kCFRunLoopBeforeWaiting = (1UL << 5),
    kCFRunLoopAfterWaiting = (1UL << 6),
    kCFRunLoopExit = (1UL << 7),
    kCFRunLoopAllActivities = 0x0FFFFFFFU
};

```

可知该observer监听main RunLoop的kCFRunLoopBeforeWaiting事件。每当main RunLoop即将休眠时，该observer被触发，同时调用回调函数_UIGestureRecognizerUpdateObserver。_UIGestureRecognizerUpdateObserver会检测当前需要被更新状态的recognizer（创建，触发，销毁）。

如果有手势被触发，在_UIGestureRecognizerUpdateObserver回调中会借助UIKit一个内部类UIGestureEnvironment 来进行一系列处理。其中会向APP的event queue中投递一个gesture event，这个gesture event的处理流程应该和上面的事件处理类似的，内部会调用__handleEventQueueInternal处理该gesture event，并通过UIKit内部类UIGestureEnvironment 来处理这个gesture event，并最终回调到我们自己所写的gesture回调中。

**手势识别 的过程可以大致总结如下：**
- - - -
当上面的 _UIApplicationHandleEventQueue()识别了一个手势时，其首先会调用 Cancel 将当前的 touchesBegin/Move/End 系列回调打断。随后系统将对应的 UIGestureRecognizer 标记为待处理。

苹果注册了一个 Observer 监测 BeforeWaiting (Loop即将进入休眠) 事件，这个 Observer 的回调函数是 _UIGestureRecognizerUpdateObserver()，其内部会获取所有刚被标记为待处理的 GestureRecognizer，并执行GestureRecognizer 的回调。

当有 UIGestureRecognizer 的变化(创建/销毁/状态改变)时，这个回调都会进行相应处理。
- - - -

# 3、页面的渲染
- 当我们调用 [UIView setNeedsDisplay] 时，这时会调用当前 View.layer 的 [view.layer setNeedsDisplay]方法。

- 这等于给当前的 layer 打上了一个标记，而此时并没有直接进行绘制工作。而是会到当前的 Runloop 即将休眠，也就是 beforeWaiting 时才会进行绘制工作。

- 紧接着会调用 [CALayer display]，进入到真正绘制的工作。CALayer 层会判断自己的 delegate 有没有实现异步绘制的代理方法 displayer:，这个代理方法是异步绘制的入口，如果没有实现这个方法，那么会继续进行系统绘制的流程，然后绘制结束。

- CALayer 内部会创建一个 Backing Store，用来获取图形上下文。接下来会判断这个 layer 是否有 delegate。

- 如果有的话，会调用 [layer.delegate drawLayer:inContext:]，并且会返回给我们 [UIView DrawRect:] 的回调，让我们在系统绘制的基础之上再做一些事情。

- 如果没有 delegate，那么会调用 [CALayer drawInContext:]。

- 以上两个分支，最终 CALayer 都会将位图提交到 Backing Store，最后提交给 GPU。

- 至此绘制的过程结束。

# 4、NSTimer

NSTimer 其实就是 CFRunLoopTimer，他们之间是 toll-free bridged 的
一"码"当先
```
typedef struct CF_BRIDGED_MUTABLE_TYPE(NSTimer) __CFRunLoopTimer * CFRunLoopTimerRef;

struct __CFRunLoopTimer {
    CFRuntimeBase _base;
    uint16_t _bits;
    pthread_mutex_t _lock;
    CFRunLoopRef _runLoop;         // timer对应的runloop
    CFMutableSetRef _rlModes;      // timer对应的runloop modes
    CFAbsoluteTime _nextFireDate;
    CFTimeInterval _interval;       /* immutable */
    CFTimeInterval _tolerance;          /* mutable */
    uint64_t _fireTSR;          /* timer本次需要被触发的时间， TSR units */
    CFIndex _order;         /* immutable */
    CFRunLoopTimerCallBack _callout;    /* timer 回调 immutable */
    CFRunLoopTimerContext _context; /* immutable, except invalidation */
};

```

Timer的触发流程大致是这样的：
- 用户添加timer到runloop的某个或几个mode下；
- 根据timer是否设置了tolerance，如果没有设置，则调用底层xnu内核的mk_timer注册一个mach-port事件，如果设置了tolerance，则注册一个GCD timer；
- 当由XNU内核或GCD管理的timer的 fire time到了，通过对应的mach port唤醒RunLoop(mk_timer对应rlm的_timerPort, GCD timer对应GCD queue port)；
- RunLoop执行__CFRunLoopDoTimers，其中会调用__CFRunLoopDoTimer， DoTimer方法里面会根据当前mach time和Timer的fireTSR属性，判断fireTSR 是否< 当前的mach time，如果小于，则触发timer，同时更新下一次fire时间fireTSR。

# 5、RunLoop和线程应用场景
- 线程和RunLoop是一一对应的,其映射关系是保存在一个全局的 Dictionary 里
- 自己创建的线程默认是没有开启RunLoop的

###怎么创建一个常驻线程？
1. 为当前线程开启一个RunLoop（第一次调用 [NSRunLoop currentRunLoop]方法时实际是会先去创建一个RunLoop）
2. 向当前RunLoop中添加一个Port/Source等维持RunLoop的事件循环（如果RunLoop的mode中一个item都没有，RunLoop会退出）
3. 启动该RunLoop
```
  @autoreleasepool {

        NSRunLoop *runLoop = [NSRunLoop currentRunLoop];

        [[NSRunLoop currentRunLoop] addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode];

        [runLoop run];

    }
```

###怎样保证子线程数据回来更新UI的时候不打断用户的滑动操作？

当我们在子请求数据的同时滑动浏览当前页面，如果数据请求成功要切回主线程更新UI，那么就会影响当前正在滑动的体验。

我们就可以将**更新UI事件放在主线程的NSDefaultRunLoopMode上**执行即可，这样就会等用户不再滑动页面，主线程RunLoop由UITrackingRunLoopMode切换到NSDefaultRunLoopMode时再去更新UI
```
[self performSelectorOnMainThread:@selector(reloadData) withObject:nil waitUntilDone:NO modes:@[NSDefaultRunLoopMode]];
```

###易错代码分析
```
NSLog(@"1");
dispatch_async(dispatch_get_global_queue(0, 0), ^{
    NSLog(@"2");
    [self performSelector:@selector(test) withObject:nil afterDelay:10];
    NSLog(@"3");
});
NSLog(@"4");

- (void)test{
    NSLog(@"5");
}
```
**打印结果:**
1423，test方法并不会执行。

**原因分析:**
如果是带afterDelay的延时函数，会在内部创建一个 NSTimer，然后添加到当前线程的RunLoop中。由于当前线程没有开启RunLoop，所以该方法会失效。

```
NSLog(@"1");
dispatch_async(dispatch_get_global_queue(0, 0), ^{
    NSLog(@"2");
    [[NSRunLoop currentRunLoop] run];
    [self performSelector:@selector(test) withObject:nil afterDelay:10];
    NSLog(@"3");
});
NSLog(@"4");

- (void)test{
    NSLog(@"5");
}
```
**打印结果:**
1423，test方法并不会执行。

**原因分析:**
如果RunLoop的mode中一个item都没有，RunLoop会退出。即在调用RunLoop的run方法后，由于其mode中没有添加任何item去维持RunLoop的时间循环，RunLoop随即还是会退出。
所以我们自己启动RunLoop，一定要在添加item。


```
NSLog(@"1");
dispatch_async(dispatch_get_global_queue(0, 0), ^{
    NSLog(@"2");
    [self performSelector:@selector(test) withObject:nil afterDelay:10];
    [[NSRunLoop currentRunLoop] run];
    NSLog(@"3");
});
NSLog(@"4");

- (void)test{
    NSLog(@"5");
}
```

**打印结果:**
14253，test方法会执行。

# 6、PerformSelector 相关
###PerformSelector 的实现原理？
当调用 NSObject 的 performSelecter:afterDelay: 后，实际上其内部会创建一个 Timer 并添加到当前线程的 RunLoop 中。所以如果当前线程没有 RunLoop，则这个方法会失效。

当调用 performSelector:onThread: 时，实际上其会创建一个 Timer 加到对应的线程去，同样的，如果对应线程没有 RunLoop 该方法也会失效。


###PerformSelector:afterDelay:这个方法在子线程中是否起作用？为什么？怎么解决？
不起作用，子线程默认没有 Runloop，也就没有 Timer。
解决的办法是可以使用 GCD 来实现：Dispatch_after


     