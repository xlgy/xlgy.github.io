---
layout:     post
title:      制作Cocoapods（私有库、开源库)
subtitle:   制作Cocoapods（私有库、开源库)
date:       2025-12-19
author:     LXY
header-img: img/home1.jpeg
catalog: true
tags:
    - iOS
---


# 前言
最近想根据最近的技术总结开发几个常用的基建库，作为日后的技术总结。本文重点介绍如何依赖Cocoapods制作自己的私有库和开源库。

# 一、创建 Pod 库模板

## 1. 相关命令
```
pod lib create LXYDemoSDK
```

## 2. 流程

- 选择语言：Objective-C 或 Swift

- 选择是否集成 Demo 项目

- CocoaPods 会生成目录结构：

```
LXYDemoSDK/
├── Example/          # Demo 工程
├── LXYDemoSDK/        # 库源码
├── LXYDemoSDK.podspec # Podspec 文件

```

## 3.记录：

```
lixueyang@lixueyangdeMacBook-Pro project % pod lib create LXYDemoSDK
Cloning `https://github.com/CocoaPods/pod-template.git` into `LXYDemoSDK`.
Configuring LXYDemoSDK template.
security: SecKeychainSearchCopyNext: The specified item could not be found in the keychain.

------------------------------

To get you started we need to ask a few questions, this should only take a minute.

If this is your first time we recommend running through with the guide:
 - https://guides.cocoapods.org/making/using-pod-lib-create.html
 ( hold cmd and click links to open in a browser. )


What platform do you want to use?? [ iOS / macOS ]
 >
ios
What language do you want to use?? [ Swift / ObjC ]
 > Objc

Would you like to include a demo application with your library? [ Yes / No ]
 >
yes
Which testing frameworks will you use? [ Specta / Kiwi / None ]
 >
specta
Would you like to do view based testing? [ Yes / No ]
 >
yes
What is your class prefix?
 > LXY
security: SecKeychainSearchCopyNext: The specified item could not be found in the keychain.
security: SecKeychainSearchCopyNext: The specified item could not be found in the keychain.
security: SecKeychainSearchCopyNext: The specified item could not be found in the keychain.
security: SecKeychainSearchCopyNext: The specified item could not be found in the keychain.
security: SecKeychainSearchCopyNext: The specified item could not be found in the keychain.
security: SecKeychainSearchCopyNext: The specified item could not be found in the keychain.

Running pod install on your new library.

Analyzing dependencies
Downloading dependencies
Installing Expecta (1.0.6)
Installing Expecta+Snapshots (3.1.1)
Installing FBSnapshotTestCase (2.1.4)
Installing LXYDemoSDK (0.1.0)
Installing Specta (1.0.7)
Generating Pods project
Integrating client project

[!] Please close any current Xcode sessions and use `LXYDemoSDK.xcworkspace` for this project from now on.
Pod installation complete! There are 5 dependencies from the Podfile and 5 total pods installed.

[!] Your project does not explicitly specify the CocoaPods master specs repo. Since CDN is now used as the default, you may safely remove it from your repos directory via `pod repo remove master`. To suppress this warning please add `warn_for_unused_master_specs_repo => false` to your Podfile.

 Ace! you're ready to go!
 We will start you off by opening your project in Xcode
  open 'LXYDemoSDK/Example/LXYDemoSDK.xcworkspace'

To learn more about the template see `https://github.com/CocoaPods/pod-template.git`.
To learn more about creating a new pod, see `https://guides.cocoapods.org/making/making-a-cocoapod`.
lixuey
```

# 二、更改podspec

```
Pod::Spec.new do |s|
  #   * 库名称
  s.name             = 'LXYDemoSDK'
  #   * 版本号
  s.version          = '0.1.0'
    #   * 简介
  s.summary          = 'A short description of LXYDemoSDK.'
    #   * 描述
  s.description      = 'A description of LXYDemoSDK.'

  s.homepage         = 'https://github.com/xlgy/LXYDemoSDK'
  # s.screenshots     = 'www.example.com/screenshots_1', 'www.example.com/screenshots_2'
  s.license          = { :type => 'MIT', :file => 'LICENSE' }
  s.author           = { 'lxy' => '906099632@qq.com' }
  s.source           = { :git => 'https://github.com/xlgy/LXYDemoSDK.git', :tag => s.version.to_s }
  # s.social_media_url = 'https://twitter.com/<TWITTER_USERNAME>'

  s.ios.deployment_target = '10.0'

  s.source_files = 'LXYDemoSDK/Classes/**/*'
  
  # s.resource_bundles = {
  #   'LXYDemoSDK' => ['LXYDemoSDK/Assets/*.png']
  # }

  # s.public_header_files = 'Pod/Classes/**/*.h'
  # s.frameworks = 'UIKit', 'MapKit'
  # s.dependency 'AFNetworking', '~> 2.3'
end
```

## 重点关注

1. s.source填写的git地址不要使用SSH，要使用HTTS，要不将库开源会出问题
2. tag 默认是版本号，默认是0.1.0，因此上传代码之后需要打tag
3. s.description 需要改下模板生成的文案，不然pod检测会失败
4. 不要有中文


# 三、推送远端仓库

```
git init
git add .
git commit -m "first commit"
git branch -M main
git remote add origin git@github.com:xlgy/LXYDemoSDK.git
git push -u origin main
git tag 0.1.0
```


# 四、本地验证

## 命令
```
pod spec lint LXYDemoSDK.podspec --allow-warnings

```

## 注意
在做本地验证的时候，需要临时将LXYDemoSDK.podspec 文件的s.source改为SSL，只需要本地改就可以，不需要提交。原因是github已经不支持HTTS用户名和密码的访问方式，因此临时改成SSL才可以验证成功

```
s.source           = { :git => 'git@github.com:xlgy/LXYDemoSDK.git', :tag => s.version.to_s }
```

## 日志记录

```
lixueyang@lixueyangdeMacBook-Pro LXYDemoSDK % pod spec lint LXYDemoSDK.podspec --allow-warnings

 -> LXYDemoSDK (0.1.0)
    - WARN  | source: Git SSH URLs will NOT work for people behind firewalls configured to only allow HTTP, therefore HTTPS is preferred.
    - WARN  | summary: The summary is not meaningful.
    - WARN  | description: The description is shorter than the summary.
    - WARN  | url: There was a problem validating the URL https://github.com/xlgy/LXYDemoSDK.git.
    - NOTE  | xcodebuild:  note: Using codesigning identity override: -
    - NOTE  | [iOS] xcodebuild:  note: Building targets in dependency order
    - NOTE  | [iOS] xcodebuild:  note: Target dependency graph (3 targets)
    - NOTE  | [iOS] xcodebuild:  note: Signing static framework with --generate-pre-encrypt-hashes (in target 'Pods-App' from project 'Pods')
    - NOTE  | [iOS] xcodebuild:  Pods.xcodeproj: warning: The iOS Simulator deployment target 'IPHONEOS_DEPLOYMENT_TARGET' is set to 10.0, but the range of supported deployment target versions is 12.0 to 26.1.99. (in target 'Pods-App' from project 'Pods')
    - NOTE  | [iOS] xcodebuild:  /var/folders/rf/mcsy8tv11w58mss9kwmyspk00000gn/T/CocoaPods-Lint-20251219-37213-wjnq94-LXYDemoSDK/App.xcodeproj: warning: The iOS Simulator deployment target 'IPHONEOS_DEPLOYMENT_TARGET' is set to 10.0, but the range of supported deployment target versions is 12.0 to 26.1.99. (in target 'App' from project 'App')
    - NOTE  | [iOS] xcodebuild:  Pods.xcodeproj: warning: The iOS Simulator deployment target 'IPHONEOS_DEPLOYMENT_TARGET' is set to 10.0, but the range of supported deployment target versions is 12.0 to 26.1.99. (in target 'LXYDemoSDK' from project 'Pods')

Analyzed 1 podspec.

LXYDemoSDK.podspec passed validation.

lixueyang@lixueyangdeMacBook-Pro LXYDemoSDK %
```


# 五、发布到私有源

## 1. 本地创建Spec Repo仓库关联LXYDemoSDK

```
//关联刚刚创建的管理库地址
pod repo add LXYDemoSDK git@github.com:xlgy/LXYDemoSDK.git
```

**这里关联的URL为SSL，不然下一步推送会报错**

完成以后前往~/.cocoapods/repo文件夹会发现多了LXYDemoSDK库

## 2.推送 .podspec管理文件到管理库
cd 到 ~/.cocoapods/repo/LXYDemoSDK

```
pod repo push LXYDemoSDK LXYDemoSDK.podspec --allow-warnings
```

看到日志

```
Updating the `LXYDemoSDK' repo


Adding the spec to the `LXYDemoSDK' repo

 - [Update] LXYDemoSDK (0.1.0)

Pushing the `LXYDemoSDK' repo
```
证明上传成功！！！！

## 3.验证私有库

新建pod项目，编辑Podfile

```
platform :ios, '12.0'

target 'Demo' do
  use_frameworks!
  pod 'LXYDemoSDK', :git => 'https://github.com/xlgy/LXYDemoSDK.git'
end
```


执行: pod install


# 六、发布到 CocoaPods 公共源 (trunk)

## 1. 注册 trunk 用户（首次操作）

```
pod trunk register your_email@example.com 'Your Name'

```

## 2. 推送 Podspec 到 trunk

进入到项目代码路径，LXYDemoSDK.podspec的路径

执行：
```
pod trunk push LXYRouter.podspec --allow-warnings
```

结果日志：

```
lixueyang@lixueyangdeMacBook-Pro LXYDemoSDK % pod trunk push LXYDemoSDK.podspec --allow-warnings
Updating spec repo `trunk`
Validating podspec
 -> LXYDemoSDK (0.2.0)
    - WARN  | summary: The summary is not meaningful.
    - WARN  | description: The description is shorter than the summary.
    - NOTE  | xcodebuild:  note: Using codesigning identity override: -
    - NOTE  | [iOS] xcodebuild:  note: Building targets in dependency order
    - NOTE  | [iOS] xcodebuild:  note: Target dependency graph (3 targets)
    - NOTE  | [iOS] xcodebuild:  note: Signing static framework with --generate-pre-encrypt-hashes (in target 'Pods-App' from project 'Pods')
    - NOTE  | [iOS] xcodebuild:  Pods.xcodeproj: warning: The iOS Simulator deployment target 'IPHONEOS_DEPLOYMENT_TARGET' is set to 10.0, but the range of supported deployment target versions is 12.0 to 26.1.99. (in target 'Pods-App' from project 'Pods')
    - NOTE  | [iOS] xcodebuild:  /var/folders/rf/mcsy8tv11w58mss9kwmyspk00000gn/T/CocoaPods-Lint-20251219-44153-22pgtg-LXYDemoSDK/App.xcodeproj: warning: The iOS Simulator deployment target 'IPHONEOS_DEPLOYMENT_TARGET' is set to 10.0, but the range of supported deployment target versions is 12.0 to 26.1.99. (in target 'App' from project 'App')
    - NOTE  | [iOS] xcodebuild:  Pods.xcodeproj: warning: The iOS Simulator deployment target 'IPHONEOS_DEPLOYMENT_TARGET' is set to 10.0, but the range of supported deployment target versions is 12.0 to 26.1.99. (in target 'LXYDemoSDK' from project 'Pods')

Updating spec repo `trunk`

--------------------------------------------------------------------------------
 🎉  Congrats

 🚀  LXYDemoSDK (0.2.0) successfully published
 📅  December 19th, 08:26
 🌎  https://cocoapods.org/pods/LXYDemoSDK
 👍  Tell your friends!
--------------------------------------------------------------
```

看到这个，恭喜你已经开发了自己的Cocoapods开源库了。