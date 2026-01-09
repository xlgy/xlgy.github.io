---
layout:     post
title:      深入理解 NSProxy：Objective-C 中被忽视的消息转发基石
subtitle:   深入理解 NSProxy：Objective-C 中被忽视的消息转发基石
date:       2026-01-09
author:     LXY
header-img: img/home1.jpeg
catalog: true
tags:
    - iOS
---

# 深入理解 NSProxy：Objective-C 中被忽视的消息转发基石

在 Objective-C 运行时体系中，**NSProxy** 是一个非常特殊、但又极其重要的类。它不是 NSObject 的子类，却几乎参与了所有高级消息转发、AOP、弱代理、远程消息等设计模式的实现。

本文将从 **设计背景 → 类结构 → 消息转发流程 → 常见使用场景 → 实战示例 → 常见坑**，系统、深入地讲清楚 NSProxy。



## 一、NSProxy 是什么？

官方文档对 NSProxy 的一句话定义是：

> NSProxy is an abstract superclass defining an API for objects that act as stand-ins for other objects.

翻译过来：

> NSProxy 是一个抽象基类，用来作为“替身对象（代理）”。

**关键词：抽象、替身、转发。**

### 1️⃣ NSProxy 不是 NSObject 的子类

```
NSObject
   ↑
  YourClass

NSProxy
   ↑
  YourProxyClass
```

这是理解 NSProxy 的第一个关键点：

* NSProxy **不继承** NSObject
* 但它同样是一个 Objective-C 对象
* 它同样可以接收消息（objc_msgSend）

NSProxy 和 NSObject **平级**，都是 Root Class。



## 二、为什么要有 NSProxy？

### 1️⃣ NSObject 太“重”了

NSObject 默认实现了大量方法：

* 生命周期管理（retain / release）
* KVC / KVO
* isEqual / hash
* respondsToSelector:
* forwardingTargetForSelector:

但在某些场景下，我们**不需要这些能力**，只想：

> 把所有消息“原封不动地转发给另一个对象”

如果继承 NSObject：

* 你需要处理大量无关方法
* 很容易拦截不到你真正想转发的方法

### 2️⃣ NSProxy 专为“消息转发”而生

NSProxy 的设计目标非常纯粹：

> **最小成本地参与 Objective-C 消息转发机制**

所以它：

* 只实现极少数方法
* 强制子类参与消息转发



## 三、NSProxy 的类结构

### 1️⃣ NSProxy 的头文件（精简后）

```objc
@interface NSProxy <NSObject>

+ (id)alloc;
+ (id)allocWithZone:(NSZone *)zone;

- (void)forwardInvocation:(NSInvocation *)invocation;
- (NSMethodSignature *)methodSignatureForSelector:(SEL)sel;

@end
```

⚠️ 注意：

* **没有 init 方法**
* **没有 ivar**
* **没有 dealloc（由 runtime 处理）**

### 2️⃣ 必须实现的两个方法

如果你继承 NSProxy，**至少要实现**：

```objc
- (NSMethodSignature *)methodSignatureForSelector:(SEL)sel;
- (void)forwardInvocation:(NSInvocation *)invocation;
```

否则：

* 程序必然 crash
* 抛出 `doesNotRecognizeSelector:`

---

## 四、NSProxy 与消息转发机制

### 1️⃣ Objective-C 消息发送完整流程

```text
objc_msgSend
   ↓
查找方法缓存
   ↓
查找方法列表
   ↓
+resolveInstanceMethod:
   ↓
-forwardingTargetForSelector:
   ↓
-methodSignatureForSelector:
   ↓
-forwardInvocation:
   ↓
-doesNotRecognizeSelector:
```

### 2️⃣ NSProxy 从哪一步开始？

**NSProxy 直接跳过前面所有步骤，进入：

```
- methodSignatureForSelector:
- forwardInvocation:
```

也就是说：

> NSProxy 是“为完整消息转发量身定制的对象”。



## 五、一个最小 NSProxy 示例

### 1️⃣ 定义代理类

```
@interface MyProxy : NSProxy
@property (nonatomic, weak) id target;
+ (instancetype)proxyWithTarget:(id)target;
@end
```

### 2️⃣ 实现

```
@implementation MyProxy

+ (instancetype)proxyWithTarget:(id)target {
    MyProxy *proxy = [MyProxy alloc];
    proxy.target = target;
    return proxy;
}

- (NSMethodSignature *)methodSignatureForSelector:(SEL)sel {
    return [self.target methodSignatureForSelector:sel];
}

- (void)forwardInvocation:(NSInvocation *)invocation {
    [invocation invokeWithTarget:self.target];
}

@end
```

### 3️⃣ 使用

```objc
MyProxy *proxy = [MyProxy proxyWithTarget:obj];
[proxy doSomething]; // 实际执行 obj 的方法
```



## 六、NSProxy 的经典使用场景

### 1️⃣ 弱代理（解决 NSTimer / CADisplayLink 循环引用）

```
NSTimer *timer = [NSTimer scheduledTimerWithTimeInterval:1
                                                   target:proxy
                                                 selector:@selector(tick)
                                                 userInfo:nil
                                                  repeats:YES];
```

### 2️⃣ AOP / 方法拦截

```objc
- (void)forwardInvocation:(NSInvocation *)invocation {
    NSLog(@"before");
    [invocation invokeWithTarget:self.target];
    NSLog(@"after");
}
```

### 3️⃣ 多对象代理（消息分发、模拟多继承）

```objc
for (id target in self.targets) {
    if ([target respondsToSelector:invocation.selector]) {
        [invocation invokeWithTarget:target];
    }
}
```



## 七、NSProxy 常见坑

### ❌ 1. methodSignatureForSelector 返回 nil

```objc
return nil; // 一定会 crash
```

### ❌ 2. target 提前释放

```objc
@property (nonatomic, weak) id target;
```

* target 为 nil
* invocation 调用无效

### ❌ 3. respondsToSelector: 不一致

建议重写：

```objc
- (BOOL)respondsToSelector:(SEL)aSelector {
    return [self.target respondsToSelector:aSelector];
}
```

**NSProxy 默认的 respondsToSelector 有什么问题？**

NSProxy 并不知道 target 能不能响应

NSProxy 本身：

- ❌ 没有方法列表

- ❌ 不走 normal method lookup

- ❌ 所有方法都要靠转发

但如果你不重写：

```
[proxy respondsToSelector:@selector(foo)]
```

得到的结果通常是：

- ❌ 返回 NO（即使 target 能处理）

或返回结果与真实能力不一致



## 八、NSProxy vs NSObject（对比总结）

| 对比项           | NSObject | NSProxy       |
| ------------- | -------- | ------------- |
| 是否 Root Class | 是        | 是             |
| 是否可直接用        | 可以       | 不可以           |
| 消息转发          | 可选       | 必须            |
| 使用复杂度         | 低        | 高             |
| 适合场景          | 普通对象     | 代理 / 转发 / AOP |



## 九、什么时候该用 NSProxy？

✅ 使用 NSProxy：

* 你需要**完全控制消息转发**
* 你要实现通用代理 / AOP
* 你不希望 NSObject 干扰转发逻辑
*  模拟多继承

❌ 不要滥用：

* 普通对象
* 只是想 override 几个方法


## 十、实战使用


### 设计MultiProxy进行消息转发


**MultiProxy.h**

```
#import <Foundation/Foundation.h>

NS_ASSUME_NONNULL_BEGIN

@interface MultiProxy : NSProxy

@property (nonatomic, copy) NSArray *targets;

+ (instancetype)proxyWithTargets:(NSArray *)targets;
@end

NS_ASSUME_NONNULL_END
```

**MultiProxy.m**

```
#import "MultiProxy.h"

@implementation MultiProxy

+ (instancetype)proxyWithTargets:(NSArray *)targets {
    MultiProxy *proxy = [MultiProxy alloc];
    proxy.targets = targets;
    return proxy;
}

- (NSMethodSignature *)methodSignatureForSelector:(SEL)sel {
    for (id target in self.targets) {
        if ([target respondsToSelector:sel]) {
            return [target methodSignatureForSelector:sel];
        }
    }
    return nil; // 不支持 → crash（符合 OC 语义）
}

- (void)forwardInvocation:(NSInvocation *)invocation {
    SEL sel = invocation.selector;

    for (id target in self.targets) {
        if ([target respondsToSelector:sel]) {
            [invocation invokeWithTarget:target];
            return;
        }
    }
}

- (BOOL)respondsToSelector:(SEL)aSelector{
    for (id target in self.targets) {
        if ([target respondsToSelector:aSelector]) {
            return true;
        }
    }
    return  false;
}

@end

```


### 使用示例

- 分别定义两个类

```
@interface Class1 : NSObject

-(void)class1Test;

@end

@interface Class2 : NSObject


-(void)class2Test;

@end
```


- 使用

```
   
    Class1 *c1 = [Class1 new];
    Class2 *c2 = [Class2 new];

    id proxy = [MultiProxy proxyWithTargets:@[c2,c1]];

    [proxy class1Test];
    [proxy class2Test];
```

>  这种方式可以借用NSProxy模拟实现多继承，好比MultiProxy同时继承与Class1和Class2

### 注意

当使用多对象消息转发的NSProxy时，isKindOfClass会失效，和respondsToSelector类似，只返回第一个Target的结果，如果想要解决这个问题，我们在使用多对象转发时也需要重写isKindOfClass

```
- (BOOL)isKindOfClass:(Class)aClass{
    for (id target in self.targets) {
        if ([target isKindOfClass:aClass]) {
            return true;
        }
    }
    return  false;
}
```


## 十一、三方库设计思路探究

>早些年使用过一个借助NSProxy  实现 AOP 的三方库：AOP-in-Objective-C，在这里简单看一下这个三方库的设计思路

先看如何使用

```

- (void) addInterceptor:(NSInvocation *) i {
    NSLog(@"ADD intercepted.");
}

- (void) removeInterceptor:(NSInvocation *) i {
    NSLog(@"REMOVE END intercepted !");
}

- (void) testAOP {
 
    NSMutableArray* testArray = [[AOPProxy alloc] initWithNewInstanceOfClass:[NSMutableArray class]];
    
    [(AOPProxy*)testArray interceptMethodStartForSelector:@selector(addObject:)
                                    withInterceptorTarget:self
                                      interceptorSelector:@selector( addInterceptor: )];
    
    [(AOPProxy*)testArray interceptMethodEndForSelector:@selector(removeObjectAtIndex:)
                                  withInterceptorTarget:self
                                    interceptorSelector:@selector( removeInterceptor: )];
    
    [testArray addObject:[NSNumber numberWithInt:1]];
    
    [testArray removeObjectAtIndex:0];
    
    [testArray release];
    
    
}
```

> 在上述代码中:
如果数组添加对象，就会在数组添加对象前执行addInterceptor
如果数组移除对象，就会在数组添加对象后执行removeInterceptor



如何实现？

```
- (void) interceptMethodStartForSelector:(SEL)sel withInterceptorTarget:(id)target interceptorSelector:(SEL)selector {

    // make sure the target is not nil
    NSParameterAssert( target != nil );

    AOPInterceptorInfo *info = [[AOPInterceptorInfo alloc] init];

    // create the interceptorInfo
    info.interceptedSelector = sel;
    info.interceptorTarget = target;
    info.interceptorSelector = selector;

    // add to our list
    [methodStartInterceptors addObject:info];

    [info release];
}

- (void) interceptMethodEndForSelector:(SEL)sel withInterceptorTarget:(id)target interceptorSelector:(SEL)selector {
    
    // make sure the target is not nil
    NSParameterAssert( target != nil );
    
    AOPInterceptorInfo *info = [[AOPInterceptorInfo alloc] init];
    
    // create the interceptorInfo
    info.interceptedSelector = sel;
    info.interceptorTarget = target;
    info.interceptorSelector = selector;
    
    // add to our list
    [methodEndInterceptors addObject:info];
    
    [info release];
}
```

> 定义两个数组methodStartInterceptors和methodEndInterceptors，分别保存AOPInterceptorInfo对象，AOPInterceptorInfo对象会保存原始target,原始Selector和要插入的Selector，这样在forwardInvocation中就可以遍历执行了

```
- (void)forwardInvocation:(NSInvocation *)anInvocation
{
    SEL aSelector = [anInvocation selector];

    // check if the parent object responds to the selector ...
    if ( [parentObject respondsToSelector:aSelector] ) {

        [anInvocation setTarget:parentObject];

        //
        // Intercept the start of the method.
        //
        
        NSAutoreleasePool *pool = [[NSAutoreleasePool alloc] init];

        for ( int i = 0; i < [methodStartInterceptors count]; i++ ) {

            // first search for this selector ...
            AOPInterceptorInfo *oneInfo = [methodStartInterceptors objectAtIndex:i];

            if ( [oneInfo interceptedSelector] == aSelector ) {

                // extract the interceptor info
                id target = [oneInfo interceptorTarget];
                SEL selector = [oneInfo interceptorSelector];

                // finally invoke the interceptor
                [(NSObject *) target performSelector:selector withObject:anInvocation];
            }
        }

        [pool release];

        //
        // Invoke the original method ...
        //
        
        [self invokeOriginalMethod:anInvocation];

        
        //
        // Intercept the ending of the method.
        //
        
        NSAutoreleasePool *pool2 = [[NSAutoreleasePool alloc] init];
        
        for ( int i = 0; i < [methodEndInterceptors count]; i++ ) {
            
            // first search for this selector ...
            AOPInterceptorInfo *oneInfo = [methodEndInterceptors objectAtIndex:i];
            
            if ( [oneInfo interceptedSelector] == aSelector ) {
                
                // extract the interceptor info
                id target = [oneInfo interceptorTarget];
                SEL selector = [oneInfo interceptorSelector];
                
                // finally invoke the interceptor
                [(NSObject *) target performSelector:selector withObject:anInvocation];
            }
        }
        
        [pool2 release];        
    } 
//    else {
//        [super forwardInvocation:invocation];
//    }
}
```



## 十二、总结

NSProxy 并不常用，但**一旦用上，必定是高级场景**。

它是 Objective-C 消息机制中：

> **最纯粹、最接近 Runtime 的抽象**

