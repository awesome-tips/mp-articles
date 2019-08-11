| 作者：Ross Butler
| 原文链接：https://medium.com/@rwbutler/nwpathmonitor-the-new-reachability-de101a5a8835

我最近写了一篇文章，来介绍 iOS 在连接新的 Wi-Fi 网络时，如何在弹出一个 web view 以让用户登录或注册之前，检测 `Captive Portals` (强制网络门户)。如果你连接过诸如酒店、酒吧或咖啡店等地的公共 Wi-Fi 网络，对这个应该会比较熟悉。如果你不熟悉 iOS 中 Captive Portals 的工作方式，可以查看 `Solving the Captive Portal Problem on iOS` 这篇文章，以了解一些背景知识。

多年来，Apple 的 `Reachability` 示例程序一直被用作 App 中检测网络访问的基础代码。搜索 `Cocoapods.org` 将会看到一个很长的第三方库列表，这些库基本上都是基于 Reachability，并考虑了 ARC 的支持或 Swift 的兼容等问题。

在 WWDC 2018 上，Apple 介绍了 iOS 12 中的一个新的框架：`Network.framework`，该框架包含了一个 `NWPathMonitor` 类。这个类为我们提供了一种监视网络状态变化的方法，而无需包含第三方库或 Apple 示例代码。

## 使用

只需简单导入 Network 框架，便可以使用 `NWPathMonitor` 类，如下创建一个 `NWPathMonitor` 实例：

```objc
let monitor = NWPathMonitor()
```

如果你只对某个特定网络适配器的状态变更感兴趣，例如 Wi-Fi，则可以使用 `init(requiredInterfaceType:)` 初始化方法，并提供 `NWInterface.InterfaceType` 值作为参数，来实例化 NWPathMonitor 对象，以监听指定类型的网络适配器，例如：

```objc
let monitor = NWPathMonitor(requiredInterfaceType: .wifi)
```

您需要确保在某处保留对 NWPathMonitor 对象的引用（例如使用 strong 属性），否则 ARC 可能会释放 NWPathMonitor 对象，从而导致指定的回调无法被调用。

可监控的网络类型包括：

* cellular
* loopback
* other (对于虚拟或未确定的网络类型)
* wifi
* wiredEthernet

要获取状态更改的通知，需要为 `pathUpdateHandler` 属性指定一个回调，该回调将在网络接口发生状态更改时调用。例如，你的手机网络从蜂窝网络切换到 Wi-Fi 网络。然后，每当发生状态更改时，将返回一个 `NWPath` 实例，可以使用该实例以确定后续的操作，如下代码：

```objc
monitor.pathUpdateHandler = { path in
    if path.status == .satisfied {
        print("Connected")
    }
}
```

使用无参初始化方法与使用指定网络适配器的初始化方法的不同点是：返回的 NWPathobject 对象的 status 属性是否是 satisfied。例如，你只想监听蜂窝网络，而你的手机连接的是 Wi-Fi 网络，则当 Wi-Fi 网络状态发生变化时，并不会调用回调方法，并且 path 的 status 也会保持 unsatisfied 状态，因为手机没有使用指定的网络连接。所以，如果你只想知道是否有网络连接，无论是 Wi-Fi 还是蜂窝，则最好使用无参数的初始化方法。

一个有趣的问题是，NWPath 在 iOS 12 中是作为 Network 框架的一部分，而实际上在 iOS 9 中就有它的身影，不过是在 `NetworkExtension.framework`，两者之间有一些细微差别。

可以查询返回的 NWPath 对象，以查看设备的网络适配器的状态信息。另一个更有趣的属性是 `isExpensive`，它标识网络接口返回的数据收费是否昂贵，如使用蜂窝数据。我们同样可以查询是否支持 DNS、IPv4 或 IPv6。我们可以调用 `usesInterfaceType` 方法，来查看哪个接口改变了状态并触发回调：

```objc
let isCellular: Bool = path.usesInterfaceType(.cellular)
```

使用 NWPathMonitor 有点类似于使用其他 iOS API，例如 CLLocationManager，我们需要调用 `start` 方法以便开始接收更新，然后在完成后调用对应的 `stop` 方法。NWPathMonitor 的 start 方法要求我们为对象提供一个队列来执行其工作：

```objc
let queue = DispatchQueue.global(qos: .background)
monitor.start(queue: queue)
```

当我们完成监听状态的变化时，我们只需在调用 `cancel()` 方法。请注意，目前在 NWPathMonitor 上调用 cancel 后，我们无法再次启动监听，而是需要实例化一个新的 NWPathMonitor 实例。

请注意，如果在调用 start() 之前访问 NWPathMonitor 的 `currentPath` 属性，将返回 nil。实际上，如果你打印返回到更新回调的 path，如下所示：

```objc
monitor.pathUpdateHandler = { path in
    print(path)
}
```

则会打印以下内容：

```objc
Optional(satisfied (Path is satisfied), interface: en0, scoped, ipv4, ipv6, dns)
```

这表明此处返回的 NWPaths 和 currentPath 属性是可选项，尽管 API 没有明确说明（我们可以推断返回的 NWPath 引用是桥接到 Swift 的 Objective-C 指针）。

## Captive Portals

Captive Portal 是在公共 Wi-Fi 热点连接时显示的网页，通常用于在授权访问 Internet（或访问其他网络资源）之前强制登录、注册或支付。在之前的一篇博客中，我谈到了从 App 开发的的角度来看，Reachability 看起来好像没什么问题，但实际上由于有 Captive Portals，它并不能很好完成任务。这可能导致 App 无法正常工作甚至于崩溃 -- 因为 App 可能期望从 RESTful API 中获取一些 JSON 数据，却从 Captive Portals 获取到了一些 HTML。

我之前很好奇 NWPathMonitor 在检测网络连接方面是否比 Reachability 有所改进。`NWPath.Status` 枚举确实提供了三种情况 -- `satisfied`、 `unsatisfied` 和 `requiresConnection`。不幸的是，Network.framework 的开发者文档并未提供这些枚举值的使用说明，而如果我们查看 NetworkExtension.framework 文档，其中的 `NWPathStatus` 对象提供了 `satisfiable` 枚举值，里面有一些相关文档描述：

> The path is not currently satisfied, but may become satisfied upon a connection attempt. This can be due to a service, such as a VPN or a cellular data connection not being activated.

requiresConnection 枚举值似乎类似于 NWPathStatus 对象的 satisfiable 值。

好消息是 NWPathMonitor 通常只在 captive portal 协商之后通知 path 被设置为 satisfiable 状态，即在弹出 web view 且用户登录后。而在没有弹出 captive portal 的情况下，将向用户显示一个 Action Sheet，提供了 `Use Without Internet` 和 `Use Other Network` 选项。如果用户选择了 Use Without Internet，则 NWPathMonitor 返回的 path 的状态是 satisfied，即便实际上并没有连网。

![1](http://)

通过使用 Charles 做的一些实验，我发现除非选择 `Use Without Internet`，否则在初始化 Wi-Fi 网络连接的同时中断连接的情况下，NWPathMonitor 没有报告 NWPath 的 Status 被置为 statisfied。但是，如果网络连接已恢复，但随后被删除，则并不能检测到这种变更，并且 path 的状态未依然是 satisfied。如果用户仅在火车或酒店上支付一小时的互联网访问费用，这种情况是可能发生的。

## Connectivity

Connectivity 是一个 MIT 许可的开源框架，其目的是复用 iOS 现有的检测 captive portal 的方法。它允许在 iOS 8+ 上使用 Reachability 准确检测真正的 Internet 连接，这意味着在无法使用 NWPathMonitor 时，我们可以使用这个方法。并且在 iOS 12 上，Connectivity 使用了 NWPathMonitor 来提供更高的准确度。

Connectivity 已经提供了对 NWPathMonitor 的支持，可用于 iOS 12+ 系统。如果 framework 属性设置为 network，则会使用 Network 框架来替代 `SystemConfiguration` 框架（Reachability），以监听网络适配器的状态变更。

```objc
let connectivity = Connectivity()
connectivity.framework = .network
```

在网络适配器中的状态更改后，Connectivity 会执行大量检查以确定 Internet 访问是否可用。另外还有一个轮询选项，可以用来轮询网络是否可用，即使状态并未发生改变。可以通过设置 `isPollingEnabled = true` 并将 `pollingInterval` 设置为适当的时间值来实现这一点。

## 总结

Network 框架引入了一些很棒的新类，包括 NWPathMonitor，可用于在 iOS 12+ 上监听设备网络适配器的状态变化。在用户与 captive portal 交互后会将 path 的状态设置为 satisfied，但不会检测后续网络访问的丢失。Connectivity 可以为支持之前 iOS 系统的 App 提供向后兼容性，并通过使用 NWPathMonitor 获取更高的准确性。


### 优点

* Apple 官方支持；
* 无需包含第三方代码 - 只需导入 iOS 12 中的 Network.framework 即可；
* 在与 captive portal 协商后，报告 NWPath 的状态为 satisfied；

### 缺点

* 不能在 iOS 12 之前使用，这意味着如果你需要支持早期版本的 iOS，就会稍显麻烦了；
* 缺乏详细文档；
* 在初始连接成功后，不会再检测 captive portals 以及其他 Internet 连接中断的情况；

#### 相关链接

* Solving the Captive Portal Problem on iOS https://medium.com/@rwbutler/solving-the-captive-portal-problem-on-ios-9a53ba2b381e
* Connectivity https://github.com/rwbutler/Connectivity

推荐阅读

Network.framework 入门
使用 iOS 12 的 Network Framework 实现 netcat
Core ML & Vision 入门教程
iOS 任务调度器：为 CPU 和内存减负

