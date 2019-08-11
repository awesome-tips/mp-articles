## 现代化的传输 API

说起 `Socket` ，我回头望了一眼书架上厚厚的 `UNIX 网络编程 卷1: 套接字联网 API（第 3 版)` ，而她的姊妹进程间通信我连塑封膜都没拆开。的确，这套最早来自 BSD 的 API 很让人头疼。虽然她们依然是跨平台程序的最佳选择，但是我想应该没有哪个小伙伴在项目中会有勇气从这些 API 开始构筑，至少是 `CFNetwork` 或者 `NSNetwork` 中的现成接口。更一般性的是选一些面向对象的第三方库，比如老牌的 `CocoaAsyncSocket`。当然作为 Swift 老法师我也会推荐你看看 IBM 出品的 `BlueSocket`。

Socket 编程有很多需要解决的问题，最重要的 3 个大问题，以及更多的细节问题：

* 建立连接

![1](http://)

* 数据传输

![2](http://)

* 连接的变动

![3](http://)

当前，`URLSession` 底层就是使用 `Network.framework` 完成基础连接的。特地查了一下，相关私有 API 是从 iOS 9 开始存在的。
在未来，Apple 希望你能够将原来的 Socket API 全部替换为全新的 `Network.framework`。（iOS 又有人要了！）

## Network.framework 的特点

* 智能建立连接
* 经优化的数据传输
* 内建的安全加密
* 无缝兼容移动网络
* 原生 Swift 支持🔥🔥🔥

## 开始你的第一次连接

Socket 主要使用的三种场景：游戏联机、流式视频传输、在线聊天。

### 使用传统 Socket 建立连接

* 使用 getaddrinfo() 查询 DNS
* 使用正确的地址族去调用 socket()
* 使用 setsockopt() 设置 socket 选项
* 调用 connect() 开始 TCP 连接
* 等待直到一个可写入的事件回调

### 使用 Network.framework 建立连接

* 使用 NWEndPoint 与 NWParameters 创建连接
* 调用 connection.start()
* 等待连接进入 .ready 的状态

![4](http://)

对就是这么简单，完全的原生 Swift 支持，又面向对象，又支持闭包。这样的接口，你不心动么？

### 连接的生命周期

在连接设置完毕以后，就会进入 准备 状态。而针对移动设备复杂的网络状态，你需要更加智能的建立连接。

![5](http://)

而使用 Network.framework ，你可以十分简单的对网络路径进行配置，比如下面的例子中，指定了仅使用蜂窝网络、使用 IPv6 协议、与禁止代理。都仅是一行命令就完成了。特别当你需要为特定连接指定连接方式时，这个框架能极大提高你的效率。

![6](http://)

在准备完毕以后，连接可能进入 **等待** 、**就绪** 或 **失败** 状态。当然在你取消连接时也会进入 **取消** 状态。

![7](http://)

### 案例：流式视频传输

该案例使用 UDP 进行视频的实时传输，出于简化考虑，并未对视频帧做任何编码，直接把裸数据封包，并通过 UDP 传输。在接收端，解包数据并重新封装为视频帧，直接进行播放。案例中也使用了 Bonjour 服务来进行快速设备配对连接。

![8](http://)

在监听端的代码异常简单，甚至连 `Bonjour` 服务也已经整合好了。你要做的仅仅是指定 `.udp` 并指定正确的 Bonjour 服务名称。

![9](http://)

### 最佳的数据传输方式

**数据的发送与接收**

单帧发送

```objc
// Send a single frame
func sendFrame(_ connection: NWConnection, frame: Data) {
    // The .contentProcessed completion provides sender-side back-pressure
    connection.send(content: frame, completion: .contentProcessed { (sendError) in
        if let sendError = sendError {
            // Handle error in sending
        } else {
            // Send has been processed, send the next frame
            let nextFrame = generateNextFrame()
            sendFrame(connection, frame: nextFrame)
        }
    })
}
```

使用 batch 发送多个数据报

```objc
// Hint that multiple datagrams should be sent as one batch
connection.batch {
    for datagram in datagramArray {
        connection.send(content: datagramArray, completion: .contentProcessed { (error) in
            // Handle error in sending
        }
    })
}
```

在接收时，提供了方便的方法来读取消息头

```objc
// Read one header from the connection
func readHeader(connection: NWConnection) {
    // Read exactly the length of the header
    let headerLength: Int = 10
    connection.receive(minimumIncompleteLength: headerLength, maximumLength: headerLength) { (content, contentContext, isComplete, error) in
        if let error = error {
            // Handle error in reading
        } else {
         // Parse out body length
        readBody(connection, bodyLength: bodyLength)
        }
    }
}
// Follow the same pattern as readHeader() to read exactly the body length
func readBody(_ connection: NWConnection, bodyLength: Int) { ... }
```

### 高级选项

**显式拥塞通知(Explicit Congestion Notification)**

在所有 TCP 连接中 ECN 是默认开启的。

在 UDP 连接中为每个数据包标记 ECN 的方法：

```objc
let ipMetadata = NWProtocolIP.Metadata() 
ipMetadata.ecn = .ect0
let context = NWConnection.ContentContext(identifier: "ECN", metadata: [ ipMetadata ])
connection.send(content: datagram, contentContext: context, completion: .contentProcessed{..})
```

**服务等级（网络队列优先级）**

为整个连接更改服务等级

```objc
let parameters = NWParameters.tls 
parameters.serviceClass = .background
```

为每个 UDP 数据包更改服务等级

```objc
let ipMetadata = NWProtocolIP.Metadata() 
ipMetadata.serviceClass = .signaling
let context = NWConnection.ContentContext(identifier: "Signaling", metadata: [ ipMetadata ])
connection.send(content: datagram, contentContext: context, completion: .contentProcessed{..})
```

**快速连接（Fast Open Connections）**

允许在连接上快速打开需要发送幂等数据

```objc
parameters.allowFastOpen = true
let connection = NWConnection(to: endpoint, using: parameters)
connection.send(content: initialData, completion: .idempotent) 
connection.start(queue: myQueue)
```

可以手动启用 TCP Fast Open 以通过 TFO 运行 TLS

```objc
let tcpOptions = NWProtocolTCP.Options() 
tcpOptions.enableFastOpen = true
```

允许失效的 DNS 查询结果

主动使用失效的 DNS 查询结果

```objc
parameters.expiredDNSBehavior = .allow
let connection = NWConnection(to: endpoint, using: parameters)
connection.start(queue: myQueue)
```

新的 DNS 查询会同步进行

## 处理网络连接的变动

### 开始连接

* .waiting 状态暗示连接还未建立
* 避免在网络连接开始前检查可用性
* 在需要时在 NWParameters 限制连接类型

### 处理网络连接状态的变化

主要是两个状态，一个是 `isViable` 当前连接是否可用，一个是 `betterPathAvailable` 是否有更佳的连接路径。她们也都提供了相应的闭包来处理

```objc
// Handle connection viability
connection.viabilityUpdateHandler = { (isViable) in
    if (!isViable) {
        // Handle connection temporarily losing connectivity
    } else {
        // Handle connection return to connectivity
    }
}

// Handle better paths
connection.betterPathUpdateHandler = { (betterPathAvailable) in
    if (betterPathAvailable) {
        // Start a new connection if migration is possible
    } else {
        // Stop any attempts to migrate
    }
}
```

## 开始实践

### 应避免的做法

![10](http://)

### 不应继续使用的接口

CoreFoundation 中 `CFStream` 绑定的相关方法及 `CFSocket`

![11](http://)

`Foundation` 中与 `NSStream` 绑定、`NSNetService` 监听、`NSSocketPort` 以及 `SystemConfiguration` 中的 `SCNetworkReachability`。

### 推荐的接口

![12](http://)

当然是 `URLSession` 和 `Network.framework`。

推荐阅读

使用 iOS 12 的 Network Framework 实现 netcat
Core ML & Vision 入门教程
iOS 任务调度器：为 CPU 和内存减负

