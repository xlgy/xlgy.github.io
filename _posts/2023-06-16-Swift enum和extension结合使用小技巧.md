---
layout:     post
title:      Swift enum和extension结合使用小技巧
subtitle:   Swift enum和extension结合使用小技巧
date:       2023-06-16
author:     LXY
header-img: img/home1.jpeg
catalog: true
tags:
    - Swift
---

## 很痛的痛点
我们经常碰到这样场景，定义枚举，代码判断switch-case，不同枚举返回不同值。这样的写法没有问题，只是枚举的逻辑无法收敛，查找枚举控制的逻辑在哪里会很麻烦~

**举个例子：**
比如我们定义一个枚举表示阅读器主题
```
enum ReadBookTheme{
    ///>男生主题
    case ReadBookThemeBoy
    ///>女生主题
    case ReadBookThemeGirl
    ///>暗黑主题
    case ReadBookThemeDark
    case......
}
```

如果我们有很多样式的颜色和主题相关，我们就需要写很多的switch-case散落在不同地方的代码，这点很难受，**增加一个枚值举需要改多个地方，很头痛！！**

这里分享一个小技巧，解决这个问题！！

## 解决方案

拯救这个痛点的关键就是——**extension**
我们可能常常会为一个class做extension， 其实enum也可以做extension！

按照上面的例子，我们可以给```ReadBookTheme```写一个```extension```

```
extension ReadBookTheme{
    var textColor : Color {
        switch self {
        case .boy :
            return Color.blue
        case .girl :
            return Color.pink
        case .girl :
            return Color.black
        default: 
            return UIColor(red: 11/255.0, green: 96/255.0, blue: 254/255.0, alpha: 1)
        }
    }'
    var backgroundColor : Color {
        switch self {
        case .boy :
            return Color.blue
        case .girl :
            return Color.pink
        case .girl :
            return Color.black
        default: 
            return UIColor(red: 11/255.0, green: 96/255.0, blue: 254/255.0, alpha: 1)
        }
    }

}
```
那我们要获取主题对应字体颜色的时候，只需要```theme.textColor```就可以获取字体颜色，获取背景颜色```theme.backgroundColor```，虽然这种方式也是需要 switch，但是所有逻辑都收敛到一个地方，增加枚举值也不需要整个工程找代码了。







