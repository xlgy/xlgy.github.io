---
layout:     post
title:      SDWebImage 源码解析(一)
subtitle:   SDWebImage 源码解析(一)
date:       2023-07-17
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


## 四、下载源码解析

Download
Download部分主要包含如下2个方法：

 - SDWebImageDownloader 图片下载控制类
 - SDWebImageDownloaderOperation NSOperation子类

### SDWebImageDownloader

SDWebImageDownloader主要包含以下内容：

- 属性及初始化实现
- 相关请求信息设置与获取
- 请求图片方法
- NSURLSession回调转发

####  a. 属性及初始化实现

**相关属性：**

SDWebImageDownloader.h中的property声明：

```
//下载完成执行顺序
typedef NS_ENUM(NSInteger, SDWebImageDownloaderExecutionOrder) {
    //先进先出
    SDWebImageDownloaderFIFOExecutionOrder,
    //先进后出
    SDWebImageDownloaderLIFOExecutionOrder
};

@interface SDWebImageDownloader : NSObject
//是否压缩图片
@property (assign, nonatomic) BOOL shouldDecompressImages;
//最大下载数量
@property (assign, nonatomic) NSInteger maxConcurrentDownloads;
//当前下载数量
@property (readonly, nonatomic) NSUInteger currentDownloadCount;
//下载超时时间
@property (assign, nonatomic) NSTimeInterval downloadTimeout;
//下载策略
@property (assign, nonatomic) SDWebImageDownloaderExecutionOrder executionOrder;
```

SDWebImageDownloader.m中的Extension：

```
@interface SDWebImageDownloader () <NSURLSessionTaskDelegate, NSURLSessionDataDelegate>
//下载的queue
@property (strong, nonatomic) NSOperationQueue *downloadQueue;
//上一个添加的操作
@property (weak, nonatomic) NSOperation *lastAddedOperation;
//操作类
@property (assign, nonatomic) Class operationClass;
//url请求缓存
@property (strong, nonatomic) NSMutableDictionary *URLCallbacks;
//请求头
@property (strong, nonatomic) NSMutableDictionary *HTTPHeaders;
// This queue is used to serialize the handling of the network responses of all the download operation in a single queue
//这个queue为了能否序列化处理所有网络结果的返回
@property (SDDispatchQueueSetterSementics, nonatomic) dispatch_queue_t barrierQueue;

// The session in which data tasks will run
//数据runsession
@property (strong, nonatomic) NSURLSession *session;

@end
```

**内部初始化方法：**

```
//初始化方法
- (id)init {
    if ((self = [super init])) {
        _operationClass = [SDWebImageDownloaderOperation class];
        _shouldDecompressImages = YES;
        _executionOrder = SDWebImageDownloaderFIFOExecutionOrder;
        _downloadQueue = [NSOperationQueue new];
        _downloadQueue.maxConcurrentOperationCount = 6;
        _downloadQueue.name = @"com.hackemist.SDWebImageDownloader";
        _URLCallbacks = [NSMutableDictionary new];
#ifdef SD_WEBP
        _HTTPHeaders = [@{@"Accept": @"image/webp,image/*;q=0.8"} mutableCopy];
#else
        _HTTPHeaders = [@{@"Accept": @"image/*;q=0.8"} mutableCopy];
#endif
        _barrierQueue = dispatch_queue_create("com.hackemist.SDWebImageDownloaderBarrierQueue", DISPATCH_QUEUE_CONCURRENT);
        _downloadTimeout = 15.0;

        NSURLSessionConfiguration *sessionConfig = [NSURLSessionConfiguration defaultSessionConfiguration];
        sessionConfig.timeoutIntervalForRequest = _downloadTimeout;

        //session初始化
        self.session = [NSURLSession sessionWithConfiguration:sessionConfig
                                                     delegate:self
                                                delegateQueue:nil];
    }
    return self;
}
```

从这边也可以看出，SDWebImageDownloader的下载方法就是用NSOperation。使用NSOperationQueue去控制最大操作数量和取消所有操作，在NSOperation中运行NSUrlSession方法请求参数。

####  b. 相关请求信息设置与获取

```
//设置hedaer头
- (void)setValue:(NSString *)value forHTTPHeaderField:(NSString *)field {
    if (value) {
        self.HTTPHeaders[field] = value;
    }
    else {
        [self.HTTPHeaders removeObjectForKey:field];
    }
}

- (NSString *)valueForHTTPHeaderField:(NSString *)field {
    return self.HTTPHeaders[field];
}

//设置多大并发数量
- (void)setMaxConcurrentDownloads:(NSInteger)maxConcurrentDownloads {
    _downloadQueue.maxConcurrentOperationCount = maxConcurrentDownloads;
}

- (NSUInteger)currentDownloadCount {
    return _downloadQueue.operationCount;
}

- (NSInteger)maxConcurrentDownloads {
    return _downloadQueue.maxConcurrentOperationCount;
}

//设置operation的类型，可以自行再继承子类去实现
- (void)setOperationClass:(Class)operationClass {
    _operationClass = operationClass ?: [SDWebImageDownloaderOperation class];
}

//暂停下载
- (void)setSuspended:(BOOL)suspended {
    [self.downloadQueue setSuspended:suspended];
}
//取消所有下载
- (void)cancelAllDownloads {
    [self.downloadQueue cancelAllOperations];
}
```

```setOperationClass```是为了可以自己实现NSOperation的子类去完成相关请求的设置。


####  c. 请求图片方法


```
- (void)addProgressCallback:(SDWebImageDownloaderProgressBlock)progressBlock completedBlock:(SDWebImageDownloaderCompletedBlock)completedBlock forURL:(NSURL *)url createCallback:(SDWebImageNoParamsBlock)createCallback {
 ...
}
```

这个方法主要对回调进行存储维护，将progressblock和completeblock存入dictionary，再添加到self.URLCallbacks[url]的array中

```
        NSMutableArray *callbacksForURL = self.URLCallbacks[url];
        NSMutableDictionary *callbacks = [NSMutableDictionary new];
        if (progressBlock) callbacks[kProgressCallbackKey] = [progressBlock copy];
        if (completedBlock) callbacks[kCompletedCallbackKey] = [completedBlock copy];
        [callbacksForURL addObject:callbacks];
        self.URLCallbacks[url] = callbacksForURL;

```
   

```
//请求图片方法
- (id <SDWebImageOperation>)downloadImageWithURL:(NSURL *)url options:(SDWebImageDownloaderOptions)options progress:(SDWebImageDownloaderProgressBlock)progressBlock completed:(SDWebImageDownloaderCompletedBlock)completedBlock {
 ...
}
```

网络请求相关代码：

```
        //这边的创建方式允许自己定义SDWebImageDownloaderOperation的子类进行替换
        operation = [[wself.operationClass alloc] initWithRequest:request
                                                        inSession:self.session
                                                          options:options
                                                         progress:^(NSInteger receivedSize, NSInteger expectedSize) {
                                                             SDWebImageDownloader *sself = wself;
                                                             if (!sself) return;
                                                             __block NSArray *callbacksForURL;
                                                             //这里用barrierQueue可以确保数据一致性
                                                             dispatch_sync(sself.barrierQueue, ^{
                                                                 //获取所有的同url的请求
                                                                 callbacksForURL = [sself.URLCallbacks[url] copy];
                                                             });
                                                             //遍历创建信息函数
                                                             for (NSDictionary *callbacks in callbacksForURL) {
                                                                 dispatch_async(dispatch_get_main_queue(), ^{
                                                                     //获取progressblock并调用
                                                                     SDWebImageDownloaderProgressBlock callback = callbacks[kProgressCallbackKey];
                                                                     if (callback) callback(receivedSize, expectedSize);
                                                                 });
                                                             }
                                                         }
                                                        completed:^(UIImage *image, NSData *data, NSError *error, BOOL finished) {
                                                            SDWebImageDownloader *sself = wself;
                                                            if (!sself) return;
                                                            __block NSArray *callbacksForURL;
                                                            //等待新请求插入和获取参数完结，再finish
                                                            dispatch_barrier_sync(sself.barrierQueue, ^{
                                                                callbacksForURL = [sself.URLCallbacks[url] copy];
                                                                //如果完结了，删除所有url缓存
                                                                if (finished) {
                                                                    [sself.URLCallbacks removeObjectForKey:url];
                                                                }
                                                            });
                                                            //调用completeblock
                                                            for (NSDictionary *callbacks in callbacksForURL) {
                                                                SDWebImageDownloaderCompletedBlock callback = callbacks[kCompletedCallbackKey];
                                                                if (callback) callback(image, data, error, finished);
                                                            }
                                                        }
                                                        cancelled:^{
                                                            SDWebImageDownloader *sself = wself;
                                                            if (!sself) return;
                                                            //删除url
                                                            dispatch_barrier_async(sself.barrierQueue, ^{
                                                                [sself.URLCallbacks removeObjectForKey:url];
                                                            });
                                                        }];
        //是否压缩图片
        operation.shouldDecompressImages = wself.shouldDecompressImages;
        
        //是否包含url的验证
        if (wself.urlCredential) {
            operation.credential = wself.urlCredential;
        } else if (wself.username && wself.password) {
            //有账号和密码则设置请求
            operation.credential = [NSURLCredential credentialWithUser:wself.username password:wself.password persistence:NSURLCredentialPersistenceForSession];
        }
        //设置operation请求优先级
        if (options & SDWebImageDownloaderHighPriority) {
            operation.queuePriority = NSOperationQueuePriorityHigh;
        } else if (options & SDWebImageDownloaderLowPriority) {
            operation.queuePriority = NSOperationQueuePriorityLow;
        }

        //开始请求[operation start]
        [wself.downloadQueue addOperation:operation];
        //如果是先进后出，链接依赖
        if (wself.executionOrder == SDWebImageDownloaderLIFOExecutionOrder) {
            // Emulate LIFO execution order by systematically adding new operations as last operation's dependency
            [wself.lastAddedOperation addDependency:operation];
            wself.lastAddedOperation = operation;
        }
    }];
```

####  d. NSURLSession回调转发

```
#pragma mark Helper methods

//获取operationQueue中的包含此task的operation
- (SDWebImageDownloaderOperation *)operationWithTask:(NSURLSessionTask *)task {
    SDWebImageDownloaderOperation *returnOperation = nil;
    for (SDWebImageDownloaderOperation *operation in self.downloadQueue.operations) {
        if (operation.dataTask.taskIdentifier == task.taskIdentifier) {
            returnOperation = operation;
            break;
        }
    }
    return returnOperation;
}

#pragma mark NSURLSessionDataDelegate

//收到返回结果
- (void)URLSession:(NSURLSession *)session
          dataTask:(NSURLSessionDataTask *)dataTask
didReceiveResponse:(NSURLResponse *)response
 completionHandler:(void (^)(NSURLSessionResponseDisposition disposition))completionHandler {

    // Identify the operation that runs this task and pass it the delegate method
    SDWebImageDownloaderOperation *dataOperation = [self operationWithTask:dataTask];

    [dataOperation URLSession:session dataTask:dataTask didReceiveResponse:response completionHandler:completionHandler];
}

//收到返回数据
- (void)URLSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask didReceiveData:(NSData *)data {

    // Identify the operation that runs this task and pass it the delegate method
    SDWebImageDownloaderOperation *dataOperation = [self operationWithTask:dataTask];

    [dataOperation URLSession:session dataTask:dataTask didReceiveData:data];
}

//是否缓存reponse
- (void)URLSession:(NSURLSession *)session
          dataTask:(NSURLSessionDataTask *)dataTask
 willCacheResponse:(NSCachedURLResponse *)proposedResponse
 completionHandler:(void (^)(NSCachedURLResponse *cachedResponse))completionHandler {

    // Identify the operation that runs this task and pass it the delegate method
    SDWebImageDownloaderOperation *dataOperation = [self operationWithTask:dataTask];

    [dataOperation URLSession:session dataTask:dataTask willCacheResponse:proposedResponse completionHandler:completionHandler];
}

#pragma mark NSURLSessionTaskDelegate

//task已经完成
- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task didCompleteWithError:(NSError *)error {
    // Identify the operation that runs this task and pass it the delegate method
    SDWebImageDownloaderOperation *dataOperation = [self operationWithTask:task];

    [dataOperation URLSession:session task:task didCompleteWithError:error];
}

//task请求验证方法
- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task didReceiveChallenge:(NSURLAuthenticationChallenge *)challenge completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition disposition, NSURLCredential *credential))completionHandler {

    // Identify the operation that runs this task and pass it the delegate method
    SDWebImageDownloaderOperation *dataOperation = [self operationWithTask:task];

    [dataOperation URLSession:session task:task didReceiveChallenge:challenge completionHandler:completionHandler];
}
```

可以看到NSURLSessionTask的代理回调都转发给SDWebImageDownloaderOperation中


### SDWebImageDownloaderOperation

SDWebImageDownloaderOperation主要包含以下内容：

 - 属性及初始化实现
 - 相关参数设置
 - 开始请求与取消请求
 - NSURLSession相关回调处理

#### a、属性及初始化实现

```
@interface SDWebImageDownloaderOperation : NSOperation <SDWebImageOperation, NSURLSessionTaskDelegate, NSURLSessionDataDelegate>
//operation的请求
@property (strong, nonatomic, readonly) NSURLRequest *request;
//operation的task
@property (strong, nonatomic, readonly) NSURLSessionTask *dataTask;
//是否压缩图片
@property (assign, nonatomic) BOOL shouldDecompressImages;
//认证令牌
@property (nonatomic, strong) NSURLCredential *credential;
//下载方式
@property (assign, nonatomic, readonly) SDWebImageDownloaderOptions options;
//文件大小
@property (assign, nonatomic) NSInteger expectedSize;
//operation的返回
@property (strong, nonatomic) NSURLResponse *response;
```

```
@interface SDWebImageDownloaderOperation ()

//进度回调
@property (copy, nonatomic) SDWebImageDownloaderProgressBlock progressBlock;
//完成回调
@property (copy, nonatomic) SDWebImageDownloaderCompletedBlock completedBlock;
//取消回调
@property (copy, nonatomic) SDWebImageNoParamsBlock cancelBlock;
//是否在执行中
@property (assign, nonatomic, getter = isExecuting) BOOL executing;
//是否完成
@property (assign, nonatomic, getter = isFinished) BOOL finished;
//图片数据
@property (strong, nonatomic) NSMutableData *imageData;
//与此操作关联的任务
@property (weak, nonatomic) NSURLSession *unownedSession;
//兜底默认任务，内部实例化
@property (strong, nonatomic) NSURLSession *ownedSession;
//sessiontask
@property (strong, nonatomic, readwrite) NSURLSessionTask *dataTask;
//当前线程
@property (strong, atomic) NSThread *thread;

#if TARGET_OS_IPHONE && __IPHONE_OS_VERSION_MAX_ALLOWED >= __IPHONE_4_0
@property (assign, nonatomic) UIBackgroundTaskIdentifier backgroundTaskId;
#endif
```

```
- (id)initWithRequest:(NSURLRequest *)request
            inSession:(NSURLSession *)session
              options:(SDWebImageDownloaderOptions)options
             progress:(SDWebImageDownloaderProgressBlock)progressBlock
            completed:(SDWebImageDownloaderCompletedBlock)completedBlock
            cancelled:(SDWebImageNoParamsBlock)cancelBlock {
    if ((self = [super init])) {
        _request = [request copy];
        _shouldDecompressImages = YES;
        _options = options;
        _progressBlock = [progressBlock copy];
        _completedBlock = [completedBlock copy];
        _cancelBlock = [cancelBlock copy];
        _executing = NO;
        _finished = NO;
        _expectedSize = 0;
        _unownedSession = session;
        responseFromCached = YES; // Initially wrong until `- URLSession:dataTask:willCacheResponse:completionHandler: is called or not called
    }
    return self;
}
```

#### b、相关参数设置

```
//重写set方法，并且设置kvo的观察回调
- (void)setFinished:(BOOL)finished {
    [self willChangeValueForKey:@"isFinished"];
    _finished = finished;
    [self didChangeValueForKey:@"isFinished"];
}

//重写excute方法，并且设置kvo的观察回调
- (void)setExecuting:(BOOL)executing {
    [self willChangeValueForKey:@"isExecuting"];
    _executing = executing;
    [self didChangeValueForKey:@"isExecuting"];
}

//是否允许并发
- (BOOL)isConcurrent {
    return YES;
}
```

重写的时候还注意到了原来相关属性的KVO值。


####  c、开始请求与取消请求

主要就是重写NSOperation的start和cancel方法，其中cancel方法还是SDWebImageOperation的回调。


start方法核心实现：

```
- (void)start {
...
              //后台任务，取消请求
            self.backgroundTaskId = [app beginBackgroundTaskWithExpirationHandler:^{
                __strong __typeof (wself) sself = wself;

                if (sself) {
                    [sself cancel];

                    [app endBackgroundTask:sself.backgroundTaskId];
                    sself.backgroundTaskId = UIBackgroundTaskInvalid;
                }
            }];
....
        self.dataTask = [session dataTaskWithRequest:self.request];
....
        //发起请求
         [self.dataTask resume];
...
self.completedBlock()
}
```

cancel方法核心实现：

```

- (void)cancel {
    @synchronized (self) {
        //有self.thread说明该operation已经开始下载了，需要把cancel方法放到下载的同一个线程，并且等待下载完成后cancel
        if (self.thread) {
            [self performSelector:@selector(cancelInternalAndStop) onThread:self.thread withObject:nil waitUntilDone:NO];
        }
        else {
            [self cancelInternal];
        }
    }
}

- (void)cancelInternal {
    ....
    //数据请求取消
    [self.dataTask cancel];
    ....
    //重置信息
    [self reset];
    ....
}
```

重点实现关注：判断self.thread，如果存在self.thread说明operation已经start，那么cancel已经无效，必须等待请求完成后，才能去cancel它，防止出现问题。这边用self.thread就可以达成这样的目的。


#### d. NSURLSession相关回调处理

我们主要关注的回调函数


```
- (void)URLSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask didReceiveData:(NSData *)data {
  ...
  self.completedBlock(image, nil, nil, NO);
  ...
}
```

从上面代码中我们可以知道，在下载过程当中，就对已经接受数据的信息并且转换成了模糊化的图片，并且回调了```self.completedBlock(image, nil, nil, NO);```，说明可以做到图片逐渐清晰的这种效果。

## 五、工具类源码解析

Utils主要包含以下3个类：

- SDWebImageDecoder：图片解码类
- SDWebImageManager： 核心的下载控制类
- SDWebImagePrefetcher：图片预下载类

### SDWebImageDecoder

SDWebImageDecoder实际上是UIIMage的一个分类

```
@interface UIImage (ForceDecode)
//解码图片
+ (UIImage *)decodedImageWithImage:(UIImage *)image;
@end
```

内部实现:

```
+ (UIImage *)decodedImageWithImage:(UIImage *)image {
    // while downloading huge amount of images
    // autorelease the bitmap context
    // and all vars to help system to free memory
    // when there are memory warning.
    // on iOS7, do not forget to call
    // [[SDImageCache sharedImageCache] clearMemory];
    // 当下载大量图片的时候，自动释放bitmap context和所有变量去帮助节约内存
    // 在ios7上别忘记调用 [[SDImageCache sharedImageCache] clearMemory];
    if (image == nil) { // Prevent "CGBitmapContextCreateImage: invalid context 0x0" error
        return nil;
    }
    
    @autoreleasepool{
        // do not decode animated images
        // 不去解码gif图片
        if (image.images != nil) {
            return image;
        }
        
        CGImageRef imageRef = image.CGImage;
        
        CGImageAlphaInfo alpha = CGImageGetAlphaInfo(imageRef);
        //获取任何的alpha轨道
        BOOL anyAlpha = (alpha == kCGImageAlphaFirst ||
                         alpha == kCGImageAlphaLast ||
                         alpha == kCGImageAlphaPremultipliedFirst ||
                         alpha == kCGImageAlphaPremultipliedLast);
        //有alpha信息的图片，直接返回
        if (anyAlpha) {
            return image;
        }
        
        // current
        //表示需要使用的色彩标准（为创建CGColor做准备）
        //例如RBG：CGColorSpaceCreateDeviceRGB
        CGColorSpaceModel imageColorSpaceModel = CGColorSpaceGetModel(CGImageGetColorSpace(imageRef));
        CGColorSpaceRef colorspaceRef = CGImageGetColorSpace(imageRef);
        
        //是否不支持这些标准
        BOOL unsupportedColorSpace = (imageColorSpaceModel == kCGColorSpaceModelUnknown ||
                                      imageColorSpaceModel == kCGColorSpaceModelMonochrome ||
                                      imageColorSpaceModel == kCGColorSpaceModelCMYK ||
                                      imageColorSpaceModel == kCGColorSpaceModelIndexed);
        //如果不支持说明是RGB的标准
        if (unsupportedColorSpace) {
            colorspaceRef = CGColorSpaceCreateDeviceRGB();
        }
        
        //获取图片高和宽
        size_t width = CGImageGetWidth(imageRef);
        size_t height = CGImageGetHeight(imageRef);
        //每一个pixel是4个byte
        NSUInteger bytesPerPixel = 4;
        //每一行的byte大小
        NSUInteger bytesPerRow = bytesPerPixel * width;
        //1Byte=8bit
        NSUInteger bitsPerComponent = 8;


        // kCGImageAlphaNone is not supported in CGBitmapContextCreate.
        // Since the original image here has no alpha info, use kCGImageAlphaNoneSkipLast
        // to create bitmap graphics contexts without alpha info.
        // kCGImageAlphaNone无法使用CGBitmapContextCreate.
        // 既然原始图片这边没有alpha信息，就使用kCGImageAlphaNoneSkipLast
        // 去创造没有alpha信息的bitmap的图片讯息
        CGContextRef context = CGBitmapContextCreate(NULL,
                                                     width,
                                                     height,
                                                     bitsPerComponent,
                                                     bytesPerRow,
                                                     colorspaceRef,
                                                     kCGBitmapByteOrderDefault|kCGImageAlphaNoneSkipLast);
        
        // Draw the image into the context and retrieve the new bitmap image without alpha
        // 将图片描绘进 图片上下文 去 取回新的没有alpha的bitmap图片
        CGContextDrawImage(context, CGRectMake(0, 0, width, height), imageRef);
        CGImageRef imageRefWithoutAlpha = CGBitmapContextCreateImage(context);
        //适应屏幕和方向
        UIImage *imageWithoutAlpha = [UIImage imageWithCGImage:imageRefWithoutAlpha
                                                         scale:image.scale
                                                   orientation:image.imageOrientation];
        
        if (unsupportedColorSpace) {
            CGColorSpaceRelease(colorspaceRef);
        }
        
        CGContextRelease(context);
        CGImageRelease(imageRefWithoutAlpha);
        
        return imageWithoutAlpha;
    }
}
```

这边解码图片主要原因就是图片的加载是lazy加载，在真正显示的时候才进行加载，这边先解码一次就会直接把图片加载完成。

看内部调用可以发现，都是在回调前才调用解析方法，外部拿到的图片都是被解析过了，可能和内部存储并不一样。

### SDWebImageManager

SDWebImageManager是核心的下载控制类方法，它内部封装了SDImageCache与SDWebImageDownloader
主要包含以下几块：

- 属性及初始化函数
- 存储与判断的一些封装方法
- 下载图片核心方法

这里只对SDWebImageManager对外使用做介绍，内部实现感兴趣的可以自行查看源码


#### a.属性及初始化函数


```
@interface SDWebImageManager : NSObject
//代理
@property (weak, nonatomic) id <SDWebImageManagerDelegate> delegate;

//缓存，这边只读，在.m的extension中，可以重写为readwrite
@property (strong, nonatomic, readonly) SDImageCache *imageCache;

//下载器
@property (strong, nonatomic, readonly) SDWebImageDownloader *imageDownloader;

//缓存路径过滤
@property (nonatomic, copy) SDWebImageCacheKeyFilterBlock cacheKeyFilter;

//单例
+ (SDWebImageManager *)sharedManager;

//初始化函数
- (instancetype)initWithCache:(SDImageCache *)cache downloader:(SDWebImageDownloader *)downloader;
```

这里重点关注cacheKeyFilter，他是缓存路径过滤器。应用场景：有些图片后台返回的URL每次都不一样（如后面的参数不同），可以根据需求将该图片的存储路径进行过滤一下，可以不用每次都去下载

#### b.存储与判断的一些封装方法

```
/**
 通过url保存图片

 @param image 需要缓存的图片
 @param url   图片url
 */
- (void)saveImageToCache:(UIImage *)image forURL:(NSURL *)url;

/**
 取消所有操作
 */
- (void)cancelAll;

//判断是否有操作还在运行
- (BOOL)isRunning;

/**
 判断是否有图片已经被缓存

 @param url 图片url

 @return 是否图片被缓存
 */
- (BOOL)cachedImageExistsForURL:(NSURL *)url;

/**
 判断是否图片只在在硬盘缓存

 @param url 图片url

 @return 是否图片只在硬盘缓存
 */
- (BOOL)diskImageExistsForURL:(NSURL *)url;

/**
 异步判断是否图片已经被缓存

 @param url             图片url
 @param completionBlock 完成block
 
 @note  完成block一直在主线程调用
 */
- (void)cachedImageExistsForURL:(NSURL *)url
                     completion:(SDWebImageCheckCacheCompletionBlock)completionBlock;

/**
 异步判断是否只在硬盘缓存

 @param url             图片url
 @param completionBlock 完成block
 */
- (void)diskImageExistsForURL:(NSURL *)url
                   completion:(SDWebImageCheckCacheCompletionBlock)completionBlock;

//返回一个url的缓存key
- (NSString *)cacheKeyForURL:(NSURL *)url;
```

#### c.下载图片核心方法

```
/**
 通过url下载图片如果缓存不存在

 @param url            图片url
 @param options        下载设置
 @param progressBlock  进度block
 @param completedBlock 完成block

 @return 返回SDWebImageDownloaderOperation的实例
 */
- (id <SDWebImageOperation>)downloadImageWithURL:(NSURL *)url
                                         options:(SDWebImageOptions)options
                                        progress:(SDWebImageDownloaderProgressBlock)progressBlock
                                       completed:(SDWebImageCompletionWithFinishedBlock)completedBlock;
```




### SDWebImagePrefetcher

```
//管理器
@property (strong, nonatomic, readonly) SDWebImageManager *manager;

//预加载并发数量，默认是3
@property (nonatomic, assign) NSUInteger maxConcurrentDownloads;


 //预加载图片选项，默认SDWebImageLowPriority
@property (nonatomic, assign) SDWebImageOptions options;

//预加载默认队列，默认主队列
@property (nonatomic, assign) dispatch_queue_t prefetcherQueue;

//代理
@property (weak, nonatomic) id <SDWebImagePrefetcherDelegate> delegate;

//单例
+ (SDWebImagePrefetcher *)sharedImagePrefetcher;

//初始化
- (id)initWithImageManager:(SDWebImageManager *)manager;

//预加载图片
- (void)prefetchURLs:(NSArray *)urls;

//预加载图片
- (void)prefetchURLs:(NSArray *)urls progress:(SDWebImagePrefetcherProgressBlock)progressBlock completed:(SDWebImagePrefetcherCompletionBlock)completionBlock;

//取消预加载操作
- (void)cancelPrefetching;

```

## 五、核心分类源码解析

SDWebImage中有很多分类供外部使用，这里主要介绍两个核心分类的使用及实现：```UIView+WebCacheOperation```与```UIImageView+WebCache```

### ```UIView+WebCacheOperation```

```
/**
 设置图片的load操作(存入一个uiview的字典)

 @param operation 操作
 @param key       存入的key
 */
- (void)sd_setImageLoadOperation:(id)operation forKey:(NSString *)key;

/**
 取消当前uiview的key的操作

 @param key 存入的key
 */
- (void)sd_cancelImageLoadOperationWithKey:(NSString *)key;

/**
 移除当前uiview的key的操作，并且不取消他们

 @param key 存入的key
 */
- (void)sd_removeImageLoadOperationWithKey:(NSString *)key;
```

内部实现:(关联对象)

```
static char loadOperationKey;

- (NSMutableDictionary *)operationDictionary {
    //通过associateobject去存入和获取dictionaryu
    NSMutableDictionary *operations = objc_getAssociatedObject(self, &loadOperationKey);
    if (operations) {
        return operations;
    }
    operations = [NSMutableDictionary dictionary];
    objc_setAssociatedObject(self, &loadOperationKey, operations, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    return operations;
}

- (void)sd_setImageLoadOperation:(id)operation forKey:(NSString *)key {
    //先取消操作
    [self sd_cancelImageLoadOperationWithKey:key];
    //然后获取dictionary并存入
    NSMutableDictionary *operationDictionary = [self operationDictionary];
    [operationDictionary setObject:operation forKey:key];
}

- (void)sd_cancelImageLoadOperationWithKey:(NSString *)key {
    // Cancel in progress downloader from queue
    //获取dictionary，并且取消在queue中的操作
    NSMutableDictionary *operationDictionary = [self operationDictionary];
    id operations = [operationDictionary objectForKey:key];
    if (operations) {
        //数组的话获取内部所有的取消
        if ([operations isKindOfClass:[NSArray class]]) {
            for (id <SDWebImageOperation> operation in operations) {
                if (operation) {
                    [operation cancel];
                }
            }
            //如果是实现SDWebImageOperation契约，直接取消
        } else if ([operations conformsToProtocol:@protocol(SDWebImageOperation)]){
            [(id<SDWebImageOperation>) operations cancel];
        }
        [operationDictionary removeObjectForKey:key];
    }
}

- (void)sd_removeImageLoadOperationWithKey:(NSString *)key {
    //获取dictionary并直接删除key
    NSMutableDictionary *operationDictionary = [self operationDictionary];
    [operationDictionary removeObjectForKey:key];
}
```


### ```UIImageView+WebCache```


```


- (NSURL *)sd_imageURL;

- (void)sd_setImageWithURL:(NSURL *)url;


- (void)sd_setImageWithURL:(NSURL *)url placeholderImage:(UIImage *)placeholder;


- (void)sd_setImageWithURL:(NSURL *)url placeholderImage:(UIImage *)placeholder options:(SDWebImageOptions)options;


- (void)sd_setImageWithURL:(NSURL *)url completed:(SDWebImageCompletionBlock)completedBlock;


- (void)sd_setImageWithURL:(NSURL *)url placeholderImage:(UIImage *)placeholder completed:(SDWebImageCompletionBlock)completedBlock;


- (void)sd_setImageWithURL:(NSURL *)url placeholderImage:(UIImage *)placeholder options:(SDWebImageOptions)options completed:(SDWebImageCompletionBlock)completedBlock;


- (void)sd_setImageWithURL:(NSURL *)url placeholderImage:(UIImage *)placeholder options:(SDWebImageOptions)options progress:(SDWebImageDownloaderProgressBlock)progressBlock completed:(SDWebImageCompletionBlock)completedBlock;


- (void)sd_setImageWithPreviousCachedImageWithURL:(NSURL *)url placeholderImage:(UIImage *)placeholder options:(SDWebImageOptions)options progress:(SDWebImageDownloaderProgressBlock)progressBlock completed:(SDWebImageCompletionBlock)completedBlock;

- (void)sd_setAnimationImagesWithURLs:(NSArray *)arrayOfURLs;

- (void)sd_cancelCurrentImageLoad;

- (void)sd_cancelCurrentAnimationImagesLoad;

- (void)setShowActivityIndicatorView:(BOOL)show;

- (void)setIndicatorStyle:(UIActivityIndicatorViewStyle)style;
```

重点关注方法：sd_setImageWithPreviousCachedImageWithURL

这个方法会判断对这个url是否有缓存，如果有缓存则用缓存图片作为placeholder图片
