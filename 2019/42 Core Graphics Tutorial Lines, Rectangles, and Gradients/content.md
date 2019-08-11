| 作者：Tom Elliott
| 链接：https://www.raywenderlich.com/475829-core-graphics-tutorial-lines-rectangles-and-gradients

Core Graphics 是 iOS 上非常酷的 API。作为开发人员，我们可以使用它来自定义 UI，并应用一些非常简洁的效果，通常甚至无需让设计人员参与其中。任何与 2D 绘图相关的东西 - 比如绘制形状、填充、渐变 - 使用 Core Graphics 都是一个很好的选择。

Core Graphics 的历史可以追溯到 OS X 的早期阶段，是目前仍在使用的最古老的 API 之一。也许这就是为什么，对于许多 iOS 开发人员来说，Core Graphics 起初可能有些令人生畏：它是一个庞大的 API，学习难度大。但是，自 Swift 3 以来，C 风格的 API 已经被封装起来，外观和感觉就像我们熟悉和喜爱的现代 Swift API！

在本教程中，我们将构建一个 Star Wars Top Trumps 卡片应用程序，该应用程序由包含 Starships 列表的主视图组成：

![](https://koenig-media.raywenderlich.com/uploads/2019/01/starship_list_finished.png)

每个 Starship 的详细视图如下：

![](https://koenig-media.raywenderlich.com/uploads/2019/01/starship_detail_finished.png)

在创建此应用程序时，我们将学习如何开始使用 Core Graphics，如何填充和描边矩形以及如何绘制线条和渐变，以制作自定义表格视图单元格和背景。

是时候与 Core Graphics 享受一些乐趣了！

## 开始

首先下载启动项目(https://koenig-media.raywenderlich.com/uploads/2019/02/CoreGraphicsTutorialLinesRectanglesAndGradients.zip)。打开起始项目并快速浏览一下。该应用程序基于 Xcode 提供的 **Master-Detail App 模板**。主视图控制器包含 Star Ships 列表，详细视图控制器显示每艘船的详细信息。

打开 **MasterViewController.swift**。在该类的顶部，注意一个 `starships` 变量，它是包含 `Starship` 类型的数组，还有 `StarshipDataProvider` 类型的 `dataProvider` 变量。

通过 **Command-单击** `StarshipDataProvider` 并选择 **Jump to Definition** 跳转到 `StarshipDataProvider.swift`。这是一个简单的类，它读取资源文件 `Starships.json`，并将内容转换为 `Starship` 数组。

我们可以在 **Starship.swift** 找到 `Starship` 的定义。它只是一个简单的结构，有 `Starships` 常用的属性。

接下来，打开 **DetailViewController.swift**。在类定义之前是一个枚举定义，`FieldsToDisplay`，它对应 Starship 的属性，是一些可理解的字符串。在这个文件中，`tableView(_:cellForRowAt:)` 只是一个很大的 `switch` 语句，用于将每个 `Starship` 属性的数据格式化为正确的格式。

构建并运行应用程序。

![](https://koenig-media.raywenderlich.com/uploads/2019/01/starship_list_starter.png)

首页是 `MasterViewController`，显示星球大战宇宙中的星舰列表。点击以选择 **X-wing**，应用程序将导航到该船的详细视图，其中显示了 X-wing 的图像，然后是各种属性，例如它的成本和飞行速度。

![](https://koenig-media.raywenderlich.com/uploads/2019/01/starship_detail_starter.png)

这是一个功能齐全，但有点无聊的应用程序。现在是时候添加一些新东西了！

## 分析 Table View 样式

在本教程中，我将为两个不同的 table view添加不同的样式。仔细看看这些变化是什么样的。

在主视图控制器中，每个单元格：

* 有从深蓝色到黑色的渐变色。
* 以黄色勾勒出轮廓，从 Cell 边界插入。

![](https://koenig-media.raywenderlich.com/uploads/2019/01/master_zoomed_in-2.png)

而在详情视图控制器中：

* table view 本身有从深蓝色到黑色的渐变色。
* 每个 cell 都有一个黄色分割线，将其与相邻 cell 分开。

![](https://koenig-media.raywenderlich.com/uploads/2019/01/detail_zoomed_in.png)

要绘制这两种样式，我们只需要知道如何使用 Core Graphics 绘制矩形，渐变和线条，这正是我们将要学习的内容。：]

## Hello, Core Graphics!

虽然本教程主要讨论在 iOS 上使用 Core Graphics，但需要知道的是 Core Graphics 可用于所有主要的 Apple 平台，包括使用 AppKit 的MacOS，使用 UIKit 的 iOS 和 tvOS 以及使用 WatchKit 的 Apple Watch。

在一些情况下可以考虑使用 Core Graphics，如在画布上绘画；绘图操作的顺序很重要。例如，如果我们绘制重叠的形状，那么添加的最后一个形状将位于顶部并与下面的重叠。

Apple 以这样的方式构建 Core Graphics，让开发人员考虑绘制什么(what)而不是在何处(where)绘制。

由 CGContext 类表示的 **Core Graphic 上下文**定义了 where。我们可以告诉上下文要执行的绘制操作。有几种 CGContext，包括用于绘制到位图图像、绘制到 PDF 文件的上下文，最常见的是直接绘制到 UIView 中。

在与绘画的类比中，Core Graphics 上下文代表画家绘制的画布。

Core Graphics 上下文是一个状态机。也就是说，当我们设置填充颜色时，可以为整个画布设置填充颜色，并且在我们更改之前，绘制的任何形状都将具有相同的填充颜色。

每个 UIView 都有自己的 Core Graphics 上下文。要使用 Core Graphics 绘制 UIView 的内容，必须在视图的 `draw(_:)` 中编写绘图代码。这是因为 iOS 在调用 `draw(_:)` 之前设置了正确的 CGContext 以便绘制到视图中。

现在我们了解了如何在 UIKit 中使用 Core Graphics 的基础知识，现在是时候更新我们的应用了！

## 绘制矩形

首先，通过在 **File** 菜单中选择 **New ▸ File…** 来创建新的视图文件。选择 **Cocoa Touch Class**，按 **Next**，然后将类名设置为 **StarshipsListCellBackground**。让这个类继承自 UIView 类，然后创建类文件。将以下代码添加到新类：

```objc
override func draw(_ rect: CGRect) {
  // 1
  guard let context = UIGraphicsGetCurrentContext() else {
    return
  }
  // 2  
  context.setFillColor(UIColor.red.cgColor)
  // 3
  context.fill(bounds)
}
```

逐行分析下这段代码：

* 首先，使用 `UIGraphicsGetCurrentContext()` 获取此 `UIView` 实例的当前 `CGContext`。请记住，iOS 会在调用`draw(_:)` 之前自动为您设置。如果由于任何原因无法获取上下文，则可以从方法中提前返回。
* 然后，在上下文中设置填充颜色。
* 最后，告诉它填充视图的边界。

如我们所见，Core Graphics API 不包含直接绘制填充颜色的形状的方法。相反，有点像画笔上蘸上颜料，我们将颜色设置为 CGContext 的一个状态，然后，告诉上下文分别用该颜色绘制什么。

我们可能还注意到，当在上下文中调用 `setFillColor(_:)` 时，没有提供标准的 `UIColor`。相反，必须使用 `CGColor`，这是 Core Graphics 内部用于表示颜色的基本数据类型。只需访问任何 UIColor 的 `cgColor` 属性，将 UIColor 转换为 CGColor 非常容易。

## 显示新的 Cell

要查看我们的新视图，请打开 **MasterViewController.swift**。 在 `tableView(_:cellForRowAt:)` 中，在方法的第一行添加以下代码：

```objc
if !(cell.backgroundView is StarshipsListCellBackground) {
  cell.backgroundView = StarshipsListCellBackground()
}
    
if !(cell.selectedBackgroundView is StarshipsListCellBackground) {
  cell.selectedBackgroundView = StarshipsListCellBackground()
}
```

此代码将单元格的背景视图设置为新视图。构建并运行应用程序，将在每个单元格中看到花哨的红色背景。

![](https://koenig-media.raywenderlich.com/uploads/2019/01/red_cells.png)

现在可以使用 Core Graphics 进行绘制了。不管信不信，我们已经学会了一系列非常重要的技巧：如何获取绘制的上下文，如何更改填充颜色以及如何用颜色填充矩形。我们可以用它制作一些非常漂亮的用户界面。

但是要更进一步，学习一种非常有用的技术来制作漂亮的用户界面：**渐变**！

## 创建新的颜色

我们将在此项目中反复使用相同的颜色，因此为 UIColor 创建一个扩展，以方便使用这些颜色。转到 **File ▸ New ▸ File…** 并创建一个名为 **UIColorExtensions.swift** 的新 Swift 文件。用以下内容替换文件的内容：

```objc
import UIKit

extension UIColor {
  public static let starwarsYellow = 
    UIColor(red: 250/255, green: 202/255, blue: 56/255, alpha: 1.0)
  public static let starwarsSpaceBlue = 
    UIColor(red: 5/255, green: 10/255, blue: 85/255, alpha: 1.0)
  public static let starwarsStarshipGrey = 
    UIColor(red: 159/255, green: 150/255, blue: 135/255, alpha: 1.0)
} 
```

这段代码定义了三种新颜色，可以在 UIColor 上以静态属性的形式访问这些颜色。

### 绘制渐变

接下来，由于我们要在此项目中绘制大量渐变，因此请添加辅助方法来绘制渐变。这样将渐变代码放在一处以重用。

选择 **File ▸ New ▸ File…** 并创建一个名为 **CGContextExtensions.swift** 的 Swift 文件。用以下内容替换文件的内容：

```objc
import UIKit

extension CGContext {
  func drawLinearGradient(
    in rect: CGRect, 
    startingWith startColor: CGColor, 
    finishingWith endColor: CGColor
  ) {
    // 1
    let colorSpace = CGColorSpaceCreateDeviceRGB()

    // 2
    let locations = [0.0, 1.0] as [CGFloat]    

    // 3
    let colors = [startColor, endColor] as CFArray

    // 4
    guard let gradient = CGGradient(
      colorsSpace: colorSpace, 
      colors: colors, 
      locations: locations
    ) else {
      return
    }
  }
}
```

这个方法做了不少事情：

1) 首先，设置正确的颜色空间。我们可以使用颜色空间做很多事情，但是几乎总是希望使用 `CGColorSpaceCreateDeviceRGB` 来使用与设备相关的标准 RGB 颜色空间。
2) 接下来，设置一个数组，跟踪渐变范围内每种颜色的位置。值 0 表示渐变的开始，1 表示渐变的结束。

> 注意：如果需要，可以在渐变中使用三种或更多颜色，并且可以设置每种颜色在渐变中的位置，就像这样的数组。这对某些效果很有用。

3) 之后，使用传递给方法的颜色创建一个数组。注意在这里使用 `CFArray` 而不是 `Array`，因为在这里我们使用的是较低级别的 C API。
4) 然后，通过初始化 `CGGradient` 对象，传入颜色空间，颜色数组和先前创建的位置来创建渐变。如果由于某种原因，可选的初始化程序失败，则可以提前返回。

现在有一个渐变引用，但它实际上还没有绘制任何东西 - 它只是一个指向稍后实际绘制时使用的信息的指针。现在到绘制渐变的时候了，但在这之前，还需要了解一些理论。

### 图形状态栈

请记住，Core Graphics 上下文是状态机。在上下文中设置状态时必须要小心，特别是在传递上下文的函数中，或者在本例中是上下文本身的方法，因为在修改上下文之前无法知道上下文的状态。请考虑 `UIView` 中的以下代码：

```objc
override func draw(_ rect: CGRect) {
  // ... get context
     
  context.setFillColor(UIColor.red.cgColor)
  drawBlueCircle(in: context)
  context.fill(someRect)    
}
  
// ... many lines later
  
func drawBlueCircle(in context: CGContext) {
  context.setFillColor(UIColor.blue.cgColor)
  context.addEllipse(in: bounds)
  context.drawPath(using: .fill)
}
```

看一下这段代码，我们可能会认为它会在视图中绘制一个红色矩形和一个蓝色圆圈，但这是错的！相反，这段代码绘制了一个蓝色矩形和一个蓝色圆圈 - 为什么呢？

![](https://koenig-media.raywenderlich.com/uploads/2019/01/all_blue.png)

因为 `drawBlueCircle(in:)` 在上下文中设置了蓝色填充颜色，并且因为上下文是状态机，所以它会覆盖先前的红色填充色。

这里就要说说 `saveGState()` 和 `restoreGState()` 了！

每个 CGContext 都维护一个图形状态的堆栈，其中包含当前绘图环境的大部分（尽管不是全部）方面。`saveGState()` 将当前状态的副本推送到图形状态堆栈，然后可以使用 `restoreGState()` 将上下文恢复到该状态，并在该过程中从堆栈中删除状态。

在上面的示例中，应该像这样修改 `drawBlueLines(in:)`：

```objc
func drawBlueCircle(in context: CGContext) {
  context.saveGState()
  context.setFillColor(UIColor.blue.cgColor)
  context.addEllipse(in: bounds)
  context.drawPath(using: .fill)
  context.restoreGState()
}
```

![](https://koenig-media.raywenderlich.com/uploads/2019/01/blue_and_red.png)

可以通过下载示例代码，打开 **RedBluePlayground.playground** 来自行测试。

### 完成渐变

了解了有关图形状态堆栈的知识，现在是时候完成绘制背景渐变了。将以下内容添加到 `drawLinearGradient(in:startingWith:finishingWith:)` 的末尾：

```objc
// 5
let startPoint = CGPoint(x: rect.midX, y: rect.minY)
let endPoint = CGPoint(x: rect.midX, y: rect.maxY)
    
// 6
saveGState()

// 7
addRect(rect)
clip()
drawLinearGradient(
  gradient, 
  start: startPoint, 
  end: endPoint, 
  options: CGGradientDrawingOptions()
)

restoreGState()
```

5) 首先计算渐变的起点和终点。可以将其设置为从矩形的顶部中间到底部中间的线。CGRect 包含一些实例变量，如 midX 和 maxY，使得这非常简单。
6) 接下来，由于即将修改上下文的状态，因此可以保存其图形状态并通过还原它来结束该方法。
7) 最后，在提供的矩形中绘制渐变。`drawLinearGradient(_:start:end:options:)` 是实际绘制渐变的方法，但除非另有说明，否则将使用渐变填充整个上下文，即整个视图。在这里，我们只想填充提供的矩形中的渐变。要做到这一点，需要了解一下裁剪。

剪切是 Core Graphics 中一个非常棒的功能，可以将绘图限制为任意形状。我们所要做的就是将形状添加到上下文中，然后，不要像通常那样填充它，而是在上下文中调用`clip()` ，然后将所有后续的绘制限制到该区域。

因此，在这种情况下，我们在调用 `drawLinearGradient(_:start:end:options:)` 绘制渐变之前，在上下文和剪辑上设置提供的矩形。

打开 **StarshipsListCellBackground.swift**，在获取当前的 `UIGraphicsContext` 之后，用以下内容替换代码：

```objc
let backgroundRect = bounds
context.drawLinearGradient(
  in: backgroundRect, 
  startingWith: UIColor.starwarsSpaceBlue.cgColor, 
  finishingWith: UIColor.black.cgColor
)
```

编译并运行程序：

![](https://koenig-media.raywenderlich.com/uploads/2019/01/ugly_gradient.png)

我们现在已成功将渐变背景添加到自定义单元格。但是，现在成品并不是很好看。现在我们用一些标准的 UIKit 主题来修复它了。

## 修复主题

打开 **Main.storyboard** 并选择 **Master scene** 中的表格视图。在 Attributes inspector 中，将 **Separator** 设置为 **None**。

![](https://koenig-media.raywenderlich.com/uploads/2019/01/plain_table_view.png)

然后，在 **Master Navigation Controller** 场景中选择 `Navigation Bar` 并将导航栏 **Style** 设置为 **Black** 并取消选择 **Translucent**。 在**Detail Navigation Controller** 场景中重复 `Navigation Bar` 的设置。

![](https://koenig-media.raywenderlich.com/uploads/2019/01/black_navigation_bar.png)

Next, open MasterViewController.swift. At the end of viewDidLoad(), add the following:

接下来，打开 **MasterViewController.swift**。在 `viewDidLoad()` 的末尾，添加以下内容：

```objc
tableView.backgroundColor = .starwarsSpaceBlue
```

然后在 `tableView(_:cellForRowAt:)` 中，在返回单元格之前，设置文本的颜色：

```objc
cell.textLabel!.textColor = .starwarsStarshipGrey
```

最后，打开 **AppDelegate.swift** 并在应用程序 `application(_:didFinishLaunchingWithOptions:)` 中返回之前添加以下内容：

```objc
// Theming
UINavigationBar.appearance().tintColor = .starwarsYellow
UINavigationBar.appearance().barTintColor = .starwarsSpaceBlue
UINavigationBar.appearance().titleTextAttributes = 
  [.foregroundColor: UIColor.starwarsStarshipGrey]
```

编译并运行：

![](https://koenig-media.raywenderlich.com/uploads/2019/01/less_ugly_gradient.png)

这样看上去好多了。

## 路径描边

在 Core Graphics 中进行描边意味着沿着路径绘制一条线，而不是像之前那样填充它。

当 Core Graphics 描边路径时，它会在路径的精确、、边缘的中间绘制描边线。这可能会导致一些常见问题。

### 边界外

首先，如果正在绘制一个矩形的边缘，例如，边框，默认情况下，Core Graphics 将会少绘制一半的描边路径。

为什么？因为为 UIView 设置的上下文仅扩展到视图的边界。想象一下，在视图边缘周围有一个点边框。因为 Core Graphics 在路径的中间向下描边，所以描边线将在视图边界之外半个点，在视图边界内半个点。

一个常见的解决方案是将描边路径在每个方向向内收缩行宽度的一半，使其位于视图内。

下图显示了一个黄色矩形，在灰色背景上有一个点宽的红色笔划。在左图中，描边路径遵循视图的边界并已被裁剪。我们可以看到，因为红线是灰色方块宽度的一半。在右图中，描边路径已插入半个点，现在就是正确的线宽。

![](https://koenig-media.raywenderlich.com/uploads/2019/01/inset_stroking.png)

### 抗锯齿

其次，需要了解可能影响边框外观的抗锯齿效果。抗锯齿是一种渲染引擎用于避免在显示图形无法完美映射到设备上的物理像素时出现的“锯齿状”边缘和线条的技术。

以前面一段视图周围的一点边框为例。如果边框遵循视图的边界，则 Core Graphics 将尝试在矩形的任一侧绘制半个点宽的线。

在非视网膜显示器上，一个点等于设备上的一个像素。不可能只点亮半个像素，因此 Core Graphics 将使用抗锯齿来绘制两个像素，但是在较浅的阴影中只能呈现单个像素的外观。

在以下几组屏幕截图中，左图是非视网膜显示器，中间图像是视网膜显示器，其缩放比例为 2，第三图像是视网膜显示器，其缩放比例为 3。

对于第一个图，请注意 2x 图像如何不显示任何抗锯齿，因为黄色矩形的两边的半点落在像素边界上。然而，在 1x 和 3x 图像中发生抗锯齿。

![](https://koenig-media.raywenderlich.com/uploads/2019/01/stroking_different_scales-1.png)

在下一组屏幕截图中，描边矩形已插入半个点，使得笔划线与点精确对齐，从而与像素边界对齐。注意没有锯齿伪像。

![](https://koenig-media.raywenderlich.com/uploads/2019/01/stroking_different_scales_pixel_boundary.png)

## 添加边框

回到应用程序！Cell 开始看起来很好，但我们会增加另一种触感，使它更美观 这一次，我们将在 Cell 边缘绘制一个明亮的黄色框。

我们已经知道如何轻松填充矩形。在他们周围描边也同样容易。

打开 **StarshipsListCellBackground.swift** 并将以下内容添加到 `draw(_:)` 的底部：

```objc
let strokeRect = backgroundRect.insetBy(dx: 4.5, dy: 4.5)
context.setStrokeColor(UIColor.starwarsYellow.cgColor)
context.setLineWidth(1)
context.stroke(strokeRect)
```

在这里，我们可以创建一个用于描边的矩形，它在 x 和 y 方向上从背景矩形中插入4.5个点。然后将描边颜色设置为黄色，将线宽设置为一个点，最后描边。构建并运行您的项目。

![](https://koenig-media.raywenderlich.com/uploads/2019/01/far_far_away.png)

现在我们的星舰列表真的看起来像是来自遥远的星系了！

## 构建卡片布局

虽然我们的主视图控制器看起来很花哨，但详情视图控制器仍然需要一些修饰！

![](https://koenig-media.raywenderlich.com/uploads/2019/01/detail_starter_vs_finished.png)

对于此视图，我们将首先使用自定义 `UITableView` 子类在表视图背景上绘制渐变。

创建一个名为 **StarshipTableView.swift** 的 Swift 文件。用以下内容替换生成的代码：

```objc
import UIKit

class StarshipTableView: UITableView {
  override func draw(_ rect: CGRect) {
    guard let context = UIGraphicsGetCurrentContext() else {
      return
    }

    let backgroundRect = bounds
    context.drawLinearGradient(
      in: backgroundRect, 
      startingWith: UIColor.starwarsSpaceBlue.cgColor, 
      finishingWith: UIColor.black.cgColor
    )
  }
}
```

现在这应该开始变得熟悉了。在新表视图子类的 `draw(_:)` 方法中，将获得当前的 `CGContext`，然后在视图的边界绘制一个渐变，从顶部的蓝色开始，在底部变成黑色。简单！

打开 **Main.storyboard** 并单击 **Detail** 场景中的 **TableView**。在 **Identity** 检查器中，将类设置为新的 **StarshipTableView**。

![](https://koenig-media.raywenderlich.com/uploads/2019/01/using_starship_table_view-1.png)

构建并运行应用程序，然后点击 **X-wing**。

![](https://koenig-media.raywenderlich.com/uploads/2019/01/detail_babckground.png)

详情视图现在具有从上到下的一个漂亮的全屏渐变，但表格视图中的单元格遮挡了效果的最佳部分。是时候解决这个问题并为 Cell 添加更多特性了。

回到 **Main.storyboard**，在 **Detail Scene** 中选择 **FieldCell**。 在 **Attributes** 检查器中，将 **background** 设置为 **Clear Color**。 接下来，打开**DetailViewController.swift**，在 `tableView(_:cellForRowAt:)` 的最底部，在返回单元格之前，添加以下内容：

```objc
cell.textLabel!.textColor = .starwarsStarshipGrey
cell.detailTextLabel!.textColor = .starwarsYellow
```

这只是将单元格的字段名称和值设置为适合星球大战主题的更合适的颜色。

然后，在 `tableView(_:cellForRowAt:)` 之后添加以下方法来设置表视图头的样式：

```objc
override func tableView(
  _ tableView: UITableView, 
  willDisplayHeaderView view: UIView, 
  forSection section: Int
) {
    view.tintColor = .starwarsYellow
    if let header = view as? UITableViewHeaderFooterView {
      header.textLabel?.textColor = .starwarsSpaceBlue
    }
  }
```

在这里，我们将表格视图的标题视图的色调颜色设置为主题黄色，为其提供黄色背景，并将其文本颜色设置为主题蓝色。

## 绘图线

最后一点，我们将在详情视图中为每个单元格添加一个分割线。创建一个新的 Swift 文件，这次称为 **YellowSplitterTableViewCell.swift**。用以下内容替换生成的代码：

```objc
import UIKit

class YellowSplitterTableViewCell: UITableViewCell {
  override func draw(_ rect: CGRect) {
    guard let context = UIGraphicsGetCurrentContext() else {
      return
    }
    
    let y = bounds.maxY - 0.5
    let minX = bounds.minX
    let maxX = bounds.maxX

    context.setStrokeColor(UIColor.starwarsYellow.cgColor)
    context.setLineWidth(1.0)
    context.move(to: CGPoint(x: minX, y: y))
    context.addLine(to: CGPoint(x: maxX, y: y))
    context.strokePath()
  }
}
```

在 `YellowSplitterTableVIewCell` 中，使用 Core Graphics 来划分单元格边界底部的一条线。注意所使用的 y 值是如何比视图的边界小半个点，以确保分割器完全在单元内绘制。

现在，需要实际绘制分割线。

要在 A 和 B 之间绘制一条线，首先移动到 A 点，这不会导致 Core Graphics 绘制任何东西。然后，将一条线添加到 B 点，该线将点 A 到点 B 的线添加到上下文中。然后，可以调用 `strokePath()` 来描边该行。

最后，再次打开 **Main.storyboard** 并使用 **Identity** 检查器将 **Detail** 场景中的 **FieldCell** 类设置为新创建的`YellowSplitterTableViewCell`。构建并运行您的应用程序。 然后，打开 X-wing 详情视图。漂亮！

![](https://koenig-media.raywenderlich.com/uploads/2019/01/starship_detail_finished.png)

## 总结

然后去哪儿？

您可以在 https://koenig-media.raywenderlich.com/uploads/2019/02/CoreGraphicsTutorialLinesRectanglesAndGradients.zip 中下载最终项目。

工程中还包括两个 playgrounds。**RedBluePlayground.playground** 包含在上下文保存/恢复部分中设置的示例，**ClippedBorderedView.playground** 演示剪切边框。

至此，我们应该熟悉 Core Graphics 的一些非常酷且功能强大的技术：填充和抚摸矩形，绘制线条和渐变以及裁剪路径。现在我们的 table view现在看起来很酷了。

