---
layout:     post
title:      flutter——手把手教你制作一个动态富文本SDK
subtitle:   flutter——手把手教你制作一个动态富文本SDK
date:       2023-05-06
author:     LXY
header-img: img/home1.jpeg
catalog: true
tags:
    - flutter
---
#一、前言
学习flutter也有一段时间了，今天挑战一下自己，手动完成一个富文本SDK。在本人的移动端职业生涯中，经历过几个大家所说的“**大厂**”的工作，发现很多“**大厂**”都有自己原生端的动态富文本库，来适配各种运营位的展示和交互，基本的实现原理都由服务端下发动态配置，再有客户端富文本SDK进行解析并渲染，实现在不发版的情况下动态更新富文本。

![](https://images.xiaozhuanlan.com/photo/2022/8f0b0826d72042bde54d42c3dd2c5a94.png)


#二、实现

##1、 创建纯dart插件 package
```
flutter create --template=package lxy_rich_text_from_json
```

##2、 确定配置json

由于是初次尝试，优先实现一个简单配置的版本，只控制富文本的颜色和字体大小
```
{
	"message": "一二三四五六七八九",
	"textSize": 20,
	"textColor": "FF0000",
	"richTexts": [{
			"textSize": 44,
			"textColor": "23238E",
			"startIndex": 0,
			"endIndex": 3
		},
		{
			"textSize": 66,
			"textColor": "545454",
			"startIndex": 5,
			"endIndex": 7
		}
	]
}
```

|参数名|类型|描述|
|--|--|--|
|message|String|文案|
|textSize|int|字号|
|textColor|String|色值|
|richTexts|Array|富文本数组|
|startIndex|int|起始index|
|endIndex|int|结束index|

**子数组的权重要高于全局的样式权重!!!**

##3、 组件实现

```
library lxy_rich_text_from_json;

import 'dart:convert';
import 'dart:core';

import 'package:flutter/material.dart';

class RichTextView extends StatefulWidget {
  String rich_text_config;

  RichTextView(this.rich_text_config);

  @override
  State<RichTextView> createState() => _RichTextViewState();
}

class _RichTextViewState extends State<RichTextView> {
  @override
  Widget build(BuildContext context) {
    return Text.rich(TextSpan(
      children: _textSpanList(),
    ));
  }

  List<TextSpan> _textSpanList(){
    List<TextSpan> spanList = [];
    var jsonMap = json.decode(widget.rich_text_config);
    String message = jsonMap['message'];
    int defaultTextSize = jsonMap['textSize'];
    String defaultTextColorStr = jsonMap['textColor'];
    Color defaultTextColor = Color(int.parse('0xff' + defaultTextColorStr));
    List richTexts = jsonMap['richTexts']??[];
    if(richTexts.isEmpty){
      TextSpan defaultTextSpan = TextSpan(text: message,
        style: TextStyle(color:defaultTextColor, fontSize: defaultTextSize.toDouble()),
      );
      return [defaultTextSpan];
    }

    int currentIndex = 0;
    for(Map richText in richTexts){
      int startIndex = richText['startIndex'];

      if(currentIndex != startIndex){
        spanList.add(
            TextSpan(text: message.substring(currentIndex,startIndex),
              style: TextStyle(color:defaultTextColor, fontSize: defaultTextSize.toDouble()),
            )
        );
      }

      int endIndex = richText['endIndex'];
      currentIndex = endIndex;

      String childText = message.substring(startIndex,endIndex);
      int childTextSize = richText['textSize'];
      double textSize = richText['textSize'] == null?defaultTextSize.toDouble():childTextSize.toDouble();
      String textColorStr = richText['textColor']??defaultTextColorStr;
      Color textColor = Color(int.parse('0xff' + textColorStr));

      spanList.add(
          TextSpan(text: childText,
            style: TextStyle(color:textColor, fontSize: textSize),
          )
      );
    }

    if(currentIndex != message.length - 1){
      spanList.add(
          TextSpan(text: message.substring(currentIndex,message.length),
            style: TextStyle(color:defaultTextColor, fontSize: defaultTextSize.toDouble()),
          )
      );
    }
    return spanList;
  }
}


```
组件实现比较简单，主要包含一下几个步骤：

**JSON解析 ——> 生成富文本数组 ——> 渲染**

##4、 调用并展示
```
RichTextView('{"message":"一二三四五六七八九","textSize":20,"textColor":"FF0000","richTexts":[{"textSize":44,"textColor":"23238E","startIndex":0,"endIndex":3},{"textSize":66,"textColor":"545454","startIndex":5,"endIndex":7}]}')
```
![结果展示](https://images.xiaozhuanlan.com/photo/2022/39a2b7530dd52053c8182a016fb1f142.png)



#三、发布前的工作
##1、将代码提交到gitbub
将仓库代码提交到github，我的仓库地址：https://github.com/xlgy/rich_text_from_json


##2、更新pubspec.yaml
```
name: lxy_rich_text_from_json
description: json配置动态富文本
version: 0.0.3
homepage: https://github.com/xlgy/rich_text_from_json.git
```
name：包名称
description：描述
version：版本号
homepage：github仓库地址

##3、更新CHANGLOG.md
更新说明文案
```
## 0.0.3

更新说明文案，完善配置信息说明
```

该版本记录会显示在pub的Changlog中：
![](https://images.xiaozhuanlan.com/photo/2022/976ed0596e476a5f58740713ffcf105e.png)

## 4、更新README.md
![](https://images.xiaozhuanlan.com/photo/2022/60809a422a907c1dcb4751e68ee993e0.png)

README会显示在pub的Readme中：
![](https://images.xiaozhuanlan.com/photo/2022/f2ababea13d860129e65de2fc149b8b2.png)

##5、运行 dry-run 命令
在package根目录下运行 dry-run 命令以查看是否都准备OK了:
```
flutter packages pub publish --dry-run
```
![](https://images.xiaozhuanlan.com/photo/2022/bc381945732a2d16954748d681780806.png)
从输出可以看出0 warnings，表示目前上传没有问题


## 6、运行发布命令
```
flutter packages pub publish --server=https://pub.dartlang.org
```
- 输入上面命令后，会提示是否publish，输入 **y** 即可。
- 如果是第一次发布或距离上次发布的时间比较久，则会输出一个Google账号授权的url。点击url浏览器打开后，进行Google账号授权。
- 账后授权后，开始Uploading…
- 发布成功，你就会看到输出Successful uploaded package。



## 7、网络环境
- 需要**科学上网**
- 终端执行"url google.com"，看看是否正常如下图：
![](https://images.xiaozhuanlan.com/photo/2022/afac51cf90af4dab698bd68867dc9cc8.png)


#四、结果链接
pub.dev链接：https://pub.dev/packages/lxy_rich_text_from_json
GitHub链接：https://github.com/xlgy/rich_text_from_json