# 前言

YYText 是业界知名富文本框架，基于 CoreText 做了大量基础设施并且实现了两个上层视图组件：YYLabel 和 YYTextView。同其它 YYKit 组件一样，YYText 在性能方面表现优异，且功能出奇的强大，可以说是业界巅峰之作。

提起 YYText，都知道它的核心优化点：异步绘制，然而这只是冰山一角，YYText 中最为复杂和篇幅最多的是基于 CoreText 的各种计算，不得不说，源码中大量的计算很容易让人眼花缭乱。

若想深入理解 YYText 或者看懂本文，必须要了解 CoreText 基础知识并且有足够的耐心。框架代码量非常大，本文主要讲解框架基于 CoreText 的底层基础部分，不会过多的讲解 YYLabel 和 YYTextView 的细节。

## 一、框架总览

**YYText GitHub** (https://github.com/ibireme/YYText)

iOS UI 组件大都必须在主线程绘制，当绘制压力过大会造成界面卡顿，得益于多线程技术，我们可以在异步线程绘制图形从而减轻主线程压力。

YYText 核心思路：在异步线程创建图形上下文，然后利用 CoreText 绘制富文本，利用 CoreGraphics 绘制图片、阴影、边框等，最后将绘制完成的位图放到主线程显示。

![](https://upload-images.jianshu.io/upload_images/2909132-19f53ae874a301c6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

步骤看起来很简单，源码中涉及到 CoreText 和 CoreGraphics 的绘制时需要大量的代码来计算位置，这也是本文的重点之一。为了简洁易懂，笔者会略过一些技术细节，比如纵向文本布局逻辑，一些奇怪的 BUG 修复代码。

希望读者朋友优先了解 CoreText 基础 ([CoreText 官方介绍](https://developer.apple.com/library/archive/documentation/StringsTextFonts/Conceptual/CoreText_Programming/Introduction/Introduction.html#//apple_ref/doc/uid/TP40005533-CH1-SW1))，这里放上两个结构图便于理解（图会有偏差）：

![结构图1](https://upload-images.jianshu.io/upload_images/2909132-90372e1edca32ec8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![结构图2](https://upload-images.jianshu.io/upload_images/2909132-2bb13095979bb8ad.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 二、CoreText 相关工具类

### 1、YYTextRunDelegate 

在富文本中插入 key 为`kCTRunDelegateAttributeName` 的`CTRunDelegateRef`实例可以定制一段区域的大小，通常使用这个方式来预留出一段空白，后面可以填充图片来达到图文混排的效果。而创建`CTRunDelegateRef`需要一系列的函数名，使用繁琐，框架使用一个类来封装以减小使用成本：

```objc
@interface YYTextRunDelegate : NSObject <NSCopying, NSCoding>
...
@property (nonatomic) CGFloat ascent;
@property (nonatomic) CGFloat descent;
@property (nonatomic) CGFloat width;
@end


static void DeallocCallback(void *ref) {
YYTextRunDelegate *self = (__bridge_transfer YYTextRunDelegate *)(ref);
self = nil; // release
}
static CGFloat GetAscentCallback(void *ref) {
YYTextRunDelegate *self = (__bridge YYTextRunDelegate *)(ref);
return self.ascent;
}
...
@implementation YYTextRunDelegate
- (CTRunDelegateRef)CTRunDelegate CF_RETURNS_RETAINED {
CTRunDelegateCallbacks callbacks;
callbacks.dealloc = DeallocCallback;
callbacks.getAscent = GetAscentCallback;
...
return CTRunDelegateCreate(&callbacks, (__bridge_retained void *)(self.copy));
}
...
```
使用`CTRunDelegateCreate()`创建一个`CTRunDelegateRef`，同时使用`__bridge_retained`转移内存管理，持有一个`YYTextRunDelegate`对象。在该类中有数个静态函数作为回调，比如当回调`GetAscentCallback()`函数时，将持有对象的`ascent`属性作为返回值。

**注意一：**这样做似乎存在内存管理问题，`CTRunDelegateRef`实例持有的`YYTextRunDelegate`对象如何释放？
答案就在`CTRunDelegateRef`释放时会走的`DeallocCallback()`回调中，将内存管理权限转移给一个`YYTextRunDelegate`局部变量自动管理内存。

**注意二：**可以看到`CTRunDelegateCreate(&callbacks, (__bridge_retained void *)(self.copy))`代码对`self`做了一个`copy`操作 (该类的 copy 为深拷贝) ，这样做是为了什么呢？
可能第一反应是想到`CTRunDelegateRef`持有`self`的副本是为了避免循环引用，然而该方法并没有让`self`持有`CTRunDelegateCreate()`后的实例，所以也不存在循环引用问题。
实际上这里应该只是创建一个副本，当该方法返回后保证配置数据的安全性 (避免被外部意外更改)。

### 2、YYTextLine

创建一个富文本，可以拿到`CTLineRef`和`CTRunRef`以及一些结构数据 (比如`ascent descent`等)，`CTRunRef`包含的数据内容并不是很多，所以框架没有专门做一个类来包装它。使用`YYTextLine`来包装`CTLineRef`计算保存一些数据便于后面的计算，比如使用`CTLineGetTypographicBounds(...);`方法来拿到`ascent descent leading`等。

#### 计算 line 位置和大小

```objc
_bounds = CGRectMake(_position.x, _position.y - _ascent, _lineWidth, _ascent + _descent);
_bounds.origin.x += _firstGlyphPos;
```
`_position`是指 line 的`origin`点位于`context`上下文的坐标转换为`UIKit`坐标系的值，那么结合上面的结构图2分析：`_position.y - _ascent`就是 line 的最小`y`值，`_ascent + _descent`就是 line 高度（没有算上行间距 leading）。

这里最小`x`值加了一个`_firstGlyphPos`，它是当前 line 第一个 run 相对于 line 的偏移，通过`CTRunGetPositions(...);`算出，可能有一种场景，line 的`origin`位置与第一个 run 的位置有偏移（笔者并没有模拟出这种情况）。

#### 找出所有的占位 run

实际上这就是找出之前说的`CTRunDelegateRef `，框架每一个`CTRunDelegateRef `都对应了一个`YYTextAttachment`，它表示一个附件（图片、UIView、CALayer），具体实现后面会单独讲。这里只需要知道基本原理就是用`CTRunDelegateRef`占位，用`YYTextAttachment`填充。

当遍历 line 里面的 run 时，若该 run 包含了`YYTextAttachment`说明这是占位 run，那么至关重要的一步是计算这个 run 的位置和大小（便于后面将附件填充到正确位置）。

```objc
runPosition.x += _position.x;
runPosition.y = _position.y - runPosition.y;
runTypoBounds = CGRectMake(runPosition.x, runPosition.y - ascent, runWidth, ascent + descent);
```

`_position`上面已经说明了意义，`runPosition`是当前 run 相对于当前 line `origin`的偏移，那么`runPosition.x + _position.x`表示了 run 相对于图形上下文的`x`方向位置，后面同理。

最终，将这个`YYTextAttachment `附件对象和 run 位置大小信息缓存起来（后面会专门分析实现逻辑）。

### 3、YYTextContainer

创建`CTFrameRef`使用`CTFramesetterCreateFrame(...)`方法，这个方法需要一个`CGPathRef `参数，为了使用简便，框架抽象了一个`YYTextContainer `类重点属性如下：

```objc
@property CGSize size;
@property UIEdgeInsets insets;
@property (nullable, copy) UIBezierPath *path;
@property (nullable, copy) NSArray<UIBezierPath *> *exclusionPaths;
```
使用者可以简单的使用`CGSize `来制定富文本的大小，也可以用内存自动管理功能强大的`UIBezierPath `来制定路径，同时包含一个`exclusionPaths `排除路径。

```
┌─────────────────────────────┐  <------- container
│                             │
│    asdfasdfasdfasdfasdfa   <------------ container insets
│    asdfasdfa   asdfasdfa    │
│    asdfas         asdasd    │
│    asdfa        <----------------------- container exclusion path
│    asdfas         adfasd    │
│    asdfasdfa   asdfasdfa    │
│    asdfasdfasdfasdfasdfa    │
│                             │
└─────────────────────────────┘
```

CoreText 是支持镂空效果的，就是由这个 exclusion path 控制。该类的属性访问都是线程安全的，还做了一些精致的容错。

## 三、YYTextLayout 核心计算类

`YYTextLayout`包含了布局一个富文本几乎所有的信息，同时还将众多的绘制相关 C 代码放在了这个文件里面，所以这个文件非常庞大。我们先不管这些绘制代码，`YYTextLayout`主要的作用是计算各种数据，为后面的绘制做准备。

核心计算在`+ (YYTextLayout *)layoutWithContainer:(YYTextContainer *)container text:(NSAttributedString *)text range:(NSRange)range;`初始化方法中，这个方法为后面的各种查询计算打下了数据基础，接下来就分析一下这个超过 500 行的初始化方法做了些什么。

### 1、计算绘制路径和路径的位置矩形

基于`YYTextContainer`对象计算得到`CGPathRef`是主要逻辑，为了避免矩阵属性出现负值，使用`CGRectStandardize(...)`来矫正。由于 UIKit 和 CoreText 坐标系的差别，最终得到的矩阵要先做一个坐标系翻转：

```objc
rect = CGRectApplyAffineTransform(rect, CGAffineTransformMakeScale(1, -1));
cgPath = CGPathCreateWithRect(rect, NULL);
```

或者

```objc
CGAffineTransform trans = CGAffineTransformMakeScale(1, -1);
CGMutablePathRef transPath = CGPathCreateMutableCopyByTransformingPath(path, &trans);
```

它们道理是一样的，都是沿着 x 轴翻转坐标系 180°，可能有人有疑问，UIKit 转换为 CoreText 坐标系不是除了翻转 180°，还要移动一个绘制区域高度么？确实这里少做了一个操作，那是因为框架是使用`CTRunDraw(...)`遍历绘制 run，在绘制 run 之前会用`CGContextSetTextPosition(...)`指定位置（这个位置是 line 相对于绘制区域计算的），所以这个地方的 y 坐标是否正确已经没有意义了。

绘制路径的矩形大小位置`pathBox`的计算：
![](https://upload-images.jianshu.io/upload_images/2909132-24fc64fb2d357b99.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)          
比如这种情况，`pathBox =  (CGRect){50, 50, 100, 100}`，可想而知`pathBox `指的就是真正绘制区域相对于绘制上下文的位置和大小，这个数据非常有用，意味着后面计算 line 和 run 的位置时，都要加上 `cgPathBox.origin`偏移，才能真正表示 line 和 run 相对于绘制上下文的位置（比如 line 的`origin`是相对于绘制区域的一个点，而不是相对于绘制上下文）。

### 2、初始化 CTFramesetterRef 和 CTFrameRef

这一步很简单，利用两个函数就搞定：`CTFramesetterCreateWithAttributedString(...) CTFramesetterCreateFrame(...)`。值得注意的是框架支持了几个 CTFrameRef 的属性，比如`kCTFramePathWidthAttributeName`，这些属性同样是通过`YYTextContainer `配置的。

### 3、计算 line 总 frame 和行数

前面已经创建了一个富文本`CTFrameRef `，那么这里只需要遍历所有的 line 做计算，可以看到如下代码获取每一个 line 的位置大小：

```objc
// CoreText coordinate system
CGPoint ctLineOrigin = lineOrigins[i]; 
// UIKit coordinate system
CGPoint position;
position.x = cgPathBox.origin.x + ctLineOrigin.x;
position.y = cgPathBox.size.height + cgPathBox.origin.y - ctLineOrigin.y;

YYTextLine *line = [YYTextLine lineWithCTLine:ctLine position:position vertical:isVerticalForm];
CGRect rect = line.bounds;
```

`lineOrigins `是通过`CTFrameGetLineOrigins(...)`得到的，所以需要转换为 UIKit 坐标系方便计算。可以看到转换时做了一个`cgPathBox.origin`的偏移，这就是之前计算的实际绘制矩形的偏移，以此得到的`position`就是相对于图形上下文的点了，然后利用这个点初始化`YYTextLine`，前面讲了`YYTextLine`的内部实现，这里就直接得到了当前 line 的位置和大小：`rect`。

然后，利用`CGRectUnion(...)`函数将每一个 line 的`rect`合并起来，得到一个包含所有 line 的最小位置矩形`textBoundingRect`。

#### 计算 line 的行数

并不是一个 line 就占有一行，当有排除路径时，一行可能有两个 line：

![](https://upload-images.jianshu.io/upload_images/2909132-00caa33b292d8329.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

所以，需要计算每个 line 所在的行，便于为后续的很多计算提供基础，比如最大行限制。

![](https://upload-images.jianshu.io/upload_images/2909132-2dbd26fff2828ded.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当当前 line 的高度大于 last line 的高度时，若当前 line 的 y0 在 baseline 以上，y1 在 baseline 以下，就说明没有换行。

![](https://upload-images.jianshu.io/upload_images/2909132-1fd06a162d69e37e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当当前 line 的高度小于 last line 的高度时，若 last line 的 y0 在 baseline 以上，y1 在 baseline 以下，就说明没有换行。

### 4、获取行上下边界数组

```objc
typedef struct {
CGFloat head;
CGFloat foot;
} YYRowEdge;
```

声明了一个`YYRowEdge *lineRowsEdge = NULL;`数组，`YYRowEdge `表示每一行的上下边界。计算逻辑大致是这样的：

遍历所有 line，当当前 line 和 last line 为同一行时，取 line 和 last line 共同的最大上下边界：

```objc
lastHead = MIN(lastHead, rect.origin.y);
lastFoot = MAX(lastFoot, rect.origin.y + rect.size.height);
```

当当前 line 和 last line 为不同行时，取当前 line 的上下边界：

```objc
lastHead = rect.origin.y;
lastFoot = lastHead + rect.size.height;
```

最终的结果可能是这样的：

![](https://upload-images.jianshu.io/upload_images/2909132-6910ffb31cc7eb9a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

`foot1`和`head2`之间会存在一个间隙，这个间隙就是行间距，框架的处理是将这个间隙均分：
![](https://upload-images.jianshu.io/upload_images/2909132-d454692a48717fa0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 5、计算绘制区域总大小
上面已经计算了绘制路径的位置矩形`pathBox`，这只是实际绘制区域的大小，业务中若设置了`YYTextContainer`的线宽或者边距，那么实际业务需要的绘制区域总大小会更大：

![](https://upload-images.jianshu.io/upload_images/2909132-5a1fc3e2a16c6a64.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

图中蓝色填充区域即为实际绘制区域`pathBox`，绘制区域总大小应该是蓝色边框所覆盖的范围（请忽略线与线之间的小缝隙）。借助`CGRectInset(...) UIEdgeInsetsInsetRect(...)`等函数能轻易的计算出来，同样的需要用`CGRectStandardize(...)`纠正负值。

### 6、line 截断

当富文本超过限制时，可能需要对最后一行可显示的行末尾做一个省略号：`aaaa...`。

首先有一个`NSAttributedString *truncationToken;`，这个 token 可以自定义，框架也有默认的，就是一个`...`省略号，然后将这个`truncationToken`拼接到最后一个`line`：

```objc
NSMutableAttributedString *lastLineText = [text attributedSubstringFromRange:lastLine.range].mutableCopy;
[lastLineText appendAttributedString:truncationToken];
```

当然，这样`lastLineText`肯定会超过绘制区域的范围，所以要使用系统提供的方法`CTLineCreateTruncatedLine(...)`来创建自动计算的截断 line，该方法返回一个`CTLineRef`，这里转换为`YYTextLine`并且作为`YYTextLayout`的一个属性`truncatedLine`。

这也就意味着，YYText 的截断总是在富文本最后的，且只有一个。

### 7、缓存各种 BOOL 值

遍历富文本对象，缓存一系列的 BOOL 值：

```objc
void (^block)(NSDictionary *attrs, NSRange range, BOOL *stop) = ^(NSDictionary *attrs, NSRange range, BOOL *stop) {
if (attrs[YYTextHighlightAttributeName]) layout.containsHighlight = YES;
if (attrs[YYTextBlockBorderAttributeName]) layout.needDrawBlockBorder = YES;
if (attrs[YYTextBackgroundBorderAttributeName]) layout.needDrawBackgroundBorder = YES;
if (attrs[YYTextShadowAttributeName] || attrs[NSShadowAttributeName]) layout.needDrawShadow = YES;
...
};
[layout.text enumerateAttributesInRange:visibleRange options:NSAttributedStringEnumerationLongestEffectiveRangeNotRequired usingBlock:block];
```

可以猜测，`YYTextBlockBorderAttributeName `等就是 YYText 定制的富文本属性，在初始化`YYTextLayout`时就将富文本中是否包含自定义 key 缓存起来。

想象一下，若此处不使用这些 BOOL 值，那么在绘制的时候框架也需要去遍历查找是否有自定义的 key，若有再执行自定义的绘制逻辑。也就是说，这个遍历是必须要做的，要么在初始化时做，要么是绘制的时候做。

按照框架的设定，初始化`YYTextLayout `和绘制都可以在主线程也可以在异步绘制执行，所以这里的目的主要不是为了将这个遍历逻辑放入异步线程，而是为了缓存。

初始化`YYTextLayout `时缓存这些 BOOL 值过后，二次绘制就不需要再遍历了，以此达到优化性能的目的。

### 8、合并所有的附件

前面有讲到，`YYTextLine`初始化时会将所有的附件及其相关位置信息装到数组里面，那么这里遍历所有的 line 将附件相关数组合并到一起，那么之后的绘制就不需要再去遍历 line 获取附件了。

### 9、小结

除开`YYTextLayout`初始化方法，还有在`#pragma mark - Query`标记下的一系列查询方法，这些查询方法都是基于上面的初始化计算数据。至于`#pragma mark - Draw`标记下的绘制相关方法后面再说。

`YYTextLayout`初始化方法非常的长，笔者试图将这个方法分解一下，发现这样会更复杂。原因是这个初始化方法里面包含了众多的需要手动管理的内存，比如`CGPathRef CTFramesetterRef CTFrameRef`等。

可能有人会说，哪个地方需要引用计数减一，手动`release`不就行了？

但是实际情况更加复杂，因为整个初始化过程随时可能会被中断。比如`calloc(...)`开辟内存可能会失败，`CGPathCreateMutableCopy(...)`创建路径可能会失败，所以，在任何情况失败需要中断初始化时，大概会如下写：

```objc
if (failed) {
CFRelease(...);
free(...);
...
return nil;
}
```

而且这个地方你必须要将前面所有手动管理的内存释放掉，当这个代码过多的时候，可能会让你疯掉。

所以作者用了一个很巧的方法，使用`goto`：

```objc
fail:
if (cgPath) CFRelease(cgPath);
if (lineOrigins) free(lineOrigins);
...
return nil;
```
那么，当某个环节失败时，直接这么写：
```
if (failed) {
goto fail;
}
```

这个场景下，`goto`的使用确实非常适合。

## 四、自定义富文本属性

前面有提到 YYText 定制的富文本属性，

我们知道，`NSMutableAttributedString`对象使用`addAttribute:value:range:`等一系列方法可以添加富文本效果，这些效果有三个要素：名字 (key)、值 (value)、范围。YYText 也拓展了一些自己的名字 (YYTextAttribute 文件)：

```objc
UIKIT_EXTERN NSString *const YYTextAttachmentAttributeName;
UIKIT_EXTERN NSString *const YYTextHighlightAttributeName;
...
```

当然为这些 key 都创建了对应的 value (类)，比如`YYTextHighlightAttributeName`对应`YYTextHighlight`。但是这些自定义的 key CoreText 是识别不了的，那么框架内部是如何处理的呢？

```objc
NSDictionary *attrs = (id)CTRunGetAttributes(run);
id anyValue = attrs[anyKey];
if (anyValue) { ... }
```
很简单，实际上就是遍历富文本，通过上面这段代码就能找到某个 run 是否包含自定义的 key，然后做相应的绘制逻辑。

### 1、图文混排实现

YYText 大部分的自定义属性都算是“装饰”文本，所以只需要绘制的时候判断有没有包含对应的 key，若包含就做相应的绘制逻辑。但是有一个自定义属性比较特殊：

```objc
YYTextAttachmentAttributeName : YYTextAttachment
```

因为这个是添加一个附件 (UIImage、UIView、CALayer)，所以需要一个空位，那么设置这个自定义属性的时候还需要设置一个`CTRunDelegateRef`：

```objc
NSMutableAttributedString *atr = [[NSMutableAttributedString alloc] initWithString:YYTextAttachmentToken];

YYTextAttachment *attach = [YYTextAttachment new];
attach.content = content; // UIImage、UIView、CALayer
...
[atr yy_setTextAttachment:attach range:NSMakeRange(0, atr.length)];

YYTextRunDelegate *delegate = [YYTextRunDelegate new];
...
CTRunDelegateRef delegateRef = delegate.CTRunDelegate;
[atr yy_setRunDelegate:delegateRef range:NSMakeRange(0, atr.length)];
```

#### (1) 对齐方式

图文混排添加图片时，业务中往往有很多对齐方式，如何来对齐通过调整`CTRunDelegateRef `的`ascent descent`来控制，框架对其方式有三种：居上，居下，居中。

居上：

![](https://upload-images.jianshu.io/upload_images/2909132-fa0d88d55964dfcc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

让占位 run 的`ascent`始终等于文本的`ascent` (若占位 run 太矮则贴着 baseline) 。

居下：

![](https://upload-images.jianshu.io/upload_images/2909132-e73a06eee977bf9c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

让占位 run 的`descent`始终等于文本的`descent ` (若占位 run 太矮则贴着 baseline) 。

居中：

![](https://upload-images.jianshu.io/upload_images/2909132-241c96962d4b26cb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

居中的计算相对复杂，需要让占位 run 的中点和文本的中点对齐 (如图)，那么图中`yOffset + (占位 run 的 height) * 0.5` 就等于占位 run 的`ascent`  (若占位 run 太矮则贴着 baseline) 。

当然，上面图中的图片可以为`UIView CALayer`。到目前为止，占位 run 的位置已经确定了，接下来就需要把 `UIImage UIView CALayer`绘制到相应的空位上了。

#### (2) 绘制附件

绘制的逻辑在`YYTextLayout`下的方法`YYTextDrawAttachment(...)`，对于`UIImage`图片的附件，还能设置`UIViewContentMode`，会根据一开始设置的占位 run 的大小做图片填充变化，然后调用 CoreGraphics API 绘制图片：

```objc
CGImageRef ref = image.CGImage;
if (ref) {
CGContextSaveGState(context);
CGContextTranslateCTM(context, 0, CGRectGetMaxY(rect) + CGRectGetMinY(rect));
CGContextScaleCTM(context, 1, -1);
CGContextDrawImage(context, rect, ref);
CGContextRestoreGState(context);
}
```

若附件的类型是`UIView CALayer`，那分别就需要额外的传入父视图、父 layer：`targetView  targetLayer`，然后的操作就是简单的将`UIView`添加到`targetView`上或者将`CALayer`添加到`targetLayer `上。

### 2、点击高亮实现

```objc
YYTextHighlightAttributeName : YYTextHighlight
```

`YYTextHighlight`包含了单击和长按的回调，还包括一些属性配置。在`YYLabel`中，通过下列方法来写触发逻辑：

```objc
- (void)touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event;
- (void)touchesMoved:(NSSet *)touches withEvent:(UIEvent *)event;
- (void)touchesEnded:(NSSet *)touches withEvent:(UIEvent *)event;
``` 

涉及到判断点击的`CGPoint`点对应富文本中的具体位置，所以有很多复杂的计算，这里不展开了。

当找到了应该触发的`YYTextHighlight `，更换具体的`YYTextLine`为高亮状态的`YYTextLine`，然后重绘。当手松开时，切换会常态下的`YYTextLine`。

这就是点击高亮的实现原理，实际上就是替换`YYTextLine`更新布局。

# 五、异步绘制

上面介绍了几种特殊的自定义富文本属性，对于其它的自定义属性，基本上都是使用 CoreGraphics API 绘制，比如边框、阴影等，当然 CoreText 自带有很多效果，YYText 做了一些改良和拓展。

可以看到绘制方法都会带有一个是否取消的 Block，比如`static void YYTextDrawShadow(YYTextLayout *layout, CGContextRef context, CGSize size, CGPoint point, BOOL (^cancel)(void));`。这个`cancel`就是用来判断是否需要取消本次绘制，这样就能在一次绘制的任意位置中断，及时的取消无用的绘制任务以提高效率。

YYText 富文本可以异步绘制，也可以在主线程绘制，创建布局类及其相关计算可以在任意线程，可以根据业务需求选择适合的策略。

具体实现有些复杂，所以关于异步绘制的具体原理可以看笔者专门的一篇博客：

**YYAsyncLayer 源码剖析：异步绘制**(https://www.jianshu.com/p/154451e4bd42)

YYAsyncLayer 就是从 YYText 里面提取出来的组件，核心就是一个支持异步绘制的`CALayer`子类，相信看完 YYAsyncLayer 的解析会对异步绘制有较深的认识。

# 后语

YYText 确实过于重量，本文只是对基础部分取重点做了解析，除此之外还有非常多的计算和逻辑，感兴趣可以自行研究。

从代码质量来看，YYText 几乎无可挑剔，细节处理非常棒，逻辑代码很精炼，笔者尝试过重写部分逻辑代码，发现优化半天又回到了源码的写法 😂，不得不佩服作者的功底。

至此，笔者已经阅读了 YYKit 大部分源码，曾多次被作者的代码技巧所折服，几乎每一句代码都经得起推敲，笔者也更加深刻的理解了性能优化，明白了优化要从细节做起。

突然想起了笔者和一位好友的笑梗。每逢佳时：

“这确实是一个非常巧妙且令人兴奋的技巧”。

### 系列文章

* YYText 源码剖析：CoreText 与异步绘制
	https://www.jianshu.com/p/b9ac1b5d8f01
* YYAsyncLayer 源码剖析：异步绘制
	https://www.jianshu.com/p/154451e4bd42
* YYCache 源码剖析：一览亮点
	https://www.jianshu.com/p/408d4d37bcbd
* YYModel 源码剖析：关注性能
	https://www.jianshu.com/p/fe30e6bbc551
* YYImage 源码剖析：图片处理技巧
	https://www.jianshu.com/p/43ac91be0cf4
* YYWebImage 源码剖析：线程处理与缓存策略
	https://www.jianshu.com/p/dd5537fa0caf


