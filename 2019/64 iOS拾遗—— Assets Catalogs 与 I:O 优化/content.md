早在 XCode 5，苹果引入了 Assets Catalogs ，它作为一个重要的开发组件，能够让开发者可以更方便的管理项目内的图片资源。

苹果也在不断的完善它的功能：

* XCode 9 中添加了对颜色、矢量图、PDF等的支持（_WWDC 2017 Session What’s New in Cocoa_<sup>[1]</sup> ）
* XCode 10 中添加了对 `High Efficiency Image` 和Mojave dark mode的支持（_WWDC 2018 Session Optimizing App Assets_<sup>[2]</sup> )

那么相比直接存储在根目录下，究竟 Assets Catalogs 有什么自己独特的优势呢？在 WWDC 2016 上提到的 _I/O 优化_<sup>[3]</sup>是怎么完成的？`imageName:`、`imageWithContentOfFile:`这些方法在不同情况下又有什么表现呢，这篇文章就是基于这种种疑问诞生的。

> 太长不看版：
>
> Assets Catalogs 将会在编译时生成一个.car文件，并在其中包含了这个图像加载所需的一切数据，当图像需要加载的时候，可以直接获取其中的数据并进行加载。

## 从一次 I/O 优化说起

相信大家现在在项目里面都会使用 Assets Catalogs 对图片资源进行管理，但很不幸，我接手的项目依然是把图片放在 Folder 中，这样看起来似乎并没有什么问题，但是如果打开 Time Profile ，就会发现把图片放在 Folder 中并使用`imageName:`加载图片所用的耗时要比放在 Assets Catalogs 中要慢得多。

**保存在 Folder ，并使用`imageName:`获取：**

![1](http://)

展开后的调用栈耗时：

![2](http://)

**保存在 Assets Cataglogs ，并使用`imageName:`获取：**

![3](http://)

展开后的调用栈耗时：

![4](http://)

而如果使用`imageWithContentOfFile:`，则两种存储方式所用的耗时则相同

**使用imageWithContentOfFile:获取：**

![5](http://)

由这几个案例，我们可以推断出：

1. 保存在 Folder 中**并不会导致查找时间的增加**，因为在`imageWithContentOfFile:`中两者加载图片的耗时一致
2. 使用`imageName:`加载图片时，两种存储方式**都调用**了底层 CoreUI.framework 的框架，但是调用的方法有所不同
3. 存储在 Folder 中的图片加载时生成的是`CUIMutableStructuredThemeStore`，而存储在 Assets Catalogs 中则是生成`CUIStructuredThemeStore`
4. `CUIMutableStructuredThemeStore`与`CUIStrucetedThemeStore`都调用到一些带有rendition字眼的类，而`CUIMutableStrucetedThemeStore`还多了一层`canGetRenditionWithKey:`的方法调用，导致了耗时的增加

从上面这些推断，我们可能会产生以下的一些问题：

* CoreUI.framework 在加载图片中负责了什么工作？
* `CUIMutalbeStructuredThemeStore`与`CUIStructuredThemeStore`是什么东西？
* rendition又是什么东西？
* 为什么 Assets Catalogs 能够提高这么多加载速度呢？
* `imageWithContentOfFile:`不对图像进行缓存，是否这个原因导致其加载速度要比`imageWithName:`要快呢？

针对这些问题，我们一个一个解决。

## 探秘 Assets Catalogs 与 .car 文件

在研究这些问题之前，我们先来从新认识一下 Assets Catalogs。

关于 Assets Catalogs ，它_详细的使用方法_<sup>[4]</sup>相信大家已经很熟悉了，苹果也在_Asset Catalog Format Reference_<sup>[5]</sup>中给出了`.xcassets`的组成。

但是可能很少人知道在 XCode 编译过程中，保存在 Assets Catalogs 中的图像资源并不是简单的复制到 APP 的 Bundle 中，而是会在编译时生成一个将资源打包并生成索引的`.car`文件，而它在苹果开发者文档上并没有介绍，在网上关于它的信息也是少之又少。

**那么`.car`文件究竟是什么？**

要知道`.car`文件究竟是什么，有什么作用，我们可以先看看它包含了什么。所以我在 Assets Catalogs 中放入了一组PNG文件：

![6](http://)

随后在 XCode 中对项目进行编译，在生成的 APP 包中我们可以找到编译完成的`.car`文件。利用 AssetCatalogTinkerer 我们可以看到在`.car`文件中，包含了各种**图像资源**：@1x的、@2x的、@3x的。而利用 XCode 自带的 `assetutil` 则能够分析.car文件：

```sh
sudo xcrun --sdk iphoneos assetutil --info ./Assets.car > ./Assets.json
```

并输出一份`json`文档：

```json
[
  {
    "AssetStorageVersion" : "IBCocoaTouchImageCatalogTool-10.0",
    "Authoring Tool" : "@(#)PROGRAM:CoreThemeDefinition  PROJECT:CoreThemeDefinition-346.29\n",
    "CoreUIVersion" : 498,
    "DumpToolVersion" : 499.1,
    "Key Format" : [
      "kCRThemeAppearanceName",
      "kCRThemeScaleName",
      "kCRThemeIdiomName",
      "kCRThemeSubtypeName",
      "kCRThemeDeploymentTargetName",
      "kCRThemeGraphicsClassName",
      "kCRThemeMemoryClassName",
      "kCRThemeDisplayGamutName",
      "kCRThemeDirectionName",
      "kCRThemeSizeClassHorizontalName",
      "kCRThemeSizeClassVerticalName",
      "kCRThemeIdentifierName",
      "kCRThemeElementName",
      "kCRThemePartName",
      "kCRThemeStateName",
      "kCRThemeValueName",
      "kCRThemeDimension1Name",
      "kCRThemeDimension2Name"
    ],
    "MainVersion" : "@(#)PROGRAM:CoreUI  PROJECT:CoreUI-498.40.1\n",
    "Platform" : "ios",
    "PlatformVersion" : "12.0",
    "SchemaVersion" : 2,
    "StorageVersion" : 15
  },
  {
    "AssetType" : "Image",
    "BitsPerComponent" : 8,
    "ColorModel" : "RGB",
    "Colorspace" : "srgb",
    "Compression" : "palette-img",
    "Encoding" : "ARGB",
    "Idiom" : "universal",
    "Image Type" : "kCoreThemeOnePartScale",
    "Name" : "MyPNG",
    "Opaque" : false,
    "PixelHeight" : 28,
    "PixelWidth" : 28,
    "RenditionName" : "My.png",
    "Scale" : 1,
    "SizeOnDisk" : 1007,
    "Template Mode" : "automatic"
  },
  {
    "AssetType" : "Image",
    "BitsPerComponent" : 8,
    "ColorModel" : "RGB",
    "Colorspace" : "srgb",
    "Compression" : "palette-img",
    "Encoding" : "ARGB",
    "Idiom" : "universal",
    "Image Type" : "kCoreThemeOnePartScale",
    "Name" : "MyPNG",
    "Opaque" : false,
    "PixelHeight" : 56,
    "PixelWidth" : 56,
    "RenditionName" : "My@2x.png",
    "Scale" : 2,
    "SizeOnDisk" : 1102,
    "Template Mode" : "automatic"
  },
  {
    "AssetType" : "Image",
    "BitsPerComponent" : 8,
    "ColorModel" : "RGB",
    "Colorspace" : "srgb",
    "Compression" : "palette-img",
    "Encoding" : "ARGB",
    "Idiom" : "universal",
    "Image Type" : "kCoreThemeOnePartScale",
    "Name" : "MyPNG",
    "Opaque" : false,
    "PixelHeight" : 84,
    "PixelWidth" : 84,
    "RenditionName" : "My@3x.png",
    "Scale" : 3,
    "SizeOnDisk" : 1961,
    "Template Mode" : "automatic"
  }
]
```

在这份`.json`文档中揭示了一些有趣的信息，可以看到每一个不同分辨率的图像都会在.car文件中去记录它们的一些数据，同时还又一个叫`keyFormatter`的东西，还有很多东西我们暂时不知道它们是什么意思，所以我们继续探究。

## 反编译 CoreUI.framework

既然知道了整个图片的加载过程是与 CoreUI.framework 密不可分，那么想要探究这些问题最好的方法，就是直接去看这些方法做了什么事情。

所以我们利用 _Hopper Disassemble_<sup>[6]</sup> 对 CoreUI.framework 进行反编译，看一下图片加载的过程中究竟发生了什么事情。

CoreUI.framework 位于 `/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/Library/CoreSimulator/Profiles/Runtimes/iOS.simruntime/Contents/Resources/RuntimeRoot/System/Library/PrivateFrameworks/CoreUI.framework/CoreUI`

Hopper 解析完成后会显示这样一个界面：

![7](http://)

随后选择右上角的这一个按钮，就可以看到反编译出来的代码了：

![8](http://)

>  Github 上也有其他人反编译的 CoreUI.framework 的头文件，我 fork 了一份，不方便的同学可以先看一下头文件。

### Folder 中加载图片的过程

**1. 基础判断**

首先关注的是保存在 Folder 中，并使用`imageName:`方法加载的例子，根据 Time Profile 中的调用栈，我们找到

```objc
[CUICatalog _resolvedRenditionKeyForName: scaleFactor: deviceIdiom: deviceSubtype: displayGamut: layoutDirection: sizeClassHorizontal: sizeClassVertical: memoryClass: graphicsClass: appearanceIdentifier: graphicsFallBackOrder: deviceSubtypeFallBackOrder:]
```

而在方法内部我们很容易关注到它对设备的型号做了一次判断，也对加载的图片的`name`进行了一次检查，随后获取了对应`name`的`baseKey`,然后调用下一层的方法

![9](http://)

而`baseKey`则是去取`renditionKey`，它首先会获取一个叫themeStore的东西，在调用栈中我们可以知道，如果图片存放在 Folder 中，则会生成`CUIMutableStructuredThemeStore`，随后它会根据图片的名字，获取CUIRenditionKey对象。

![10](http://)

而且从这里我们可以猜测到应该每一个`rendition`都有与之对应的`renditionKey`，在一张图片资源里，它们可能是一对一的形式，即一个`rendition`对应一个`renditionKey`。

**2. 图片加载前的最后准备工作**

而在下一层的

```objc
[CUICatalog _resolvedRenditionKeyFromThemeRef: withBaseKey: scaleFactor: deviceIdiom: deviceSubtype: displayGamut: layoutDirection: sizeClassHorizontal: sizeClassVertical: memoryClass: graphicsClass: graphicsFallBackOrder: deviceSubtypeFallBackOrder: iconSizeIndex: appearanceIdentifier:]
```

这一个方法是负责完成加载图片前最后的准备工作，包括对应图像的分辨率、放大倍数、方向、水平尺寸、垂直尺寸等参数的设置

![11](http://)

同时在此方法内，我们会注意到有很多地方调用canGetRenditionWithKey:这个方法

![12](http://)

而在开始调用`canGetRenditionWithKey:`之前，会调用`renditionInfoForIdentifier:`去获取`rendition`，如果能够成功获取，则不会再进入到多次调用`canGetRenditionWithKey:`的流程中，这一点十分**重要**，因为只有在 Folder 中加载图片才不能在这步成功获取`rendition`，所以可以假设`rendition`是 Assets Catalogs 中**附带的一些属性**，在 Assets Catalogs 中能够直接获取，而在 Folder 中则是需要重复调用`canGetRenditionWithKey:`来手动获取。

**3. canGetRendition 的判断**

在`canGetRenditionWithKey:`方法内部可以看到它本质上是调用了`renditionWithKey:`的方法，再判断该方法返回值是否为空：

![13](http://)

而在`renditionWithKey:`方法内，它主要做了两件事：

* 根据上一层传入的`[CUIRenditionKey keyList]`获取`keySignature`
* 根据`[CUIRenditionKey keyList]`与`keySignature`获取`rendition`

![14](http://)

先看一下这个keyList：

![15](http://)

它其实是获取自身的的属性，是一个 getter 方法，拿到的值其实不是一个 List ，而是一个结构体：

![16](http://)

里面包含了`identifier`与`value`。

所以利用这个`keyList`，`CUIMutableStructuredThemeStore`获取到了`keySignature`，并根据它获取到了对应的`rendition`：

![17](http://)

可以看到这个方法被加了一个线程同步锁`objc_sync_enter`，以确保它是**线程安全**的，所以它的耗时会高很多。另一方面，在获取`keySignature`的时候，还执行了一个叫做`__CUICopySortedKeySignature`的方法，这个方法是对`keySignature`进行各种位操作，也是会导致耗时的增加。

**4. 小结**

从上面的分析可以看出，在 Folder 中加载导致耗时增加的原因如下：

> 加载图片过程中由于没有办法直接获取`rendition`，所以需要调用`canGetRenditionWithKey:`方法进行判断，而该方法会调用两个比较耗时的操作，一个是对`keySignature`的 copy 操作，另一个是在添加了线程锁并从`CUIMutableStructuredThemeStore`的字典中取出`rendition`的操作，这两个操作是导致耗时增加的元凶。

所以`CUIMutableStructuredThemeStore`在 CoreUI.framework 中起到了一个类似 imageSet 的作用，其中包括了一个**可变字典**，能够存放`rendition`，所以`rendition`就是我们需要加载的图片，而`renditionKey`则是这个图像资源的一种标识，能够通过`renditionKey`获取到对应的`rendition`，同时`renditionKey`中包含了各种`attribute`，是代表该图片的分辨率、垂直大小、水平大小等参数，这些参数这也和我们之前解析的`.json`文件的数据也能一一对应：

```json
{
    "AssetType" : "Image",
    "BitsPerComponent" : 8,
    "ColorModel" : "RGB",
    "Colorspace" : "srgb",
    "Compression" : "palette-img",
    "Encoding" : "ARGB",
    "Idiom" : "universal",
    "Image Type" : "kCoreThemeOnePartScale",
    "Name" : "MyPNG",
    "Opaque" : false,
    "PixelHeight" : 28,
    "PixelWidth" : 28,
    "RenditionName" : "My.png",
    "Scale" : 1,
    "SizeOnDisk" : 1007,
    "Template Mode" : "automatic"
  },
```

所以在 Folder 中加载图片将会生成`CUIMutableStructuredThemeStore`，把图片转成`rendition`并保存到其可变数组中，并根据图片名称生成`renditionKey`，随后根据`CUINamedImageDescription`这个类，获取图片的相关信息，并填充到`renditionKey`中，在需要加载图片的时候，先根据`renditionKey`获取对应的图片资源，然后再从`renditionKey`中读取各种`attribute`信息，并交由 Image I/O 框架对图片进行渲染工作。

### 从 Assets Catalogs 中加载图片

**1. 获取 Rendition**

在 Assets Catalogs 中加载图片则是另外一条路径，在 Time Profile 中能够看到是调用

```objc
[CUICatalog _namedLookupWithName: scaleFactor:  deviceIdiom: deviceSubtype: displayGamut: layoutDirection: sizeClassHorizontal: sizeClassVertical:]
```

其里面也调用了在与上面一样的那两个`resolveXXXX`的方法，但是在耗时上并没有像在 Folder 中加载那样耗费大量时间在`canGetRenditonWithKey:`中，所以可以猜测在`renditionInfoForIdentifier:`中，已经获取了所需的`rendition`。所以我们来关注一下这个函数：

![18](http://)

略去缓存的情况不谈，这个BOM树是一个比较有意思的东西，BOM——(Bill Of Material)这是一个继承自 NeXTSTEP 的文件格式，而且是在 macOS 的各种 installer 中用来决定哪些文件要进行安装、移除或者更新，我们可以在`man 5 bom`中找到这些信息：

> The Mac OS X Installer uses a file system “bill of materials” to determine which files to install, remove, or upgrade. A bill of materials, bom, contains all the files within a directory, along with some information about each file. File information includes: the file’s UNIX permissions, its owner and group, its size, its time of last modification, and so on. Also included are a checksum of each file and information about hard links.

很显然这里的 BOM 树表示其内是以**树的形式**存储数据，在其中应该是存储关于资源文件的一些东西，同时在 CoreUI.framework 中引用了 BOM.framework 中的相关 API 对这个 BOM 文件进行解析并得到相关数据，所以我们可以猜测在 Assets Catalogs 中，编译完成的`.car`文件应该会**包含 BOM 数据**，更进一步，可能`keySignature`就是用于在树中获取对应的`rendtion`与`renditionKey`。

**2. CUIStructuredThemeStore**

在接下来的流程中，能够看到生成的`ThemeStore`是`CUIStructuredThemeStore`，不同于 Folder 中读取时所使用的`CUIMutableStructuredThemeStore`，从名字上就可以猜测，它是“不可变的”，根据上文其实也很容易推断出为什么是不可变了，因为它已经获取到所需要的`rendition`了，不同于 Folder 需要**动态的获取**。

**3. 小结**

从两个加载方法的对比来看，`rendition`的获取是整体耗时的关键，在 Assets Catalogs 中获取的图像资源，其`rendition`能够从一个 BOM 文件中获取，大大加快了加载的速度，另一方面其`renditionKey`也同样作为数据**被保存到 BOM 文件**中，同样`attribute`也在编译过程中获取了，所以无需要再在加载时候进行多余的操作，可以一步到位直接获取所需的图片资源以及其相关信息，并交由渲染引擎进行渲染。

另一方面，虽然在 Folder 中生成的是`CUIMutableStructuredThemeStore`，但是在读取新的图片时，仍然会生成新的`themeStore`，所以在 I/O 上会消耗较大，而在 Assets Catalogs 中，由于所有图像资源都是保存在同一个`.xcassets`中，所以只需要读取一次，就可以获取到所有的图像信息，那么在 I/O 次数上有了显著的优化。

## 问题回顾

所以我们来回顾一下开头提出的问题，现在应该都可以清楚的回答了：

> CoreUI.framework 在加载图片中负责了什么工作？
> `CUIMutalbeStructuredThemeStore`与`CUIStructuredThemeStore`是什么东西？
> rendition又是什么东西？
> 为什么 Assets Catalogs 能够提高这么多加载速度呢？
> `imageWithContentOfFile:`不对图像进行缓存，是否这个原因导致其加载速度要比`imageWithName:`要快呢？

在现在我们可以一一解答了：

> CoreUI.framework 在加载图片中负责了什么工作？

CoreUI.framework 负责进行图片加载的准备工作，UIImage其实是对 CoreUI 的上层封装。

> `CUIMutalbeStructuredThemeStore`与 `CUIStructuredThemeStore`是什么东西？

我们可以将它们理解成 imageSet ，其中包含了不同的图像资源。

> `rendition`又是什么东西？

`rendition`是 CoreUI.framework 对某一图像资源的不同样式的统称，如@1x，@2x，每一个`rendition`有一个`renditionKey`与之对应，`renditionKey`包含了不同的`attribute`，用于记录图片资源的参数。

> 为什么 Assets Catalogs 能够提高这么多加载速度呢？

因为在编译过程中其会生成一个`.car`文件，其中包含了 BOM 文件，BOM文件能够在加载图片时直接获取`rendition`和`renditionKey`以及`attribute`，不同于 Folder 中加载需要先读取图像获取其参数，再生成`rendition`和`renditionKey`，并进行需要大量耗时的`canGetRenditionWithKey`操作。

> `imageWithContentOfFile:`不对图像进行缓存，是否这个原因导致其加载速度要比`imageNamed:`要快呢？

不是，只不过是`imageWithContentOfFile:`不需要转换成`rendition`与生成`renditionKey`等耗时操作。

## 总结

如果你的项目里面还没有使用 Assets Catalogs ，你应该马上使用，因为它不只是能够更方便的管理图像，还可以提供包括切图等一系列方便的功能，更不用说它在 I/O 上性能的显著提升了。

那将图片保存在 Folder 上是否就永远不可取呢？其实也不一定，因为保存在 Assets Catalogs 中的图像无法通过`imageWithContentOfFile:`获取，所以一些**不常用、占用内存**多的图片，可以放在 Folder 中，并通过`imageWithContentOfFile:`获取，另一方面，如果你的应用是“内存紧张”的，或者是想应用更长时间存活在后台，那么可以将图片都存放在 Folder，以减少`imageNamed:`对图片的缓存，换取更低的内存占用。不过我还是建议使用 Assets Catalogs 进行图像的管理。

### 参考资料

1. https://developer.apple.com/videos/play/wwdc2017/207/
2. https://developer.apple.com/videos/play/wwdc2018/227/
3. https://developer.apple.com/videos/play/wwdc2016/719
4. https://help.apple.com/xcode/mac/current/#/dev10510b1f7
5. https://developer.apple.com/library/archive/documentation/Xcode/Reference/xcode_ref-Asset_Catalog_Format/index.html
6. https://www.hopperapp.com/
7. Analysing Assets.car file in iOS : https://stackoverflow.com/questions/22630418/analysing-assets-car-file-in-ios
8. iOS-Asset-Extractor: https://github.com/Marxon13/iOS-Asset-Extractor
9. UIImage加载图片的方式以及Images.xcassets对于加载方法的影响 https://github.com/Marxon13/iOS-Asset-Extractor
10. How to use create and use a UIImageAsset in iOS 8: https://blog.timac.org/2018/1018-reverse-engineering-the-car-file-format/
11. UIImageAsset: https://developer.apple.com/documentation/uikit/uiimageasset
12. Unleashing the power of asset catalogs and bundles on iOS: https://rambo.codes/ios/2018/10/03/unleashing-the-power-of-asset-catalogs-and-bundles-on-ios.html

