---
layout:     post
title:      最简单的 Mac配置gitlab ssh密钥方法
subtitle:   最简单的 Mac配置gitlab ssh密钥方法
date:       2023-05-25
author:     LXY
header-img: img/home1.jpeg
catalog: true
tags:
    - 微经验
---


## 前言
之前尝试过按照网上的方法配置密钥，虽然配置成功了但是每次进行任何操作还是得输入密码（不用输账号，只是输入 .rsa.pub的passphrase），还是很不方便，自己重新配置了下，尝试了一下，不用输密码了。

## 操作
在有了gitlab账号后：

- 在终端(根目录就行)输入 ssh-keygen -t rsa -C  + gitlab上的email。
 
	**比如 ssh-keygen -t rsa -C 12345678@xx.com.**
			
- 回车之后会让你输入存储id_rsa和id_rsa.pub的目录，不用管直接继续回车即可，此时存储的就是默认目录
 
- 回车之后会出现让输入密码，关键的来了，这个密码~**不要输入任何东西，直接回车**（不然每次进行git和远程仓库有关系的操作的时候都得输入这个密码）。这两步直接enter之后密钥对就创建成功了

 ![](https://p.ipic.vip/3r9gej.png)
 
 
- 接下来去前往文件夹(command+shift+G)
点开之后直接在输入框里输入 ~/.ssh 然后回车，就会出现id_rsa和id_rsa.pub两个文件。右键打用文本编辑打开id_rsa.pub，将里面的东西全部复制

- 配置gitlab
打开gitlab，点击右上角红框位置打开settings，进入settings后，点击左侧SSH Keys，把刚才复制的id_rsa.pub里的东西粘贴到秘钥的输入框里，（ title可以随便写，也可以什么都不写）然后点击添加秘钥
![](https://p.ipic.vip/72zzr9.png)
 