## 背景

本文讨论 WKWebview 在加载 h5 页面时，Objective-C 里的 `WKNavigationDelegate`、`window.performance.timing`、`WKUserScriptInjectionTimeAtDocumentStart`、`WKUserScriptInjectionTimeAtDocumentEnd`，以及和前端最常用的`document.readystate\domContentLoaded\document.onload` 事件等时间维度的关系，为 native 和前端在相互调用时，能够明确沟通的时机。

## 图解

**普通的200请求，注意先看图例**

本次数据采集的页面是 https://mp.weixin.qq.com/s/X_WDv1-vqdXYcg0eLpMAhA，

![1](http://)
​
**带有302跳转的页面**

本次数据采集的页面是 https://lq.163.com/platform/wap/entry?merchantCode=M32412338855&before_login=1

![2](http://)

**请求出错时的序列**

本次数据采集的页面是 https://hite.me，出错原因是因为证书不正确。

![3](http://)
​
## 对比，结论

**首先注意几个等价事件。**

* didStartProvisionalNavigation = navigationStart
* didCommitNavigation = domLoading  =  WKUserScriptInjectionTimeAtDocumentStart，此时刚刚开始创建 DOM
* WKUserScriptInjectionTimeAtDocumentEnd = domContentLoade = document.readystate = interactive，此时 CSSOM 和 DOM 都已经构建完毕，等待图片资源等下载
* didFinishNavigation = domComplete = document.onload，注意此时进度条才结束
* decidePolicyForNavigationResponse 在 transfer-encoding =chunked 的情况下，不等价于 responseEnd

**注意使用estimatedProgress来显示进度条的方案有缺陷。**

`0%` 不是从 `LoadRequest` 开始的，甚至不是从 `didStartProvisionalNavigation` 开始。所以一个符合用户视角的进度条应该自己写 timer 来显示进度。从 `decidePolicyForNavigationResponse` 开始显示进度会有很长时间的空白和无进度条阶段，从这次用例来看大概 1s 的空白时间。

**白屏时间**

从 `loadRequest` 到 `decidePolicyForNavigationAction` 之间的白屏时间耗时，使用预加载也无法消除，目前我没有找到很好的方法来缩短这 500 多毫秒的耗时，如有有读者有更好的方法希望告知下。

**关于 TTFB 的起始时间**

我这里计算的 TTFB 和 web 上的 TTFB 不一样，我认为在 WKWebview 里起始时间应在  `decidePolicyForNavigationAction`，而不是 `responseStart`，以 decidePolicyForNavigationAction 更符合用户视角，我对 TTFB 的改动和下面一条理念是一样的。

**关于FP、FMP（first-meaniful-paint）**

Safari 和 Chrome 都实现了 `performance.timing` 接口，但是 Chrome 更详细一些。对用户而言 FMP 才是最重要的。详见参考链接

**使用CSP，无法阻止 WKUserScriptInjectionTimeAtDocumentStart的注入脚本执行**

所有的测试代码和都在我即将发布的一个JSBridge框架里，大概在月底可以发布到 github 上，欢迎大家提意见。

### 参考

* http://kaaes.github.io/timing/info.html
* https://developers.google.com/web/fundamentals/performance/user-centric-performance-metrics#user-centric_performance_metrics
* https://w3c.github.io/paint-timing/
* https://developer.mozilla.org/en-US/docs/Web/API/PerformanceObserver/PerformanceObserver#Example
* https://en.wikipedia.org/wiki/Time_to_first_byte
* https://www.w3.org/TR/navigation-timing-2/#processing-model
* https://github.com/micmro/performance-bookmarklet

