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

    <div id="d" style="width: 200px;height: 200px;background-color: red;">

    </div>
</body>
</html>

```

发布这个网页，这里推荐使用使用VSCode中live server插件运行发出这个网页，获取这个网页的url

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

![](https://p.ipic.vip/d6p7e5.png)

![](https://p.ipic.vip/24w8j8.png)

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
    <div id="d" style="width: 200px;height: 200px;background-color: red;">

    </div>

    <script type="text/javascript">
        var oDiv = document.getElementById("d");
        oDiv.addEventListener("click", function(){
            WebJSBridge.call("哈哈哈") 
        });
    </script>

</body>
</html>

```

**运行结果：**

- **点击红色区域**

![](https://p.ipic.vip/d6p7e5.png)

![](https://p.ipic.vip/lm0f1n.png)

## 三、高阶用法

#### 1. Native 打印 前端页面的console.log

- **给webView注册log事件**

```
webView.configuration.userContentController.add(self, name: "log")
```

- **重写console.log方法，并添加到前端页面**

```
        let consoleJsString = """
        console.log = (function(oriLogFunc){\

            return function(str)\

            {\

            window.webkit.messageHandlers.log.postMessage(str);\

            oriLogFunc.call(console,str);\

            }\

            })(console.log);

        """
        let consoleJs = WKUserScript(source: consoleJsString, injectionTime: WKUserScriptInjectionTime.atDocumentStart, forMainFrameOnly: true)
        webView.configuration.userContentController.addUserScript(consoleJs)
```

- **再重写的console.log 事件中，执行了：**

```
window.webkit.messageHandlers.log.postMessage(str);
```

- **在Native方法中，打印console.log 的meaasge：**

```
    func userContentController(_ userContentController: WKUserContentController, didReceive message: WKScriptMessage) {
        

        print(message.name)
        if let body:String =  message.body as? String{
            print(body)
            
            if let dic  = getDictionaryFromJSONString(jsonString: body){
                let callBackId:String = dic["callbackId"] as! String
                let callBackJsString = "window.bridgeCallback(\"\(callBackId)\", \"success\")"
                webView.evaluateJavaScript(callBackJsString)
            }
            
        }
        
    }
```


#### 2. JS 调Native  callback回调

**实现原理概述：**

JS 内部维护 {callBackId:func}，Native 只想JS 提供的入口方法，传入callBackId和入参，JS内部通过callBackId 获取到callBack函数并执行

- **JS 相关代码：**

  定义callbackMap，存储所有的callback回调，当Native执行回调时，通过callbackId拿到回调并执行
  
```
let callbackId = 0;
let callbackMap = {};

class WebJSBridge{
    static call(name, params, callback) {
        //生成callbackId
        let cbId = this.genCallbackId();
        //添加到callbackMap
        this.add(cbId, callback);

        //组装方法和参数
        let config = {name: name, params: params, callbackId: cbId};
        let string = JSON.stringify(config);
        window.webkit.messageHandlers.bridgeTest.postMessage(string); 
    }
    
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
    
};

//注入全局方法，用于Native向h5回调
window.bridgeCallback = function(callbackId, res) {
    let cb = callbackMap[callbackId];
    let callback = cb["callback"];
    if (callback) {
        callback(res);
    }
    delete callbackMap.callbackId
}
  
```

- **HTML 相关代码：**

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
    <div id="d" style="width: 200px;height: 200px;background-color: red;">

    </div>

    <script type="text/javascript">
        var oDiv = document.getElementById("d");
        oDiv.addEventListener("click", function () {
            WebJSBridge.call("jsbridgeMethod", "param", function (res) {
                console.log("recieve callback")
                console.log(res)
            });
        });
    </script>

</body>

</html>
```

- **Native 相关代码：**

```
    func userContentController(_ userContentController: WKUserContentController, didReceive message: WKScriptMessage) {
        if message.name == "log" {
            print("console.log: \(message.body)")
        }

        if let body:String =  message.body as? String{
            if let dic  = getDictionaryFromJSONString(jsonString: body){
                if let callBackId:String = dic["callbackId"] as? String {
                    jsCallBack(callBackId, param:"success")
                }
            }
        }
    }
    
    func jsCallBack(_ callBackId:String,param:String){
        let callBackJsString = "window.bridgeCallback(\"\(callBackId)\", \"\(param)\")"
        webView.evaluateJavaScript(callBackJsString)
    }
    
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

**运行结果：**

- **点击红色区域**

![](https://p.ipic.vip/d6p7e5.png)

![](https://p.ipic.vip/eb98w0.png)