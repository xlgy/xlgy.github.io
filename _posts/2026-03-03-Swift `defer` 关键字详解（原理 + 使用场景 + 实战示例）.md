---
layout:     post
title:      Swift `defer` 关键字详解（原理 + 使用场景 + 实战示例）
subtitle:   Swift `defer` 关键字详解（原理 + 使用场景 + 实战示例）
date:       2026-03-03
author:     LXY
header-img: img/home1.jpeg
catalog: true
tags:
    - Swift
---




> 一篇真正让你 **彻底理解 Swift defer 执行机制** 的文章。

------------------------------------------------------------------------

## 一、什么是 defer？

`defer` 是 Swift 中用于 **延迟执行代码** 的关键字。

核心作用：

👉 **在当前作用域结束前，一定执行指定代码。**

无论函数：

-   正常 return
-   提前 return
-   throw error
-   break / continue

`defer` 中的代码 **都会执行**。

------------------------------------------------------------------------

## 基本语法

``` swift
defer {
    // 作用域结束时执行
}
```

------------------------------------------------------------------------

## 二、执行时机

**离开当前作用域时执行**。

``` swift
func test() {
    defer { print("结束") }
    print("开始")
}
```

输出：

    开始
    结束

------------------------------------------------------------------------

## 三、多个 defer 执行顺序（重点）

> 后写的 defer 先执行（LIFO 栈结构）

``` swift
func test() {
    defer { print("1") }
    defer { print("2") }
    defer { print("3") }
    print("start")
}
```

输出：

    start
    3
    2
    1

------------------------------------------------------------------------

## 四、设计目的

`defer` 主要解决：

-   资源释放一致性
-   多 return 清理遗漏
-   错误路径 cleanup

------------------------------------------------------------------------

## 五、经典使用场景

### 1️⃣ Lock / Unlock

``` swift
lock.lock()
defer { lock.unlock() }

if error {
    return
}
```

------------------------------------------------------------------------

### 2️⃣ 文件操作

``` swift
func readFile() throws {
    let file = openFile()
    defer { file.close() }
    try file.read()
}
```

------------------------------------------------------------------------

### 3️⃣ 数据库事务

``` swift
db.beginTransaction()
defer { db.commit() }
```

------------------------------------------------------------------------

### 4️⃣ 性能统计

``` swift
func loadData() {
    let start = Date()

    defer {
        print("耗时:", Date().timeIntervalSince(start))
    }

    heavyTask()
}
```

------------------------------------------------------------------------

## 六、defer + Error Handling

``` swift
func process() throws {
    setup()
    defer { cleanup() }
    try dangerousTask()
}
```

即使 throw，cleanup 仍执行。

------------------------------------------------------------------------

## 七、作用域陷阱

``` swift
if true {
    defer { print("A") }
}
print("B")
```

输出：

    A
    B
    
    
``` swift
if true {
    defer { print("A") }
}
defer { print("C") }
print("B")
```

输出：

    A
    B
    C
    
defer的执行时机是在作用域执行之后

**第一个例子中if{}作用域在前，先打印A，之后打印B**

* **第二个例子中打印B和打印C都在if以外的作用域，打印C被defer修饰，会在该作用域最后执行，因此先打印B再打印C**

    

------------------------------------------------------------------------

## 八、async/await 中的 defer

### 1.defer + async 的执行顺序

``` swift
func test() async {
    print("1")

    defer {
        print("3")
    }

    await Task.sleep(1_000_000_000)

    print("2")
}
```

执行顺序：

    1
    (挂起)
    2
    3

关键理解：

👉 `await` **不会触发 defer**。

👉 只有函数真正结束才执行。

------------------------------------------------------------------------

### 2.Actor 切换 + defer

``` swift
@MainActor
func updateUI() async {

    defer {
        print(Thread.isMainThread)
    }

    await backgroundWork()
}
```

问题：

👉 defer 在哪个线程执行？

答案：

    MainActor

原因：

-   defer 属于函数作用域
-   函数被 MainActor 隔离
-   cleanup 必须回到 Actor

即使中途离开主线程。

------------------------------------------------------------------------

### 3.Task Cancel 时 defer 会执行吗？

答案：

✅ 会。

示例：

``` swift
func load() async {
    defer { print("cleanup") }

    try? await Task.sleep(nanoseconds: 5_000_000_000)
}
```

当：

``` swift
task.cancel()
```

发生：

    CancellationError
    ↓
    函数退出
    ↓
    defer 执行

这就是为什么 defer 非常适合释放资源。

------------------------------------------------------------------------

### 4.defer + throwing async

``` swift
func fetch() async throws {

    start()

    defer { end() }

    try await request()
}
```

执行路径：

    request throw
          ↓
    函数退出
          ↓
    end() 执行

无需重复 cleanup。

### 5.async/await 中的 defer

    defer 在 async 函数中：

    ✔ 不会在 await 时执行
    ✔ 会在函数真正结束执行
    ✔ Actor 隔离保持一致
    ✔ Task cancel 仍执行
    ✔ 类似 finally


## 九、底层原理

编译器近似转换为：

``` swift
do {
   body
} finally {
   cleanup()
}
```

------------------------------------------------------------------------



## 十、最佳实践

固定模板：

``` swift
func action() async throws {

    setup()

    defer { teardown() }

    try await work()
}
```

适用于：

-   UI 状态恢复
-   数据库事务
-   网络 loading
-   锁管理
-   Metrics 统计

------------------------------------------------------------------------

## 十一、常见问题及总结
**总结**

`defer` = **作用域结束时自动执行 cleanup**
> 谁打开资源，谁用 defer 关闭。

**Q：return 后执行吗？**

A：会。

**Q：throw 后执行吗？**

A：会。

**Q：多个 defer 顺序？**

A：后进先出。


**Q： defer适合使用再哪些场景？**

-   Lock
-   File
-   DB Transaction
-   性能统计
-   状态恢复




