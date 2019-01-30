---
title: "Hybrid-Native Birdge"
date: 2019-01-29T14:50:23+08:00
lastmod: 2019-01-29T20:5201+08:00
draft: false
tags: ["Hybrid"]
categories: ["iOS"]
---


Hybird-JS侧的逻辑参考这里[同事简书](https://www.jianshu.com/p/0f46941d55ef)，以后有时间自己再完善一下。

iOS侧Native Bridge基于WKWebView开发，虽然messagehandler 可以直接处理JS发送过来的对象，但考虑到iOS、Android的数据统一性，这里还是在JS侧将要发送的消息转化为`JSON.stringify`，Native收到消息后，再解析即可。

先上流程图：
![Hybird Native.png](https://upload-images.jianshu.io/upload_images/273788-792040e8a2d2b98a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

进行简要说明，主要分为两大部分：
```
NativeBridge：处理JS消息接收与Native执行结果回调。
Plugin：Native plugin，执行相应Native能力。
```
`dhjsbridge.js`是封装在Native侧的，并不直接提供给JS使用，在webView初始化时注入。`WKScriptMessageHandler`的代理方法在webView初始化时也代理给了`Native Bridge`统一处理，下面会介绍消息处理过程。

JS给Native发消息之后在Native 侧收到的消息格式如下：![msg数据格式.png](https://upload-images.jianshu.io/upload_images/273788-dfbe18925a5778a8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

收到消息后Bridge按照约定格式将消息进行处理，
```objc
- (instancetype)initWithDictionary:(NSDictionary *)dict
{
    self = [super init];
    if (self && [dict isKindOfClass:[NSDictionary class]]) {
        NSDictionary *data = dict[@"data"];
        if (data) {
            NSArray *plugin = [dict[@"data"][@"plugin"] componentsSeparatedByString:@"."];
            if (plugin.count == 2) {
                _handler = plugin[0];
                _action = plugin[1];
            }
        }
        _params = dict[@"params"];
        _callbackId = dict[@"callbackId"];
        _callbackFunction = @"window.handleMessageFromNative"; //dict[@"callbackFunction"];
    }
    return self;
}
```
根据`handler`和`action`在`self.handlerMap`找到对应的`plugin`，self.handlerMap是保存`plugin`能力的一个字典，plugin插件装载的时候会保存在这里。
保存过程如下：
```objc
// 注册Native能力
- (void)registerHandler:(NSString *)handlerName action:(NSString *)actionName handler:(HandlerBlock)handler
{
    if (handlerName && actionName && handler) {
        NSMutableDictionary *handlerDic = [self.handlerMap objectForKey:handlerName];
        if (!handlerDic) {
            handlerDic = [[NSMutableDictionary alloc] init];
        }
        // 注册时设置handlerMap 及保存 handler
        [self.handlerMap setObject:handlerDic forKey:handlerName];
        [handlerDic setObject:handler forKey:actionName];
    }
}
```
可以看到根据模块名handlerName存储一个字典到handlerMap里，里面存放着模块下对应能力的回调函数HandlerBlock，以方法名做key。

这样当JS消息处理完成之后：
```objc
DWKJSBridgeMessage *msg = [[DWKJSBridgeMessage alloc] initWithDictionary:msgBody];
NSDictionary *handlerDic = [self.handlerMap objectForKey:msg.handler];
HandlerBlock handler = [handlerDic objectForKey:msg.action];
```
这样就和刚才`plugin`注册的时候存储的`handler`一一对应了，如果查找不到的话，将消息准发给Native处理。

找到plugin之后，先别着急将消息回调给plugin处理，在此之前还需判断JS是否需要回掉，JS发过来的消息中callbackId还没使用，这个id是回调Native处理结果给JS的凭证。
```objc
if (msg.callbackFunction && msg.callbackId) {
    // 生成OC的回调block 异步执行
    __weak typeof(self) weakSelf = self;
    JSResponseCallback callback = ^(id responseData){
         // 执行OC 主动 Call JS 的编码与通信
         [weakSelf injectMessageFuction:msg.callbackFunction callbackId:msg.callbackId withParams:responseData withCallback:nil];
    };
    [msg setCallback:callback];
}
```
生成回掉函数存储到msg中去，之后将msg作为参数回掉给plugin处理，这样plugin就有了相应的参数及回调函数了
```
handler(msg);
```

如果有回调即msg中存在callback回掉，则执行完成之后，将结果返回。
```
[msg callback:result];
```

和上面解析JS发过来的消息一样，回掉消息也是在同一个消息处理中心进行转换，转换如下：
```objc
 Native执行完能力之后调用此方法，回调数据给JS
 
 @param result 用户回传的数据
 @param status 是否调用成功
 @param complete 默认为1 不需要连续回调，0代表需要连续回调，
 @param errorMessage 如果失败，失败原因
 
NSDictionary *message = @{@"status":@(status),
                          @"complete":@(complete),
                          @"errorMessage":errorMessage,
                          @"data":result
                          };
self.callback(message);
```
callback触发执行
```
JSResponseCallback callback = ^(id responseData) {
     // 执行OC 主动 Call JS 的编码与通信
     [weakSelf injectMessageFuction:msg.callbackFunction callbackId:msg.callbackId withParams:responseData withCallback:nil];
};
```
```objc
// 主线程执行evaluateJavaScript:
- (void)injectMessageFuction:(NSString *)action callbackId:(NSString *)callbackId withParams:(NSDictionary *)message withCallback:(void (^)(_Nullable id, NSError * _Nullable error))handler
{
    if (!message) {
        message = @{};
    }
    
    NSMutableDictionary *response = [message mutableCopy];
    [response setObject:callbackId forKey:@"callbackId"];
    
    NSString *paramsString = [DWKUtility serializeMessageData:response];
    NSString *paramsJSString = [DWKUtility transcodingJavascriptMessage:paramsString];
    NSString* javascriptCommand = [NSString stringWithFormat:@"%@('%@');", action, paramsJSString];
    if ([[NSThread currentThread] isMainThread]) {
        [self.webView evaluateJavaScript:javascriptCommand completionHandler:handler];
    } else {
        __strong typeof(self)strongSelf = self;
        dispatch_sync(dispatch_get_main_queue(), ^{
            [strongSelf.webView evaluateJavaScript:javascriptCommand completionHandler:handler];
        });
    }
}
```
由于语言特性，还需要对结果进行转码，这里不再详述这个过程可以[看这里](http://awhisper.github.io/2018/03/06/hybrid-webcontainer/#.WwqaY1vZcZo.sinaweibo)
至此，JS调用Native完成。

还有一点需要说明的是，加载plugin过程，写了一个基类，里面只有一个方法
```objc
@interface DWKPlugin : NSObjec
- (void)registPlugin;
@end
```

需要开发plugin的话，继承这个基类即可，重写`registPlugin`，并且需要将plugin信息写到一个`DPluginRegisterFile.json`文件中，如下：
```objc
- (void)registPlugin
{
    [[DWKJSBridge shareJSBridge] registerHandler:@"camera" action:@"getImage" handler:^(DWKJSBridgeMessage * _Nonnull msg) {
        // Native 执行任务
        [msg callback:@{@"imageData":result} status:1 complete:1 errorMessage:nil];
    }];
}
```
```json
{
    "DWKCameraPlugin": {
        "handler": "camera",
        "actions": ["getImage"]
    },
    "DWKCommonPlugin": {
        "handler": "common",
        "actions": [
            "commonAsyncParams",
            "commonSyncParams",
            "nativeLog"
        ]
    }
}
```
这样在初始化的时候Native birdge的时候，通过反射机制注册响应能力
```objc
- (void)registerHandler
{
    NSString *path = [[NSBundle mainBundle] pathForResource:@"DPluginRegisterFile" ofType:@"json"];
    
    NSData *data = [NSData dataWithContentsOfFile:path];
    if (data) {
        NSError *error = nil;
        NSDictionary *json = [NSJSONSerialization JSONObjectWithData:data
                                                             options:NSJSONReadingAllowFragments
                                                               error:&error];
        NSMutableDictionary *handlerMap = [NSMutableDictionary new];
        [handlerMap addEntriesFromDictionary:json];
        if (error) {
            NSLog(@"error %@ ", error);
        }
        for (NSString *plugin in handlerMap) {
            id obj = [NSClassFromString(plugin) new];
            SEL selector = NSSelectorFromString(@"registPlugin");
            if ([obj respondsToSelector:selector]) {
                [obj performSelector:selector withObject:nil afterDelay:0];
            }
        }
    }
}
```



