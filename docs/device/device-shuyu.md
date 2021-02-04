---
title: 基础术语定义
nav:
  title:   设备管理
  order: 1
---

# 常见名词(功能)说明

**产品**: 对某一型设备的分类,通常是已经存在的某一个设备型号.

**设备**: 具体的某一个设备.

**网络组件**: 用于管理各种网络服务(MQTT,TCP等),动态配置,启停. 只负责接收,发送报文,不负责任何处理逻辑。

**协议**: 用于自定义消息解析规则,用于认证、将设备发送给平台报文解析为平台统一的报文，以及处理平台下发给设备的指令。

**设备网关**: 负责平台侧统一的设备接入,使用网络组件处理对应的请求以及报文,使用配置的协议解析为平台统一的设备消息(`DeviceMessage`),然后推送到事件总线。

**事件总线**: 基于topic和事件驱动,负责进程内的数据转发.(设备告警,规则引擎处理设备数据都是使用事件总线订阅设备消息)。
## 平台统一设备消息定义

平台使用自定义的协议包将设备上报的报文解析为平台统一的消息,来进行统一管理。

平台统一消息基本于物模型中的定义相同,主要由`属性(property)`,`功能(function)`,`事件(event)`组成.

### 消息组成

消息主要由`deviceId`,`messageId`,`headers`,`timestamp`组成.

`deviceId`为设备的唯一标识,`messageId`为消息的唯一标识,`headers`为消息头,通常用于对自定义消息处理的行为,如是否异步消息, 是否分片消息等.

常用的:

1.  **async** 是否异步,boolean类型.
2.  **timeout** 指定超时时间. 毫秒.
3.  **frag\_msg\_id** 分片主消息ID,为下发消息的`messageId`
4.  **frag\_num** 分片总数
5.  **frag\_part** 当前分片索引
6.  **frag\_last** 是否为最后一个分片,当无法确定分片数量的时候,可以将分片设置到足够大,最后一个分片设置:`frag_last=true`来完成返回.
7.  **keepOnline** 与`DeviceOnlineMessage`配合使用,在TCP短链接,保持设备一直在线状态,连接断开不会设置设备离线.
8.  **keepOnlineTimeoutSeconds** 指定在线超时时间,在短链接时,如果超过此间隔没有收到消息则认为设备离线.
~~~
TIP

messageId通常由平台自动生成,如果设备不支持消息id,可在自定义协议中通过Map的方式来做映射,将设备返回的消息与平台的messageId进行绑定.
~~~
### 属性相关消息

1.  `获取设备属性(ReadPropertyMessage)`对应设备回复的消息`ReadPropertyMessageReply`.
2.  `修改设备属性(WritePropertyMessage)`对应设备回复的消息`WritePropertyMessageReply`.
3.  `设备上报属性(ReportPropertyMessage)` 由设备上报.

注意

设备回复的消息是通过messageId进行绑定,messageId应该注意要全局唯一,如果设备无法做到,可以在编解码时通过添加前缀等方式实现.

消息定义:

```
ReadPropertyMessage{
    Map<String,Object> headers;
    String deviceId; 
    String messageId;
    long timestamp; //时间戳(毫秒)
    List<String> properties;//可读取多个属性
}

ReadPropertyMessageReply{
    Map<String,Object> headers;
    String deviceId;
    String messageId;
    long timestamp; //时间戳(毫秒)
    boolean success;
    Map<String,Object> properties;//属性键值对
}
```

```
WritePropertyMessage{
    Map<String,Object> headers;
    String deviceId; 
    String messageId;
    long timestamp; //时间戳(毫秒)
    Map<String,Object> properties;
}

WritePropertyMessageReply{
    Map<String,Object> headers;
    String deviceId;
    String messageId;
    long timestamp; //时间戳(毫秒)
    boolean success;
    Map<String,Object> properties; //回复被修改的属性最新值
}
```

```
ReportPropertyMessage{
    Map<String,Object> headers;
    String deviceId;
    String messageId;
    long timestamp; //时间戳(毫秒)
    Map<String,Object> properties;
}
```

### 功能相关消息

调用设备功能到消息(`FunctionInvokeMessage`)由平台发往设备,对应到返回消息`FunctionInvokeMessageReply`.

消息定义:

```
FunctionInvokeMessage{
    Map<String,Object> headers;
    String functionId;//功能标识,在元数据中定义.
    String deviceId;
    String messageId;
    long timestamp; //时间戳(毫秒)
    List<FunctionParameter> inputs;//输入参数
}

FunctionParameter{
    String name;
    Object value;
}

FunctionInvokeMessageReply{
    Map<String,Object> headers;
    String deviceId;
    String messageId;
    long timestamp;
    boolean success;
    Object output; //输出值,需要与元数据定义中的类型一致
}
```

###  事件消息

事件消息`EventMessage`由设备端发往平台.

消息定义:

```
EventMessage{
    Map<String,Object> headers;
    String event; //事件标识,在元数据中定义
    Object data;  //与元数据中定义的类型一致,如果是对象类型,请转为java.util.HashMap,禁止使用自定义类型.
    long timestamp; //时间戳(毫秒)
}
```

###  其他消息

1.  `DeviceOnlineMessage` 设备上线消息,通常用于网关代理的子设备的上线操作.
2.  `DeviceOfflineMessage` 设备上线消息,通常用于网关代理的子设备的下线操作.
3.  `ChildrenDeviceMessage` 子设备消息,通常用于网关代理的子设备的消息.
4.  `ChildrenDeviceMessageReply` 子设备消息回复,用于平台向网关代理的子设备发送消息后设备回复给平台的结果.

消息定义:

```
DeviceOnlineMessage{
    String deviceId;
    long timestamp;
}

DeviceOfflineMessage{
    String deviceId;
    long timestamp;
}

```

```
ChildDeviceMessage{
    String deviceId;
    String childDeviceId;
    Message childDeviceMessage; //子设备消息
}
```

父子设备消息处理[请看这里](http://doc.jetlinks.cn/best-practices/device-gateway-connection.html)

### 设备消息对应事件总线topic

协议包将设备上报后的报文解析为平台统一的设备消息后,会将消息转换为对应的topic 并发送到事件总线,可以通过从事件总线订阅消息来处理这些消息。

所有设备消息的`topic`的前缀均为: `/device/{productId}/{deviceId}`.

如:产品`product-1`下的设备`device-1`上线消息: `/device/product-1/device-1/online`.

可通过通配符订阅所有设备的指定消息,如:`/device/*/*/online`,或者订阅所有消息:`/device/**`.

TIP
~~~
1.  此topic和mqtt的topic没有任何关系,仅仅作为系统内部通知的方式
2.  使用通配符订阅可能将收到大量的消息,请保证消息的处理速度,否则会影响系统消息吞吐量.
~~~

<table><thead><tr><th>topic</th> <th>类型</th> <th>说明</th></tr></thead> <tbody><tr><td>/online</td> <td>DeviceOnlineMessage</td> <td>设备上线</td></tr> <tr><td>/offline</td> <td>DeviceOfflineMessage</td> <td>设备离线</td></tr> <tr><td>/message/event/{eventId}</td> <td>DeviceEventMessage</td> <td>设备事件</td></tr> <tr><td>/message/property/report</td> <td>ReportPropertyMessage</td> <td>设备上报属性</td></tr> <tr><td>/message/send/property/read</td> <td>ReadPropertyMessage</td> <td>平台下发读取消息指令</td></tr> <tr><td>/message/send/property/write</td> <td>WritePropertyMessage</td> <td>平台下发修改消息指令</td></tr> <tr><td>/message/property/read/reply</td> <td>ReadPropertyMessageReply</td> <td>读取属性回复</td></tr> <tr><td>/message/property/write/reply</td> <td>WritePropertyMessageReply</td> <td>修改属性回复</td></tr> <tr><td>/message/send/function</td> <td>FunctionInvokeMessage</td> <td>平台下发功能调用</td></tr> <tr><td>/message/function/reply</td> <td>FunctionInvokeMessageReply</td> <td>调用功能回复</td></tr> <tr><td>/register</td> <td>DeviceRegisterMessage</td> <td>设备注册,通常与子设备消息配合使用</td></tr> <tr><td>/unregister</td> <td>DeviceUnRegisterMessage</td> <td>设备注销,同上</td></tr> <tr><td>/message/children/{childrenDeviceId}/{topic}</td> <td>ChildDeviceMessage</td> <td>子设备消息,{topic}为子设备消息对应的topic</td></tr> <tr><td>/message/children/reply/{childrenDeviceId}/{topic}</td> <td>ChildDeviceMessage</td> <td>子设备回复消息,同上</td></tr> <tr><td>/message/direct</td> <td>DirectDeviceMessage</td> <td>透传消息</td></tr> <tr><td>/message/tags/update</td> <td>UpdateTagMessage</td> <td>更新标签消息 since 1.5</td></tr> <tr><td>/firmware/pull</td> <td>RequestFirmwareMessage</td> <td>拉取固件请求 (设备-&gt;平台)</td></tr> <tr><td>/firmware/pull/reply</td> <td>RequestFirmwareMessageReply</td> <td>拉取固件请求回复 (平台-&gt;设备)</td></tr> <tr><td>/firmware/report</td> <td>ReportFirmwareMessage</td> <td>上报固件信息</td></tr> <tr><td>/firmware/progress</td> <td>UpgradeFirmwareProgressMessage</td> <td>上报更新固件进度</td></tr> <tr><td>/firmware/push</td> <td>UpgradeFirmwareMessage</td> <td>推送固件更新</td></tr> <tr><td>/firmware/push/reply</td> <td>UpgradeFirmwareMessageReply</td> <td>固件更新回复</td></tr> <tr><td>/log</td> <td>DeviceLogMessage</td> <td>设备日志</td></tr></tbody></table>
