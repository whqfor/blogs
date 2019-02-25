---
title: "钉钉E应用调研"
date: 2019-02-25T14:50:23+08:00
lastmod: 2019-02-25T14:5201+08:00
draft: false
tags: ["小程序"]
categories: ["Web", "前端"]
---

##### 简速


在钉钉内运行的"小程序"叫做E应用，[钉钉小程序概览入口](https://open-doc.dingtalk.com/microapp?spm=a2115p.8777639.4570797.11.22baa7dblUfzqH)，又分为三种类型：
```
企业内部开发：企业或组织内部使用，该类应用无需钉钉团队审核，企业内部自行开发并使用即可。

第三方企业应用：基于钉钉的开放能力开发应用，并上架至钉钉应用市场，供钉钉上的企业/组织使用，需要钉钉团队审核。

第三方个人应用：基于钉钉的开放能力开发应用，提供给钉钉个人用户使用。此种类型应用不感知企业信息。应用可以通过群转发、应用市场、群应用使用历史、个人应用使用历史等钉钉客户端入口传播和分发，需要钉钉团队审核。
```
第三方个人应用示例：
![第三方个人应用示例](https://cdn-pub.yuque.com/lark/0/2018/png/18251/1530586998522-f0e59135-db9b-4633-996a-e11ea69a7cde.png)

大致浏览了一下开发者文档，并试用了一下IDE，下面简单介绍下体验。
##### 文档解读

E应用开发和微信小程序很相似，也是分为 `app` 和 `page` 两层。`app` 用来描述整体程序，`page` 用来描述各个页面。

`app` 由三个文件组成，必须放在项目的根目录。

文件 | 必填 | 作用
----- | ----- | -----
app.js | 是 | E应用逻辑
app.json | 是  | E应用公共设置
app.acss | 否 | E应用公共样式表

`page` 由四个文件组成，分别是：

文件类型 | 必填 | 作用
----- | ----- | -----
js | 是 | 页面逻辑
axml | 是 | 页面结构
acss | 否 | 页面样式表
json | 否 | 页面配置

`acss` `axml` 中的a的是ant蚂蚁的首字母，和支付宝小程序通用，语法和前端开发使用类似，上手容易。


[`API`可以看这里](https://open-doc.dingtalk.com/microapp/dev/framework-app)
组件还算齐全，满足绝大多数UI功能开发。不过目前`多媒体能力欠缺`，开发音视频体验受限，不能像微信小程序一样提供丰富的多媒体API供开发者调用，微信小程序的多媒体能力是直接使用Native能力供小程序开发者调用。

##### IDE
[IDE下载](https://open-doc.dingtalk.com/microapp/kn6zg7/zunrdk)

打开ide之后新建应用看到如下界面：
![创建应用.png](https://upload-images.jianshu.io/upload_images/273788-101e22d655899c9f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
同一个IDE可以开发`支付宝小程序`，`钉钉E应用`，还有基于`mPaaS框架的应用`。因为都采用`acss` `axml` `js` `json`，差异只是平台不同，大部分功能可以复用。

目前支付宝小程序不接受个人开发者试用，只有企业可以申请。

移动开发平台（Mobile PaaS，简称 mPaaS）是源于支付宝 App 的移动开发平台，为移动开发、测试、运营及运维提供云到端的一站式解决方案，[文档](https://tech.antfin.com/docs/2/49549)支持热更新及跨平台。

ide内置demo功能，可以先查看组件示例，简单体验一下里面的组件。

工程打开后的界面如下图：
![ide预览.png](https://upload-images.jianshu.io/upload_images/273788-d4258e02b814cbe5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240) 和微信小程序很相似，并且把模拟器和代码编辑区分开，更方便调试。

真机预览会生成一个二维码，需要使用钉钉扫描查看。

![上传.png](https://upload-images.jianshu.io/upload_images/273788-5cd919ec302e45d6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
上传操作很简单，点击上传即可，上传之后可以在钉钉后台看到，这时候并不是提交审核。
这个地方有个小坑，`点击这里`按钮跳转到的页面支付宝小程序管理后台。因此需要用户自己搜索进入钉钉开放平台，界面如下图
![管理.png](https://upload-images.jianshu.io/upload_images/273788-d429d2bc5b5f649f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
审核通过之后可以灰度发布，也可以版本回退。
灰度测试后，开发者点击发布，钉钉用户即可使用该版本的应用，用户可通过`扫码或分享`进入应用。
E应用的入口在钉钉里隐藏的比较深，钉钉->我的->最下侧E应用，可能是还没有大规模推广的原因。
目前可以看到的个人开发的E应用比较少，面向企业和组织内部的也不是太多。如果进行开发的话，可以先小范围试用。







