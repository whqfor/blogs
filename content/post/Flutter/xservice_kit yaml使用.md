---
title: "xservice_kit yaml使用"
date: 2019-06-25T14:50:23+08:00
lastmod: 2019-06-25T20:5201+08:00
draft: false
tags: ["Flutter"]
categories: ["Flutter"]
---

xservice_kit 的功能是管理channel通信，详细介绍之后开篇再讲，接上一篇，这篇主要介绍使用：[官方文档](https://pub.dev/packages/xservice_kit) 里面有介绍、集成说明、使用文档、原理介绍，如果看完明白了，就没必要往下看你啦。
Xservice的优势：

	•	统一标准化配置
	•	一份配置三端代码生成，告别重复手写代码工作。
	•	消息类型显示指定，方便消息定义和调用，增加类型安全性。
	•	方便支持双向一对一，一对多消息代码生成。
	•	自定义消息过滤支持，防止自定义类型序列化过程中的异常。
看完是不是很心动呐，安装介绍pub文档即可，里面有详细说明。在使用的过程中 yaml的生成规则，感觉描述的不是很清楚，接下来结合yaml自动生成channel三端代码，介绍一下自己理解的Xservice使用。

Xservice的使用步骤官方文档给出如下：

	•	在Flutter工程pubspec.yaml加入xservice_kit依赖: xservice_kit:^0.0.27
	•	安装node（代码生成工具使用node开发），然后npm install xservice -g
	•	编写配置文件
	•	运行xservice命令行生成代码
	•	将生成代码移动到项目，然后在程序的开始调用Serviceloader 
前两步自行安装即可。
##### 编写配置文件：
先给自己编写的yaml文件
```
name : TestService

messages:

 -
  name : MessageToFlutter
  returnType : Map 
  messageType : native
  channelType : method
  args : 
   -
    name : url
    type : String
   -
    name : params
    type : Map

 -
  name : MessageToNative
  returnType : Map
  messageType : flutter
  channelType : method
  args : 
   -
    name : params
    type : List
```
下面针对配置参数介绍下：

```
name : TestService  			// 自定义Service名字

messages:

 -
  name : MessageToFlutter		// NA调用的Flutter方法名
  returnType : Map				// 返回数据类型 
  messageType : native			// 在native侧生成的代码
  channelType : method			// Channel类型
  args : 
   -
    name : MessageToFlutterArg1	// 参数名字
    type : String				// 参数类型  
   -							// - 要缩进对齐，格式有要求
    name : MessageToFlutterArg2 // 参数名字
    type : Map					// 参数类型 
    
    // 以上有俩参数  一个是字典一个是string 则native侧会创建一个【字典】包装俩参数。
	// 只有一个参数时，type为什么类型，参数即是什么类型，如下：List在Android侧对应的
args : 
   -
    name : MessageToFlutterArg1
    type : List
```
Type 支持的数据类型如下：
```
"int":"0",
"double":"0.0",
"bool":"false",
"String":'""',
"List”:"[]"
"Map”:"{}"
```

##### 运行xservice命令行生成代码
命令如下：
```
xservice -o flutterService -p channel -t yaml /Users/whqfor/Desktop/ServicesYaml 
```
在`/usr/local/lib/node_modules/xservice/index.js`这个文件里可以找到生成channel代码的逻辑，这里只做简单解读：
先介绍命令里的参数
```
.arguments('<directory>')
.option('-o , --output <output>' , 'A directory to save the output')
.option('-t , --type <type>' , 'input file type support json and yaml default is json')
.option('-p , --package <package>' , 'Java package')
.action(function(directory){
    program.directory = directory;
    start(directory);
}).parse(process.argv);
```
这一段是index.js 里的命令解释，结合自己的使用说明一下，
`-o flutterService` 定义输出的路径，默认是在用户目录下，命令执行完之后，生成的三端代码会存放在这里。
`-p channel`主要是给Android小伙伴使用，Android的package 路径里添加的例如：
`package channel.TestService.handlers;`
目录结构和package是对应的的，如下：
![目录结构.png](https://upload-images.jianshu.io/upload_images/273788-2cf462cb5376649e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
`-t yaml`配置文件的类型，推荐yaml格式，也支持json格式
`/Users/whqfor/Desktop/ServicesYaml`存放yaml文件的地址
执行完生成的结构如下：
![文件结构](https://gw.alicdn.com/tfs/TB1DUM4IOrpK1RjSZFhXXXSdXXa-1230-1050.png)

最后将生成的文件拷贝到项目里即可，iOS的拷贝打开iOS工程，copy if need，Android和dart的代码可以直接拖到项目里。


