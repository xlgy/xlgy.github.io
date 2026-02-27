---
layout:     post
title:      iOS Hybrid 原理介绍
subtitle:   iOS Hybrid 原理介绍
date:       2023-08-11
author:     LXY
header-img: img/home1.jpeg
catalog: true
tags:
    - iOS
---


***目前绝大部分App 已经采取Hybrid模式开发，即一部分页面由Native编写，一部分页面由前端技术编写。随着页面数量增多，前端和Native页面之间的通信是必须解决的问题。前端页面需要通过js代码调用到Native的功能，获取Native页面的数据等，Native页面也需要执行js代码获取前端页面的信息。***


## 一、Native Call  JS

#### 1. API 介绍

WKWebView 提供API可以直接执行JS 方法的调用

```
open func evaluateJavaScript(_ javaScriptString: String) async throws -> Any
```


#### 2. 举个例子

**HTML代码：**

- **写一个简单的网页代码，并发布页面**

```
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>

<body>
    <script type="text/javascript" src="./WebJSBridge.js"></script>
    <div id="mydiv" style="width: 200px;height: 200px;background-color: red;">
    </div>
</body>
</html>

```

发布这个网页，这里推荐使用使用VSCode中live server插件运行发出这个网页，获取这个网页的url

![](https://raw.githubusercontent.com/xlgy/xlgy.github.io/refs/heads/master/_posts/image/hybrid1.png)


**iOS Native 代码：**

- **在ViewController里面先添加一个WKWebview，并且布局WKWebview使其占满整个页面**

```
    let webView:WKWebView = WKWebView()

    override func viewDidLoad() {
        super.viewDidLoad()
        
        webView.frame = self.view.bounds
        webView.navigationDelegate = self
        self.view.addSubview(webView)
        
        if let url = URL(string: "http://xxxxx"){
            let request = URLRequest(url: url)
            webView.load(request)
        }
    }
```

- **当网页加载完成后，执行js方法 window.location.href --> 获取页面的URL**

```
    func webView(_ webView: WKWebView, didFinish navigation: WKNavigation!) {
        
        webView.evaluateJavaScript("window.location.href") { res, err in
            print(res ?? "")
        }
    }
```



## 二、JS Call  Native

- **JS 调用Native时首先Native代码需要注册一个jsbridge通道，指定通道的名字，然后js调用这个jsbridge**

```
webView.configuration.userContentController.add(self, name: "bridgeTest")
```
- **这里注册了名为bridgeTest的jsbridg通道，这样webview加载的网页可以通过以下代码调用到Native:**

```
window.webkit.messageHandlers.bridgeTest.postMessage()
```

同时，Native代码需要实现WKScriptMessageHandler协议，拿到bridgeTest的传参

**JS 代码：**

- **写一个WebJSBridge 类，实现call方法**

```
class WebJSBridge{
    static call(params) {
        //执行native注册的bridgeTest方法，并传递参数
        window.webkit.messageHandlers.bridgeTest.postMessage(params); 
    }
}
```

**HTML代码：**

- **给红色的div添加点击事件，执行WebJSBridge的call方法**

```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>
<body>

    <script type="text/javascript" src="./WebJSBridge.js"></script>
    <div id="mydiv" style="width: 200px;height: 200px;background-color: red;">

    </div>

    <script type="text/javascript">
        var oDiv = document.getElementById("mydiv");
        oDiv.addEventListener("click", function(){
            WebJSBridge.call("哈哈哈") 
        });
    </script>

</body>
</html>

```

**native检测代码**

```
func userContentController(_ userContentController: WKUserContentController, didReceive message: WKScriptMessage) {
        print("message.name:" + message.name)
        if let body:String =  message.body as? String{
            print("message.body:" + body)
        }
    }
```



**运行结果：**

- **点击红色区域**

![](https://raw.githubusercontent.com/xlgy/xlgy.github.io/refs/heads/master/_posts/image/hybrid1.png)

- **日志：**

```
message.name:bridgeTest
message.body:哈哈哈
```

## 三、高阶场景


### 1.console.log工具插件


***目标：***
设计一个可以使用 Native 打印 前端页面的console.log的插件

***设计思路：***

- hook前端console.log方法
- 注册native方法
- 当前端调用 console.log 时重定向调用native方法


***代码实现：***

- **编写js方法，重写 console.log 方法**

```
// 1. 保留原有 console.log 功能
// 2. 同时把日志发送给 iOS（WKWebView）
// 3. 实现 JS → Native 的日志桥接

console.log = (function (oriLogFunc) {
    
    // 返回一个新的函数，用来替换原来的 console.log
    return function (str) {
        
        // ① 把日志发送给 iOS
        // 这里的 "nativelog" 必须和 iOS 端注册的 name 一致
        // userContentController.add(handler, name: "nativelog")
        window.webkit.messageHandlers.nativelog.postMessage(str);
        
        // ② 调用原始 console.log，保证浏览器控制台仍然输出
        // call(console, str) 的作用：
        //   - 保持 this 指向 console
        //   - 执行原始 log 方法
        oriLogFunc.call(console, str);
    };
    
})(console.log); // 把原始 console.log 作为参数传进去
```

- **将重写的console.log方法，添加到前端页面**

```
        let consoleJsString = """
        console.log = (function(oriLogFunc){\

            return function(str)\

            {\

            window.webkit.messageHandlers.nativelog.postMessage(str);\

            oriLogFunc.call(console,str);\

            }\

            })(console.log);

        """
        let consoleJs = WKUserScript(source: consoleJsString, injectionTime: WKUserScriptInjectionTime.atDocumentStart, forMainFrameOnly: true)
        webView.configuration.userContentController.addUserScript(consoleJs)
```


- **给webView注册postMessage消息事件**

```
webView.configuration.userContentController.add(self, name: "nativelog")
```




- **在Native方法中，打印console.log 的meaasge：**

```
    func userContentController(_ userContentController: WKUserContentController, didReceive message: WKScriptMessage) {
        if message.name == "nativelog",let body:String =  message.body as? String{
            print(body)
        }
    }
```


***插件完整代码：(直接可用)***

```
import UIKit
import WebKit

class WebViewUtil: NSObject, WKScriptMessageHandler {
    
    static let share = WebViewUtil()
    
    ///>使用native打印前端log
    func hookLog(_ webView:WKWebView){
        let consoleJsString = """
        console.log = (function(oriLogFunc){\

            return function(str)\

            {\

            window.webkit.messageHandlers.nativelog.postMessage(str);\

            oriLogFunc.call(console,str);\

            }\

            })(console.log);

        """
        let consoleJs = WKUserScript(source: consoleJsString, injectionTime: WKUserScriptInjectionTime.atDocumentStart, forMainFrameOnly: true)
        webView.configuration.userContentController.addUserScript(consoleJs)
        webView.configuration.userContentController.add(self, name: "nativelog")
        
    }

    func userContentController(_ userContentController: WKUserContentController, didReceive message: WKScriptMessage) {
        if message.name == "nativelog",let body:String =  message.body as? String{
            print(body)
        }
    }
}

```

使用：

```
WebViewUtil.share.hookLog(webView)
```


### 2. JS 调 Native 异步 callback回调

***实现原理概述：***

- JS 内部维护键值对 {callBackId:func}，key为callbackId，value是callBack函数方法
- Native 只向 JS 提供的入口方法，传入callBackId和入参
- JS内部通过callBackId 获取到callBack函数并执行

***代码实现：***

- **JS 方法设计：**

  
```
    static call(params, callback) {
        //生成callbackId
        let cbId = this.genCallbackId();
        //添加到callbackMap
        this.add(cbId, callback);

        //组装方法和参数
        let config = {name: name, params: params, callbackId: cbId};
        let string = JSON.stringify(config);
        
        //执行native注册的bridgeTest方法，并传递参数
        window.webkit.messageHandlers.bridgeTest.postMessage(string); 
    }  
```

- **JS 完整代码：**

```
let callbackId = 0;
let callbackMap = {};

//注入全局方法，用于Native向h5回调，名字叫bridgeCallback
window.bridgeCallback = function(callbackId, res) {
    let cb = callbackMap[callbackId];
    let callback = cb["callback"];
    if (callback) {
        callback(res);
    }
    delete callbackMap.callbackId
}


class WebJSBridge{
    
    //生成callbackId
    static genCallbackId() {
        return `Webview_callback_${callbackId++}`
    }

    //添加callback
    static add(callbackId, callback) {
        callbackMap[callbackId] = {
            callback: callback
        }
    }

    static call(params) {
        //执行native注册的bridgeTest方法，并传递参数
        window.webkit.messageHandlers.bridgeTest.postMessage(params); 
    }

    
    static call(params, callback) {
        //生成callbackId
        let cbId = this.genCallbackId();
        //添加到callbackMap
        this.add(cbId, callback);

        //组装方法和参数
        let config = {name: name, params: params, callbackId: cbId};
        let string = JSON.stringify(config);
        
        //执行native注册的bridgeTest方法，并传递参数
        window.webkit.messageHandlers.bridgeTest.postMessage(string); 
    }
}
```

- **方法调用代码：**

```
            WebJSBridge.call("param", function (res) {
                console.log("recieve callback")
                console.log(res)
            }); 
```

- **Native 注册方法：**

```
webView.configuration.userContentController.add(self, name: "bridgeTest")
```

- **Native 方法回调：**

```
    func userContentController(_ userContentController: WKUserContentController, didReceive message: WKScriptMessage) {
        if let body:String =  message.body as? String,
           let dic  = getDictionaryFromJSONString(jsonString: body),
           let callBackFuncName:String = dic["callbackFuncName"] as? String,
           let callBackId:String = dic["callbackId"] as? String{
               jsCallBack(callBackFuncName, callBackId: callBackId, param:"success")
           }
    }
    
    
    ///js回调
    func jsCallBack(_ callBackFuncName:String, callBackId:String,param:String){
        let callBackJsString = "window.\(callBackFuncName)(\"\(callBackId)\", \"\(param)\")"
        webView.evaluateJavaScript(callBackJsString)
    }
    
    ///字符串专字典
    func getDictionaryFromJSONString(jsonString: String?) -> [String: Any]? {
        guard let mJsonString = jsonString else {
            return nil
        }
        let jsonData: Data = mJsonString.data(using: .utf8)!
        guard let dict = try? JSONSerialization.jsonObject(with: jsonData, options: .mutableContainers) else {
            return nil
        }
        return dict as? [String: Any]
    }
```

**执行代码：**

```
            WebJSBridge.call("param", function (res) {
                console.log("recieve callback")
                console.log(res)
            }); 
```

**日志：**

```
recieve callback
success
```

