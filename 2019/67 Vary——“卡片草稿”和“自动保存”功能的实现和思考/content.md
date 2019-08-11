## 前言

Vary 是一款我十分喜欢的工具，官方 slogan 为“分享不拘一格”，是“轻量级的社交网络，重量级的创造工具”。从去年 8 月 9 日加入 Vary 开发组到现在已经整整过去半年了……在 17 年初时，从知乎上得知到了 Dandy Weng 对 Vary 的宣传。没记错的话，当时应该是在上软件工程课，刷知乎看到了 *这篇文章*<sup>[1]</sup>。刷完后我又把作者的所有相关信息都看了一遍，赶紧申请了 TestFlight 的测试资格，非常幸运的申请到了。

一打开 Vary app，当时直接抓了坐在旁边的妹子来一同把玩，瞬间被 Vary 捕获了芳心！我从未想到 app 还可以这么做！交互体验还可以这么好！UI 实在太精美了！可以说直到现在，我都没遇到第二个能够让我发出如此赞叹的 app。我持续使用到了现在，并且也加了作者的微信，在使用过程中一直在持续不断的给作者报 bug，就是因为太喜欢了，所以报 bug 报到去年 8 月 9 日，我发现了一个复现周期比较长的 bug，Dandy 直接邀请我加入了开发团队！大家感兴趣可以阅读 *这篇文章*<sup>[2]</sup>。

当时我正在搬砖，直接从椅子上喊了一声站了起来！我居然可以加入 Vary 的开发了！我就要加入自己十分喜爱的 app 开发工作了！后来，我们对接了目前的开发工作，当时确定要进行的内容有：

* 文字编辑时插入新版 `Emoji` 保存时闪退；（这个 bug 让我好几次编辑的长文都没了）
* 首页信息流缓存；
* 消息持久化；
* 卡片草稿；
* iPad 适配；
* iOS 12 兼容性测试；

刚开始我负责的是修复插入新版 Emoji 时保存的闪退问题，最后因为工作事情太多了，这部分工作又交回给 Dandy 了哈哈哈～过了一段时间后，开始优化首页信息流卡片高度缓存的问题，Vary 在架构上还是有一定的细节问题存在的，导致前期看代码时有些恍惚，后来 Dandy 几乎是每周都来问我进度，但当时正值秋招火热时，忙着三方和课设等各种琐碎的事情，每天只有额外多余的两个小时的空余时间来做自己的事情，所以最后实在被 Dandy 问烦了直接逼着自己弄完了首页信息流卡片缓存的一期优化，但偶尔还是会出现高度不准确的问题，只能放到二期优化中去做了。

后来时间实在是推不开，跟 Dandy 说了等到放了寒假再继续跟进。我以为 Dandy 会“放过”我了哈哈哈，从寒假到现在磕磕碰碰的做出了“卡片草稿”和“定时保存”两个主需求。在下文中，我将在脱离 Vary 核心代码的前提下讲解开发这两个有趣的需求时遇到的问题和思考。

## 梳理需求

### 卡片草稿

卡片草稿的需求比较容易梳理，但因为对相关逻辑代码的不熟悉导致开发流程修改了好几次，最后确定的开发流程是这样的：

* 在选择“✗”和“✓”，点击“✗”后弹出的菜单中，新增一个 action “存为草稿”，回调中处理卡片草稿相关逻辑；
* 下次点击“+”时，先判断是否有草稿（缓存），如果有直接展示“模块编辑”页面，并载入缓存模块，如果没有则走原逻辑；

### 定时保存

定时保存跟 Dandy 讨论了好几次，在讨论的过程中发现之前写好的“卡片草稿”功能的实现思路不是正确的，导致又继续返工到现在。最后我们讨论的结果是：

* “文字编辑”模块采用类似“石墨”等在线编辑文档应用的模式，直接缓存用户编辑内容；
* 其它如“视频”、“语音”和“位置”等模块因为操作逻辑较为简单且不需要长时间留存，所以在触发各个模块的“确定”事件时再进行缓存。

## 熟悉代码

对 Vary 的开发并没有一直在持续，只是有时间就修修补补。针对上文中所梳理出的两个需求，着重看了 Vary 中相关逻辑，让我感到意外的是，Vary 中对卡片的渲染并不是利用 Native 能力进行渲染的，我说怎么有时候感觉不跟手，以为这个有点“卡顿”的情况只是个小问题，没想到当我看到了*这篇文章*<sup>[3]</sup>后，深深的感受到是我错了，原来 webKit 里连这都有点小问题。

接下来继续熟悉代码。发现了原来每一张卡片都是 WKWebView，直接渲染了下载的 HTML 字符串，而且在 Native 中做了不少 JS 代码注入的东西，看到这里后我内心开始有些许的失落，因为并没有看到我想看的东西，Vary 中的自动排版引擎实际上并不是我想的那么“高大上”，简单来说是下了很多“苦功夫”，没有很多吸引我眼球的地方。不过看到了一个让我惊喜到笑出了声的实现。

在 Dandy 的文章中对 Vary 的宣传点其中有一部分是这样的：

> 同时我也在不断研究如何通过技术手段来引导交互层面的创新，例如 Vary 的 iOS App 可以检测手指接触屏幕的面积，并以此来调整卡片的滚动速度：用指尖滑动时，一次只会滚动一张卡片；整个手指贴在屏幕上滑动时，则会根据你滑动速度的惯性来连续滚动多张卡片。这听起来也许有些玄乎，却非常容易上手，也很实用。

当我翻到了相关实现后，真的就惊喜的笑了出来，至于为什么会这般惊喜的笑了出来，该功能核心是利用了 `touch` 对象的 `majorRadius` 属性，如果你真的很想推测出 Vary 具体的实现到底是怎么样的，推荐你仔细把玩十几分钟后也能够猜的出来。

随后，我开始研究“模块编辑”页面的代码，该模块中的代码逻辑十分清晰，嗯，就是你现在脑海中直觉冒出来的那个实现思路。Vary 中最有趣的地方莫过于看各个模块的实现了，其中最让我感到兴奋的是“图片”和“语音”模块，又仔细研究一番后，我都已经摩拳擦掌的给自己心理准备，一上午就好好研究这两个模块的实现好了，但等我都看完具体实现后，一看时间，原来只过去了半小时。如果有经常关注我 github 的同学，“语音”模块是用了开源库。“图片”模块之所以能够在我内心中占据这么大的份额，就是因为它精美的 UI，本以为实现也很完美，但实际上让我惊叹的是之前开发同学的新奇思路而已，这点最感到失望！

## 正式开发

零零碎碎的熟悉了相当长的时间后，开始在返工和继续开发以及做了一些微小的重构工作中来来回回，原本想着直接给 Vary 来一波大的重构，但想到现在这个时间点还是先不要给自己找太多不必要的事情了。

### 卡片草稿

卡片草稿的核心围绕着 `NSFileManager` 进行展开。因为只保存单一数据，没有必要上 `Core Data`，但在首页卡片信息流的后续开发中有涉及到缓存一系列相同数据的地方，而且还会涉及到部分卡片的更新，如果这个时候再使用 `NSFileManager` 进行缓存数据的管理就会显得有些拘谨，此时使用 Core Data 是一件再适合不过的事情。

首先，针对 Vary 中的缓存策略需要涉及的逻辑写了封装了一个简单的基于 `NSFileManager` 的管理类 `PJCache`，这与核心业务无关，具体实现如下：

**PJCache.h**

```
//
//  PJCache.h
//  Vary
//
//  Created by PJHubs on 2019/2/10.
//  Copyright © 2019 Vary iOS Team. All rights reserved.
//


#import <Foundation/Foundation.h>

@interface PJCache : NSObject

// 创建文件
+(BOOL)cacheFileSet:(NSString*)fileName contents:(NSString*)contents;

// 通过 NSData 创建缓存文件
+(BOOL)cacheFileSetNSdata:(NSString*)fileName contents:(NSData*)contents;

// 读取缓存文件，返回 NSData
+(NSData*)cacheFileGetNSdata:(NSString*)fileName;

// 缓存文件 是否存在
+(BOOL)cacheFileExists:(NSString*)fileName;

// 缓存文件 删除
+(BOOL)cacheFileDelete:(NSString*)fileName;

@end
```

**PJCache.m**

```objc
//
//  PJCache.m
//  Vary
//
//  Created by PJHubs on 2019/2/10.
//  Copyright © 2019 Vary iOS Team. All rights reserved.
//

#import "PJCache.h"

@implementation PJCache

// 创建文件
+(BOOL)cacheFileSet:(NSString*)fileName
           contents:(NSString*)contents {
    NSError *err;
    NSString *path = [self cacheFilePath:fileName];
    NSFileManager *fm = [NSFileManager defaultManager];
    if ([fm fileExistsAtPath:path]) {
        [fm removeItemAtPath:path error:&err];
    }
    return [fm createFileAtPath:path
                       contents:[contents dataUsingEncoding:NSUTF8StringEncoding]
                     attributes:nil];
}

// 通过 NSData 创建缓存文件
+(BOOL)cacheFileSetNSdata:(NSString*)fileName
                 contents:(NSData*)contents {
    NSError *err;
    NSString *path = [self cacheFilePath:fileName];
    NSFileManager *fm = [NSFileManager defaultManager];
    if ([fm fileExistsAtPath:path]) {
        [fm removeItemAtPath:path error:&err];
    }
    return [fm createFileAtPath:path
                       contents:contents
                     attributes:nil];
}

// 读取缓存文件，返回 NSData
+(NSData*)cacheFileGetNSdata:(NSString*)fileName {
    NSString *path = [self cacheFilePath:fileName];
    NSFileManager *fm=[NSFileManager defaultManager];
    if ([fm fileExistsAtPath:path]) {
        return [fm contentsAtPath:path];
    }
    return nil;
}

// 判断缓存文件是否存在
+(BOOL)cacheFileExists:(NSString*)fileName {
    NSString *path = [self cacheFilePath:fileName];
    NSFileManager *fm=[NSFileManager defaultManager];
    return [fm fileExistsAtPath:path];
}

// 删除缓存文件
+(BOOL)cacheFileDelete:(NSString*)fileName {
    NSString *path = [self cacheFilePath:fileName];
    NSFileManager *fm=[NSFileManager defaultManager];
    if ([fm fileExistsAtPath:path]) {
        NSError *err;
        return [fm removeItemAtPath:path error:&err];
    }
    return YES;
}

@end
```

写入缓存时需要注意的时如缓存数据过大，注意必须进行异步现场的写入和读写。卡片草稿一开始根本没有考虑太多，根据实际业务逻辑调整好相关新增代码后，缓存的时预览卡片时接收到渲染完成的 html 字符串。当时 Dandy 还提了一个细节问题，“当存在缓存时，用户下次再创建卡片时应该直接载入上一次未发布卡片内容”，这句话我当时并没有理解得很好，以为要做到用户一点击“+”时，要以最快速度加载渲染完成的卡片。实际上的应该要做的对卡片哥哥模块进行 `encoder`，完成后直接把归档完成的“模块”对象数组进行缓存。

### 自动保存

这块功能当时因为在完成“卡片草稿”需求时走错了路子，导致在此基础上进行的“自动保存”功能开发做设计时搞了比较复杂的一套流程，因为当时最终的目的要的是卡片渲染完成的 html 字符串，这就导致了“自动保存”时要每次都以拿到 html 字符串为“保存”成功的标志，这更加耗费服务器资源，幸亏在跟 Dandy 多次讨论的过程中被及时制止了。

“自动保存”功能的触发在“模块编辑”页面，主要分为两种情况：

* 在“文件编辑”页面，利用 GCD 每隔 3s 针对用户输入的内容做“清洗判断”，保证每次缓存时的内容与上一次缓存的内容完全不一致且一定不为空。在触发“确定”事件时，带上最终用户输入的文本内容以及停止缓存且立即更新缓存至最新，以免出现用户在刚好完成上一次缓存 3s 间隔时，立马输入了个标点符合后直接退出，而导致文本内容的缺失。
* 其它模块的“自动保存”上文也以及说了大概，因为用户不会长时间滞留在非“文本”模块中的其它模块（不排除遗忘可能），所以这块比较简单粗暴，在各个模块的“确定”事件中，各自触发一次缓存。

“自动保存”功能如果需要保存的数据结构较简单，同样可以直接上 NSFileManager。

## 总结

完成以上两个内容后，这次的版本迭代的主要任务也就完成了。因为每天只有区区不到三个小时的完整时间去做迭代维护，很多细节还是没能考虑好，Vary 依然有很多需要优化的地方，但因时间关系导致实在是难以投入大量的时间，真希望能有一个月的完整时间来对 Vary 好好的做一个大的维护呢～

真心希望能够给大家带来体验更好的 Vary，找到一个能够认真分享自己内心世界的地方。

### 参考

1. https://zhuanlan.zhihu.com/p/25751703
2. https://blog.dandyweng.com/2017/02/nine-months-of-work/
3. https://blog.csdn.net/weixin_40785245/article/details/80594023

