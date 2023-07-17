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

```
//单例
+ (SDImageCache *)sharedImageCache;
//初始化一个namespace空间
- (id)initWithNamespace:(NSString *)ns;
//初始化一个namespace空间和存储的路径
- (id)initWithNamespace:(NSString *)ns diskCacheDirectory:(NSString *)directory;
//存储路径
-(NSString *)makeDiskCachePath:(NSString*)fullNamespace;
//添加只读cache的path
- (void)addReadOnlyCachePath:(NSString *)path;
```

**内部核心方法实现：**

```
- (id)initWithNamespace:(NSString *)ns diskCacheDirectory:(NSString *)directory {
    if ((self = [super init])) {
        NSString *fullNamespace = [@"com.hackemist.SDWebImageCache." stringByAppendingString:ns];

        // initialise PNG signature data
        //初始化PNG图片的标签数据
        kPNGSignatureData = [NSData dataWithBytes:kPNGSignatureBytes length:8];

        // Create IO serial queue
        //创建 serial queue
        _ioQueue = dispatch_queue_create("com.hackemist.SDWebImageCache", DISPATCH_QUEUE_SERIAL);

        // Init default values
        //默认存储时间一周
        _maxCacheAge = kDefaultCacheMaxCacheAge;

        // Init the memory cache
        //初始化内存cache对象
        _memCache = [[AutoPurgeCache alloc] init];
        _memCache.name = fullNamespace;

        // Init the disk cache
        //初始化硬盘缓存路径
        if (directory != nil) {
            _diskCachePath = [directory stringByAppendingPathComponent:fullNamespace];
        } else {
            //默认路径
            NSString *path = [self makeDiskCachePath:ns];
            _diskCachePath = path;
        }

        // Set decompression to YES
        //需要压缩图片
        _shouldDecompressImages = YES;

        // memory cache enabled
        //需要cache在内存
        _shouldCacheImagesInMemory = YES;

        // Disable iCloud
        //禁止icloud
        _shouldDisableiCloud = YES;

        dispatch_sync(_ioQueue, ^{
            _fileManager = [NSFileManager new];
        });

#if TARGET_OS_IOS
        // Subscribe to app events
        //app相关warning，包括内存warning，app关闭notification，到后台notification
        [[NSNotificationCenter defaultCenter] addObserver:self
                                                 selector:@selector(clearMemory)
                                                     name:UIApplicationDidReceiveMemoryWarningNotification
                                                   object:nil];

        [[NSNotificationCenter defaultCenter] addObserver:self
                                                 selector:@selector(cleanDisk)
                                                     name:UIApplicationWillTerminateNotification
                                                   object:nil];

        [[NSNotificationCenter defaultCenter] addObserver:self
                                                 selector:@selector(backgroundCleanDisk)
                                                     name:UIApplicationDidEnterBackgroundNotification
                                                   object:nil];
#endif
    }

    return self;
}
```

**源码重点关注:**

- 存储策略默认存储时间是**一周

```
static const NSInteger kDefaultCacheMaxCacheAge = 60 * 60 * 24 * 7; // 1 week

_maxCacheAge = kDefaultCacheMaxCacheAge;
```

-  默认存储沙盒地址

```
.../Documents/default/com.hackemist.SDWebImageCache.default
```
此时"default"为namespace空间

- ```kPNGSignatureData```是用来判断是否是PNG图片的前缀标签数据，它的初始化数据kPNGSignatureBytes的来源如下：

```
// PNG图片的标签值
static unsigned char kPNGSignatureBytes[8] = {0x89, 0x50, 0x4E, 0x47, 0x0D, 0x0A, 0x1A, 0x0A};
```

### b. 存储图片

存储图片主要分为2块：

- 存储到内存
	- 直接计算图片大小后，用```NSCache *memCache```存储

- 存储到硬盘
	- 先将UIImage转为NSData，然后通过```NSFileManager *_fileManager```创建存储路径文件夹和图片存储文件

**对外使用：**

```
//存储image到key
- (void)storeImage:(UIImage *)image forKey:(NSString *)key;

//存储image到key，是否硬盘缓存
- (void)storeImage:(UIImage *)image forKey:(NSString *)key toDisk:(BOOL)toDisk;

//存储image，存储数据是否需要image转化(recalculate = true 则存储image，否则存储imageData)
- (void)storeImage:(UIImage *)image recalculateFromImage:(BOOL)recalculate imageData:(NSData *)imageData forKey:(NSString *)key toDisk:(BOOL)toDisk;

//硬盘缓存，key
- (void)storeImageDataToDisk:(NSData *)imageData forKey:(NSString *)key;
```

**内部核心方法实现：**

```
//存储图片基础方法
- (void)storeImage:(UIImage *)image recalculateFromImage:(BOOL)recalculate imageData:(NSData *)imageData forKey:(NSString *)key toDisk:(BOOL)toDisk {
    if (!image || !key) {
        return;
    }
    // if memory cache is enabled
    //是否缓存在内存中国年
    if (self.shouldCacheImagesInMemory) {
        //获取图片大小
        NSUInteger cost = SDCacheCostForImage(image);
        //设置缓存
        [self.memCache setObject:image forKey:key cost:cost];
    }

    //是否硬盘存储
    if (toDisk) {
        dispatch_async(self.ioQueue, ^{
            NSData *data = imageData;

            if (image && (recalculate || !data)) {
#if TARGET_OS_IPHONE
                int alphaInfo = CGImageGetAlphaInfo(image.CGImage);
                //判断是否有alpha
                BOOL hasAlpha = !(alphaInfo == kCGImageAlphaNone ||
                                  alphaInfo == kCGImageAlphaNoneSkipFirst ||
                                  alphaInfo == kCGImageAlphaNoneSkipLast);
                //有alpha肯定是png
                BOOL imageIsPng = hasAlpha;

                //查看图片前几个字节，是否是png
                if ([imageData length] >= [kPNGSignatureData length]) {
                    imageIsPng = ImageDataHasPNGPreffix(imageData);
                }

                //png图
                if (imageIsPng) {
                    data = UIImagePNGRepresentation(image);
                }
                //jpeg图
                else {
                    data = UIImageJPEGRepresentation(image, (CGFloat)1.0);
                }
#else
                data = [NSBitmapImageRep representationOfImageRepsInArray:image.representations usingType: NSJPEGFileType properties:nil];
#endif
            }
            //存储到硬盘
            [self storeImageDataToDisk:data forKey:key];
        });
    }
}
```

其中对于获取key的完整路径处理```NSString *cachePathForKey = [self defaultCachePathForKey:key];```，还做了md5的加密，具体实现如下：

```
//path组合key
- (NSString *)cachePathForKey:(NSString *)key inPath:(NSString *)path {
    NSString *filename = [self cachedFileNameForKey:key];
    return [path stringByAppendingPathComponent:filename];
}

//默认路径的keypath
- (NSString *)defaultCachePathForKey:(NSString *)key {
    return [self cachePathForKey:key inPath:self.diskCachePath];
}

#pragma mark SDImageCache (private)

//MD5加密key
- (NSString *)cachedFileNameForKey:(NSString *)key {
    const char *str = [key UTF8String];
    if (str == NULL) {
        str = "";
    }
    unsigned char r[CC_MD5_DIGEST_LENGTH];
    CC_MD5(str, (CC_LONG)strlen(str), r);
    NSString *filename = [NSString stringWithFormat:@"%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%@",
                          r[0], r[1], r[2], r[3], r[4], r[5], r[6], r[7], r[8], r[9], r[10],
                          r[11], r[12], r[13], r[14], r[15], [[key pathExtension] isEqualToString:@""] ? @"" : [NSString stringWithFormat:@".%@", [key pathExtension]]];

    return filename;
}
```

### c. 删除图片

**对外使用：**

```
//通过key，删除图片
- (void)removeImageForKey:(NSString *)key;

//异步删除图片，调用完成block
- (void)removeImageForKey:(NSString *)key withCompletion:(SDWebImageNoParamsBlock)completion;

//删除图片，同时从硬盘删除
- (void)removeImageForKey:(NSString *)key fromDisk:(BOOL)fromDisk;

//异步删除，同时删除硬盘
- (void)removeImageForKey:(NSString *)key fromDisk:(BOOL)fromDisk withCompletion:(SDWebImageNoParamsBlock)completion;
```

**内部核心方法实现：**

```
//删除图片基础方法
- (void)removeImageForKey:(NSString *)key fromDisk:(BOOL)fromDisk withCompletion:(SDWebImageNoParamsBlock)completion {
    
    //不存在key返回
    if (key == nil) {
        return;
    }

    //需要用memory，先从cache删除图片
    if (self.shouldCacheImagesInMemory) {
        [self.memCache removeObjectForKey:key];
    }
    
    //硬盘删除
    if (fromDisk) {
        dispatch_async(self.ioQueue, ^{
            //直接删除路径下key对应的文件名字
            [_fileManager removeItemAtPath:[self defaultCachePathForKey:key] error:nil];
            
            if (completion) {
                dispatch_async(dispatch_get_main_queue(), ^{
                    completion();
                });
            }
        });
    } else if (completion){
        completion();
    }
    
}
```

### d.删除缓存


**对外使用：**

```
//删除缓存图片
- (void)clearMemory;

//删除硬盘存储图片，调用完成block
- (void)clearDiskOnCompletion:(SDWebImageNoParamsBlock)completion;

//删除硬盘存储图片
- (void)clearDisk;
```


### e.清理缓存

清理缓存的目标是将过期的图片和确保硬盘存储大小小于最大存储许可。
清理缓存的方法也是分为两步：

- 清除所有过期图片
- 在硬盘存储大小大于最大存储许可大小时，将旧图片进行删除，删除到一半最大许可以下

**对外使用：**

```
//清理硬盘缓存图片
- (void)cleanDiskWithCompletionBlock:(SDWebImageNoParamsBlock)completionBlock;

//清理硬盘缓存图片
- (void)cleanDisk;
```

**内部核心方法实现：**

```
//清理硬盘基础方法
//清理硬盘的目的是为了缓解硬盘压力，以及过期图片，超大小图片等
- (void)cleanDiskWithCompletionBlock:(SDWebImageNoParamsBlock)completionBlock {
    dispatch_async(self.ioQueue, ^{
        //硬盘存储路径获取
        NSURL *diskCacheURL = [NSURL fileURLWithPath:self.diskCachePath isDirectory:YES];
        //是否是目录，获取修改日期，获取size大小
        NSArray *resourceKeys = @[NSURLIsDirectoryKey, NSURLContentModificationDateKey, NSURLTotalFileAllocatedSizeKey];

        //迭代器遍历
        NSDirectoryEnumerator *fileEnumerator = [_fileManager enumeratorAtURL:diskCacheURL
                                                   includingPropertiesForKeys:resourceKeys
                                                                      options:NSDirectoryEnumerationSkipsHiddenFiles
                                                                 errorHandler:NULL];
        //获取应该过期的日期
        NSDate *expirationDate = [NSDate dateWithTimeIntervalSinceNow:-self.maxCacheAge];
        //缓存路径与3个key值
        NSMutableDictionary *cacheFiles = [NSMutableDictionary dictionary];
        //获取总的目录下文件大小
        NSUInteger currentCacheSize = 0;
        
        //需要删除的路径存储
        NSMutableArray *urlsToDelete = [[NSMutableArray alloc] init];
        //遍历路径下的所有文件于文件夹
        for (NSURL *fileURL in fileEnumerator) {
            //获取该文件路径的3种值
            NSDictionary *resourceValues = [fileURL resourceValuesForKeys:resourceKeys error:NULL];

            //跳过目录
            if ([resourceValues[NSURLIsDirectoryKey] boolValue]) {
                continue;
            }

            //获取修改日期
            NSDate *modificationDate = resourceValues[NSURLContentModificationDateKey];
            //在有效期日期之前的文件需要删除，添加到
            if ([[modificationDate laterDate:expirationDate] isEqualToDate:expirationDate]) {
                [urlsToDelete addObject:fileURL];
                continue;
            }

            NSNumber *totalAllocatedSize = resourceValues[NSURLTotalFileAllocatedSizeKey];
            //获取文件大小
            currentCacheSize += [totalAllocatedSize unsignedIntegerValue];
            //存储值
            [cacheFiles setObject:resourceValues forKey:fileURL];
        }
        
        //删除所有的过期图片
        for (NSURL *fileURL in urlsToDelete) {
            [_fileManager removeItemAtURL:fileURL error:nil];
        }

        //如果当前硬盘存储大于最大缓存，则删除一半的硬盘存储，先删除老的图片
        if (self.maxCacheSize > 0 && currentCacheSize > self.maxCacheSize) {
 
            //目标空间
            const NSUInteger desiredCacheSize = self.maxCacheSize / 2;

            
            //排序所有filepath，时间排序
            NSArray *sortedFiles = [cacheFiles keysSortedByValueWithOptions:NSSortConcurrent
                                                            usingComparator:^NSComparisonResult(id obj1, id obj2) {
                                                                return [obj1[NSURLContentModificationDateKey] compare:obj2[NSURLContentModificationDateKey]];
                                                            }];

            //删除文件，直到小于目标空间停止
            for (NSURL *fileURL in sortedFiles) {
                if ([_fileManager removeItemAtURL:fileURL error:nil]) {
                    NSDictionary *resourceValues = cacheFiles[fileURL];
                    NSNumber *totalAllocatedSize = resourceValues[NSURLTotalFileAllocatedSizeKey];
                    currentCacheSize -= [totalAllocatedSize unsignedIntegerValue];

                    if (currentCacheSize < desiredCacheSize) {
                        break;
                    }
                }
            }
        }
        //回调block
        if (completionBlock) {
            dispatch_async(dispatch_get_main_queue(), ^{
                completionBlock();
            });
        }
    });
}
```

当app进入后台时，会调用 ```backgroundCleanDisk```，内部调用就是清理硬盘内存。所以：每次进入后台的时候都会触发清理磁盘操作！！！

### f.获取硬盘缓存大小

```
typedef void(^SDWebImageCalculateSizeBlock)(NSUInteger fileCount, NSUInteger totalSize);
//获得硬盘存储空间大小
- (NSUInteger)getSize;

//获得硬盘空间图片数量
- (NSUInteger)getDiskCount;

//异步计算硬盘空间大小
- (void)calculateSizeWithCompletionBlock:(SDWebImageCalculateSizeBlock)completionBlock;
```


### g.判断key是否存在在硬盘

```
//异步判断是否存在key的图片
- (void)diskImageExistsWithKey:(NSString *)key completion:(SDWebImageCheckCacheCompletionBlock)completionBlock;

//同步判断是否存在key的图片
- (BOOL)diskImageExistsWithKey:(NSString *)key;
```

###总结

- SDImageCache的存储包括内存及磁盘
- SDImageCache会在ApplicationDidEnterBackground时清理资源，默认磁盘存储时间是一周
- 图片的key使用md5加密，对资源做映射
- 获取图片数据前缀标签数据，判断图片是否是png


