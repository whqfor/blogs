---
title: "Hybrid资源管理"
date: 2019-03-24T17:57:23+08:00
lastmod: 2019-03-25T17:57:23:01+08:00
draft: false
tags: ["Hybrid"]
categories: ["iOS"]
---

为了提高webView加载速度，将需要加载的资源文件提前下载的APP，下面将对资源下载流程进行介绍。

![hybird资源管理.png](https://upload-images.jianshu.io/upload_images/273788-d043553adabdda3d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

资源管理主要分为三个部分，如上图：
```
下载：黄色部分，主要负责文件的下载操作。
匹配：橙色部分，每次拉取完成配置文件后进行匹配。
文件管理：绿色部分。
```

配置文件内容如下：
```json
{
    "moduleA": {
        "h5_modul_id": "12345765432",
        "h5_resource_name": "serviceName",
        "h5_resource_version": "1.1",
        "h5_resource_download_url": "http://cdn.XXX.com/client/html.zip",
        "h5_resource_route_uri": "Test://hybrid/moduleA",
        "h5_resource_index": "/modules/test/test.html"
    },
    "moduleB": {
        "h5_modul_id": "12345760001",
        "h5_resource_name": "serviceName",
        "h5_resource_version": "1.0",
        "h5_resource_download_url": "http://cdn.XXX.com/client/html.zip",
        "h5_resource_route_uri": "Test://hybrid/moduleB",
        "h5_resource_index": "/modules/test/test.html"
    }
}
```
每一个module是一个具体功能，对应一个zip压缩包，比如moduleA、moduleB，和`h5_resource_route_uri`字段下的module对应。每次下发的配置文件时全量的hybrid功能，不过不用担心，下载的时候只会下载和上次有变更的内容，这即是橙色部分的功能。会将上次拉取的配置文件和本次拉取的配置文件做比较，并生成一份和存储相对应的目录，下面会详细介绍。

和APP交互的地方有两个：
一是：APP启动的时候，指定配置文件下载地址，资源管理会按照配置文件进行更新。
另一个是：APP在需要使用hybrid资源的时候，根据路由查找，及`h5_resource_route_uri`中定义的字段，先查找下发的配置文件，如果有相应的module，再去文件系统里查找是否有下载好的文件，则下载，下载完成解压后，`callback`查找到`fileUrl`给APP。

这里贴一下存储在APP中的目录结构：
![存储目录.png](https://upload-images.jianshu.io/upload_images/273788-6b7f818862cf730a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

文件存储在Documents下创建的hybrid中，hybrid里面存放着各个module，与之并列的是两个字典文件，`preJsonValue`里面是最新拉取的json配置里的数据。`preMergeValue`里存储着新旧两次合并之后的信息。使用`NSDictionary`自带存储方式即可
```objc
// 储存
[jsonValue writeToFile:preJsonValueFilePath atomically:YES];

// 读取
[NSDictionary dictionaryWithContentsOfFile:preJsonValueFilePath];
```
每个module下面最多保两个版本，如上图中`moduleB`下面有着两个版本，1.0、1.1，当再有个1.2版本时，会把module下面最小的一个版本1.0删除。假如只保留一个版本的时候，如果程序启动下载资源比较慢，这时APP已经加载了之前之前版本的web页面，按照一个文件的逻辑，下载完新的资源需要把老的资源删除，这样就对正在显示的页面有了影响。保留两个文件能够解决这个问题`这一块可能不是最优解，欢迎一起探讨`。也不用担心会加载老版本的问题，路由查找的时候会对配置里的版本号进行比对，保证加载的都是配置文件中指定的版本。

关于下载和文件管理比较简单，不做过多介绍。主要介绍一下匹配策略。
##### 匹配

匹配策略是资源管理的中枢，如流程图上所示，当下载模块将配置文件拉取之后，转交给匹配`Map`中，在`Map`中做新旧配置文件的对比。

步骤如下：
```
0.获取到最新配置文件，储存一份到APP，并保存一份全局变量供查找使用。
1.对比新老配置文件，找到不在新配置文件中的modul，整个module删除。
2.遍历新配置文件，查找是否存在旧配置文件中，
··如果已经存在则不需要更新。
··如果不存在module的新版本
····将module的新版本加入到下载队列。
······如果已经存储的module下只有一个版本，符合最多保存两个版本的规则，不做处理。
······如果已经存在两个版本，则对两个版本做比较，删除最小的版本。
3.更新合并之后的配置信息，存储到APP中。
```

##### 查找

```
"moduleA": {
        "h5_modul_id": "12345765432",
        "h5_resource_name": "serviceName",
        "h5_resource_version": "1.1",
        "h5_resource_download_url": "http://cdn.XXX.com/client/html.zip",
        "h5_resource_route_uri": "Test://hybrid/moduleA",
        "h5_resource_index": "/modules/test/test.html"
    }
```
```
h5_modul_id：资源文件唯一id，可以用作埋点
h5_resource_name：功能名称
h5_resource_version：版本号
h5_resource_download_url：资源下载地址
h5_resource_route_uri：用作路由
h5_resource_index：资源中的入口地址
```
```objc
// 查找
__weak typeof(self) weakSelf = self;
    [[DHSourseMap shareSourseMap] fileWithRoute:@"Test://hybrid/moduleA" callBack:^(NSURL * _Nullable fileUrl, NSError * _Nullable error) {
        NSLog(@"hybrid文件 FileUrl %@ if error %@", fileUrl, error);
        if (!error) {
            [weakSelf.webView loadFileURL:fileUrl allowingReadAccessToURL:fileUrl];
        }
    }];
```
根据`Test://hybrid/moduleA`中的moduleA，在配置文件中找到相应的moduleA，这样在文件管理中拼接处文件路径，
`hybrid/moduleA/1.1/modules/test/test.html`

文件管理查找到有这么文件，则回调给APP，APP加载`fileUrl`
如果没找到，则通过KVC告知下载模块系在，下载完成并解压后，再回调给APP，APP加载`fileUrl`
```
[weakSelf.webView loadFileURL:fileUrl allowingReadAccessToURL:fileUrl];
```

如果有更好的思路或者错误欢迎指正、交流。

