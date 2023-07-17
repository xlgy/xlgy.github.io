---
layout:     post
title:      SDWebImage 源码解析(一)
subtitle:   SDWebImage 源码解析(一)
date:       2023-07-017
author:     LXY
header-img: img/home1.jpeg
catalog: true
tags:
    - iOS
---

## 一、前言

SDWebImage是一个十分有名的第三方开源框架，用于加载网络图片、及相关缓存管理

## 二、框架结构


- Downloader(下载图片)：
	- SDWebImageDownloader
	- SDWebImageDownloaderOperation 

- Cache(图片缓存)：
	- SDImageCache

- Utils：
	- SDWebImageDecoder(管理类)
	- SDWebImageManager(图片解压缩类)
	- SDWebImagePrefetcher(图片预加载类)

- Categories：
	- NSData+ImageContentType
	- UIImage+GIF
	- UIImage+MultiFormat
	- UIImageView+WebCache
	- UIImageView+HighlightedWebCache
	- UIButton+WebCache
	- UIView+WebCacheOperation

- other：
	- SDWebImageCompat (protocol)
	- SDWebImageCompat (宏定义、常量、内联函数)


## 三、缓存源码解析


SDImageCache 是用来缓存图片到内存以及硬盘，它主要包含以下几类方法：

- 创建Cache空间和路径
- 存储图片
- 读取图片
- 删除图片
- 删除缓存
- 清理缓存
- 获取硬盘缓存大小
- 判断key是否存在在硬盘

### a. 创建Cache空间和路径

**对外使用：**




