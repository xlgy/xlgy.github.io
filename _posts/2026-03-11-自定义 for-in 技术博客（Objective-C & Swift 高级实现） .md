---
layout:     post
title:      自定义 for-in 技术博客（Objective-C & Swift 高级实现）
subtitle:   自定义 for-in 技术博客（Objective-C & Swift 高级实现）
date:       2026-03-11
author:     LXY
header-img: img/home1.jpeg
catalog: true
tags:
    - iOS
    - Swift
---


> 本文深入讲解 **for-in 循环底层机制**，以及如何在 **Objective-C 与 Swift 中实现自定义可迭代对象**，包含高级用法与完整示例代码。  
> 适合 iOS 中高级开发者阅读。

---

# 一、for-in 的本质

很多开发者认为：

```
for element in collection
```


只是语法糖。

实际上它的本质是：

👉 **迭代器模式（Iterator Pattern）**

即：


不断向集合请求“下一个元素”


不同语言实现不同：

| 语言 | 实现机制 |
|---|---|
| Objective-C | NSFastEnumeration |
| Swift | Sequence + IteratorProtocol |

---

# 二、Objective-C：for-in 底层实现

OC 的 `for-in` 依赖协议：`NSFastEnumeration`



```
@protocol NSFastEnumeration

- (NSUInteger)countByEnumeratingWithState:(NSFastEnumerationState *)state objects:(id __unsafe_unretained _Nullable [_Nonnull])buffer count:(NSUInteger)len;

@end
```
                                    

系统：

```
NSArray

NSDictionary

NSSet
```
全部实现了它。

## 2.1 自定义支持 for-in 的容器

### Step 1：定义类

```
@interface MyCollection : NSObject <NSFastEnumeration>

@end

```

### Step 2：实现枚举方法

```
@interface MyCollection()
@property (nonatomic, strong) NSArray *items;
@end

@implementation MyCollection

- (instancetype)initWithArray:(NSArray *)array {
    self = [super init];
    if (self) {
        self.items = array;
    }
    return self;
}
- (NSUInteger)countByEnumeratingWithState:(NSFastEnumerationState *)state
                                  objects:(id __unsafe_unretained [])buffer
                                    count:(NSUInteger)len {

    if (state->state >= self.items.count) {
        return 0;
    }

    state->itemsPtr = buffer;

    NSUInteger count = 0;

    while (state->state < self.items.count && count < len) {
        buffer[count] = self.items[state->state];
        state->state++;
        count++;
    }

    state->mutationsPtr = &state->extra[0];

    return count;
}

@end

```

### Step 3：使用

```
MyCollection *coll = [[MyCollection alloc]initWithArray:@[@1,@2,@3]];
for (NSNumber *n in coll) {
   NSLog(@"%i",n.intValue);
}
```

✅ 成功支持 for-in。

## 2.2 OC 高级技巧

✅ Lazy 枚举（不存数据）

```
buffer[count] = @(state->state * state->state);
```

动态生成：

```
1,4,9,16...
```

几乎零内存。

✅ 修改检测机制

```
state->mutationsPtr
```

用于检测遍历期间集合是否被修改。

这就是：
```
*** Collection mutated while being enumerated
```
崩溃的原因。

# 三、Swift：for-in 实现原理

Swift 基于两个协议：

`Sequence`

`IteratorProtocol`

调用链：

```
for-in
 ↓
makeIterator()
 ↓
next()
 ↓
Element?
```

## 3.1 自定义 Iterator

```
struct CountdownIterator: IteratorProtocol {

    var current: Int

    mutating func next() -> Int? {
        guard current > 0 else { return nil }
        defer { current -= 1 }
        return current
    }
}
```

## 3.2 自定义 Sequence

```
struct Countdown: Sequence {

    let start: Int

    func makeIterator() -> CountdownIterator {
        CountdownIterator(current: start)
    }
}

```

使用

```
for num in Countdown(start: 5) {
    print(num)
}
```

输出：

```
5 4 3 2 1
```

# 四、Swift 高级 for-in 用法

## 4.1 Lazy Fibonacci Sequence


```
struct Fibonacci: Sequence, IteratorProtocol {

    var a = 1
    var b = 2

    mutating func next() -> Int? {
        let result = a*2+b*3
        defer{
            a += 1
            b += 1
        }
        return result
    }
}
```

使用：

```
for n in Fibonacci().prefix(10) {
    print(n)
}
```

特点：

- 无限序列
- 按需生成
- 极低内存占用
- 集合内容可以按照自己的规则生产

## 4.2 自定义 Step Range

```
struct StepSequence: Sequence {

    let start: Int
    let end: Int
    let step: Int

    func makeIterator() -> AnyIterator<Int> {

        var current = start

        return AnyIterator {
            guard current <= end else {
                return nil
            }
            defer {
                current += step
            }
            return current
        }
    }
}

```

使用：

```

for i in StepSequence(start: 12, end: 30, step: 2) {
    print(i)
}
```

结果：

```
12
14
16
18
20
22
24
26
28
30
```

step步数为2，因此打印间隔都是2

# 五、OC vs Swift 对比
|语言|Objective-C|Swift|
|----|----|----|
|底层协议|	NSFastEnumeration	|Sequence|
|类型安全|弱	|强|
|Lazy支持	|手动	|原生|
|泛型	|无	|强|
|可读性	|一般	|高|



# 六、性能理解（高级）

for-in 实际执行：

```
makeIterator()
↓
next()
↓
Optional<Element>
```

优化建议：

- Iterator 使用 struct

- 避免对象分配

- 使用 Lazy Sequence

- 减少中间数组

# 七、总结

Objective-C

- 实现 NSFastEnumeration

- 手动管理 buffer

- 批量高性能遍历

Swift

- 实现 Sequence

- 定义 IteratorProtocol

- 天然支持 Lazy / Async

## ⭐ 核心理解：

👉 **for-in 不是遍历集合，而是不断请求 next()。**

👉 **for-in 是迭代器模式（Iterator Pattern）**

