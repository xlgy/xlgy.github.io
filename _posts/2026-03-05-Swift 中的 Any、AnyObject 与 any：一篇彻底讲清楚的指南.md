---
layout:     post
title:      Swift 中的 Any、AnyObject 与 any：一篇彻底讲清楚的指南
subtitle:   Swift 中的 Any、AnyObject 与 any：一篇彻底讲清楚的指南
date:       2026-03-05
author:     LXY
header-img: img/home1.jpeg
catalog: true
tags:
    - Swift
---



在日常使用 Swift 开发时，经常会遇到三个非常相似但含义完全不同的关键字：**Any、AnyObject、any**
很多开发者在刚接触时都会产生疑惑：

-   Any 和 AnyObject 有什么区别？
-   any 为什么是小写？
-   any Protocol 和 Protocol 有什么不同？
-   为什么 Swift 5.7 之后推荐使用 any？

本文会从 概念 → 原理 → 使用场景 → 最佳实践 一步一步讲清楚这三个关键字。

------------------------------------------------------------------------

# 一、Any：可以表示"任何类型"

Any 是 Swift 中最通用的类型。

> Any 可以表示 **任意类型的值**。

包括：

-   Int
-   String
-   Struct
-   Class
-   Enum
-   Function
-   Optional

示例：

``` swift
let a: Any = 10
let b: Any = "Hello"
let c: Any = [1,2,3]
let d: Any = { print("hello") }
```

也就是说：

    Any = 任意类型

甚至数组也可以存放不同类型：

``` swift
let arr: [Any] = [1, "Swift", true, 3.14]
```

------------------------------------------------------------------------

## 类型转换

因为 Any 没有具体类型，所以在使用时通常需要 **类型转换**。

``` swift
let value: Any = 10

if let number = value as? Int {
    print(number + 1)
}
```

------------------------------------------------------------------------

# 二、AnyObject：只能表示"类类型"

AnyObject 的含义就严格很多。

> AnyObject 只能表示 **class 类型实例**。

例如：

``` swift
class Person {}

let obj: AnyObject = Person()
```

但是：

``` swift
let value: AnyObject = 10
```

会直接报错，因为 Int 是 struct。

------------------------------------------------------------------------

## Objective-C 兼容

AnyObject 的设计初衷主要是为了兼容 Objective-C。

Objective-C 中存在：

    id

Swift 对应关系：

  Objective-C   Swift
  ------------- -----------
  id            AnyObject

------------------------------------------------------------------------

# 三、any：协议存在类型（Protocol Existential）

any 是 Swift 5.7 引入的新关键字。

它的作用是：

> 表示 **协议存在类型（Existential Type）**

示例：

``` swift
protocol Animal {
    func run()
}

var a: any Animal
```

意思是：

    a 可以是任何遵循 Animal 协议的类型

例如：

``` swift
struct Dog: Animal {
    func run() { print("Dog run") }
}

struct Cat: Animal {
    func run() { print("Cat run") }
}

let a: any Animal = Dog()
```

------------------------------------------------------------------------

# 四、any vs 泛型

很多人会混淆：

    any Protocol

和

    <T: Protocol>

### 使用 any

``` swift
func run(_ animal: any Animal) {
    animal.run()
}
```

特点：

    运行时动态类型

### 使用泛型

``` swift
func run<T: Animal>(_ animal: T) {
    animal.run()
}
```

特点：

    编译期确定类型

------------------------------------------------------------------------

## 性能区别

| 方式| 性能 |
|----|----|
|any Protocol |  较慢|
|泛型      |      更快 | 


  

原因：

    any 使用动态派发
    泛型使用静态派发

------------------------------------------------------------------------

# 五、三者区别总结


|关键字|含义|可表示| 
|----|----|----|
|Any |任意类型|struct / class / enum|
|AnyObject|任意类对象|class|
|any|协议存在类型|protocol|

简单理解：

    Any       = 任意类型
    AnyObject = 任意 class
    any       = 任意遵循协议的类型

------------------------------------------------------------------------

# 六、实际开发建议

在实际 Swift 项目中，一般建议：

### 优先使用泛型

``` swift
func process<T: Codable>(_ value: T)
```

原因：

    性能更好
    类型安全

### 必须存储不同类型时使用 any

例如：

``` swift
var animals: [any Animal]
```

### 与 Objective-C 交互时使用 AnyObject

例如：

``` swift
weak var delegate: AnyObject?
```

------------------------------------------------------------------------

# 七、一个经典面试题

下面代码是否合法？

``` swift
protocol Test {}

let arr: [Test]
```

在 Swift 5.7 之后会出现警告：

    Use 'any Test'

正确写法：

``` swift
let arr: [any Test]
```

------------------------------------------------------------------------

# 结语

Any、AnyObject、any 是 Swift 类型系统中非常重要的三个关键字。

理解它们的区别，可以帮助我们写出 **更安全、更高性能、更符合 Swift
风格的代码**。

最后用一句话总结：

    Any       表示任意类型
    AnyObject 表示任意类对象
    any       表示任意遵循协议的类型
