---
layout:     post
title:      iOS-define、const 、static、extern详解
subtitle:   iOS-define、const 、static、extern详解
date:       2025-12-08
author:     LXY
header-img: img/home1.jpeg
catalog: true
tags:
    - iOS
---

# 一、define

## 1.定义及使用

我们使用**#define**来定义常量，这个预处理指令会将**宏**替换为定义值

新建一个头文件DefineA.h

```
#ifndef DefineA_h
#define DefineA_h

#define test 1

#endif /* DefineA_h */
```

引入DefineA.h头文件，执行代码

```
NSLog(@"define : %i",test);
```

打印：

**define : 1**


可见#define 是在 编译期 完成的。

它属于 预处理指令，在编译器真正编译代码之前就被替换掉了。

```
预处理（处理 #define 等宏）
↓
编译（生成目标文件 .o）
↓
链接（生成可执行文件）
↓
运行
```

## 2.重复定义

如果重复定义相同的**宏**，则会产生定义冲突，只会产生警告，并不会报错


新建一个头文件DefineB.h，重复定义宏test

```
#ifndef DefineB_h
#define DefineB_h

#define test 2
#endif /* DefineB_h */
```

此时宏test的值被重复定义，具体的值和引入头文件顺序有关系，如果有相同的宏定义，后引用的会覆盖之前引用的宏定义

如果引用顺序为

```
#import "DefineA.h"
#import "DefineB.h"
```

此时打印日志为

**define : 2**


# 二、const

## 1.定义

const对变量的修饰在<编译阶段>执行，被const修饰的变量在<编译阶段>会进行编译检查，会报编译错误。

被const修饰的变量仅在<编译阶段>初始化一次，在<常量区>为它分配一份内存，一直到程序结束运行由系统回收。

**const的作用：**

1. 将位于const右部的(全局/局部)变量修饰为(全局/局部)常量
2. 被const修饰局部变量是只读的，不能被修改

## 2.使用

### 基础数据类型：

对于基础数据类型，const关键关键字放在类型前还是变量前都可以，起到相同的效果

```
    const int intVar1 = 1;
    int const intVar2 = 1;
```

这段代码中intVar1和intVar2都是不可变的，如果给他们重新复制，会产生变异报错：

```
Cannot assign to variable 'intVar1' with const-qualified type 'const int'
Cannot assign to variable 'intVar2' with const-qualified type 'const int'
```

### 指针类型：

对于指针类型，和基础数据类型不同，先看代码

```
    const NSString *string1 = @"s";
    NSString const *string2 = @"s";
    NSString * const string3 = @"s";
    
    string1 = @"1";
    string2 = @"1";
    string3 = @"1";
```

对于上述代码，只有一个变量更改值会编译报错，是string3。

由此可见，对于指针类型，想要锁定变量值需要在变量名前使用const关键字；有是const 右边紧跟varName时，varName才变为常量且无法被修改。


# 三、static

static对变量的修饰在<编译阶段>执行，被static修饰的变量在<编译阶段>会进行编译检查，会报编译错误。

被static修饰的变量仅在<编译阶段>初始化一次，在<全局/静态区>为它分配一份内存，一直到程序结束运行由系统回收。

static的作用:

1. static关键字修饰局部变量，可延长局部变量的生命周期，直到程序结束运行为止。
2. static关键字修饰局部变量，不会改变局部变量的作用域，该局部变量还是只能在当前作用域范围内使用。
3. static关键字修饰(在.m中声明)的全局变量，使其只在本.m文件内部有效，而其他文件不可连接或引用该变量（即使在外部使用extern关键字也无法访问）。

解释一下上述第三点作用：

没有使用static关键字修饰(不管是在.m还是.h中声明)的全局变量，在其他.m和.h文件中定义同名的全局变量，在编译时是会报重复声明错误的，也就是此时的全局变量的作用域是整个项目。

而使用static关键字修饰(在.m中声明)的全局变量后，在其他.m和.h文件中定义同名的全局变量就不会报错了，因此我们得到的上述第三点作用。（PS：如果使用static关键字修饰(在.h中声明)的全局变量，只要在文件中#import该.h文件，还是可以使用该全局变量，所以第三点作用强调的是(在.m中声明)的全局变量）。


***全局变量是不安全的，因为它可能会被外部修改，所以在.m中定义全局变量时推荐使用static关键字修饰。***

```
//只有以下两种用法，且效果一样
static int staticNum1 = 1;
int static  staticNum2 = 1;
```

# 四、extern

### 作用：
1. 在别的文件声明变量（“只是告诉别人有这个变量”）

2. extern关键字修饰全局变量是表示对该全局变量的访问，而不是定义该全局变量，所以并不会分配内存。

3. extern关键字会先在当前文件查找有没有该全局变量，没有找到，才会去整个项目中的文件去查找。




### 定义值
实例代码：
定义class Extern1，头文件定义externNum，extern关键字修饰

头文件：

```
@interface Extern1 : NSObject
extern int externNum;
@end
```

实现文件：

```
@implementation Extern1

int externNum = 4;

@end
```

此时访问externNum时值为4


### 寻找值&更改值

可以不在实现文件中给变量赋值，这样extern变量会去整个项目中的文件去查找

实例代码：
定义class Extern1，实现文件中不给赋值

```
#import "Extern1.h"

@implementation Extern1

//int externNum = 4;

@end
```

定义另一个class Extern2，在其实现文件中给变量赋值，同时定义方法来更改值

```
@implementation Extern2
int externNum = 5;

+ (void)changeExternNum {
    externNum = 100;
}

@end
```

执行代码：

```
    NSLog(@"extern : %i",externNum);
    [Extern2 changeExternNum];
    
    NSLog(@"extern : %i",externNum);
```

日志：

```
extern : 4
extern : 100
```

PS: extern也可以使用苹果官方定义的宏：UIKIT_EXTERN来进行替换，你看哪个你觉得舒服就用哪个，效果一样。



# 五、组合使用

## static与const

static与const同时修饰一个变量时，该变量变为静态的只读变量，无法被外部文件访问也无法被修改。

通常用于全局的数据常量或字符串常量，这些常量定义之后就不需要也不能修改，且作用域仅在本.m文件中。

类似下面的栗子，这些常量仅在一个.m文件使用，且定义之后就不需要也不能修改，就应该使用static与const同时修饰。

```
static NSString *const titleOfViewController = @"首页";
 
static NSInteger const PI = 3.1415926;
```

## extern与const

多个文件中都经常使用的相同的字符串常量，就需要使用extern与const同时修饰，可供外部文件访问且不可修改。

日常使用比较多的就是通知关键字使用，我们先看一下系统通知关键字的定义：

```
UIKIT_EXTERN NSNotificationName const UIApplicationDidEnterBackgroundNotification       API_AVAILABLE(ios(4.0)) API_UNAVAILABLE(watchos) NS_SWIFT_NONISOLATED;
UIKIT_EXTERN NSNotificationName const UIApplicationWillEnterForegroundNotification      API_AVAILABLE(ios(4.0)) API_UNAVAILABLE(watchos) NS_SWIFT_NONISOLATED;
```

可见系统就是使用了extern与const的组合

UIKIT_EXTERN定义：

```
#ifdef __cplusplus
#define UIKIT_EXTERN       extern "C" __attribute__((visibility ("default")))
#else
#define UIKIT_EXTERN           extern __attribute__((visibility ("default")))
#endif
```

NSNotificationName定义：

```
typedef NSString *NSNotificationName NS_TYPED_EXTENSIBLE_ENUM;
```

把 NSString * 定义为一个新的类型名 NSNotificationName。


因此我们自定义通知消息的时候，也可以使用这种组合定义～

举个🌰：
定义一个通知管理类，为外暴露通知消息名称：

头文件

```
@interface LoginMgr : NSObject

extern const NSNotificationName loginMgrNot_loginSuccess;

@end
```

实现文件

```
@implementation LoginMgr

NSNotificationName const loginMgrNot_loginSuccess = @"loginMgrNot_loginSuccess";


- (void)loginSuccess{
    [[NSNotificationCenter defaultCenter]postNotificationName: loginMgrNot_loginSuccess object:nil];
}

@end
```

外部使用监听消息

```
[[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(loginSuccess) name:loginSuccess object:nil];
```




