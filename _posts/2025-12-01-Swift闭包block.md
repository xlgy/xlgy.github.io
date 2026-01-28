---
layout:     post
title:      Swift闭包block
subtitle:   Swift闭包block
date:       2025-12-01
author:     LXY
header-img: img/home1.jpeg
catalog: true
tags:
    - iOS
---


## Swift闭包与Objective-C Block的技术演进背景

在现代iOS开发中，函数式编程思想的引入使得代码更加简洁、可读性更强。Swift语言自诞生起便深度集成闭包（Closure）这一核心特性，而Objective-C则通过Block实现类似功能，二者均用于解决回调处理、异步任务封装及高阶函数支持等关键问题。尽管语法与底层机制不同——Block基于C语言扩展并依赖运行时结构体复制，Swift闭包则融合了类型推断与值语义优化——但其技术演进均受ARC内存管理普及和Cocoa框架现代化驱动。理解两者的设计动因与互操作基础，不仅有助于掌握捕获语义与生命周期控制的核心机制，也为混编项目中的内存安全与架构演进提供理论支撑。



# 一、Block定义

**1.Block 定义及使用**

Swift中的block比较简单，相比OC更加简洁

```
let block变量名 =  {(形参列表) in
	return 返回值
}

			
let myBlock = { (string: String) in
    print("\(string)")
    return 1
}

```

**2. 项目中使用格式**

在项目中，通常会重新定义block的类型的别名，然后用别名来定义block的类型

```
typealias myBlock = (String) -> String


let block:myBlock = {s in
	return "hello \(s)"
}


print(myBlock("block"))
```

# 二、Block变量截获

swift 没有__block关键字，默认闭包获取外部变量是地址获取(类似OC中__block)

```
        var testNum = 1
        var testString:String = "1"
        let block = {(num:Int)  in
            print("num:\(num)  testNum:\(testNum)  testString:\(testString)")
            testNum = 3
            testString = "3"
        }
        testNum = 2
        testString = "2"
        block(1)
        
        print("testNum:\(testNum)  testString:\(testString)")
```

输出结果：

```
num:1  testNum:2  testString:2
testNum:3  testString:3
```

可见闭包外部对变量的更改会对闭包内部的变量产生影响，同样闭包内部对变量的更改也同样响应外部变量


### 有没有方法可以避免闭包内部更改外部的变量值？

```
        var testNum = 1
        var testString:String = "1"
        let block = { [testNum,testString] (num:Int)  in
            print("num:\(num)  testNum:\(testNum)  testString:\(testString)")
        }
```

这种情况闭包对变量的捕获是值捕获，内部更改变量值也会变异报错！！

# 三、逃逸闭包

当闭包在函数返回后仍需执行（如异步操作、回调等），需用@escaping修饰。
@escaping是Swift中用于标记闭包的属性，表示该闭包的作用域会超出其定义函数的范围，需特别注意内存管理和循环引用问题。

代码示例：

```
    @objc func test(mblock:@escaping (String) -> Void) {
        DispatchQueue.global().async {
            
            DispatchQueue.main.asyncAfter(deadline: .now() + 2.0) {
                mblock("Some String")
            }
        }
    }
```

在示例中闭包mblock会在方法执行完之后被调用，会超出函数test的范围，所以需要标识为逃逸闭包，否则会变异报错







