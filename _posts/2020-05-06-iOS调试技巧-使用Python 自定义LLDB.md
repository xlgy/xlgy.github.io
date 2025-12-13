---
layout:     post
title:      iOS调试技巧-使用Python 自定义LLDB
subtitle:   iOS调试技巧-使用Python 自定义LLDB
date:       2020-05-06
author:     LXY
header-img: img/home1.jpeg
catalog: true
tags:
    - iOS
---

##一、类介绍

在使用Python 自定义LLDB之前，先了解一下LLDB的一些类型

- SBTarget 正在被调试的程序
- SBProcess 和程序关联的具体的进程
- SBThread 执行的线程
- SBFrame 和线程关联的一个栈帧
- SBVariable 变量，寄存器或是一个表达式

一般情况下，我们取到SBFrame就可以进行方法调用来打印关键信息


##二、断点调试示例
在写Python前，先使用Xcode断点执行一下

自定义类**MyClass**
.h文件
```
@interface MyClass : NSObject

+ (NSString *)lldbTest;

@end
```

.m文件

```
@implementation MyClass

+ (NSString *)lldbTest {
    return @"lldb test successed";
}

@end
```
中断程序
![](https://images.xiaozhuanlan.com/photo/2021/b8874e34ab74d993bdb5598bfa572f8d.png)

打开lldb控制台
![](https://images.xiaozhuanlan.com/photo/2021/d5a9a379a7561d511bec2c432eaaa3ae.png)

下面就开始写lldb的命令
**预期目标，打印出[MyClass lldbTest]的返回值**

输入script
```
(lldb) script
Python Interactive Interpreter. To exit, type 'quit()', 'exit()'.
>>> 
```
定义变量test接收MyClass lldbTest]的返回值
```
>>> test = lldb.frame.EvaluateExpression('(NSString *)[MyClass lldbTest]').GetObjectDescription()
```
打印变量test
```
>>> print(test)
```
![](https://images.xiaozhuanlan.com/photo/2021/6a6a20a184934bbfdc0158cbe6a8fcd5.png)

至此，直接在Xcode中使用lldb打印出[MyClass lldbTest]的返回值就完成了

##三、编写Python

**如果想把这个功能打包起来，使用一句命令调用，就需要使用Python来扩展我们的lldb命令**

####1、新建Python文件
这里将Python文件命名问lldbtest.py

####1、引入lldb头文件
```
import lldb
```

####2、初始化函数
```
def __lldb_init_module(debugger, internal_dict):
    debugger.HandleCommand()
```
在HandleCommand中添加命令：
```
'command script add lldb_test -f lldbtest.test'
```
lldb_test表示命令名称，lldbtest是Python文件名，test是自定义方法名

初始化函数最终
```
def __lldb_init_module(debugger, internal_dict):
    debugger.HandleCommand('command script add lldb_test -f lldbtest.test')
```

####3、自定义Python方法

**获取当前的frame栈帧**
```
  target = debugger.GetSelectedTarget()
  process = target.GetProcess()
  thread = process.GetSelectedThread()
  currentFrame = thread.GetSelectedFrame()
```

**调用方法**
```
def test(debugger, command, result, internal_dict):
  target = debugger.GetSelectedTarget()
  process = target.GetProcess()
  thread = process.GetSelectedThread()
  currentFrame = thread.GetSelectedFrame()
  test = currentFrame.EvaluateExpression('(NSString *)[Person lldbTest]').GetObjectDescription()
  print("result:%s" % test)
```

整个Python文件
```
#自定义lldb命令 
import lldb

def test(debugger, command, result, internal_dict):
  target = debugger.GetSelectedTarget()
  process = target.GetProcess()
  thread = process.GetSelectedThread()
  currentFrame = thread.GetSelectedFrame()
  test = currentFrame.EvaluateExpression('(NSString *)[Person lldbTest]').GetObjectDescription()
  print("result:%s" % test)

def __lldb_init_module(debugger, internal_dict):
    debugger.HandleCommand('command script add lldb_test -f lldbtest.test')

```



##四、自动加载python脚本
**原理：xcode启动的时候会读取一个默认文件:~/.lldbinit**
只需要将命令command script import /Users/xx/Desktop/lldbtest.py 写入这个文件即可。
/Users/xx/Desktop/lldbtest.py是Python文件路径

测试：
![](https://images.xiaozhuanlan.com/photo/2021/02b68081b12c1a3f1f0fe6db362a78b2.png)



