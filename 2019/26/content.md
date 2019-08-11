有个朋友问了我一个问题:

> 他们通过 `WKWebView`，访问了一个其他的页面，然后希望原生获得用户的输入信息。

其实，我之前接触 `WKWebView` 并不多，但是这个问题我觉得很有意思。这篇文章便是我解决这个问题的全部思路，与最终的解决办法。

## 思路分析 与 代码实践

这个问题其实很具象了，就是希望原生获得H5的用户输入内容(这样子感觉有些不地道-_-)

接下来我们就需要分析这个需求了。

首先我们先需要抓住两个点，1个是H5，1个是原生。

所以这个问题现在被我拆分出了1个额外的问题

### HTML能否和原生交互?如何进行?

这个问题其实是一个很关键的问题，因为我们只有实现了原生和HTML的交互，才能获得相关信息(这里我们假定网页是我们自己写的，完全受操纵于我们自己)

于是便搜寻资料，发现 `WKWebView` 提供了一个很方便的交互渠道 `WKScriptMessageHandler`，我们通过对 `WKWebView` 进行相关的定制操作便可以解决。

我们先创建一个工程，这里我把整个工程命名为 **InjectHTML**
为了防止循环引用，我们先构建一个中间层 `ScriptHandler`

```objc
import Foundation
import WebKit
// 这里我们使用一个中间层来解除循环WebView和Controller间的循环引用问题
class ScriptHandler: NSObject, WKScriptMessageHandler {

    weak var delegate: WKScriptMessageHandler?

    func userContentController(_ userContentController: WKUserContentController, didReceive message: WKScriptMessage) {
        delegate?.userContentController(userContentController, didReceive: message)
    }

    init(delegate: WKScriptMessageHandler? = nil) {
        self.delegate = delegate
    }
}
```

在 ViewController 中的代码

```objc
import UIKit
import WebKit
class ViewController: UIViewController {
    var webview: WKWebView!

    static let scriptKey = "InjectHTML"

    override func viewDidLoad() {
        super.viewDidLoad()
        //初始化 Configuration
        let configuration = WKWebViewConfiguration()
        configuration.userContentController = WKUserContentController()
        // 给Configuration 增加一个js script处理器
        // 采用了中间层的因素，避免循环引用导致无法释放问题
        configuration.userContentController.add(ScriptHandler(delegate: self), name: ViewController.scriptKey)

        webview = WKWebView(frame: view.bounds, configuration: configuration)

        view.addSubview(webview)
        // 设置导航处理器
        webview.navigationDelegate = self
        // 我们先从本地读网页方便自我改动测试
        let fileURL = Bundle.main.url(forResource: "index", withExtension: "html")
        // 加载网页
        webview.loadFileURL(fileURL!, allowingReadAccessTo: fileURL!)
    }
}

extension ViewController: WKScriptMessageHandler {
    // 遵守协议
    func userContentController(_ userContentController: WKUserContentController, didReceive message: WKScriptMessage) {
        // 因html可能传递多种类型名称，我们这里须指定key
        if message.name == ViewController.scriptKey {
            guard let dic = message.body as? [String: String] else {
                return
            }
            // 交给真实处理解析函数
            receiveInputValue(para: dic)
        }
    }
    // 解析函数可以负责更具体的内容，因为demo，故此只是打印
    func receiveInputValue(para: [String: String]) {
        let title = para["title"] ?? "无值"

        let message = para["message"] ?? "无值"

        let `id` = para["id"] ?? "无值"

        print("title: \(title)")
        print("message: \(message)")
        print("id: \(id)")
    }
}
// 遵循导航协议，方便我们知道何时网页加载完成
extension ViewController: WKNavigationDelegate {
    func webView(_ webView: WKWebView, didFinish navigation: WKNavigation!) {

    }
}
```

这样子我们的 ViewController 就构建完成，访问了一个本地 `index.html` 的网页。

我们在项目中新建一个html文件(从外界拖入也可)

html 页面只用于测试，因此我们也就不做类似自适应之类的css样式了

html 代码如下:

```html
<html>
    <head>
        <title>注入测试网页</title>
    </head>
    <body bgcolor="#FFFFFF">
        <h1>Test for inject js</h1>
        <!--添加一个button按钮用以测试点击事件-->
        <button onclick="onClickTest()">Test Button</button>
    </body>
    <script type="text/javascript">
        function onClickTest() {
            // 注意，这里是需要注意的是我们在ViewController中定义的Script Name需要作为messageHandler的一个属性
            window.webkit.messageHandlers.InjectHTML.postMessage({title: 'test title', message:'test message', id: 'test id'})
            // 若我们未注册此名称，则无法触发对应回调，postMessage中的参数可传为任意的，但我们在原生中定义为字典了，则我们在这里需要传入字典
            window.webkit.messageHandlers.InjectHTMLS.postMessage({title: 'test title', message:'test message', id: 'test id'})
        }
    </script>
</html>
```

其中对应部分均已加上注释，接下来我们来跑一遍测试结果:

```
title: test title
message: test message
id: test id
```

当我们点击 WebView 上的按钮时候，我们可以打印出来对应的js端返回结果，说明成功了。

所以，这个需求我们可以说解析了3分之1了。

接下来我们需要继续分析了，这个需求需要我们能监控所有的输入控件。

所以我们需要从网页端剖析了，问题回归HTML端。

我们继续拆解问题，我们既然要监控所有的输入控件，那么我们首先就得知道，我们能否获得所有的输入控件(甚至说，我们能否获得页面的所有控件，然后进行遍历过滤也可以)

### HTML如何获取指定元素

这个问题，通过搜索与询问前端朋友得知:

HTML提供了对应的Api，直接获取指定标签的内容，因为我们是要获得输入框，输入框在HTML中的标签是input。所以我们获得页面中所有输入框元素的方法就已经出来了

```javascript
// 获得所有input数组
var inputs = document.getElementsByTagName("input")
```

那么问题就迎刃而解了，然后我们还是需要测试一下，毕竟万一这个方法不好用怎么办

这里就不在手机端进行测试，因为如果模拟器还是比较费事的，并且我们如果想打印`console`日志的话，看起来也不那么容易，因此我们将接下来的测试网页步骤，直接挪到`Google Chrome`上。

这里还是给出对应的html测试代码

```html
<html>
    <head>
        <title>注入测试网页</title>
    </head>
    <body bgcolor="#FFFFFF">
        <h1>Test for inject js</h1>
        <!--添加一个button按钮用以测试点击事件-->
        <button onclick="onClickTest()">Test Button</button>

        <input type="text" placeholder="Test Input1">
        <input type="text" placeholder="Test Input2">
    </body>
    <script type="text/javascript">
        // 获得所有input数组
        var inputs = document.getElementsByTagName("input");
        // 打印input数组
        console.log(inputs)
    </script>
</html>
```

然后使用浏览器(下文浏览器均为Google Chrome)打开。

我们按Command+Shift+c即可打开页面审查和网页控制台，在这里我们看见了我们执行网页的log，打印结果如下

![1](http://)

说明我们获取input成功了

接下来我们只需要进行过滤即可获得我们需要的页面元素了(如过滤checkbox之类的)

然后接下来新的问题来了

### JS代码如何动态添加事件

我们已经通过js代码获得input了，所以问题就变到了如何添加点击事件。毕竟原生的输入框还有至少监听输入事件，或者监听输入完成类的，那么JS端理应也存在。

这里感谢**菜鸟教程**，查询到了一个函数`addEventListener`

这个函数的功能就是给HTTP的DOM元素增加对应的事件，也就是说我们可以通过这个方法额外的增加点击事件。

同时，新增加的事件并不会覆盖原有事件，一个元素可以拥有多个同样的事件(如一个按钮可以同时出发两个onClick事件)

这个函数给了我们新的天地啊。因此我们可以给input增加事件，这里我选择了两个事件，一个是input事件(输入框文字发生变化)，一个是change事件(输入框失去焦点)

我们修改上文的html代码，修改结果如下:

```html
<html>
    <head>
        <title>注入测试网页</title>
    </head>
    <body bgcolor="#FFFFFF">
        <h1>Test for inject js</h1>
        <!--添加一个button按钮用以测试点击事件-->
        <button onclick="onClickTest()">Test Button</button>
        <!--这里给input增加一些回调-->
        <input type="text" placeholder="Test Input1" onchange="onChange()" oninput="onInput()">
        <input type="text" placeholder="Test Input2" onchange="onChange()" oninput="onInput()">
    </body>
    <script type="text/javascript">
        // 这里是为了测试原有对应事件是否会被覆盖
        function onChange(input) {
            console.log("原有 失去焦点");
            console.log(input);
        }

        // 这里是为了测试原有对应事件是否会被覆盖
        function onInput(input) {
            console.log("原有 键盘输入");
            console.log(input);
        }
    </script>
    <script type="text/javascript">
        function demoOnchange(input) {
            console.log("失去焦点");
            console.log(input);
        }
        function demoOnInput(input) {
            console.log("键盘输入");
            console.log(input);
        }
        function demoSet() {
            var inputs=document.getElementsByTagName("input");
            for(var i=0;i < inputs.length;i++) {
                var input = inputs[i];
                // 这里我们增加一些过滤条件，因为我们有时并不需要所有的input，这里我只是允许了text(文字输入框)
                if(input.type=="text") {
                    input.addEventListener('change', demoOnchange);
                    input.addEventListener('input', demoOnInput)
                }
            }
        }
        demoSet();
    </script>
</html>
```

继续进入浏览器测试网址，我们可以看到，当我们输入的时候，同时触发了两个回调，说明后期注入有效

接下来我们需要继续考虑，我们需要能让原生接收到对应事件啊

### 原生如何收到对应事件

我们在上文中已经知道了js如何调用原生，那么这一步也就我们也就可以直接通过结合便可以实现了。

这一步还是更改html代码

```html
<html>
    <head>
        <!--HTML页面内容需告知为utf8，否则会出现乱码问题-->
        <meta http-equiv="Content-Type" content="text/html;charset=utf-8">
        <title>注入测试网页</title>
    </head>
    <body bgcolor="#FFFFFF">
        <h1>Test for inject js</h1>
        <!--添加一个button按钮用以测试点击事件-->
        <button onclick="onClickTest()">Test Button</button>
        <!--这里给input增加一些回调-->
        <input type="text" placeholder="Test Input1" onchange="onChange()" oninput="onInput()">
        <input type="text" placeholder="Test Input2" onchange="onChange()" oninput="onInput()">
    </body>
    <script type="text/javascript">
        // 这里是为了测试原有对应事件是否会被覆盖
        function onChange(input) {
            console.log("原有 失去焦点");
            console.log(input);
        }

        // 这里是为了测试原有对应事件是否会被覆盖
        function onInput(input) {
            console.log("原有 键盘输入");
            console.log(input);
        }
    </script>
    <script type="text/javascript">
        function demoOnchange(input) {
            // 这里，input是一个点击事件，target才是真正的input元素，我们可以通过此任意获取input的相关信息，如class，id以及其他的一些信息等等
            window.webkit.messageHandlers.InjectHTML.postMessage({title: '输入框失去焦点', message:input.target.value, id: input.target.id});
        }
        function demoOnInput(input) {
            window.webkit.messageHandlers.InjectHTML.postMessage({title: "输入框正在输入", message:input.target.value, id: input.target.id});
        }
        function demoSet() {
            var inputs=document.getElementsByTagName("input");
            for(var i=0;i < inputs.length;i++) {
                var input = inputs[i];
                // 这里我们增加一些过滤条件，因为我们有时并不需要所有的input，这里我只是允许了text(文字输入框)
                if(input.type=="text") {
                    input.addEventListener('change', demoOnchange);
                    input.addEventListener('input', demoOnInput)
                }
            }
        }
        demoSet();
    </script>
</html>
```

重新启动刚才的项目App，在输入框中输入，可以看到控制台中返回了数据

```
title: 输入框正在输入
message: 1
id:
title: 输入框失去焦点
message: 1
id:
```

我们的打印出来了。到这里我们自有网页的测试全部通过，那么就剩下最后一步了，如何在三方网页上执行?

### JS代码注入

WKWebView既然能让js调用OC，那么OC能否调用js代码呢?

答案是可以的，WKWebView提供给我们原生的方法可以动态的执行js代码

```objc
webview.evaluateJavaScript(someTest, completionHandler: closure)
```

这个方法可以让我们动态的执行由原生生成的js代码。

那么我们需要执行什么方法呢?

自然就是我们上述研究出来的获取HTML的元素并增加事件回调的事情啊。

这里我们在项目中新建一个js文件，方便我们以后修改js代码，名称就叫`Inject.js`

从项目中的html文件中把如下方法复制进来

```javascript
function demoOnchange(input) {
    window.webkit.messageHandlers.InjectHTML.postMessage({title: "输入框失去焦点", message:input.target.value, id: input.target.id});
}
function demoOnInput(input) {
    window.webkit.messageHandlers.InjectHTML.postMessage({title: "输入框正在输入", message:input.target.value, id: input.target.id});
}
function demoSet() {
    var inputs=document.getElementsByTagName("input");
    for(var i=0;i < inputs.length;i++) {
        var input = inputs[i];
                
        if(input.type=="text") {
            input.addEventListener('change', demoOnchange);
            input.addEventListener('input', demoOnInput)
        }
    }
}
demoSet();
```

然后我们决定采用直接从项目中读取文件的形式将js文件转成字符串，我们在ViewController中的

```javascript
func webView(_ webView: WKWebView, didFinish navigation: WKNavigation!)
```

函数中加入js代码注入，使我们每次加载成功网页都注入对应的js代码
完整函数如下:

```objc
func webView(_ webView: WKWebView, didFinish navigation: WKNavigation!) {
    guard let jsString = try? String(contentsOfFile: Bundle.main.path(forResource: "Inject", ofType: "js") ?? "") else {
            // 没有读取出来则不执行注入
        return
    }
    // 注入语句
    webView.evaluateJavaScript(jsString, completionHandler: { _, _ in
        print("代码注入成功")
    })
}
```

我们将应用程序中的index.html网页中对应的js代码删除。使这个网页更像是一个三方网页(不会主动支持该功能)。这里，就不再详述HTML代码了。

我们执行测试后发现，测试网页回调成功

### 三方网页集成测试

当我们以三方网页测试的时候，便会发现这样或者那样的问题。

我们以 掘金 为例，试试我们的自有脚本。我们的js代码在掘金网上的登录竟然失效了。

但是作为程序，函数是具有幂等性的（在相同情况下进行无限次的操作，结果一定相同）。那么只能是我们的环境出现了问题，而不是我们的js代码失败了。

所以我们需要观察环境究竟哪里出现问题了。

首先在我们的自有测试网站上，输入框是直接就存在的。但是在掘金上却不是，它需要我们点击登录按钮后作为弹框出现。而我们的js代码注入是在网页渲染完成，因此我们接下来尝试给我们的页面增加一个按钮。点击的时候再进行js代码注入

为了简化代码，我们给ViewController加入了一个导航栏，并在导航栏右上角增加一个注入按钮。(导航控制器通过StoryBoard增加)

我们在 `ViewController` 的 `viewDidLoad` 中增加如下代码:

```objc
self.navigationItem.rightBarButtonItem = UIBarButtonItem(title: "注入", style: .plain, target: self, action: #selector(injectJS))
```

同时在类中增加如下方法

```objc
@objc func injectJS() {
    guard let jsString = try? String(contentsOfFile: Bundle.main.path(forResource: "Inject", ofType: "js") ?? "") else {
        // 没有读取出来则不执行注入
        return
    }
    // 注入语句
    webview.evaluateJavaScript(jsString, completionHandler: { _, _ in
        print("代码注入成功")
    })
}
```

重新运行代码

发现头部被挡在导航栏下面了-_-

继续改尺寸（代码就不列出来了）

这阵我们点击网页上的登录按钮，随后点击导航栏上注入

然后再进行输入，我们就可以看见控制台打印出我们正在输入的账户名和密码了-_-

如果你问我怎么自动注入？我给的思路就是获取到登录的元素，然后再多做一步回调，我们可以先注入一次js再注入我们的目标js。

但是具体怎么实现么，不好意思，不说了。

## 总结
其实这篇文章的主要目的介绍如何分析具体的需求，并将其转换为代码。根本在于**从一个相对于具象的需求中，我们一步一步抽离问题，组成更细小的问题的组合，然后逐步的去实验小问题的可行性，最终完成复杂问题**

所有的需求均可以如此实现，只是有的我们可以实现，有的问题分析到最后发现，细节无法实现或实现起来成本过高(比如平安银行的自动识别手机壳颜色)。

当我写完这篇文章的demo的时候发现，我们的用户信息实在是太容易泄漏了，比如我的掘金账户和密码，如果在别人app上的内嵌网页登录的话，不是直接就泄漏了吗(手动滑稽)。

ps:这篇文章可以有个别名:**震惊！！！你还敢在你的手机上登录账户么**

本文所述demo链接:

https://link.juejin.im/?target=https%3A%2F%2Fgithub.com%2Fchouheiwa%2FInjectJSDemo


