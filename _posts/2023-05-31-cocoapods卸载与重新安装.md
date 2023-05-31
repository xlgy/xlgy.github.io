---
layout:     post
title:      cocoapods卸载与重新安装
subtitle:   cocoapods卸载与重新安装
date:       2023-05-31
author:     LXY
header-img: img/home1.jpeg
catalog: true
tags:
    - 微经验
---


### 1、检查残留及删除

检查是否有安装残留（删除CocoaPods）
如果之前装过cocopods，最好先卸载掉，卸载命令：

```
sudo gem uninstall cocoapods
```

先查看本地安装过的cocopods相关东西，命令如下：

```
gem list --local | grep cocoapods
```

会显示如下：

```
cocoapods-core (0.39.0)
cocoapods-downloader (0.9.3)
cocoapods-plugins (0.4.2)
cocoapods-search (0.1.0)
cocoapods-stats (0.6.2)
cocoapods-trunk (0.6.4)
cocoapods-try (0.5.1)

```

然后逐个删除吧

```
示例  sudo gem uninstall cocoapods-core
```

### 2、检查残留及删除

##### a.安装Homebrew:

```
/bin/zsh -c "$(curl -fsSL https://gitee.com/cunkai/HomebrewCN/raw/master/Homebrew.sh)"
```
然后按指示操作


##### b.查看安装Homebrew:

```
brew -v
```

M1新机此处应该会有两个致命错误,大概就是说要设置cask、core两个文件路径为设置为safe.directory,如果没有错误则忽略此处.
按提示操作即可:

```
git config --global --add safe.directory /opt/homebrew/Library/Taps/homebrew/homebrew-core
git config --global --add safe.directory /opt/homebrew/Library/Taps/homebrew/homebrew-cask
```

##### c.安装Cocoapods:

```
brew install cocoapods
```

##### d.设置Cocoapods:

```
pod setup
```
