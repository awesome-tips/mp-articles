## 1. 背景

在作者将 App 里的 UIWebView 切换到 WKWebView 之后，有些功能得到了加强，包括烦人的 Cookie 也很好的解决了，唯独离线包和 webView 里的资源请求拦截一直是个心病，没有很优雅的解决方案。

坊间流传的自定义拦截 https、http 的方案，可以实现一部分功能。但是有很大的缺点，

1. 调用私有的 API，审核和以后版本升级有隐患
2. XHR 请求丢失 body 的问题，所以要封装一层 myXHR 代替 XHR
3. 拦截 form 请求里的 body 参数。

而缓存和资源拦截作为 webView 框架应该提供的基本功能，AppHost【注1】也尝试用一个更优雅的方式来实现，所以尝试用 iOS11+ 上提供的新接口。

## 2. WKURLSchemeHandler 的 checklist

WKURLSchemeHandler 被引入是在 *WWDC 2017 Customized Loading in WKWebView*<sup>[1]</sup>，当时演示的时候是以**加载自定义图片为例**的。

**请记住这句话“加载自定义图片为例”。**

结合我们的需求，我们需要加载**本地资源**、**拦截请求**、**发出 ajax 请求**，同时这些请求都包含正确的 **Cookie**，我们需要解决以下几个子问题；

### 自定义 scheme ，可以定义哪些 scheme？

根据自己测试和阅读源码，下面类型的都属于内置协议不可用

```c++
static const StringVectorFunction functions[] {
            builtinSecureSchemes,
            builtinSchemesWithUniqueOrigins,
            builtinEmptyDocumentSchemes,
            builtinCanDisplayOnlyIfCanRequestSchemes,
            builtinCORSEnabledSchemes,
        };
```

结论：已知的约定俗称的都不能定义，包括，https、http、about，当然也包括，data、blob、ftp 等，如果你这样做了，会收到一个错误`''https' is a URL scheme that WKWebView handles natively'`

### 自定义 scheme ，哪些 HTML 里元素会触发自定义请求？

根据作者在常见的 html 元素里搜集的会触发资源下载或者 navigation 逻辑的标签，整理了下面的表格。

![1](http://)

总结一下；按照 *MDN 里混合内容*<sup>[2]</sup> 的说法，**混合被动/显示内容**在任何情况下都会触发，**混合活动内容** 在 https 下不会触发。

在 https 下的 xhr 的请求和混合活动内容，都不能触发的原因其实是两个机制；

**1. 混合活动内容是受限 CSP，目前基本无法绕过。**

```html
<meta http-equiv="Content-Security-Policy" content="default-src 'self'; img-src https://*; script-src 'self' apphost: ;">
```

上面的写法并没有生效，依然阻止 `apphost://a.js` 的文件加载。（可能是我姿势不对）

**2. xhr 不能发出去，原因是 same-origin 策略导致的。**

> Definition of an origin *Section*<sup>[3]</sup>
> Two URLs have the *same origin* if the protocol, port (if specified), and host are the same for both.

### 自定义 scheme ，可以带 Cookie 吗？

作者通过网上搜索、阅读了 RFC6265 里对 Cookie 的规范、实践，得出了以下结论。

1. 自定义 scheme 是可以带 Cookie，而且和相同 domain 的其他协议共享 Cookie。这就厉害了，然后实际上不完全是——初步测试，`http`,`https`,`ftp`等 scheme 其实是完全没问题，**包括设置 session 级别的 Cookie；在此之外，设置自定义 scheme 的 session Cookie 会 fail，持久化 Cookie 是可行的**。 没想到吧

### 自定义 scheme ，可以发 post 请求吗？

是的。在 webkit 实现有 bug 的情况下，可以用 form 发送 post 请求。不仅如此，使用 form post，**可以成功将 body 发送到 native**，这个就厉害了。

### 自定义 scheme ，可以拦截 xhr 请求？是否丢失 POST 请求的 body？

事实上是可以拦截的。在上面表格里虽然 http、https 下都是 ❌，但是那是因为 domain 是标准协议。如果是

```objc
[self.webView loadHTMLString:html baseURL:[NSURL URLWithString:@"apphost://test.com/index.2.html"]];
```

那么 xhr 依然也可以发送出去，但是不幸的时，xhr 里 body 还是丢了。

## 3. 实践

有了上述的前提。我们有一个简单的离线包渲染的方案。

* 把加载页面的 domain 改为 自定义协议如 apphost；

```objc
[self.webView loadHTMLString:html baseURL:[NSURL URLWithString:@"apphost://test.com/index.2.html"]];
```

* 加载的 HTML 文件应用的资源全部用`://user/a.png`这种相对的写法。这样 http 和 自定义协议都可以用。
* HTML 里面  ajax 请求使用绝对路径，修改后端的 Response 头，使用 Cross-Origin Resource Sharing (CORS) 技术，配合对 ajax 请求追加几个参数，也可以实现带上 Cookie 的请求

上面的做法，可以拦截基本上传统 UIWebview 时代可拦截的所有类型。但是缺点也很明显。

- iOS 11+ 以上才能用
- https 下支持有限
- 对 xhr js 代码小幅度的改动
- 最最严重的是，不支持 session 级别的 Cookie 设置。

在 AppHost 框架里，作者没有采用这种方式，而是采用了读 html，解析静态资源的方式来实现加载离线包的功能，同时不影响 session 级别的 Cookie 设置。

总的来说 WKURLSchemeHandler 的使用场景还是比较有限，不能拦截 http 请求，有些结合 NSURLSession，先下载 html 的方式，然后再代理的方式，对 webview 的加载速度反而是拖累，还需要想想其它办法。

**结论：WKURLSchemeHandler 最好的场景还是用在加载本地图片上，其它方面不稳定，慎用。**

## 4. 注

AppHost.framework 是作者在网易有钱、严选工作中，抽离出来 JSBridge 的库，实现 native 和 h5 之间的通讯，内置诸多常用的功能，在业务简单的情况下可开箱即用，复杂情况允许灵活定制。预计在4月底开源

## 5. 参考

1. https://developer.apple.com/videos/play/wwdc2017/220/
2. https://developer.mozilla.org/zh-CN/docs/Security/MixedContent
3. https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy#Definition_of_an_origin
4. network-scheme 的定义 : https://fetch.spec.whatwg.org/#network-scheme
5. RFC6265里对 Cookie domain 的说明 : https://tools.ietf.org/html/rfc6265#section-4.1.1
6. Preflighted requests : https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS#Preflighted_requests
7. 关于 ajax 跨域携带 Cookie : https://blog.csdn.net/qq_25835645/article/details/78622349
8. 测试 demo 源码 : https://github.com/hite/URLSchemeHandlerTest

