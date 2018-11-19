
---
title: "pod私有库搭建及podspec编辑简介"
date: 2018-11-01T14:50:23+08:00
lastmod: 2018-11-09T14:5201+08:00
draft: false
tags: ["podspec", "CocoaPods"]
categories: ["Tools", "iOS"]
---


### 简述
`Cocoapods`是个非常好的`iOS`依赖管理工具，可以方便的进行管理和更新使用到的第三方库，以及在做代码模组件化管理的时候，可以用它来管理私有库。
`Cocoapods`的[安装](https://segmentfault.com/a/1190000011428874)、[使用](https://segmentfault.com/a/1190000012269216)比较基础，这里不再赘述，网络上有很多资料。这篇文章主要介绍在`Git`环境下如何搭建自己的私有仓库，用以管理项目中的小模块。
在搭建自己的私有库之前先看一下pod的工作过程：
![pod工作工程](https://image-static.segmentfault.com/385/861/3858613500-5a38be353416c)
当我们执行
```
$ pod search afnetworking
``` 
的时候会先搜索`本地repo`，其所在路径是`~/.cocoapod/repos`，如果本地没有则会默认执行
```
pod setup/pod update
```
去更新repo（将podspec文件下载到本地），这个过程比较慢，也可以自己去[github](https://github.com/CocoaPods/Specs)下载完成后放到上面路径中。
`cocoapods`其实就是利用所维护的podspec文件，在使用方和提供方之间建立起一个桥梁，并利用与项目关联的pod项目去维护所有第三方。
如果想搭建一个自己的私有仓库，则在本地需要搭建一个自己的`repo`仓库来管理保存自己的podspec文件。

### 创建步骤
```
1.创建私有的Specs git库
2.在私有库项目中创建podspec文件
3.验证私有库的合法性
4.提交私有库的版本信息
5.向Spec Repo提交podspec
6.更新维护podspec
7.示例地址
```
#### 1.创建私有的Specs git库
将私有`repo` 添加到 `Cocoapods`的格式是
```
$ pod repo add [repo名] [repo git地址]
```
举例：
a.首先创建一个`Git`仓库地址，例如`https://github.com/xxx/HQSpecs`，然后将其添加到`Cocoapods`列表中（多个工程podspec可以共用一个私有repo）。
```
$ pod repo add HQSpecs https://github.com/whqfor/HQSpecs.git
```
验证是否创建成功可以执行查看
```
$ pod repo list
```
创建成功后list中即会展现出刚才创建的repo
```
HQSpecs
- Type: git (master)
- URL:  https://github.com/xxx/HQSpecs.git
- Path: /Users/whqfor/.cocoapods/repos/HQSpecs

master
- Type: git (master)
- URL:  https://github.com/CocoaPods/Specs.git
- Path: /Users/whqfor/.cocoapods/repos/master
```
目前本地的私有repo是个空文件，先不要着急，一会还会对它进行操作，不需要手动更改。
#### 2.在私有库项目中创建podspec文件
按照[官方教程](https://guides.cocoapods.org/syntax/podspec.html)来编辑即可，下面是在写本文时创建的示例podspec

```

Pod::Spec.new do |s|

  s.name         = "TNetwork"
  s.version      = "0.0.1"
  s.summary      = "TNetwork base on AFNetworking and YTKNetwork."
  s.homepage     = "https://github.com/whqfor/TNetwork"
  s.license      = 'Code is MIT, then custom font licenses.'
  s.author       = { "whqfor" => "whqfor@126.com" }

  s.source       = { :git => "git@github.com:whqfor/TNetwork.git", :tag => "#{s.version}" }
  s.source_files  = "TNetwork/**/*.{h,m}"
  s.public_header_files = 'TNetwork/**/*.h'
  s.requires_arc = true
  s.ios.deployment_target = "8.0"
  s.frameworks = "Foundation", "UIKit"

  s.user_target_xcconfig = { 'CLANG_ALLOW_NON_MODULAR_INCLUDES_IN_FRAMEWORK_MODULES' => 'YES' }

  s.dependency "AFNetworking", "~> 3.0"
  s.dependency "YTKNetwork"

end
```
如果不清楚的话，网上相应的文章挺多的，这一步并不复杂。
编写完之后放到git仓库下即可，和工程同级目录。
#### 3.验证私有库的合法性
这是我在创建私有库时遇到问题最多的地方，尝试编译了半天。
在本地git仓库目录下，选择执行如下命令
```
pod lib lint （从本地验证你的pod能否通过验证）
pod spec lint （从本地和远程验证你的pod能否通过验证）

pod lib lint --verbose （加--verbose可以显示详细的检测过程，出错时会显示详细的错误信息）
pod lib lint --allow-warnings (允许警告，用来解决由于代码中存在警告导致不能通过校验的问题)
pod lib lint --help （查看所有可选参数，可选参数可以加多个）
```
这篇文章主要是介绍搭建私有仓库，所以首先验证本地pod是否能通过
```
$ pod lib lint --allow-warnings
```
执行这个命令的过程可能会花点时间，此外也会遇到各种错误。只需关注错误信息即可，最常见的是
```
error: include of non-modular header inside framework module
```
下面说下我在验证时总结的经验。

```
1. 引入类，使用@class XXX; 不能像平时一样直接引入.h，可以在.m文件中引入。
2. 引入协议，使用@protocol XXX; 。
3. 当需要继承别的文件时，按照@class XXX；引入会报错，此时只能引入.h文件。
```
针对第3条，参考[这篇文章给出了很多示例错误信息](https://blog.csdn.net/blog_jihq/article/details/52614156?utm_source=blogxgwz2)第7条，将
```
s.user_target_xcconfig = { 'CLANG_ALLOW_NON_MODULAR_INCLUDES_IN_FRAMEWORK_MODULES' => 'YES' }
```
这条加入到podspec中解决了问题，原理是改变了本工程Xcode的编译设置。
如果通过的话，会有相应的的提示
```
TNetwork passed validation.
```
#### 4.提交私有库的版本信息
podspec文件中获取Git版本控制的项目还需要tag号，所以我们要打上一个tag，在工程目录下，终端执行
```
$ git tag -m "first release" "0.0.1"
$ git push --tags     #推送tag到远端仓库
```
#### 5.向Spec Repo提交podspec
接下来将刚才的0.0.1版本的podspec提交到私有仓库中
```
$ pod repo push HQSpecs /Users/whqfor/TNetwork/TNetwork.podspec --allow-warnings
```
如果选择忽略警告的话可以加上--allow-warnings
#### 6.更新维护podspec
之后如果需要发新的版本，和上诉流程基本一致，编辑好自己的库文件，打上tag提交到远端后，更新下podspec文件，执行第四步验证过之后，就可以继续按照第五步提交repo了。

```
~/.cocoapods/repos/HQSpecs 内的目录如下
├── LICENSE
├── TNetwork
│   ├── 0.0.1
│   │   └── TNetwork.podspec
│   └── 0.0.2
│       └── TNetwork.podspec
└── README.md
```

删除本地私有库
```
$ pod repo remove WTSpecs
```
还可以再添加回来
```
$ pod repo add HQSpecs https://github.com/CocoaPods/Specs.git
```

#### 7.示例地址

在尝试的过程中，版本号没变化的话，之前pod install可能有缓存。在`~/Library/Caches/CocoaPods/`路径下找到缓存的库，直接删除即可。

HQSpecs仓库里放置的是podspec文件，有自己对应的git地址，podspec对应的仓库是另一个git地址，容易搞浑了。

以上即是只做了一个简单的私有库制作，后续会再完善下subspec制作，如果以后组件化的路上积攒更多的经验会持续更新。

[本文HQSpec地址](https://github.com/whqfor/HQSpecs)  
[本文podspec文件对应工程地址](https://github.com/whqfor/TNetwork)


### 参考文章
[使用Cocoapods创建私有podspec](http://www.cocoachina.com/ios/20150228/11206.html)  
[Cocoapods整理（三）——编写podspec文件](https://segmentfault.com/a/1190000012269307)  
[podSpec文件相关知识整理](https://www.cnblogs.com/richard-youth/p/6272932.html)



