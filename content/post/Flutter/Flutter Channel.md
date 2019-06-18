---
title: "Flutter Channel"
date: 2019-06-13T14:50:23+08:00
lastmod: 2019-06-13T20:5201+08:00
draft: false
tags: ["Flutter"]
categories: ["Flutter"]
---

Flutter定义了三种不同类型的Channel，它们分别是
	▪	BasicMessageChannel：用于传递字符串和半结构化的信息。
	▪	MethodChannel：用于传递方法调用（method invocation）。
	▪	EventChannel: 用于数据流（event streams）的通信。
	
	
原理讲解咸鱼团队的技术文章讲解的很通透了： [深入理解Flutter Platform Channel](https://mp.weixin.qq.com/s/FT7UFbee1AtxmKt3iJgvyg)
这篇文章在官方`platform_channel`的示例基础上，补全了示例子中没有实践的种方式。


### Fultter侧实现
#### BasicMessageChannel
1 创建messageChannel
```dart
static const BasicMessageChannel messageChannel =
BasicMessageChannel('samples.flutter.io/message',  StandardMessageCodec());
```

1.1 发送数据到NA `flutter -> NA`
```dart
// result接收异步回调的数据
String result = await messageChannel.send('0987654321');
```

1.2 接收NA发过来的数据 `NA -> flutter`
```dart
Future<String> naText(parama) async {
  print('log ' + parama);
  _naToFlutterText = parama;
  
  // 回调结果给NA
  return '$parama + 321';
}

// void setMessageHandler(Future<T> handler(T message)) 
messageChannel.setMessageHandler(naText);
```
#### MethodChannel
2 创建MethodChannel
```dart
static const MethodChannel methodChannel =
    MethodChannel('samples.flutter.io/battery');
```
2.1 flutter调用NA方法 
```dart
try {
  final int result = await methodChannel.invokeMethod('getBatteryLevel');
  // 处理成功结果
} on PlatformException {
  // 处理异常结果
}
```
2.2 NA调用flutter方法 `NA -> flutter`
```dart
Future<String> methodHandler (MethodCall call) async {
  print(call.method);
  print(call.arguments);
  
  return '回调结果';
}

// void setMethodCallHandler(Future<dynamic> handler(MethodCall call)) 
methodChannel.setMethodCallHandler(methodHandler);
```

#### EventChannel
3 创建EventChannel
```dart
static const EventChannel eventChannel =
    EventChannel('samples.flutter.io/charging');
```
3.1 接收NA发送过来的stream `NA -> flutter`
```dart
// Stream<dynamic> receiveBroadcastStream([ dynamic arguments ])
// StreamSubscription<T> listen(void onData(T event), {Function onError, void onDone(), bool cancelOnError});

eventChannel.receiveBroadcastStream().listen(_onEvent, onError: _onError);

void _onEvent(Object event) {
  // 正常数据流
}

void _onError(Object error) {
  // 失败数据流
}
```

### iOS侧实现

1 创建channel
```objc
// 创建massageChannel
    FlutterBasicMessageChannel *massageChannel = [FlutterBasicMessageChannel messageChannelWithName:@"samples.flutter.io/message" binaryMessenger:controller];
    
    // 创建methodChannel
    FlutterMethodChannel *methodChannel = [FlutterMethodChannel
    methodChannelWithName:@"samples.flutter.io/battery" binaryMessenger:controller];
    
    // 创建eventChannel 代理方法里处理stream流 delegate设置为self
    FlutterEventChannel *eventChannel = [FlutterEventChannel eventChannelWithName:@"samples.flutter.io/charging" binaryMessenger:controller];
    [eventChannel setStreamHandler:self];
```

2 接收flutter发送过来的message和method `flutter -> NA`
```objc
	// 接收flutter发过来的message
		[self.massageChannel setMessageHandler:^(id  _Nullable message, FlutterReply  _Nonnull callback) {
        NSLog(@"receive flutter message0 %@ ", message);
        callback([NSString stringWithFormat:@"%@ 12349876543211111", message]);
    }];
    
    
    // 接收flutter调用的方法
    __weak typeof(self) weakSelf = self;
    [methodChannel setMethodCallHandler:^(FlutterMethodCall* call,
                                          FlutterResult result) {
        
        if ([@"getBatteryLevel" isEqualToString:call.method]) {
            int batteryLevel = [weakSelf getBatteryLevel];
            if (batteryLevel == -1) {
                result([FlutterError errorWithCode:@"UNAVAILABLE"
                                           message:@"Battery info unavailable"
                                           details:nil]);
            } else {
                result(@(batteryLevel));
            }
        } else {
          result(FlutterMethodNotImplemented);
        }
    }];
```

3 NA发送message和method到flutter `NA -> flutter`
```objc
		//  na to flutter 发送普通消息
    [self.massageChannel sendMessage:@"na message 参数" reply:^(id  _Nullable reply) {
        NSLog(@"na->flutter message callback %@ ", reply);
    }];
    
    //  na to flutter 调用flutter方法
    [self.methodChannel invokeMethod:@"naCallFlutter" arguments:@"na method 参数" result:^(id  _Nullable result) {
        NSLog(@"na->flutter method callback %@ ", result);
    }];
```
### Android侧实现
1 创建channel
```java
  // 创建massageChannel
     BasicMessageChannel channel = new BasicMessageChannel(registrar.messenger(), "flutter_message_plugin", StringCodec.INSTANCE );
      channel.setMessageHandler(new BasicMessageChannel.MessageHandler() {
      @java.lang.Override
      public void onMessage(java.lang.Object object, BasicMessageChannel.Reply reply) {  //flutter发送消息到平台测
        Log.i("BasicMessageChannel", "接收到Flutter的消息:" + object);
      }
    });
    channel.send("发送消息到Flutter");
  // 创建methodChannel
        new MethodChannel(getFlutterView(), "samples.flutter.io/battery").setMethodCallHandler(
          // 直接 new MethodChannel，然后设置一个Callback来处理Flutter端调用
                new MethodCallHandler() {
                    @Override
                    public void onMethodCall(MethodCall call, Result result) {
                        // 在这个回调里处理从Flutter来的调用
                        if (call.method.equals("getBatteryLevel")) {
                            int batteryLevel = getBatteryLevel();
                            //result是给Flutter的返回值
                            if (batteryLevel != -1) {
                                result.success(batteryLevel);
                            } else {
                                result.error("UNAVAILABLE", "Battery level not available.", null);
                            }     
                          }
                        } else {
                          result.notImplemented();
                        }
                    }
                });
    
    // 创建eventChannel 代理方法里处理stream流 delegate设置为self
    new EventChannel(getFlutterView(), "samples.flutter.io/charging").setStreamHandler(
        new StreamHandler() {
          // 接收电池广播的BroadcastReceiver。
          private BroadcastReceiver chargingStateChangeReceiver;
          @Override
         // 这个onListen是Flutter端开始监听这个channel时的回调，第二个参数 EventSink是用来传数据的载体。
          public void onListen(Object arguments, EventSink events) {
            chargingStateChangeReceiver = createChargingStateChangeReceiver(events);
            registerReceiver(
                chargingStateChangeReceiver, new IntentFilter(Intent.ACTION_BATTERY_CHANGED));
          }

          @Override
          public void onCancel(Object arguments) {
            // 对面不再接收
            unregisterReceiver(chargingStateChangeReceiver);
            chargingStateChangeReceiver = null;
          }
        }
    );
```


