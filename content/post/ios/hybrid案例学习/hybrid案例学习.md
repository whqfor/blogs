---
title: "Hybrid案例学习"
date: 2018-12-20T14:50:23+08:00
lastmod: 2018-12-26T20:5201+08:00
draft: false
tags: ["Hybrid"]
categories: ["iOS"]
---

	花了两三天时间试用了一下Cordova，也粗略看了一下VasSonic源码，两者在对hybrid的探索上都有很好的参考价值，只是初步使用，理解有限，简单说下自己的见解。Cordova主要侧重于框架封装，使用这套框架可以比较方便的和Native进行交互，提供原生Api给web开发使用。VasSonic侧重于性能优化，极致的提高了web页面的加载速度。
	
### [Cordova](https://cordova.apache.org/docs/en/latest/)
可以很方便的根据[官网提供的指令](https://cordova.apache.org/docs/en/latest/guide/cli/index.html)创建初始Cordova工程，也可以[稍微麻烦一点](https://www.jianshu.com/p/cb400e3888f0)集成进已有工程，不做重点讨论。说一下体验，按照官网教程，创建体验demo，在cordova build的时候失败，也运行不起来，完全按照官方流程一步一步运行却出错，对于初学者很不友好，排查错误也没找到好的解决方法，网络上相关文档不多，当然这不是本节重点。按照上面教程将cordova集成进已有APP，本节重点在于分析其运行原理及提供plugin给web端使用的能力。
	
	Cordova对web的性能提升没做优化，甚至由于其经过Cordova.js的处理还略有降低，比如在Cordova编写的web页面中，想要打开一个链接，由于Cordova.js中的消息拦截并不能直接打开，需要相应的plugin去处理，否则无法打开。

Cordova的优势在于，使用这套框架可以便捷的和原生Api进行交互。其框架如下图：
![架构图](https://raw.githubusercontent.com/whqfor/whqfor.github.io/master/post/iOS/Hybrid案例学习/cordova.png)

Cordova框架 主要由三部分组成：`WebView` `Web App` `Plugins`。
使用这套框架可以进行跨平台开发或者单端平台开发。
>WebvView：提供开发的承载容器，可以作为组件使用，在iOS中采用UIWebView作为承载。
>Web App：应用程序本身是作为web页面实现的，和常规web页面开发方式区别不大，有一个非常重要的文件config.xml，是配置一些配置plugin及关键配置的地方。
>Plugins：Cordova不可或缺的一部分，为Cordova和本地组件提供了相互通信的接口，并提供了与Native Api的绑定，使JS能够方便的调用Native代码。

Cordova提供了很多封装好plugin，可以方便的集成，也可以准守其plugin开发规范，开发出自己想要的插件。在官网提供的plugin中，如右图所示，大多数plugin更新都在两三年前。此外看到的现象是使用者越来越少，相关博客较少，大多数相关博文，也都是很久之前跟新的内容，停留在基础使用层次。

#### 通信流程
在自己需要进行开发的web页里引入外部脚本文件`cordova.js[, cordova_plugins.js]`，`cordova.js`是JS一侧的处理消息的地方，`cordova_plugins.js`是 plugins的JS接口。当JS需要和Native进行交互时最终消息都会通过`cordova.exec`执行

```
cordova.exec(function(winParam) {}, 
             function(error) {},
             "service",
             "action",
             ["firstArgument", "secondArgument", 42, false]);
第一个参数是成功的回调函数，
第二个参数是失败的回调参数，
service 即 plugin的名字，
Action 是调用的native中的函数名字，
最后是一个数组，可以传给native的参数。
```
```
调用这个函数之后，这并不是真正发给native的消息，因为假跳转拦截有两个弊端：
1.连续发送两条消息，只能收到1条，会丢失其中一条，
2.受HTTP协议限制，url的长度有限制，最大不超过2K字符左右
```
Cordova进行的处理是进行再封装：
```js
if (successCallback || failCallback) {
        callbackId = service + cordova.callbackId++;
        cordova.callbacks[callbackId] =
            {success:successCallback, fail:failCallback};
 }
```
在cordova.js中维持一个数组，根据service和自增的`callbackid` 拼接的字符串做为回调的`callbackid`，由`successCallback`、`failCallback`作为其`value`。
然后将这些参数组成一个数组，
```
var command = [callbackId, service, action, actionArgs];
```
再将数组转换为JSON String 存入一个指令集中
```js
commandQueue.push(JSON.stringify(command));
```
commandQueue是一个	`队列`，里面放着`JS->Native`的信息。
最后执行pokeNative 函数，如果有多条消息，则间隔50毫秒发送一条固定的`gap://ready` 消息，这样间接解决了假跳转拦截的两个弊端。
这样就将**JS->Native**的信息发送出去了。

在Native侧，通过无差别的拦截request，当拦截到`gap://ready` 消息后，Native会创建一个CDVViewController类来处理这个消息，通过
```
stringByEvaluatingJavaScriptFromString:
```
执行JS提供的固定函数：
```
cordova.require(‘cordova/exec’).nativeFetchMessages()
```
，来获取刚才存储到commandQueue中的参数，将其JSON序列化之后得到需要执行的command数据，Native也维护一个`队列`，每次取以一条消息执行，如果有多条消息，也是间隔一个固定时间去执行。

command消息包含`callbackId`、`className`、`methodName`、`arguments`
四个属性，`callbackId`是native执行完消息后需要回调给JS的唯一标示，`className`也即使要执行的plugin，这样就知道了哪一个类需要执行什么方法，需要什么参数。当native处理完了之后，将执行的结果添加上`CDVCommandStatus`标识后封装成`CDVPluginResult`类型，回调的结果支持基本数据类型，当然最后都是转换位JSON，最后通过`CDVCommandDelegateImpl`根据`CDVPluginResult`和`callbackID`生成参数回调给JS，也是通过webView的`stringByEvaluatingJavaScriptFromString`进行。

再根据`callbackid`找到之前保存的
```
var callback = cordova.callbacks[callbackId]
```
这样拿到之前定义好的的`callback.success` 或者 `callback.fail`将结果返回给调用方。
至此一个完整的基本调用过程就完成了。

#### Cordova总结
```
	通信流程不算复杂，这里iOS平台使用了UIWebView作为基础webView，但是UIWebView在iOS12之后要被淘汰，那么Cordova框架将会有很大的改变，并且消息需要转换为JSON String去处理，JS和Native都要做相应的转码，此外有多条消息时需要等待一段时间才能处理，只能异步回调。
	分析完Cordova的执行的流程之后，基本就能了解为什么自定义plugin这么方便了。因为plugin要做的事情很简单：
1.只需继承CDVPlugin，
2.提供一个函数包含CDVInvokedUrlCommand类型的参数
3.如果需要回掉，则让父类CVDPlugin中的代理commandDelegate，执行相应方法即可sendPluginResult: callbackId:
	不需额外关注其他的事情，在上一篇文章里介绍了通信的方式，这里分析完Cordova，学习到了设计通信框架的一个思路，即简洁的给Native提供通信接口，使native功能模块只需关注实现独立的功能即可。可以借鉴Cordova上已有的plugin，封装咱们自己的Native组件。
```

### [VasSonic](https://github.com/Tencent/VasSonic)
腾讯的VasSonic解决的问题是web性能问题，WebView加载慢主要有几个问题：
```
1.webView初始化
2.页面资源加载
3.数据更新问题
```
针对这几个问题，VasSonic给出了相应的解决方案:
```
1.webView首次启动初始化慢，预先创建webView池
2.在资源加载上做的优化是，初始化webView和request数据并行处理，CDN加速，域名解析优化等
3.数据更新问题，则使用动态加载，缓存数据，局部刷新及预加载技术，局部刷新又包含h5模板拆分等功能点。
```
开源的VasSonic对webView加载速度优化很强大，能给用户带来更好的体验，但是涉及到了很多的技术，需要web端、iOS、Android、后端一起进行优化，需要有人统筹，短期内各端可以借鉴单端的优化，比如iOS可以借鉴其中的预先创建webView池及并行加速技术，缓存策略也是需要学习借鉴的一个技术点，这些需要进行持续研究。

关于性能优化可以参考：
腾讯团队：[腾讯祭出大招VasSonic，让你的H5页面首屏秒开！](https://mp.weixin.qq.com/s/5SASDtiBCHzoCN-YBZy1nA)
美团技术：[WebView性能、体验分析与优化](https://tech.meituan.com/WebViewPerf.html)







