---
layout:     post
title:      Swift Currying（柯里化）：隐藏在Swift语言背后的设计模式
subtitle:   Swift Currying（柯里化）：隐藏在Swift语言背后的设计模式
date:       2026-03-02
author:     LXY
header-img: img/home1.jpeg
catalog: true
tags:
    - Swift
---


很多人第一次看到时的反应：

- 🤔 为什么函数还能返回函数？
- 🤔 Swift 里几乎没人写，但为什么面试常问？
- 🤔 为什么说 Swift 方法本质是 Currying？

看完本文，你会真正理解 Swift 函数模型。



# 一、什么是 Currying？

## ✅ 一句话定义


**Currying = 把多参数函数拆成一连串单参数函数**



## 普通函数 vs Currying

```
普通函数
──────────────
add(a, b)
   │
   ▼
 result


Currying
──────────────
add(a)
   │
   ▼
返回函数 ──► (b) -> result
```

---

# 二、从普通函数开始

## 普通写法

```swift
func add(_ a: Int, _ b: Int) -> Int {
    a + b
}

add(2, 3) // 5
```

执行流程：

```
(2,3) ───► add ───► 5
```



## Currying 写法

```swift
func add(_ a: Int) -> (Int) -> Int {
    return { b in
        a + b
    }
}
```

调用：

```swift
let add2 = add(2)
add2(3)
```



## 执行过程（核心理解）

```
Step 1
add(2)
   │
   ▼
返回函数 f(b)

Step 2
f(3)
   │
   ▼
5
```

本质：

```
add(2)(3)
```


# 三、函数类型的真实含义（关键）

函数签名：

```swift
(Int) -> (Int) -> Int
```

不要读成两个参数。

应该读作：

```
输入 Int
   │
   ▼
返回 一个函数
          │
          ▼
        输入 Int
          │
          ▼
         Int
```



## 类型结构

```
A -> B -> C

等价于：

A -> (B -> C)
```

Swift 默认 **右结合**。



# 四、Currying 为什么存在？

核心目的只有一个：

## ✅ 参数预绑定（Partial Application）



### 示例：生成专用函数

```swift
func multiply(_ a: Int) -> (Int) -> Int {
    { b in a * b }
}

let double = multiply(2)
let triple = multiply(3)
```

---

## 函数工厂

```
multiply(2) ──► double(x) = 2*x
multiply(3) ──► triple(x) = 3*x
```

你创建了新的函数。



# 五、🔥 Swift 方法本质就是 Currying

这是理解 Swift 的关键。



## 定义类

```swift
class Person {
    func say(_ msg: String) {
        print(msg)
    }
}
```


## 你以为的方法类型

```
(String) -> Void
```



## 实际类型（重要）

```swift
Person.say
```

类型是：

```
(Person) -> (String) -> Void
```



## 方法真实调用

```
Person.say
     │
     ▼
需要 Person 实例
     │
     ▼
得到函数 (String)->Void
     │
     ▼
传入参数
```

等价：

```swift
let fn = Person.say
fn(person)("Hello")
```

其实就是：

```swift
person.say("Hello")
```

👉 Swift 自动帮你做了第一步。



# 六、为什么 Swift 3 后很少写 Currying？

官方逐渐弱化原因：

```
❌ 可读性下降
❌ 学习成本高
❌ closure 更直观
```



现代推荐：

```swift
let add2 = { add(2, $0) }
```



## 现代替代方案

```
Closure Partial Apply

add(2, _)
     │
     ▼
{ add(2, $0) }
```

效果一致，但更清晰。



# 七、多级 Currying（理解函数链）

```swift
func sum(_ a: Int)
 -> (Int)
 -> (Int)
 -> Int {

    { b in
        { c in
            a + b + c
        }
    }
}
```

调用：

```swift
sum(1)(2)(3)
```



## 函数洋葱模型 🧅

```
sum(1)
   │
   ▼
function(b)
   │
   ▼
function(c)
   │
   ▼
result
```

一层层展开。



# 八、iOS 开发中你早就在用

虽然你没意识到。



## 1️⃣ Target-Action

```
button.sendAction(...)
```

本质：

```
(UIButton) -> (...) -> Void
```

先绑定实例。



## 2️⃣ Dependency Injection（依赖注入）

```swift
func request(baseURL: String)
    -> (String) -> Void {

    { path in
        print(baseURL + path)
    }
}
```

---

图示：

```
baseURL 固定
        │
        ▼
生成 API 函数
        │
        ▼
只需要 path
```

---

## 3️⃣ Combine / RxSwift

函数链：

```
map → filter → flatMap
```

背后依赖函数组合思想（与 currying 强相关）。






# 九、总结
```
Currying =
函数接收一个参数
返回新的函数
直到得到最终结果
```

或者：

```
Function → Function → Result
```

更深层：

> Swift 实例方法调用，本质就是一次 Currying。

理解 Currying 后，你会突然明白：

✅ 为什么方法必须有 `self`  
✅ 为什么函数可以像变量一样传递  
✅ 为什么 Swift 支持函数组合  

这不是语法技巧，而是 **Swift 的底层设计思想**。


Swift 看起来是：

```
面向对象语言
```

但内核更像：

```
函数式语言 + 类型系统
```

而 Currying，就是连接两者的桥梁。



