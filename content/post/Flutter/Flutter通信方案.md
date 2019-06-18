---
title: "Flutter通信方案"
date: 2019-06-18T14:50:23+08:00
lastmod: 2019-06-18T20:5201+08:00
draft: false
tags: ["Flutter"]
categories: ["Flutter"]
---

上一篇讲了简单实践了Flutter 的几种channel互发消息的过程，直接使用感觉不太方便，发送一条消息，Flutter、iOS、Android要写三套代码，可否进行一些封装，让使用起来更方便呐？

首先想到的是集中处理消息，接收的入口统一，类似于路由做统跳。
其次收发消息和网络层的处理过程有些相似，想到的是采用C/S架构，比如Flutter通过channel给NA发送消息或者调用方法，这时Flutter扮演C端角色，NA扮演S端，NA处理完之后再通过channel回调给Flutter。思路如下
![Flutter-思路.png](https://upload-images.jianshu.io/upload_images/273788-3e2c38dc75cfd677.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

发送消息或者调用方法根据业务可以根据实际需要分散在不同的地方，接收消息在channel里统一处理。

再进一步思考，一个channel能满足所有需求吗，业务刚开始的时候应该问题不大，随着业务量的增加，路由跳转、APNS推送之类的事件，要不要和业务上的通信分开来处理呐，一个channel处理所有通信会不会太臃肿，这样考虑多channel的需求是有的。

那么如何方便的创建第二个、第三个channel呐？创建channel的流程是一样的，相同channel类型的不同的channel唯一的区别是标识的name不同。
```
// Creates a [MethodChannel] with the specified [name]
const MethodChannel(this.name, [this.codec = const StandardMethodCodec()]);
```
设想可否通过创建一个channel的管理中心呐，来管理所有的call和Handler。带着这些设想来进行调研，看了咸鱼的通信方案，想法不谋而合。

基于这个思路之前的框架图就演变为：
![Flutter-通信思路2.png](https://upload-images.jianshu.io/upload_images/273788-f14d749c30c74118.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

中间多了一层Gateway，类似于我们的Hybrid通信方案，只不过这里区分的不是Hybrid消息的唯一Id，而是channel的`specified [name]`。每个channel里都有接收和发送的逻辑，Gateway的作用便是将`specified [name]`和 收发逻辑对应起来。channel层不再实际创建具体的channel，只需关注收发，Gateway负责创建channel及 收发逻辑的管理。

将上面的思路进一步完善，时序图如下：
![Flutter-通信时序图.png](https://upload-images.jianshu.io/upload_images/273788-ac9413c7ec89286d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

根据Flutter channel类型的不同，具体使用也会有些差异，比如MethodChannel 和 EventChannel，总的思路不变。

上面说完Channel的思路，那么找到具体Channel之后分发怎么处理呐？也就是时序图里面的`oncallDispatch`，调用一个方法，需要知道方法名和参数，类似于网络层的C/S结构，并且考虑到和路由扩展Channel侧接口大致设计如下：
iOS
```objc
// 发送
+ (void)MessageToFlutter:(void (^)(NSDictionary *))resultCallback url:(NSString *)url params:(NSDictionary *)params;

// 接收
- (void)onCall:(void (^)(NSDictionary *))result url:(NSString *)url params:(NSDictionary *)params {
   // 根据url及params进行分发
    NSLog(@"NA收到flutter method call：url %@ params %@", url, params);
    result(@{@"key2":@"onCall NA"});
}
```
Android侧
```java
// 接收
private boolean onCall(MessageResult<Map> result, String url, Map params) {
    // 根据url及params进行分发
    System.out.println("flutter to na onCall:url" + url + "   params:" + params);
    Map<String, String> resultMap = new HashMap<>();
    resultMap.put("a", "a");
    result.success(resultMap);
    return true;
}

// 发送
public static void MessageToFlutter(final MessageResult<Map> result, String url,Map params){});
```
Flutter侧
```dart
// 接收
Future<Map> onCall(String url,Map params) async{
  // 根据url及params进行分发
  print('onCall Flutter: $url  $params');
  return {'key3':'value3'};
}

// 发送
static Future<Map> MessageToNative(String url,Map params,[onInvocationException onException]) {});
```
以上都是基于Method Channel来做的处理，至于EventChannel类型，iOS基于代理、Android基于接口来实现，可以做到哪里用在那里写，不需要通过oncallDispatch进行分发即可方便使用，如果有事件流的场景可以考虑。

至于Gateway侧，基于上面的调研结果，方案可以采用咸鱼团队的`xservice_kit`。

甚至于channel侧创建的代码也可以通过工具来生成。约定好`参数类型`、`方法名`、`channelName`即可。下一篇文章会介绍`xservice_kit`的使用，及使用npm工具生成channel侧的代码。



