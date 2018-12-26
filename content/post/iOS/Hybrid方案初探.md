---
title: "Hybrid方案初探"
date: 2018-12-16T14:50:23+08:00
lastmod: 2018-12-20T20:5201+08:00
draft: false
tags: ["Hybrid"]
categories: ["iOS"]
---


		移动端的两大平台Android和iOS，基于自身平台的特性，开发相应的native应用能保证用户最好的体验效果。两个平台的开发技术和运行方式不同，针对同一套业务逻辑需要实现两遍，这样就有了跨平台开发的需求，同时需要保证用户体验。关于跨平台目前主要有两个方向，一种是Hybrid，一种是基于React Native/Flutter/Weex 等平台进开发，两者各有优劣先不做讨论，本文主要给出自己对Hybrid 方案的调研分析，经验有限，如有错误盼望指正。
### 1.Hybrid App的[类型划分](https://www.cnblogs.com/dailc/p/5930231.html)
#### a.多View混合型
	这种模式主要特点是将webview作为Native中的一个view组件，当需要的时候在独立运行显示,也就是说主体是Native，web技术只是起来一些补充作用。
	大多数APP都会或多或少的使用，比如加载一个使用帮助之类的页面，功能比较单一，一般都是静态展示，跨平台能力较弱。
#### b.单View混合型
	这种模式是在同一个view内，同时包括Native view和webview（互相之间是层叠的关系），比如一些应用会用H5来加载百度地图作为整个页面的主体内容，然后再webview之上覆盖一些原生的view，比如搜索什么的。
	这种用户体验好，但是开发成本较大，更适合移动端人员使用，不够灵活。
#### c.Web主体型
	这种模式算是传统意义上的Hybrid开发,很多Hybrid框架都是基于这种模式的,比如PhoneGap、Cordova、AppCan、Ionic等。这种模式的一个最大特点是，Hybrid框架已经提供各种api，打包工具，调试工具，然后实际开发时不会使用到任何原生技术，实际上只会使用H5和js来编写，然后js可以调用原生提供的api来实现一些拓展功能。往往程序从入口页面，到每一个功能都是h5和js完成的。
	理论上来说，这种模式应该是最佳的一种模式（因为用H5和js编写最为快速，能够调用原生api，功能够完善），但是由于一些webview自身的限制，导致了这种模式在性能上损耗不小，包括在一些内存控制上的不足，所以导致体验要逊色于原生不少。
	
#### d.多主体共存型（灵活型）
	这种模式的存在是为了解决web主体型的不足，这种模式的一个最大特点是，原生开发和h5开发共存，也就是说，对于核心模块、交互性强的的界面仍是Native 开发为主，对于一些通用型模块，用h5和js来完成。
	在开发原生应用的基础上嵌入webView，但是整体的架构使用原生应用提供，一般这样的开发由Native开发人员和Web前端开发人员组成。Native开发人员写好基本的架构以及API让Web开发人员开发界面以及大部分的渲染。保证到交互设计，以及开发都有一个比较折中的效果出来，优化的好也会有很棒的效果。

	结合目前的研发方向，我们更趋向于客户端提供框架，前端人员可以选择性灵活的使用框架提供的Api和Native进行交互，从而在web开发的基础上，最大限制的发挥native的能力，保证跨平台的能力和比较不错的体验效果。这样看来比较符合符合多主体共存型这种类型。

### 2.通信方式
	既然选择了多主体共存这种方式，就需要考虑框架的设计，那么首先要考虑的是JS和Native怎么通信的问题，有了通信之后还需要再结合JS/Android/iOS三个台各自的特性进行封装，做到在JS侧差异内化解决。这里主要说一下通信方式：
		
#### Native调用JS

JS是脚本语言，任何一个JS引擎都是可以在任意时机直接执行任意的JS代码，我们可以把任何Native想要传递的消息/数据直接写进JS代码里，这样就能传递给JS了，Android和iOS都可以通过执行`evaluateJavascript` 直接注入执行JS代码，Android 4.4之前和4.4之后执行方式不太一致，考虑到目前的用户分布可以考虑将APP最低适用版本设置到4.4，根据自身APP过去30天启动次数统计占比只有0.1%。
Android4.4之前，可以通过loadUrl方式调用
Android4.4之后
```js
webView.evaluateJavascript("javascript: 方法名('参数,需要转为字符串')", new ValueCallback( ) {
	@Override
	public void onReceiveValue(String value) {
		//这里的value即为对应JS方法的返回值
	}
});

```
iOS
```
[webView stringByEvaluatingJavaScriptFromString:@“方法名(参数)”];
```
		此外WKWebView 还有一个新的Api，WKUserScript 可以在预先准备好JS代码，通过 addUserScript方法，当WKWebView加载Dom的时候，执行当条JS代码，这种Native主动调用JS，只能在WebView加载时期发起，并不能在任意时刻发起通信。
		所以Native调用JS就选取evaluateJavascript即可。

#### JS调用Native
主要有三种方式，Android、iOS都适用：
```
* 假跳转的请求拦截，A标签跳转、原地跳转、iframe跳转，提前约定好URL。
* 弹窗拦截，alert()、prompt()、confirm()。
* JS上下文注入:
	a. iOS JavaScriptCore注入
	b. Android addJavascriptInterface( ) 注入
	c. iOS scriptMessageHandler注入
```
JS调用Native实现方式主要有以上三种，具体怎么实现不做过多说明，简单的进行一下优缺点对比：
##### 1.假跳转的请求拦截
[WebViewJavascriptBridge](https://github.com/marcuswestin/WebViewJavascriptBridge) 和[cordova](https://cordova.apache.org) 都是使用这种方式
```
优点：兼容性好，iOS6之前只有这一种方式，唯一同时支持Android webview/iOS UIWebView/iOS WKWebView的通信方式。
缺点：
	1.经过测试当连续发送消息的时候会丢失消息
	2.URL长度有限制，在http协议里没有对URL长度做限制，但是还要受制于不同浏览器及web服务器的最大处理能力，一般URL长度最好不操作过2083个字符，一个汉字如果经过UTF8编码的话占用9个字符，也即是最多支持230个左右汉字。
```
##### 2.弹窗拦截
```
优点：支持 同步/异步 数据返回。
这个几乎没啥严重缺点，主要是需要将需要传递的内容json序列化成字符串，APP端需要再进行解析。
UIWebView不支持弹窗拦截，但UIWebView有更好的JS上下文注入的方式，JSContext不仅支持直接传递对象无需json序列化，UIWebView在iOS12及以后被废弃了，目前重新设计框架的时候不再考虑。
安卓一切正常，不会出现丢消息的情况。
WKWebView一切正常，也不会出现丢消息的情况，但其实WKWebView苹果给了更好的API，可以直接传递对象无需进行json序列化的。
```
##### JS上下文注入
>a. iOS JavaScriptCore注入
只有UIWebView可以使用，考虑到iOS12以后UIWebView 被废弃，不做过多讨论。
>b. Android addJavascriptInterface( ) 注入
Android的Interface( ) 功能比较强大，可以同步返回数据、无需json序列化传输数据、可以注入Native对象，缺点好像是有漏洞，可以参考这篇文章。
>c.iOS scriptMessageHandler注入
优点：无需json序列化传递数据，不会丢失消息。
缺点：不支持同步返回数据，可以通过其他方式实现Native同步返回数据到JS。
考虑到WKWebView强大的特性，这个是优点考虑方案。

##### 横向对比

通信方案 | 版本支持 | 丢消息 | 支持同步返回 | 传递对象 | 注入原生对象 | 数据长度限制
-----| -----| -----| ----- | ----- | ----- | -----
假跳转 | 全版本全平台 | 会丢失 | 不支持 | 不支 | 不支持 | 有限制
弹窗拦截 | UIWebView不支持 | 不丢失 | 支持 | 不支持 | 不支持 | 无限制
JSContext注入 | 只有UIWebView支持 | 不丢失 | 支持 | 支持 | 支持 | 无限制
安卓interface注入 | 安卓全版本 | 不丢失 | 支持 | 支持 | 支持 | 无限制
MessageHandler注入 | 只有WKWebView支持 | 不丢失 | 不支持 | 不支持不支持 | 无限制

综合来说可以考虑：
```
NativeToJS : evaluatingJavaScript

JSToNative : 
	iOS：异步数据返回采用messageHandler注入+同步数据返回prompt()
	Android：弹窗拦截
```
在调研的工程中简单写了一下 OC与JS的通信方式见[demo](https://github.com/whqfor/githubDemo/tree/master/HybridWK)

在整理的过程中发现一些比较不错的文章，本文不少内容整理于此**感谢**：
[从零收拾一个hybrid框架（一）-- 从选择JS通信方案开始](awhisper.github.io/2018/01/02/hybrid-jscomunication/)
[Hybrid APP基础篇(三)->Hybrid APP之Native和H5页面交互原理](https://www.cnblogs.com/dailc/p/5931322.html)
[WebViewJavascriptBridge](https://github.com/marcuswestin/WebViewJavascriptBridge)
[Android：你要的WebView与 JS 交互方式 都在这里了](https://blog.csdn.net/carson_ho/article/details/64904691)
[你不知道的 Android WebView 使用漏洞](https://www.jianshu.com/p/3a345d27cd42)

也查看了一些资料：

[cordova](https://cordova.apache.org)
[https://www.w3cschool.cn/cordova/](https://www.w3cschool.cn/cordova/)
```
1.可以使用原生的js、html、css来构建一个应用。
2.支持很多的插件来去调原生的API的，这种插件的库和它的生态是非常完善的，也就是说一个前端开发者不需要懂原生就可以做。
相对来说比较稳定，值得借鉴学习。
缺点:
1.这个框架是一个比较重的框架，做Hybrid开发的话，集成在原生的app里面，使得整个APP比较重
```
[ionic](www.ionic-china.com/)
[runoob ionic教程](http://www.runoob.com/ionic/ionic-tutorial.html)
[ionic中文教程](http://www.ionic.wang/js_doc-index.html)
`Ionic 约等于 Cordova + Angular + UI 组件库。`
```
Ionic提供了一个免费且开源的移动优化HTML，CSS和JS组件库，来构建高交互性的应用。它可以用框架中的CSS 实现有 native 风格的设计，不过相对于使用完整的 Ionic，更建议搭配 AngularJS 一起开发，从而创建完美的应用。
它有如下特点：
1 . 性能高，运行速度快，操作最少的DOM，非 jQuery且和硬件加速过渡；
2 . 设计简单，并且实用，它为当前移动设备呈现了完美的设计；
3 . 以原生SDK为蓝本，便于移动端开发人员的理解，完成时通过PhoneGap发布，达到一次开发，处处使用的效果；
4 . 核心架构是为开发专业应用创建，框架轻量级；
5 .一个命令就可以创建，构建，测试，部署你的应用程序在任何平台上,只需要npm install -g ionic 就可以创建您的应用。
6 . 代码标准，后台维护人员专注，具有强大的社区。
缺点
在了解Ionic的同时，还需要了解AngularJS，为开发增加了一定的复杂以及难度；
```

[VasSonic](https://github.com/Tencent/VasSonic) 
`优化网页加载速度`

