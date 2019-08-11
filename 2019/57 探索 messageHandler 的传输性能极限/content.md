本次测试的目标是 iOS 8+ 之后提供的 WKWebview 提供的 JS 和 Native 通讯机制之 `messageHandler`，比较浅显的探索下它各方面的极限。

> 测试 Demo 请参考文后链接 1

## 主要的测试场景包括

1、messageHandler 可传输的数据类型有哪些？

2、以字符串为例，messageHandler 最多可传输多少字节？

3、承接 2，在不同量级下的传输效果（是否丢失、速度等）如何？

4、如果是 native 向 WKWebview 传输数据，是否有类似的表现？

5、探索使用 messageHandler 的最佳方法。

## 测试方法

1、测试设备，iPhone XR 模拟器，Xcode 10.2

2、硬件，MacBook Pro (13-inch, 2016），Mem 16 GB 2133 MHz LPDDR3,3.1 GHz Intel Core i5

3、测试流程。启动 App，将 Xcode 切换到 Debug Navigation，观察内存消耗；同时打开 Safari 开发者工具，切换到 Timing tab 里，观察内存的占用情况

## 测试用例

### 1. 传输数据类型

根据 MDN 上对 postMessage 描述，

>**注意:** 在 Gecko 6.0 (Firefox 6.0 / Thunderbird 6.0 / SeaMonkey 2.3)之前， 参数 `message 必须是一个字符串。` 从 Gecko 6.0 (Firefox 6.0 / Thunderbird 6.0 / SeaMonkey 2.3)开始，参数 `message被`使用 **结构化克隆算法**<sup>[2]</sup>进行序列化。这意味着您可以将各种各样的数据对象安全地传递到目标窗口，而不必自己序列化它们。

**Supported types （表1）**

![1](http://)

按照上面的数据，我尝试了所有的数据类型，包括 FormData。得出的结论和 AppleDocumentation 一致，

> Allowed types are NSNumber, NSString, NSDate, NSArray, NSDictionary, and NSNull.

对于 JavaScript 侧，支持

> Primitive Type, Boolean, String, Date, Array(包括 TypedArray）。

那些不在 **表1** 中的数据类型，在调用 postMessage 即出错，如`postMessage(document)`；而那些在表1中，但不在

> Primitive Type, Boolean, String, Date, Array(包括 TypedArray）

中的数据，传到 native 时，`message.body` 是个空对象 `{}`;

**对应 native 向 JS 传输数据的格式**

只有一种，那就是字符串，凡是可以用字符串转换的数据接口都可以借助字符串来实现打散、传输、重建。

### 2. 传输数据速度。

我们以字符串为例，使用 `for`,字符串数组`join`等方式来模拟传输 `1\10\100\1000 M`数据试验。

*（详细的实现方式可以参考代码，这种测试代码准确度不高，因为循环和静态字符串、临时变量是否主动销毁等都会影响内存占用，所以这里得出来的数据，在数据级和相对关系上可供参考）*

下表是通过运行我的 Demo<sup>[1]</sup> 跑了 3 次，大概的数据统计。**注意：webview 和 Xcode 里的内存占用是独立的。**

**表2， messageHandler 向 native 传输 数据**

![2](http://)

352M/max, 180M  表示峰值是 252M，之后维持在 180 M

**表3， native 向 messageHandler 传输 数据**

同时，我还测试了 native 执行 `webview evaluateJavaScript` 来传数据的情况。

![3](http://)

从表2、表3对比可见，对于 Xcode 而已，内存占用是稳定的，而对于 JS 端，容易出现内存泄漏。从这点可见 iOS 的 ARC 的优点。

### 3.可能有好奇的宝宝会追问，那到底到底有没有最大值呢？

答：都没有。

1、h5 到 native 传输的情况，在我的测试用例中，我用人肉和 2 分法（见代码 index.html ），

```
[Log] 2149581562.5 is good, try hard. (index.html, line 67)
[Log] 2149581718.75 is good, try hard. (index.html, line 67)
[Log] 2149581796.875 is good, try hard. (index.html, line 67)
[Log] 2149581835.9375 is too much cost (index.html, line 64)
[Log] 2149581816.40625 is good, try hard. (index.html, line 67)
[Log] 2149581826.171875 is too much cost (index.html, line 64)
[Log] The choosed one is 2149581826.171875 (index.html, line 72)
```

测试出来 **2G** 是 messageHandler 传输的极限。如果传大于 2G的，则会出现 `Out of Memory`的错误。但我相信这是 JS 方法，

```javascript
array.join('')
```

的极限，如果是使用 `File `对象获取的大于 2G 的文件我相信也是可以传输的。

2、native 传输 h5情况，基本没有限制，传入 10000M 的时候，大概执行了 5 分钟，内存也飙到 11 G，但两端都没有出现问题。

![传 10000 M 时的内存占用](https://upload-images.jianshu.io/upload_images/277783-8a87eb8d947dacc4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

理论上是无限的，现实是有限的，真正确定上限的是你的硬件，如我的电脑上执行 10000 M 情况时，已经报警了，再超就会被杀进程了。

![内存警告](https://upload-images.jianshu.io/upload_images/277783-6cdbc42299cd2129.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### 5. 最佳参数

**2 ~ 10 M**，传输速度很快，可以传 1M+ 的数据，这时候的耗时也很小。对于 messageHandler 这么好的通道，还可以发掘的用途还很多，我想到的，

1. 可以将整个界面截图（注意是滚动截图）获得的数据传到 native，让 native 持久化。
2. 保存整个渲染完毕的 html 结构，这个功能可作为个性化的渲染结果，代替使用 phatom 这种无头浏览器渲染获得渲染结果的方式。

### 6. messageHandler 最多可以注册多少个？

我相信是无数个，因为机器配置有点差，就不再尝试让它包内存警告了。`window.webkit.messageHandlers` 对象是延迟初始化，在内存支持的情况下，会自然扩展。

## 结论

1、native 调用 h5 传数据， JS 的入参（如 Index.html 文件中的，_content），因为是被跨进程持有，会造成泄漏，即使是即将  _content 置为 null，这个情况很严重，说明 webview evaluateJavaScript 的参数不能很长，不知如何化解这个问题？

![内部对象持有的](https://upload-images.jianshu.io/upload_images/277783-cc2e7e082a4d7b57.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

2、最佳参数是 2 ~ 10 M，所以如果大于 10 M 的数据，可以考虑使用断点续传的方式，这时候需要考虑网络传输的机制，保证时效（ack_seq）正确和不丢失。

3、传 100M 左右的数据很慢，会卡住主线程。我们的好朋友`worker.js` 这时候不能执行 `postMessage`方法，也帮不了忙。但是可以在 `worker.js` 里做大数据 slice 的工作。

4、`window.webkit.messageHandlers.postMessage` 应该是 `window.postMessage` 的封装[ 从黑盒测试的角度来看，需要看源码确认]。所以他可以很好的实现 frame 和 iframe 之间的跨域问题。例如`WebViewJavascriptBridge` 因为它用了 iframe 的`_fetchQueue`，导致 `WebViewJavascriptBridge`在 iframe 是失效的。这也是 messageHandlers 的长处之一。

5、messageHandler 传不了 Blob 或者 FormData 等二进制文件

### 参考

1. https://github.com/hite/MessageHandlerIOLimitTest
2. https://developer.mozilla.org/en-US/docs/DOM/The_structured_clone_algorithm
3. Structured_clone_algorithm
	https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Structured_clone_algorithm
4. 理解DOMString、Document、FormData、Blob、File、ArrayBuffer数据类型
	https://www.zhangxinxu.com/wordpress/2013/10/understand-domstring-document-formdata-blob-file-arraybuffer/
5. window.postMessage
	https://developer.mozilla.org/zh-CN/docs/Web/API/Window/postMessage

