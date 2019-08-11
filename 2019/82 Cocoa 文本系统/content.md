## 文字排版

在开始文本系统介绍之前，我们先了解一下文字是怎么排版的，而要了解文字的排版就必须先有一些基本概念。

我这里只做简单地介绍，具体请参考：*Typographical Concepts*<sup>[1]</sup>

### 字符（Characters）与字形（Glyphs）

![1](http://)

上图表示的是连字（Ligatures），连字由字符 "f" 以及字符 "l" 组成，它们组合后成为一个字形（Glyph）。
与此类似的还有 "é"，它由 "e" 与 "´" 组合而成。

可以看到，字符与字形不一定是一一对应的关系，当然在一般情况下，它们可以看做是一一对应的。

**字符编码**

计算机通过编码表将字符存储为数字。在 Cocoa 平台的编码方案为 Unicode 标准。Unicode 标准为世界上每种现代书面语言中的每个字符提供了一个惟一的数字，其独立于所使用的平台、程序和编程语言。这个通用标准解决了一个长期存在的问题，即不同的计算机系统使用数百种相互冲突的编码方案。它还具有简化处理双向文本和上下文表单的功能。

**字形结构**

![2](http://)

### 字体（Fonts）

上面介绍了字符与字形的关系，那么它们的关系具体又是什么呢？这就需要用到字体了。

![3](http://)

字符加字体可以得到字形，在 Cocoa 中我们通过字体可以得到 CGGlyph，渲染的时候我们使用 CTFont 的方法传入 CGGlyph 就可以渲染出实际的文字。

## 文本系统架构

![4](http://)

无论是 macOS 还是 iOS，苹果的文本系统的架构都是一样的，如上图所示。
在 Typesetter 以及 Glyph generator 之下是 CoreText，所以系统的整个文本系统是构建在 CoreText 之上。

> 在 iOS 平台，系统隐藏了 Typesetter、 Glyph generator。

整个系统遵循 MVC 的架构设计：

- Model：`NSTextStorage`、`NSTextContainer`；
- View：在 macOS 是 `NSTextView`，在 iOS 是 `UITextView`；
- Controller：`NSLayoutManager`。

### 类职能简介

- `NSTextStorage` 保存富文本数据；
- `NSTextContainer` 提供布局区域；
- `TextView` 真实地展示文本；
- `NSLayoutManager` 来管理所有的布局以及缓存布局信息，其持有 `NSGlyphGenerator` 与 `NSTypesetter` 实例，其中 `NSGlyphGenerator` 用来生成 Glyph，`NSTypesetter` 进行具体的排版操作。

> `NSTypesetter` 是一个抽象类，`NSLayoutManager` 默认使用 `NSATSTypesetter`（Apple Type Services (ATS)）进行排版。

### 常见配置

![5](http://)

![6](http://)

一个 `NSTextStorage` 可以配置多个 `NSLayoutManager`，一个 `NSLayoutManager` 可以配置多个 `NSTextContainer`，每个 `NSTextContainer` 可以关联一个 `NSTextView `。

所以我们可以很方便地实现这些功能：
- 电子书阅读器：一个 `NSTextStorage`，一个 `NSLayoutManager`，`NSLayoutManager` 管理多个 `NSTextContainer`；
- 附带实时预览功能的 Markdown 编辑器：一个 `NSTextStorage`，多个 `NSLayoutManager`，每个 `NSLayoutManager` 管理一个 `NSTextContainer`。

## CoreText 

以上文本系统称之为 `TextKit`, 而整个 `TextKit` 基于 `CoreText` 构建。

![7](http://)

目前流行文本框架如 *TTTAttributedLabel*<sup>[2]</sup>、*YYText*<sup>[3]</sup> 都是基于 `CoreText` 进行开发，并且直接使用 `CTFramesstter` 相关接口。

`CTFramesstter` 内部使用 `CTTypesetter` 进行文字排版，`CTTypesetter` 可以生成 `CTLine`，`CTLine` 由多个 `CTRun` 组成，而 `CTRun` 由具有相同 `attributes` 的文字组成。最终，多个 `CTLine` 合成为 `CTFrame`。

![8](http://)

图片来源：blog.devtang.com

> 1. 绝大部分场景我们首先都应该基于更高层的 `TextKit` 进行开发，尽量避免对底层 `CoreText` 的使用。并且需要注意的是，直接使用 `CoreText` 与 `TextKit` 进行渲染的效果是不一致的，这是由于 `CoreText` 与 `TextKit` 的 fix attributes 是不完全一致的，并且它们在排版细节可能也会有差异（依赖于 `TextKit` 的实现）；
> 2. 由于 `TextKit` 基于 `CoreText`，所以无需担心性能问题，并且其更易使用与扩展。

## UILabel 的实现

通过 `Instruments` 查看 `UILabel` 的调用栈，我们知道其实际基于 `TextKit` 实现。见下图：

![9](http://)

![10](http://)

![11](http://)

可以看到 `UILabel` 首先会调用 `-[NSConcreteMutableAttributedString fixAttributesInRange:]`，然后使用 `_NSStringDawingEngine` 进行文本大小计算以及渲染。

并且可以发现，我们常用的文本大小计算方法 `-[NSAttributedString(NSExtendedStringDrawing) boundingRectWithSize:options:context:]` 也是基于 `TextKit` 实现。

## FixAttributes

`TextKit` 在进行文本排版之前，都会先对 `NSTextStorage` 执行 `fixAttribtesInRange:` 方法。而这个方法可能是非常耗时的，所以有时候也会造成 `TextKit` 性能不好的假象。

那么为什么需要进行这步操作呢？我们观察到 `fixAttribtesInRange:` 方法实际执行了另外 3 个方法，分别是：

1. `fixFontAttributeInRange:`
2. `fixParagraphStyleAttributeInRane:`
3. `fixGlyphInfoAttributeInRange:`

结合文档 *fixAttribtesInRange 方法介绍*<sup>[4]</sup>，我们知道，其只要是为了修复一些不正常 `attributes`，例如：

- 文字设置了不正确的字体，例如不能为汉字和阿拉伯字符分配Times-Roman字体，修复后会为它设置适合的字体；
- 为非 `NSAttachmentCharacter` 添加了 `NSAttachmentAttribute`，修复后会删除掉错误的 `NSAttachmentAttribute`；
- etc.

> 请注意：`TextKit` fallback 到其他的字体，系统会为 `NSTextStorage` 添加 key 为 `NSOriginalFont`，value 为原始字体的 `attribute` ，但是排版依然会使用原始字体进行排版，也就是说文本计算的大小依然是使用原始字体计算。

## TextKit 踩坑

- `UILabel` 当只有一行时候如果设置了 linespacing，linespacing 仍然会生效，这种场景其实我们是不希望有多余的 linespacing；
- `UILabel` 没有使用 `FontLeading` 进行排版；
- 不能自定义截断文本（TruncationToken），系统内部默认截断文本为三个点：`UTF16Char ellipsis = 0x2026`，不过能参考 *Texture*<sup>[5]</sup> 实现自定义截断；
- 直接使用 `TextKit`，当 `NSTextContainer` 设置了 maxNumberOfLines 文本产生截断的时候，同 `UILabel`，最后一行会有多余的 linespacing，解决方案参考：*Neat*<sup>[6]</sup>。

## 总结

1. 介绍了文字排版的基础：字符、字形、字体，字符 + 字体 -> 字形；
2. 介绍了 Cocoa 文本系统 `TextKit` 的架构，系统遵循 MVC 的架构：
	- Model：`NSTextStorage` 保存富文本数据，`NSTextContainer` 提供布局区域；
	- View：在 macOS 是 `NSTextView`，在 iOS 是 `UITextView`，负责真实地展示文本；
	- Controller：`NSLayoutManager`，负责文本布局的管理。
3. 介绍了 `TextKit` 的底层技术支持：`CoreText`，它是先进的布局文本和处理字体的底层技术；
4. 介绍了 `UILabel`，其内部实现基于 `TextKit` 提供的高性能、高质量排版引擎；
5. 介绍了为什么需要 `FixAttributes`；
6. 介绍了使用 `TextKit` 的一些踩坑经历以及其对应的解决方案。

### 参考文档

* [1] https://developer.apple.com/library/archive/documentation/TextFonts/Conceptual/CocoaTextArchitecture/TypoFeatures/TextSystemFeatures.html#//apple_ref/doc/uid/TP40009459-CH6-64585
* [2] https://github.com/TTTAttributedLabel/TTTAttributedLabel
* [3] https://github.com/ibireme/YYText
* [4] https://developer.apple.com/documentation/foundation/nsmutableattributedstring/1533823-fixattributesinrange
* [5] https://github.com/texturegroup/texture
* [6] https://github.com/leavez/Neat
* [7] https://developer.apple.com/library/archive/documentation/StringsTextFonts/Conceptual/CoreText_Programming/Introduction/Introduction.html#//apple_ref/doc/uid/TP40005533-CH1-SW1
* [8] https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/TextAttributes/TextAttributes.html#//apple_ref/doc/uid/10000088-SW1
* [9] https://developer.apple.com/library/archive/documentation/TextFonts/Conceptual/CocoaTextArchitecture/Introduction/Introduction.html#//apple_ref/doc/uid/TP40009459
* [10] https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/TextLayout/TextLayout.html#//apple_ref/doc/uid/10000158-SW1
* [11] https://developer.apple.com/library/archive/documentation/StringsTextFonts/Conceptual/TextAndWebiPhoneOS/Introduction/Introduction.html#//apple_ref/doc/uid/TP40009542-CH1-SW1

