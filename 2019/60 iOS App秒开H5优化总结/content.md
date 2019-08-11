## 背景

为了快递迭代、更新，公司app有一大模块功能使用H5实现，但是体验比原生差，这就衍生了如何提高H5加载速度，优化体验的问题。此文，记录一下自己的心路历程。
腾讯bugly发表的一篇文章`《移动端本地 H5 秒开方案探索与实现》`<sup>[1]</sup>中分析，H5体验糟糕，是因为它做了很多事：

> 初始化 webview -> 请求页面 -> 下载数据 -> 解析HTML -> 请求 js/css 资源 -> dom 渲染 -> 解析 JS 执行 -> JS 请求数据 -> 解析渲染 -> 下载渲染图片

一般页面在 dom 渲染后才能展示，可以发现，H5 首屏渲染白屏问题的原因关键在于，如何优化减少从请求下载页面到渲染之间这段时间的耗时。所以，减少网络请求，采用加载离线资源加载方案来做优化。

## 离线包

### 离线包的分发

使用公司的CDN实现离线包的分发，在对象存储中放置离线包文件和一个额外的 info.json 文件(例如：https://xxx/statics/info.json)：

```json
{
    "version":"4320573858a8fa3567a1",
    "files": [
       "https://xxx/index.html",
       "https://xxx/logo.add928b525.png",
       "https://xxx/main.c609e010f4.js",
       "https://xxx/vender.821f3aa0d2e606967ad3.css",
       "https://xxx/manifest.json"
    ]
}
```

其中，app存储当次的version，当下次请求时version变化，就说明资源有更新，需更新下载。

### 离线包的下载

* **离线包内容**：css，js，html，通用的图片等
* **下载时机**：在app启动的时候，开启线程下载资源，注意不要影响app的启动。
* **存放位置**：选用沙盒中的/Library/Caches。
* 因为资源会不定时更新，而/Library/Documents更适合存放一些重要的且不经常更新的数据。
* **更新逻辑**：请求CDN上的info.json资源，返回的version与本地保存的不同，则资源变化需更新下载。注：第一次运行时，需要在/Library/Caches中创建自定义文件夹，并全量下载资源。

1、获取CDN和沙盒中资源：

```objc
NSMutableArray *cdnFileNameArray = [NSMutableArray array];
//todo 获取CDN资源

NSArray *localExistAarry = [[NSFileManager defaultManager] contentsOfDirectoryAtPath:dirPath error:nil];
```

2、本地沙盒有但cdn上没有的资源文件，需要删除，以防文件越积越多：

```objc
//过滤删除操作
NSPredicate *predicate = [NSPredicate predicateWithFormat:@"NOT (SELF IN %@)", cdnFileNameArray];
NSArray *filter = [localExistAarry filteredArrayUsingPredicate:predicate];
if (filter.count > 0) {
 [filter enumerateObjectsUsingBlock:^(id  _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
     NSString *toDeletePath = [dirPath stringByAppendingPathComponent:obj];
     if ([fileManager fileExistsAtPath:toDeletePath]) {
         [fileManager removeItemAtPath:toDeletePath error:nil];
     }
 }];
}
```

3、 已经下载过的文件跳过，不需要重新下载浪费资源；
4、下载有变化的资源文件，存储至对应的沙盒文件夹中：

```objc
NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:[NSURL URLWithString:cssUrl]];
request.timeoutInterval = 60.0;
request.HTTPMethod = @"POST";
NSURLSession *session = [NSURLSession sharedSession];
NSURLSessionDownloadTask *downLoadTask = [session downloadTaskWithRequest:request completionHandler:^(NSURL * _Nullable location, NSURLResponse * _Nullable response, NSError * _Nullable error) {
    if (!location) {
        return ;
    }
         
    // 文件移动到documnet路径中
    NSError *saveError;
    NSURL *saveURL = [NSURL fileURLWithPath:[dirPath stringByAppendingPathComponent:fileName]];
    [[NSFileManager defaultManager] moveItemAtURL:location toURL:saveURL error:&saveError];
}];
[downLoadTask resume];
```

> 注：如果是zip包，还需要解压处理。

## 拦截并加载本地资源包

### NSURLProtocol

公司的项目从 UIWebView 迁移到了 WKWebView。WKWebView性能更优，占用内存更少。

对H5请求进行拦截并加载本地资源，自然想到NSURLProtocol这个神器了。

NSURLProtocol能拦截所有当前app下的网络请求，并且能自定义地进行处理。使用时要创建一个继承NSURLProtocol的子类，不应该直接实例化一个NSURLProtocol。

### 核心方法

**+ (BOOL)canInitWithRequest:(NSURLRequest *)request**

判断当前protocol是否要对这个request进行处理（所有的网络请求都会走到这里）。

**+ (NSURLRequest *)canonicalRequestForRequest:(NSURLRequest *)request**

可选方法，对于需要修改请求头的请求在该方法中修改，一般直接返回request即可。

**- (void)startLoading**

重点是这个方法，拦截请求后在此处理加载本地的资源并返回给webview。

```objc
- (void)startLoading
{
    //标示该request已经处理过了，防止无限循环
    [NSURLProtocol setProperty:@YES forKey:URLProtocolHandledKey inRequest:self.request];
    
    NSData *data = [NSData dataWithContentsOfFile:filePath];
    NSURLResponse *response = [[NSURLResponse alloc] initWithURL:self.request.URL
                                                            MIMEType:mimeType
                                               expectedContentLength:data.length
                                                    textEncodingName:nil];
        
    //硬编码 开始嵌入本地资源到web中
    [[self client] URLProtocol:self didReceiveResponse:response cacheStoragePolicy:NSURLCacheStorageNotAllowed];
    [[self client] URLProtocol:self didLoadData:data];
    [[self client] URLProtocolDidFinishLoading:self];
}
```

**- (void)stopLoading**

对于拦截的请求，NSURLProtocol对象在停止加载时调用该方法。

### 注册

**[NSURLProtocol registerClass:[NSURLProtocolCustom class]];**

其中NSURLProtocolCustom就是继承NSURLProtocol的子类。

但是开发时发现NSURLProtocol核心的几个方法并不执行，难道WKWebview不支持NSURLProtocol?

原来由于网络请求是在非主进程里发起，所以 NSURLProtocol 无法拦截到网络请求。除非使用私有API来实现。使用WKBrowsingContextController和registerSchemeForCustomProtocol。 通过反射的方式拿到了私有的 class/selector。通过把注册把 http 和 https 请求交给 NSURLProtocol 处理。

```objc
Class cls = NSClassFromString(@"WKBrowsingContextController");
SEL sel = NSSelectorFromString(@"registerSchemeForCustomProtocol:");
if ([(id)cls respondsToSelector:sel]) {
    // 把 http 和 https 请求交给 NSURLProtocol 处理
    [(id)cls performSelector:sel withObject:@"http"];
    [(id)cls performSelector:sel withObject:@"https"];
}

// 这下 NSURLProtocolCustom 就可以用啦
[NSURLProtocol registerClass:[NSURLProtocolCustom class]];
```

毕竟使用苹果私有api，这是在玩火呀。这篇文章`《让 WKWebView 支持 NSURLProtocol》`<sup>[2]</sup>有很好的说明。比如我使用私有api字串拆分，运行时在组合，绕过审核。还可以对字符串加解密等等。。。

### 实际问题

通过以上处理，可以正常拦截处理，但是又发现拦截不了post请求(拦截到的post请求body体为空)，即使在canInitWithRequest:方法中设置对于POST请求的request不处理也不能解决问题。内流。。。

经了解，算是 WebKit 的一个缺陷吧。首先 WebKit 进程是独立于 app 进程之外的，两个进程之间使用消息队列的方式进行进程间通信。比如 app 想使用 WKWebView 加载一个请求，就要把请求的参数打包成一个 Message，然后通过 IPC 把 Message 交给 WebKit 去加载，反过来 WebKit 的请求想传到 app 进程的话（比如 URLProtocol ），也要打包成 Message 走 IPC。出于性能的原因，打包的时候 HTTPBody 和 HTTPBodyStream 这两个字段被丢弃掉了，这个可以参考 WebKit 的源码，这就导致 -[WKWebView loadRequest:] 传出的 HTTPBody 和 NSURLProtocol 传回的 HTTPBody 全都被丢弃掉了。
所以如果通过 NSURLProtocol 注册拦截 http scheme，那么由 WebKit 发起的所有 http POST 请求就全都无效了，这个从原理上就是无解的。

当然网上也出现一些解决方案，但是本人尝试没有成功。同时拦截后对ATS支持不好。再结合又使用了苹果私有API有被拒风险，最终决定弃用NSURLProtocol拦截的方案。

### WKURLSchemeHandler
iOS 11上, WebKit 团队终于开放了WKWebView加载自定义资源的API：WKURLSchemeHandler。

根据 Apple 官方统计结果，目前iOS 11及以上的用户占比达95%。又结合自己公司的业务特性和面向的用户，决定使用WKURLSchemeHandler来实现拦截，而iOS 11以前的不做处理。

着手前，要与前端统一 URL-Scheme，如：customScheme，H5网页的js、css等资源使用该scheme：customScheme://xxx/path/xxxx.css。native端使用时，先注册customScheme，WKWebView请求加载网页，遇到customScheme的资源，就会被hock住，然后使用本地已下载好的资源进行加载。

客户端使用直接上代码：

### 注册

```objc
@implementation ViewController
- (void)viewDidLoad {    
    [super viewDidLoad];    
    WKWebViewConfiguration *configuration = [WKWebViewConfiguration new];
    //设置URLSchemeHandler来处理特定URLScheme的请求，URLSchemeHandler需要实现WKURLSchemeHandler协议
    //本例中WKWebView将把URLScheme为customScheme的请求交由CustomURLSchemeHandler类的实例处理    
    [configuration setURLSchemeHandler:[CustomURLSchemeHandler new] forURLScheme: @"customScheme"];    
    WKWebView *webView = [[WKWebView alloc] initWithFrame:self.view.bounds configuration:configuration];    
    self.view = webView;    
    [webView loadRequest:[NSURLRequest requestWithURL:[NSURL URLWithString:@"http://www.test.com"]]];
}
@end
```

注意：

1. setURLSchemeHandler注册时机只能在WKWebView创建WKWebViewConfiguration时注册。
2. WKWebView 只允许开发者拦截自定义 Scheme 的请求，不允许拦截 “http”、“https”、“ftp”、“file” 等的请求，否则会crash。
3. 【补充】WKWebView加载网页前，要在user-agent添加个标志，H5遇到这个标识就使用customScheme,否则就是用原来的http或https。

### 拦截

```objc
#import "ViewController.h"
#import <WebKit/WebKit.h>

@interface CustomURLSchemeHandler : NSObject<WKURLSchemeHandler>
@end

@implementation CustomURLSchemeHandler
//当 WKWebView 开始加载自定义scheme的资源时，会调用
- (void)webView:(WKWebView *)webView startURLSchemeTask:(id <WKURLSchemeTask>)urlSchemeTask
API_AVAILABLE(ios(11.0)){
    
    //加载本地资源
    NSString *fileName = [urlSchemeTask.request.URL.absoluteString componentsSeparatedByString:@"/"].lastObject;
    fileName = [fileName componentsSeparatedByString:@"?"].firstObject;
    NSString *dirPath = [kPathCache stringByAppendingPathComponent:kCssFiles];
    NSString *filePath = [dirPath stringByAppendingPathComponent:fileName];

    //文件不存在
    if (![[NSFileManager defaultManager] fileExistsAtPath:filePath]) {
        NSString *replacedStr = @"";
        NSString *schemeUrl = urlSchemeTask.request.URL.absoluteString;
        if ([schemeUrl hasPrefix:kUrlScheme]) {
            replacedStr = [schemeUrl stringByReplacingOccurrencesOfString:kUrlScheme withString:@"http"];
        }
        
        NSMutableURLRequest * request = [NSMutableURLRequest requestWithURL:[NSURL URLWithString:replacedStr]];
        NSURLSessionConfiguration *config = [NSURLSessionConfiguration defaultSessionConfiguration];
        NSURLSession *session = [NSURLSession sessionWithConfiguration:config];
       
        NSURLSessionDataTask *dataTask = [session dataTaskWithRequest:request completionHandler:^(NSData * _Nullable data, NSURLResponse * _Nullable response, NSError * _Nullable error) {
        
            [urlSchemeTask didReceiveResponse:response];
            [urlSchemeTask didReceiveData:data];
            if (error) {
                [urlSchemeTask didFailWithError:error];
            } else {
                [urlSchemeTask didFinish];
            }
        }];
        [dataTask resume];
    } else {
        NSData *data = [NSData dataWithContentsOfFile:filePath];
        
        NSURLResponse *response = [[NSURLResponse alloc] initWithURL:urlSchemeTask.request.URL
                                                            MIMEType:[self getMimeTypeWithFilePath:filePath]
                                               expectedContentLength:data.length
                                                    textEncodingName:nil];
        [urlSchemeTask didReceiveResponse:response];
        [urlSchemeTask didReceiveData:data];
        [urlSchemeTask didFinish];
    }
}

- (void)webView:(WKWebView *)webVie stopURLSchemeTask:(id)urlSchemeTask {
}

//根据路径获取MIMEType
- (NSString *)getMimeTypeWithFilePath:(NSString *)filePath {
    CFStringRef pathExtension = (__bridge_retained CFStringRef)[filePath pathExtension];
    CFStringRef type = UTTypeCreatePreferredIdentifierForTag(kUTTagClassFilenameExtension, pathExtension, NULL);
    CFRelease(pathExtension);
    
    //The UTI can be converted to a mime type:
    NSString *mimeType = (__bridge_transfer NSString *)UTTypeCopyPreferredTagWithClass(type, kUTTagClassMIMEType);
    if (type != NULL)
        CFRelease(type);
    
    return mimeType;
}

@end
```

分析，这里拦截到URLScheme为customScheme的请求后，读取本地资源，并返回给WKWebView显示；若找不到本地资源，要将自定义 Scheme 的请求转换成 http 或 https 请求用NSURLSession重新发出，收到回包后再将数据返回给WKWebView。

## 总结

经过测试，加载速度快了很多，特别是弱网下，效果显著，谁用谁知道！WKURLSchemeHandler相比于用 NSURLProtocol 拦截的方案更可靠。
由于是优化功能，开发时也要注意添加开关，以防上线后出现问题，可以关闭开关实现降级处理。

本文是记录总结自己在开发中遇到的问题，同时也是学习NSURLProtocol和WKURLSchemeHandler的用法，加深理解，希望对你也有所帮助。

文章最后附带 腾讯Bugly的 `《WKWebView 那些坑》`<sup>[3]</sup> 以便开发时填坑。

### 参考链接

1. [移动端本地 H5 秒开方案探索与实现](https://mp.weixin.qq.com/s/0OR4HJQSDq7nEFUAaX1x5A)
2. [让 WKWebView 支持 NSURLProtocol](https://blog.moecoder.com/2016/10/26/support-nsurlprotocol-in-wkwebview/)
3. [WKWebView 那些坑](https://mp.weixin.qq.com/s/rhYKLIbXOsUJC_n6dt9UfA?)

