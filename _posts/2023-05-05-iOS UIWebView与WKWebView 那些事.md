---
layout:     post
title:      iOS UIWebView与WKWebView 那些事
subtitle:   iOS UIWebView与WKWebView 那些事
date:       2023-05-05
author:     LXY
header-img: img/home1.jpg
catalog: true
tags:
    - iOS
---


## 一、前言介绍

**UIWebView** 是 **iOS 2** 中推出的网页容器，UIWebView是最占内存的控件；直到 **iOS 8** 以后，苹果推出了 **WebKit** 框架，其中 **WKWebView** 正式被推出来接替 **UIWebView** 的位置；**iOS 12** 中，苹果正式弃用 **UIWebView**，要求开发者用 WKWebView 全面替换 UIWebView，[apple 官方文档](https://developer.apple.com/documentation/Webkit/replacing_uiWebView_in_your_App)

## 二、特点对比

**UIWebView 特点：**
- 1、加载速度慢
- 2、内存占用大，App停留在后台很容易被系统杀死
- 3、原生容器完全不带进度条，需要自定义开发

**WKWebView 特点：**
- 1、在性能、稳定性、功能方面有很大提升（最直观的提升就是加载网页是占用的内存很少,测试加载不同网页其内存性能提升3倍到4倍,而且没有缓存）
- 2、允许JavaScript的Nitro库加载并使用（UIWebView中限制）
- 3、支持更多的 HTML5 特性
- 4、与 Safari 具有相同的 JavaScript 引擎

## 三、能力提供

#### a、加载能力

- UIWebView不仅可以加载 HTML 页面，还支持 pdf、word、txt 以及各种图片的显示； 
- 相比 UIWebView 而言，WKWebView 也支持各种文件格式，并新增了加载本地文件，即新增了 LoadFileURL 函数。

**1.1 UIWebView加载网页请求**
```
- (void)loadRequest:(NSURLRequest *)request;
```
**1.2 WKWebView加载网页请求**
```
- (nullable WKNavigation *)loadRequest:(NSURLRequest *)request;
```
**2.1 UIWebView加载HTML**
```
- (void)loadHTMLString:(NSString *)string baseURL:(nullable NSURL *)baseURL;
```
**2.2 WKWebView加载HTML**
```
- (nullable WKNavigation *)loadHTMLString:(NSString *)string baseURL:(nullable NSURL *)baseURL;
```

**3.1 UIWebView加载文件，并指定 MIME 类型和编码类型**

```
- (void)loadData:(NSData *)data MIMEType:(NSString *)MIMEType textEncodingName:(NSString *)textEncodingName baseURL:(NSURL *)baseURL;
```

**3.2 WKWebView加载文件，并指定 MIME 类型和编码类型**

```
- (nullable WKNavigation *)loadData:(NSData *)data MIMEType:(NSString *)MIMEType characterEncodingName:(NSString *)characterEncodingName baseURL:(NSURL *)baseURL;
```
**4 WKWebView加载本地文件，UIWebView做不到**
```
- (nullable WKNavigation *)loadFileURL:(NSURL *)URL allowingReadAccessToURL:(NSURL *)readAccessURL
```
#### b、导航刷新相关

| UIWebView 网页导航相关 | WKWebView 网页导航相关 | 说明 | 
|-|-|-|
| canGoBack | canGoBack | 是否可以后退 |
| canGoForward | canGoForward | 是否可以前进 |
| isLoading | isLoading | 是否正在加载 |

**1.1 UIWebView 相关方法**
```
//刷新
- (void)reload;
//停止加载
- (void)stopLoading;
//后退
- (void)goBack;
//前进
- (void)goForward;
```

**1.2 WKWebView 相关方法**
```
//刷新
- (nullable WKNavigation *)reload;
//停止加载
- (void)stopLoading;
//后退
- (nullable WKNavigation *)goBack;
//前进
- (nullable WKNavigation *)goForward;

```

***注：区别于UIWebVie， WKWebView 的方法是有返回值的（ stopLoading 除外），返回值类型为 WKNavigation ，主要用于跟踪网页加载进度。***

**1.3 WKWebView 独有方法**

会比较网络数据变化，如果没有变化，则使用缓存，否则重新请求
```
- (nullable WKNavigation *)reloadFromOrigin;
```

跳转到某个指定的历史界面
```
- (nullable WKNavigation *)goToBackForwardListItem:(WKBackForwardListItem *)item;
```

#### c、代理协议


- UIWebView 的代理协议主要是UIWebViewDel- 第一行
- WKWebView 的代理协议主要有 3 个，分别是 WKNavigationDelegate、WKUIDelegate 和 WKScriptMessageHandler 。

**UIWebViewDelegate & WKNavigationDelegate**

其中 UIWebViewDelegate 和 WKNavigationDelegate 的等效项如下所示

**UIWebViewDelegate**
```
//开始加载网页
- (void)webViewDidStartLoad:(UIWebView *)webView ;
//网页加载完成
- (void)webViewDidFinishLoad:(UIWebView *)webView;
//网页加载错误
- (void)webView:(UIWebView *)webView didFailLoadWithError:(NSError *)error;
//是否允许加载网页，或者获取JS即将打开的URL，通过截取此URL可与JS交互
- (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType;
```


**WKNavigationDelegate**
```
//开始加载网页
-  (void)webView:(WKWebView *)webView didStartProvisionalNavigation:(null_unspecified WKNavigation *)navigation;
//网页加载完成
- (void)webView:(WKWebView *)webView didFinishNavigation:(null_unspecified WKNavigation *)navigation;
//网页加载错误
- (void)webView:(WKWebView *)webView didFailNavigation:(null_unspecified WKNavigation *)navigation withError:(NSError *)error;
- (void)webView:(WKWebView *)webView didFailProvisionalNavigation:(null_unspecified WKNavigation *)navigation withError:(NSError *)error;
//是否允许加载网页，或者获取JS即将打开的URL，通过截取此URL可与JS交互
- (void)webView:(WKWebView *)webView decidePolicyForNavigationAction:(WKNavigationAction *)navigationAction decisionHandler:(void (^)(WKNavigationActionPolicy))decisionHandler;
- (void)webView:(WKWebView *)webView decidePolicyForNavigationResponse:(WKNavigationResponse *)navigationResponse decisionHandler:(void (^)(WKNavigationResponsePolicy))decisionHandler;
```
WKNavigationDelegate 拦截加载的代理方法 并不像 UIWebViewDelegate中等效的函数返回 BOOL，而是通过block中 decisionHandler 决定是否可以跳转，返回 allow 或者 cancel 。
 
**WKWebView 独有代理**
**WKScriptMessageHandler 用于App 与 JS 的交互，提供从网页中收消息的回调方法**
```
- (void)userContentController:(WKUserContentController *)userContentController didReceiveScriptMessage:(WKScriptMessage *)message;
```
WKScriptMessageHandler 是必须实现的函数，是用于App 与 JS 的交互，提供从网页中收消息的回调方法，响应从网页的 JavaScript 代码发送的消息。使用 message 参数获取消息内容并确定原始 Web 视图。

**WKUIDelegate 是 UI 界面相关的代理协议，主要用于处理三种提示框：输入、确认、警告。因为在 UIWebView 中，Alert、Confirm、Prompt 等视图是可以直接执行的，但在 WKWebView 上，需要通过这个协议接收通知，然后通过 iOS 原生执行，即需要将 Web 提示框拦截然后再通过原生做处理。**
```
//创建一个新的WebView
- (nullable WKWebView *)webView:(WKWebView *)webView createWebViewWithConfiguration:(WKWebViewConfiguration *)configuration forNavigationAction:(WKNavigationAction *)navigationAction windowFeatures:(WKWindowFeatures *)windowFeatures;
```
//经常用于在项目中处理 H5 界面中含有 target = __blank 标签（表示新建一个页面打开网页）或者网页中点击无响应的情况。
```
//调用 JS 的 alert 方法
- (void)webView:(WKWebView *)webView runJavaScriptAlertPanelWithMessage:(NSString *)message initiatedByFrame:(WKFrameInfo *)frame completionHandler:(void (^)(void))completionHandler;
//调用JS的confirm方法
- (void)webView:(WKWebView *)webView runJavaScriptConfirmPanelWithMessage:(NSString *)message initiatedByFrame:(WKFrameInfo *)frame completionHandler:(void (^)(BOOL result))completionHandler;
//调用 JS 的 prompt 方法
- (void)webView:(WKWebView *)webView runJavaScriptTextInputPanelWithPrompt:(NSString *)prompt defaultText:(nullable NSString *)defaultText initiatedByFrame:(WKFrameInfo *)frame completionHandler:(void (^)(NSString * _Nullable result))completionHandler;
//通知 App，DOM 窗口已成功关闭
- (void)webViewDidClose:(WKWebView *)webView;
```
## 四、WKWebView 历史栈缓存策略
#### a、 WebKit 基础概念

WKWebView 运行时有三种进程协同工作：UIProcess 进程、WebContent 进程、Networking 进程。

**1、WebContent 进程**
网页 DOM 及 JS 所处进程。进程数量可能有多个，取决于一些细节策略。
在该进程初始化时会创建唯一的 WebProcess 实例，并且作为 IPC::Connection 的 client，与其它进程通信的代理。

**2、UIProcess 进程**
应用程序对应的进程。
初始化后，WebPageProxy 做为了 IPC::Connection 的 client，与其它进程通信的代理。
WebPageProxy / WebProcessProxy 分别对应了 WebContent 进程的 WebPage / WebProcess。
WebProcessPool（关联 WKWebViewConfiguration 的 WKProcessPool 对象）抽象了 WebContent 进程池，也就是说一个 WKWebView 是可以对应多个 WebContent 进程。

**3、Networking 进程**
负责网络相关处理，创建多个 WKWebView 也仅只有一个进程。

#### b、 历史栈缓存策略

**简述:**
WKWebView 可以通过goBack/goForward接口进行历史栈的切换，切换时有一套缓存策略，命中时能省去请求网络的时间。
WebContent 进程的 BackForwardCache 是一个单例，管理着历史栈缓存。
UIProcess 进程的 WebProcessPool 抽象了 WebContent 进程池，每一个 WebProcessPool 都有唯一的 WebBackForwardCache 表示历史栈缓存，对应着 WebContent 进程池子里的各个 BackForwardCache 单例。
BackForwardCache 用了一个有序 hash 表存储缓存元素，并设定了最大缓存数量：
```
ListHashSet<RefPtr<HistoryItem>> m_items;
unsigned m_maxSize {0};
```

**1、缓存淘汰策略**

BackForwardCache 和 WebBackForwardCache 的策略基本一致，现以 BackForwardCache 为例说明。
WebContent 进程 在切换页面时，会将当前页面通过:
```
BackForwardCache::singleton().addIfCacheable(...);
```
添加缓存：
```
bool BackForwardCache::addIfCacheable(HistoryItem& item, Page* page) {
 ...
item.setCachedPage(makeUnique<CachedPage>(*page));
item.m_pruningReason = PruningReason::None;
 m_items.add(&item);
 ...
}
```

最大缓存数量源码：
```
namespace WebKit {
voidcalculateMemoryCacheSizes(...){
uint64_t memorySize = ramSize() / MB; 
 ...
// back/forward cache capacity (in pages)
if (memorySize >= 512)
 backForwardCacheCapacity = 2;
elseif (memorySize >= 256)
 backForwardCacheCapacity = 1;
else
 backForwardCacheCapacity = 0;
 ...
 }
...
```
可以看出是实现了一个简单的 LRU 淘汰策略。

**2、最大缓存数量**

前面说到 WebContent 进程最多两个历史栈缓存，实际上这个缓存数量是 UIProcess 进程决定的。在 UIProcess 进程中，WebProcessPool 初始化 WebBackForwardCache 时会设置最大缓存数量，并且在创建 WebProcessProxy 时通过 IPC 通知到对应的 WebContent 进程去设置 BackForwardCache 的m_maxSize。 

WebProcessPool 的 WebBackForwardCache 对应了 WebContent 进程池里每一个的 BackForwardCache 单例，是一个一对多的模式，WebBackForwardCache 在修剪缓存元素析构时会自动触发 IPC 通知到 WebContent 进程去清理对应缓存：
```
WebBackForwardCacheEntry::~WebBackForwardCacheEntry() {
if (m_backForwardItemID && !m_suspendedPage) {
auto& process = this->process();
 process.sendWithAsyncReply(Messages::WebProcess::ClearCachedPage(m_backForwardItemID), [] { });
 }
}
```
所以缓存最大数量取决于 WebProcessPool 的数量，一个 WebProcessPool 就最多两个历史栈缓存，不管它的进程池有多少个 WebContent。

**3、状态同步**

在历史栈缓存状态发生变化时，WebContent 进程会调用notifyChanged()通过 IPC 通知到 UIProcess 进程的对应 WebBackForwardCache 去同步状态：
```
notifyChanged() 最终调用到：
static void WK2NotifyHistoryItemChanged(HistoryItem& item) {
 WebProcess::singleton().parentProcessConnection()->send(Messages::WebProcessProxy::UpdateBackForwardItem(toBackForwardListItemState(item)), 0);
}
```

## 五、WKWebView 中常见问题及解决方案

**a、POST 类型请求 Request Body 丢失**

**原因分析：**
当使用网络拦截后，WebKit的IPC进程的请求会转到主进程执行，由于进程切换回导致性能下降，所以WebKit会主动丢弃request的body。

**解决方案：**
1、在 NSURLProtocol 子类中的 - startLoading 通过获取 request.HTTPBodyStream 来填充 request.HTTPBody 实现。但是此方法在有些时候(如Ajax)会失败；

2、通过 hook js XMLHTTPRequest 相关方法实现。具体的来说，注入 XHR requestType 为 post 的 send 方法，把 requstBody 和对应的 URL 通过桥的方式提前传给客户端，客户端接收端数据后，保存起来，在 NSURLProtocol 子类中的 -initWithRequest:cachedResponse:client: 方法拦截中填充保存在客户端的 requstBody 到 request 中，即可保证 requestBody 不会丢失；


**b、白屏问题**

**原因分析：**
1、内存占用过大导致 WebContent process 进程崩溃；
2、内存占用过大导致 WebContent process 被挂起。

**解决方案：**
1、当 WKWebView 占用内存过大的时候，会导致 WebContent process crash，会回调 -webViewWebContentProcessDidTerminate:，可在此方法中添加 [webView reload]，重新载入页面解决；

2、当 WKWebView 占用内存过大的时候(多见于选择相册) ，会导致 WebContent process 被挂起，此情况不会调用 -webViewWebContentProcessDidTerminate:，可通过在 WebViewController 的 -viewWillAppear: 方法中检测 webView.title 是否存在，如果不存在可认为进程被挂起，可在此方法中添加 [webView reload]，重新载入页面解决；

**c、显示HTML页面不是最新的内容**

**原因分析：**
WKWebView有缓存；

**解决方案：**
为了保证每次加载的都是最新的页面, 可以在加载的链接后面加上一个时间戳；

**d、当前页面无导航时不能填充状态栏（iOS11+ 会下移状态栏的高度）**

**原因分析：**
在页面无导航的情况下，系统会自动调节滚动视图的contentInset，使其视图永远处于状态栏之下;

**解决方案：**
```
if (@available(iOS 11.0, *)) {
      self.webView.scrollView.contentInsetAdjustmentBehavior = UIScrollViewContentInsetAdjustmentNever;
} else {
      // Fallback on earlier versions
      self.automaticallyAdjustsScrollViewInsets = NO;
 }
```
