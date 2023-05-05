---
layout:     post
title:      iOS 冷知识——内联函数 inline
subtitle:   iOS 冷知识——内联函数 inline
date:       2023-05-05
author:     LXY
header-img: img/home1.jpeg
catalog: true
tags:
    - iOS
---

#内联函数(inline function)
##inline
- 使用 inline 修饰函数的声明或者实现，可以使其变为联函数
-当函数被声明为内联函数之后, 编译器会将其内联展开, 而不是按通常的函数调用机制进行调用

**在框架中出现inline时,如YYKit框架.我们稍加观察就会发现它出现在.h文件中. such as:**
```
static inline CGFloat CGFloatFromPixel(CGFloat value) {
    return value / YYScreenScale();
}

//YYScreenScale()方法说明:
CGFloat YYScreenScale() {
    static CGFloat scale;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        scale = [UIScreen mainScreen].scale;
    });
    return scale;
}
```
## 内联函数优缺点
###优点：
- 编译器会将函数调用直接展开为函数体代码
- **解决函数调用效率的问题：**可以减少函数调用的开销；不用inline修饰的函数, 汇编时会出现 call 指令，函数之间调用，是内存地址之间的调用，当函数调用完毕之后还会返回原来函数执行的地址；函数调用有时间开销，内联函数就是为了解决这一问题。
- 对于存取函数以及其它函数体比较短, 性能关键的函数, 建议使用内联.
- 所以可以认为是 空间换时间思想的体现

###缺点：
- 会增加代码体积

 
## 内联函数 VS 宏定义

- 内联函数避免了宏的缺点:需要预编译.因为inline内联函数也是函数,不需要预编译。
- 编译器在调用一个内联函数时，会首先检查它的参数的类型，保证调用正确。然后进行一系列的相关检查，就像对待任何一个真正的函数一样。这样就消除了它的隐患和局限性。
- 可以使用所在类的保护成员及私有成员。

##注意：
- 尽量不要内联超过 10 行代码的函数，内联函数不能承载大量的代码；如果内联函数的函数体过大,编译器会自动放弃内联；有些函数即使声明为 inline 也不一定会被编译器内联，比如递归函数；
- 内联函数只是我们向编译器提供的申请,编译器不一定采取inline形式调用函数.
- 内联函数内不允许使用循环语句或开关语句
- 内联函数的定义须在调用之前。






