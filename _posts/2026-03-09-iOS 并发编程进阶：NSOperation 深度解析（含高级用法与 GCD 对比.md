# 
---
layout:     post
title:      iOS 并发编程进阶：NSOperation 深度解析（含高级用法与 GCD 对比）
subtitle:   iOS 并发编程进阶：NSOperation 深度解析（含高级用法与 GCD 对比）
date:       2026-03-09
author:     LXY
header-img: img/home1.jpeg
catalog: true
tags:
    - iOS
---

## 一、NSOperation 简介

`NSOperation` 是 Apple 在 `Foundation` 层提供的一套
**面向对象的并发抽象**，构建在 GCD 之上，但提供了更高层的控制能力。

核心类结构：

    NSOperation
     ├── NSBlockOperation
     └── 自定义 Operation
     
    NSOperationQueue

NSOperation 的核心特点：

-   面向对象封装任务
-   支持任务依赖关系
-   支持取消
-   支持优先级
-   支持 KVO 监听任务状态
-   可控制最大并发数

相比直接使用 GCD，NSOperation 更适合 **复杂任务编排**。

------------------------------------------------------------------------

# 二、NSOperation 核心概念

## 1 Operation 生命周期

Operation 有几个重要状态：

    isReady
    isExecuting
    isFinished
    isCancelled

状态流转：

    ready -> executing -> finished
            ↘ cancelled

这些状态都是 **KVO 可监听**。

示例：

``` objc
[operation addObserver:self
            forKeyPath:@"isFinished"
               options:NSKeyValueObservingOptionNew
               context:nil];
```

监听任务完成。

------------------------------------------------------------------------

## 2 NSOperationQueue

`NSOperationQueue` 负责调度 Operation。

``` objc
NSOperationQueue *queue = [[NSOperationQueue alloc] init];
[queue addOperation:operation];
```

### 控制并发数

``` objc
queue.maxConcurrentOperationCount = 3;
```

作用：

-   控制最大并发任务数量

-   实现串行队列(当把最大并发数设置成1)

```
 maxConcurrentOperationCount = 1
```
   

即可实现 **串行执行**。

### 🌰代码


- 新建四个NSBlockOperation，每一个NSBlockOperation模拟执行一个3秒的任务

```
    NSBlockOperation *op1 = [NSBlockOperation blockOperationWithBlock:^{
        NSLog(@"op1开始");
        sleep(3);
        NSLog(@"op1完成");
    }];
    
    NSBlockOperation *op2 = [NSBlockOperation blockOperationWithBlock:^{
        NSLog(@"op2开始");
        sleep(3);
        NSLog(@"op2完成");
    }];
    
    NSBlockOperation *op3 = [NSBlockOperation blockOperationWithBlock:^{
        NSLog(@"op3开始");
        sleep(3);
        NSLog(@"op3完成");
    }];
    
    NSBlockOperation *op4 = [NSBlockOperation blockOperationWithBlock:^{
        NSLog(@"op4开始");
        sleep(3);
        NSLog(@"op4完成");
    }];
    
```

- 最大并发数为2

```
    NSOperationQueue *queue = [NSOperationQueue new];
    queue.maxConcurrentOperationCount = 2;
    [queue addOperations:@[op1,op2,op3,op4] waitUntilFinished:NO];
    NSLog(@"===== 开始任务 ======");
```

- 执行结果：

```
===== 开始任务 ======
op2开始
op1开始
op2完成
op1完成
op3开始
op4开始
op3完成
op4完成
```

通过结果可以看出来，op1和op2先执行，当op1和op2执行结束后op3和op4才开始

- 最大并发数为1

```
    NSOperationQueue *queue = [NSOperationQueue new];
    queue.maxConcurrentOperationCount = 1;
    [queue addOperations:@[op1,op2,op3,op4] waitUntilFinished:NO];
    NSLog(@"===== 开始任务 ======");
```

- 执行结果：

```
===== 开始任务 ======
op1开始
op1完成
op2开始
op2完成
op3开始
op3完成
op4开始
op4完成
```

通过结果可以看出来，所有任务都是一个一个执行的，上一个执行完成下一个才开始，符合预期！！

------------------------------------------------------------------------

# 三、NSBlockOperation 使用

最简单的 Operation。

``` objc
NSBlockOperation *operation = [NSBlockOperation blockOperationWithBlock:^{
    NSLog(@"task1 %@", [NSThread currentThread]);
}];

[operation addExecutionBlock:^{
    NSLog(@"task2 %@", [NSThread currentThread]);
}];
```

特点：

-   可以添加多个 block
-   可能并发执行

------------------------------------------------------------------------

# 四、Operation 依赖关系（核心能力）

这是 NSOperation 最强的功能之一。

    A -> B -> C

代码：

``` objc
[operationB addDependency:operationA];
[operationC addDependency:operationB];
```

执行顺序：

    A 完成 -> B 执行 -> C 执行

但 **仍然是并发队列**。

依赖关系 ≠ 串行队列。

Operation 依赖关系适用于：

-   网络请求链
-   数据处理流水线
-   复杂任务编排

------------------------------------------------------------------------

# 五、取消任务

GCD 无法优雅取消任务，而 NSOperation 支持。

    [operation cancel];

在任务中判断：

``` objc
if (self.isCancelled) {
    return;
}
```

例如：

``` objc
- (void)main {

    if (self.isCancelled) return;

    // task1

    if (self.isCancelled) return;

    // task2
}
```

------------------------------------------------------------------------

# 六、自定义 NSOperation（高级用法）

自定义 Operation 主要有两种方式：

1 覆写 `main` 2 实现 **异步 Operation**

## 1 同步 Operation

``` objc
@interface DownloadOperation : NSOperation
@end

@implementation DownloadOperation

- (void)main {

    @autoreleasepool {

        if (self.isCancelled) return;

        NSLog(@"start download");

        [NSThread sleepForTimeInterval:2];

        NSLog(@"finish download");
    }
}

@end
```

------------------------------------------------------------------------

## 2、异步 Operation（高级）

当任务是 **异步 API**（如网络请求）时，需要自定义状态。

关键点：

    isExecuting
    isFinished
    
核心：

    KVO 手动控制状态

示例：

``` objc
@interface AsyncOperation : NSOperation

@property (nonatomic, assign) BOOL executing;
@property (nonatomic, assign) BOOL finished;

@end
```



``` objc

#import "AsyncOperation.h"


@implementation AsyncOperation

@synthesize executing = _executing;
@synthesize finished = _finished;

#pragma mark - 状态

- (BOOL)isAsynchronous {
    return YES;
}

- (BOOL)isExecuting {
    return _executing;
}

- (BOOL)isFinished {
    return _finished;
}

#pragma mark - start

- (void)start {
    
    if (self.isCancelled) {
        [self finish];
        return;
    }
    
    [self willChangeValueForKey:@"isExecuting"];
    _executing = YES;
    [self didChangeValueForKey:@"isExecuting"];
    
    [self execute];
}

#pragma mark - 子类重写

- (void)execute {
    
    NSLog(@"子类实现任务");
    
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, 2 * NSEC_PER_SEC), dispatch_get_global_queue(0, 0), ^{
        
        NSLog(@"任务完成");
        
        [self finish];
        
    });
}

#pragma mark - 完成任务

- (void)finish {
    
    [self willChangeValueForKey:@"isExecuting"];
    [self willChangeValueForKey:@"isFinished"];
    
    _executing = NO;
    _finished = YES;
    
    [self didChangeValueForKey:@"isExecuting"];
    [self didChangeValueForKey:@"isFinished"];
}

@end

```

任务完成：

    executing = NO
    finished = YES

这就是 **真正的异步 Operation**。

很多网络库都使用这种模式。

------------------------------------------------------------------------

# 七、NSOperation 高级技巧

## 1 暂停队列

``` objc
queue.suspended = YES;
```

恢复：

    queue.suspended = NO

------------------------------------------------------------------------

## 2 等待任务完成

``` objc
[queue waitUntilAllOperationsAreFinished];
```

或者：

``` objc
[queue addOperations:ops waitUntilFinished:YES];
```

------------------------------------------------------------------------

## 3 任务优先级

``` objc
operation.queuePriority = NSOperationQueuePriorityHigh;
```

优先级：

    VeryHigh
    High
    Normal
    Low
    VeryLow

------------------------------------------------------------------------

# 八、NSOperation vs GCD

|对比|NSOperation|GCD|
|----|----|----|
|抽象层         |高            |低|
|编程模型       |面向对象      |函数式|
|依赖关系       |✅支持        |❌不支持|
|取消任务       |✅支持        |❌困难|
|优先级         |✅支持        |部分|
|KVO            |✅支持        |❌|
|复杂任务编排   |⭐⭐⭐⭐      |⭐|
|性能           |稍低          |更高|

## 总结：

### 使用 GCD

适合：

-   简单并发
-   轻量任务
-   性能敏感

例如：

    dispatch_async
    dispatch_group
    dispatch_semaphore

------------------------------------------------------------------------

### 使用 NSOperation

适合：

-   大型项目
-   复杂任务依赖
-   任务取消
-   下载管理器
-   任务调度系统

例如：

    图片下载框架
    视频处理
    复杂网络请求链

------------------------------------------------------------------------

# 九、实际架构示例

###例如网络请求数据解析框架：

    downloadOperation
          ↓
    parseOperation
          ↓
    updateUIOperation

依赖关系：

    decode 依赖 download
    cache 依赖 decode


```
    NSOperationQueue *queue = [NSOperationQueue new];
    queue.maxConcurrentOperationCount = 4;

    NSBlockOperation * downloadOperation = [NSBlockOperation blockOperationWithBlock:^{
        NSLog(@"下载开始");
        sleep(3);
        NSLog(@"下载完成");
    }];

    NSBlockOperation * parseOperation = [NSBlockOperation blockOperationWithBlock:^{
        NSLog(@"解析开始");
        sleep(2);
        NSLog(@"解析完成");
    }];

    NSBlockOperation * updateUIOperation = [NSBlockOperation blockOperationWithBlock:^{
        NSLog(@"更新UI");
    }];

```

代码中解析op依赖下载op，UI更新op依赖解析op，即使不在同一个Queue中(UI 必须要在主线程)，也可以保证执行时序


代码：

``` objc
    [parseOperation addDependency: downloadOperation];
    [updateUIOperation addDependency: parseOperation];

    //>主线程更新UI
    [[NSOperationQueue mainQueue] addOperation: updateUIOperation];
    [queue addOperations:@[downloadOperation, parseOperation] waitUntilFinished:NO];
```

好处：

-   逻辑清晰
-   解耦
-   易维护

------------------------------------------------------------------------

# 十、最佳实践

推荐规则：

### 1 不要混用过多 GCD

如果使用 OperationQueue，尽量统一。

------------------------------------------------------------------------

### 2 Operation 内部不要阻塞主线程

避免：

    dispatch_sync(main)

------------------------------------------------------------------------

### 3 Operation 应该尽量独立

不要耦合 UI。

------------------------------------------------------------------------

### 4 使用依赖关系代替回调地狱

    A -> B -> C

而不是

    A(completion B(completion C))

------------------------------------------------------------------------

# 十一、总结

NSOperation 是 iOS 并发编程中 **最被低估的 API**。

核心优势：

-   任务依赖
-   任务取消
-   状态管理
-   面向对象
-   适合复杂系统

简单任务：

    GCD

复杂任务：

    NSOperationQueue

在大型 App（如下载管理器、图片处理管线）中，NSOperation 的可维护性远高于
GCD。
