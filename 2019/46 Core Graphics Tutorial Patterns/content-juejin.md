在本教程中，我们将使用 Core Graphics 来创建模式，并使用它们来完成一个名为 `Recall` 的模式识别游戏。在此过程中，我们将强化基于路径的绘图等概念。

如果我们是 Core Graphics 的新手，那么最好先看一些有关该主题的一些入门级教程。可以查阅使用我们的直线，矩形和渐变以及弧和路径的教程，以更好地理解我们此文的基础。

## 开始

首先，下载本教程的示例程序(https://koenig-media.raywenderlich.com/uploads/2019/01/LearningAgenda-2.zip)。 构建并运行起始应用程序。我们会看到以下的界面：

![](https://koenig-media.raywenderlich.com/uploads/2018/12/Recall_starter.png)

Recall 从 **Left vs Right** 应用程序中的游戏中获取灵感。游戏的目标是为视图中的对象选择流行的方向。一旦做出选择，就会显示一组新对象。在比赛结束前我们有五次尝试。

Recall 对四个象限中的模式对象进行分组。入门应用中的每个象限都有一个标签。文本表示方向，背景颜色表示填充颜色。

现在这个游戏还很粗糙。

![](https://koenig-media.raywenderlich.com/uploads/2018/12/RageFace_Not_Impressed.png)

我们的任务是使用新发现的 Core Graphics 印章将其转换为下面的完成的应用程序：

![](https://koenig-media.raywenderlich.com/uploads/2018/12/Recall_final.png)

在 Xcode 中查看项目。这些是主要文件：

* **GameViewController.swift**：控制游戏玩法并显示游戏视图。
* **ResultViewController.swift**：显示最终得分和重启游戏的按钮。
* **PatternView.swift**：显示游戏视图中其中一个象限的模式视图。

我们将增强 `PatternView` 以显示所需的模式。

对于初学者来说，我们可以在 Playgrounds 中对新的和改进的 PatternView 进行原型设计。这使我们可以更快地迭代，同时了解 Core Graphics 模式的细节。完成后，将相关代码复制到 **Recall** 启动项目。

在 Xcode 中，转到 **File ▸ New ▸ Playground…** . 选择 **Single View** 模板。单击 **Next**，将 playground 命名为 **PatternView.playground**，然后单击 **Create**。我们的 playground 包含一个带有单个视图的视图控制器。

选择 **Editor ▸ Run Playground** 以执行 playground。单击 **Show the Assistant Editor** 以显示启动器视图。它显示了标志性的“Hello World！”：

![](https://koenig-media.raywenderlich.com/uploads/2018/12/RunPlayground.gif)

在浏览下一节时，我们将使用模式视图替换起始视图。准备开始了吗？

![](https://koenig-media.raywenderlich.com/uploads/2018/12/RageFace_Lets_Go.png)

## 剖析模式

在之前的 Core Graphics 教程中，我们已经了解了如何定义和绘制这样的路径：

![](https://koenig-media.raywenderlich.com/uploads/2018/12/CoreGraphics_Fill_Paint.png)

上面的示例显示我们使用颜色填充左侧的路径。请记住，我们还可以使用颜色描边路径。

使用 Core Graphics，我们还可以使用模式描边或填充路径。下面的示例显示填充路径的彩色图案：

![](https://koenig-media.raywenderlich.com/uploads/2018/12/CoreGraphics_Fill_Patterns.png)

我们可以通过执行以下操作来设置模式：

1. 编写一个绘制单个模式单元格的方法；
2. 使用包含如何绘制和放置单个单元格的参数创建模式；
3. 定义模式将使用的颜色信息；
4. 使用创建的模式绘制所需的路径。

现在，检查一个略有不同的模式单元格和额外的填充。细黑边框显示单元格的边界：

![](https://koenig-media.raywenderlich.com/uploads/2018/12/PatternCell.png)

要在单元格的边界内绘制，请为模式单元格编写 draw 方法。Core Graphics 会裁剪界限之外的任何内容。Core Graphics 还希望我们每次都以完全相同的方式绘制模式单元格。

设置图案单元格时，绘制方法可以应用颜色。这是一个着色模式。无色或 masking 模式是在 draw 方法之外应用填充颜色的模式。这使我们可以灵活地设置图案颜色。

Core Graphics 重复调用draw方法来设置模式。模式创建参数定义模式的外观。下面的示例显示了一个基本的重复模式，其中单元格彼此相邻排列：

![](https://koenig-media.raywenderlich.com/uploads/2018/12/PatternCell_Repeated.png)

配置模式时，可以指定模式单元格之间的间距：

![](https://koenig-media.raywenderlich.com/uploads/2018/12/PatternCell_Repeated_Offset-250x250.png)

我们还可以应用变换来更改图案的外观。下图显示了由模糊边框表示的空间内绘制的图案：

![](https://koenig-media.raywenderlich.com/uploads/2018/12/PatternCell_Repeated_Transforms.png)

第一个显示未更改的模式。在第二个中，您会看到平移后的模式。第三个显示旋转的模式。同样，模式单元周围的黑色边框突出了它的边界。

配置模式时，我们有很多选项可用。我们将在下一节开始把所有这些放在一起。

## 创建第一个模式

在 **PatternView.playground** 中的视图控制器类之前添加以下代码：

```objc
class PatternView: UIView {
  override func draw(_ rect: CGRect) {
    // 1
    let context = UIGraphicsGetCurrentContext()!
    // 2
    UIColor.orange.setFill()
    // 3
    context.fill(rect)
  }
}
```

这表示模式的自定义视图。在这里，我们重写 `draw(_:)` 以执行以下操作：

1. 获取视图的图形上下文。
2. 设置上下文的当前填充颜色。
3. 使用当前填充颜色填充整个上下文。

将图形上下文视为可以绘制的画布。上下文包含将填充或描边路径的颜色等信息。在使用上下文的颜色信息绘制路径之前，我们可以在画布中绘制路径。

在 `MyViewController` 内部，使用以下内容替换与 `label` 相关的 `loadView()` 中的代码：

```objc
let patternView = PatternView()
patternView.frame = CGRect(x: 10, y: 10, width: 200, height: 200)
view.addSubview(patternView)
```

这将创建模式视图的实例，设置其框架并将其添加到视图中。

按 **Shift+Command+Return** 以运行 playground。之前的标签已经消失，取而代之的是橙色子视图：

![](https://koenig-media.raywenderlich.com/uploads/2018/12/Playground_Basic_Fill.png)

着色只是旅程的开始。

![](https://koenig-media.raywenderlich.com/uploads/2018/12/RageFace_BringItOn.png)

在 `PatternView` 的顶部添加以下属性：

```objc
let drawPattern: CGPatternDrawPatternCallback = { _, context in
  context.addArc(
    center: CGPoint(x: 20, y: 20), radius: 10.0,
    startAngle: 0, endAngle: CGFloat(2.0 * .pi),
    clockwise: false)
  context.setFillColor(UIColor.black.cgColor)
  context.fillPath()
}
```

上面的代码在图形上下文中绘制一个圆形路径，并用黑色填充它。

这给出了模式单元格的绘制方法，其类型为 CGPatternDrawPatternCallback。该方法接受指向与模式关联的私有数据的指针。我们没有使用私有数据，因此使用了未命名的参数。该方法还接受绘制模式单元格时使用的图形上下文。

将以下代码添加到 `draw(_:)` 结尾处：

```objc
var callbacks = CGPatternCallbacks(
  version: 0, drawPattern: drawPattern, releaseInfo: nil)
```

我们为 `drawPattern` 提供回调，`releaseInfo` 接受系统释放模式时调用的回调。如果在模式中使用私有数据，通常会设置一个 release 回调函数。由于我们不在 draw 方法中使用私有数据，因此将 nil 传递给此回调。

在 `callbacks` 分配后立即添加以下内容：

```objc
let pattern = CGPattern(
  info: nil,
  bounds: CGRect(x: 0, y: 0, width: 20, height: 20),
  matrix: .identity,
  xStep: 50,
  yStep: 50,
  tiling: .constantSpacing,
  isColored: true,
  callbacks: &callbacks)
```

这会创建一个模式对象。在上面的代码中，我们传递以下参数：

* **info**：指向要在模式回调中使用的任何私有数据的指针。由于没有使用任何私有数据，所以传递的是 `nil`；
* **bounds**：模式 cell 的边界框。
* **matrix**：表示要应用的变换的矩阵。我们传递了 `identity` 矩阵，因为您没有应用任何变换；
* **xStep**：模式单元格之间的水平间距。
* **yStep**：模式单元格之间的垂直间距。
* **tiling**：更改为用户空间单位和设备像素之间的差异。
* **isColored**：模式单元格绘制方法是否应用颜色。将此设置为true，因为绘制方法设置了颜色。
* **callbacks**：指向保存模式回调的结构的指针。

在模式分配后立即添加以下代码：

```objc
var alpha : CGFloat = 1.0
context.setFillPattern(pattern!, colorComponents: &alpha)
context.fill(rect)
```

上面的代码设置了图形上下文的填充模式。对于彩色图案，还必须传入 alpha 值以指定模式的不透明度。模式绘制方法提供颜色。最后，代码使用模式绘制视图的框架区域。

按 **Shift+Command+Return** 运行 playground。模式没有显示出来。这又是怎么回事？

我们需要为 Core Graphics 提供有关模式颜色空间的信息，以便它知道如何处理图案颜色。

在 alpha 声明上面添加以下内容：

```objc
// 1
let patternSpace = CGColorSpace(patternBaseSpace: nil)!
// 2
context.setFillColorSpace(patternSpace)
```

这是代码的作用：

1. 创建模式颜色空间。对于彩色模式，基本空间参数应为 `nil`。这会将着色委托给模式单元格绘制方法。
2. 将填充颜色空间设置为定义的模式颜色空间。

运行 playground。现在应该看到一个圆形的黑色图案：

![](https://koenig-media.raywenderlich.com/uploads/2018/12/Playground_CirclePattern_1.png)

如何更好地配置模式呢？

![](https://koenig-media.raywenderlich.com/uploads/2018/12/RageFace_SoExcite.png)

## 配置模式

在 playground 中，更改设置模式的间距参数，如下所示：

```objc
xStep: 30,
yStep: 30,
```

运行 playground。请注意，圆点似乎彼此更接近：

![](https://koenig-media.raywenderlich.com/uploads/2018/12/Playground_CirclePattern_Change_Step_1.png)

我们缩小了模式单元格之间的步长。

现在，更改间距参数，如下所示：

```objc
xStep: 20,
yStep: 20,
```

运行 playground。圈子变成了四分之一：

![](https://koenig-media.raywenderlich.com/uploads/2018/12/Playground_CirclePattern_Change_Step_2.png)

要了解原因，请注意绘制方法返回一个半径为 10 的圆，其中心位于（20,20）。模式的水平和垂直位移为 20。单元格的边界框在原点 (0,0) 处为 20✕20。这导致重复的四分之一圆从右下边缘开始。

更改绘制圆的 drawPattern 代码，如下所示：

```objc
context.addArc(
      center: CGPoint(x: 10, y: 10), radius: 10.0,
      startAngle: 0, endAngle: CGFloat(2.0 * .pi),
      clockwise: false)
```

将中心点更改为(10, 10)）而不是 (20, 20)。

运行 playground。 由于圆中的移动，现地显示了整个圆：

![](https://koenig-media.raywenderlich.com/uploads/2018/12/Playground_CirclePattern_Change_Step_3.png)

模式单元格边界也与圆完美匹配，导致每个单元格与另一个单元格相邻。

我们可以通过许多有趣的方式转换模式。 在 `draw(_:)` 中，用以下内容替换模式：

```objc
// 1
let transform = CGAffineTransform(translationX: 5, y: 5)
// 2
let pattern = CGPattern(
      info: nil,
      bounds: CGRect(x: 0, y: 0, width: 20, height: 20),
      matrix: transform,
      xStep: 20,
      yStep: 20,
      tiling: .constantSpacing,
      isColored: true,
      callbacks: &callbacks)
```

我们已通过传递转换矩阵来修改 CGPattern。 再仔细看看：

1. 创建表示转换的仿射变换矩阵。
2. 通过将其传递给 matrix 参数来配置模式以使用此转换。

运行 playground。注意模式如何向右和向下移动以匹配定义的平移：

![](https://koenig-media.raywenderlich.com/uploads/2018/12/Playground_CirclePattern_Change_Transform.png)

除了平移模式外，还可以缩放和旋转模式单元格。我们将在后面为应用程序构建模式时看到如何旋转模式。

下面是我们如何填充和描绘彩色模式。 在 drawPattern 中替换：

```objc
context.setFillColor(UIColor.black.cgColor)
context.fillPath()  
```

还有以下代码：

```objc
context.setFillColor(UIColor.yellow.cgColor)
context.setStrokeColor(UIColor.darkGray.cgColor)
context.drawPath(using: .fillStroke)
```

这里，将填充颜色更改为黄色并设置描边颜色。 然后使用填充和描边路径的选项调用 `drawPath(using:)`。

运行 playground，检查模式现在显示的新填充颜色和描边：

![](https://koenig-media.raywenderlich.com/uploads/2018/12/Playground_CirclePattern_StrokeFill.png)

到目前为止，我们已经使用了彩色图案并在模式绘制方法中定义了颜色。在完成的应用程序中，必须创建具有不同颜色的模式。我们可能意识到为每种颜色编写绘图方法不是可行的方法。 这就是蒙板模式(masking pattern)发挥作用的地方了。

## Masking Patterns

蒙板模式在模式单元格 `draw` 方法之外定义其颜色信息。 这允许更改图案颜色以满足我们的需要。

这是一个没有颜色的蒙板模式的例子：

![](https://koenig-media.raywenderlich.com/uploads/2018/12/Pattern_Stencil_3.png)

有了模式后，我们现在就可以应用颜色。下面的第一个示例显示应用于蒙版的蓝色，第二个示例显示橙色：

![](https://koenig-media.raywenderlich.com/uploads/2018/12/Pattern_Stencil_Repeated.png)

现在，把之前的模式更改为蒙版模式。

替换下面 drawPattern 的代码：

```objc
context.setFillColor(UIColor.yellow.cgColor)
context.setStrokeColor(UIColor.darkGray.cgColor)
context.drawPath(using: .fillStroke)
```

改为：

```objc
context.fillPath()
```

这会将代码恢复为填充路径。

用以下内容替换 `pattern` 赋值：

```objc
let pattern = CGPattern(
      info: nil,
      bounds: CGRect(x: 0, y: 0, width: 20, height: 20),
      matrix: transform,
      xStep: 25,
      yStep: 25,
      tiling: .constantSpacing,
      isColored: false,
      callbacks: &callbacks)
```

这会将 `isColored` 参数设置为 `false`，从而将模式更改为蒙版模式。我们还将垂直和水平间距增加到 25。现在需要为模式提供颜色空间信息。

用以下内容替换 patternSpace 赋值：

```objc
let baseSpace = CGColorSpaceCreateDeviceRGB()
let patternSpace = CGColorSpace(patternBaseSpace: baseSpace)!
```

在这里，我们将获得对标准设备相关 RGB 颜色空间的引用。然后，将模式颜色空间更改为此值，而不是之前的 nil 值。

替换下面些代码：

```objc
var alpha : CGFloat = 1.0
context.setFillPattern(pattern!, colorComponents: &alpha)
```

改为以下代码：

```objc
let fillColor: [CGFloat] = [0.0, 1.0, 1.0, 1.0]
context.setFillPattern(pattern!, colorComponents: fillColor)
```

填充模式时，会在蒙版下面创建一种颜色。

运行 playground。模式的颜色变以青色，以反映在 draw 方法之外配置的设置：

![](https://koenig-media.raywenderlich.com/uploads/2018/12/Playground_CirclePattern_Stencil.png)

是时候看看如何描边和填充蒙板模式了。

使用以下内容替换 drawPattern 定义中的 `context.fillPath()` 行：

```objc
context.setStrokeColor(UIColor.darkGray.cgColor)
context.drawPath(using: .fillStroke)
```

虽然在 `draw(_:)` 中设置了描边颜色，但模板颜色仍然在方法之外设置。

运行 playground 以查看描边模式：

![](https://koenig-media.raywenderlich.com/uploads/2018/12/Playground_CirclePattern_Stencil_FillStroke.png)

现在我们已经积累了不同模式配置和屏蔽模式的经验。现在可以开始构建 Recall 所需的模式了。

## 创建游戏的模式

将以下 playground 中的代码添加到视图控制器中：

```objc
extension UIBezierPath {
  // 1
  convenience init(triangleIn rect: CGRect) {
    self.init()
    // 2
    let topOfTriangle = CGPoint(x: rect.width / 2, y: 0)
    let bottomLeftOfTriangle = CGPoint(x: 0, y: rect.height)
    let bottomRightOfTriangle = CGPoint(x: rect.width, y: rect.height)
    // 3
    self.move(to: topOfTriangle)
    self.addLine(to: bottomLeftOfTriangle)
    self.addLine(to: bottomRightOfTriangle)
    // 4
    self.close()
  }
}
```

逐步分析一下代码：

1. 扩展 UIBezierPath 以创建三角形路径；
2. 指定构成三角形的三个点；
3. 从顶部开始绘制三角形。 move(to:) 开始路径，addLine(to:) 将路径添加到路径中。
4. 关闭路径。

将以下结构添加到 `PatternView` 的顶部：

```objc
public struct Constants {
  static let patternSize: CGFloat = 30.0
  static let patternRepeatCount = 2
}
```

这些是在设置模式时将使用的常量。`patternSize` 定义模式单元格大小。 `patternRepeatCount` 定义模式视图中的模式单元格数。

在常量定义后添加以下内容：

```objc
let drawTriangle: CGPatternDrawPatternCallback = { _, context in
  let trianglePath = UIBezierPath(triangleIn:
    CGRect(x: 0, y: 0,
           width: Constants.patternSize,
           height: Constants.patternSize))
  context.addPath(trianglePath.cgPath)
  context.fillPath()
}
```

这定义了一个用于绘制三角形图案的新功能。调用 `UIBezierPath(triangleIn:)` 返回表示三角形的路径。然后，在绘制之前将此路径添加到上下文中。

请注意，该函数未指定填充颜色，因此它可以是蒙板模式。

在 `draw(_:)` 中，将回调更改为以下内容：

```objc
var callbacks = CGPatternCallbacks(
  version: 0, drawPattern: drawTriangle, releaseInfo: nil)
```

我们现在正在使用三角绘图功能。

删除 `drawPattern`，因为它不再需要它了。

同样在 `draw(_:)` 中，用以下代码替换分配 `transform` 和 `pattern` 的代码：

```objc
// 1
let patternStepX: CGFloat =
  rect.width / CGFloat(Constants.patternRepeatCount)
let patternStepY: CGFloat =
  rect.height / CGFloat(Constants.patternRepeatCount)
// 2
let patternOffsetX: CGFloat = (patternStepX - Constants.patternSize) / 2.0
let patternOffsetY: CGFloat = (patternStepY - Constants.patternSize) / 2.0
// 3
let transform = CGAffineTransform(translationX: patternOffsetX, y: patternOffsetY)
// 4
let pattern = CGPattern(
  info: nil,
  bounds: CGRect(
    x: 0, 
    y: 0, 
    width: Constants.patternSize, 
    height: Constants.patternSize),
  matrix: transform,
  xStep: patternStepX,
  yStep: patternStepY,
  tiling: .constantSpacing,
  isColored: false,
  callbacks: &callbacks)
```

分析下代码：

1. 使用视图的宽度和高度以及视图中的模式单元格数计算水平和垂直步长；
2. 计算出尺寸，使模式单元在其边界内水平和垂直居中；
3. 根据您定义的居中变量设置 `CGAffineTransform` 转换；
4. 根据计算出的参数创建模式对象

运行 playground。应该在每个方向上看到两个三角形，在其边界内垂直和水平居中：

![](https://koenig-media.raywenderlich.com/uploads/2018/12/Playground_TrianglePattern_1.png)

现在，我们将获得更接近 Recall 应用程序的背景颜色。

在 `MyViewController` 中，更改 `loadView()` 中的背景颜色设置，如下所示：

```objc
view.backgroundColor = .lightGray
```

接下来转到 `PatternView` 并在 `draw(_:)` 中更改上下文填充设置，如下所示：

```objc
UIColor.white.setFill()
```

运行 playground。主视图背景现在应该是灰色的，模板视图为白色背景：

![](https://koenig-media.raywenderlich.com/uploads/2018/12/Playground_Background_Switch.png)

### 自定义模板视图

现在我们已正确显示基本模式，可以进行更改以控制模式方向。

在 PatternView 顶部附近添加以下枚举：

```objc
enum PatternDirection: CaseIterable {
  case left
  case top
  case right
  case bottom
}
```

这表示三角形可以指向的不同方向。它们与起始应用中的路线相符。

在 `PatternDirection` 定义之后添加以下类属性：

```objc
var fillColor: [CGFloat] = [1.0, 0.0, 0.0, 1.0]
var direction: PatternDirection = .top
```

这是我们将使用的蒙版模式的颜色和模式方向。该类设置默认颜色为红色，默认方向为 top。

删除在 `draw(_:)` 底部附近的 `fillColor` 声明。这将确保我们使用 class 属性。

用以下内容替换 `transform` 赋值：

```objc
// 1
var transform: CGAffineTransform
// 2
switch direction {
case .top:
  transform = .identity
case .right:
  transform = CGAffineTransform(rotationAngle: CGFloat(0.5 * .pi))
case .bottom:
  transform = CGAffineTransform(rotationAngle: CGFloat(1.0 * .pi))
case .left:
  transform = CGAffineTransform(rotationAngle: CGFloat(1.5 * .pi))
}
// 3
transform = transform.translatedBy(x: patternOffsetX, y: patternOffsetY)
```

以下是代码的分析：

1. 为模式转换声明一个 `CGAffineTransform` 变量。
2. 如果模式方向为 top，则将变换设置为单位矩阵。否则，变换是基于方向的旋转。 例如，如果模式指向右侧，则旋转为 `π/2` 弧度或顺时针 90º。
3. 应用 `CGAffineTransform` 转换以使模式单元在其边界内居中。

运行 `playground`。根据设置的默认模式填充颜色，三角形为红色：

![](https://koenig-media.raywenderlich.com/uploads/2018/12/Playground_TrianglePattern_2.png)

现在是设置代码来控制和测试模板颜色和方向的好时机。

在 `PatternView` 的类属性定义之后添加以下方法：

```objc
init(fillColor: [CGFloat], direction: PatternDirection = .top) {
  self.fillColor = fillColor
  self.direction = direction
  super.init(frame: CGRect.zero)
}
  
required init?(coder aDecoder: NSCoder) {
  super.init(coder: aDecoder)
}
```

这将设置一个初始化器，它接收填充颜色和模板方向。`direction` 参数具有默认值。

还在 storyboard 初始化视图时添加了所需的初始化程序。将代码传输到应用程序后，将需要此功能。

在 `MyViewController` 中，由于我们更改了初始化程序，所以更改 `patternView` 赋值：

```objc
let patternView = PatternView(
  fillColor: [0.0, 1.0, 0.0, 1.0],
  direction: .right)
```

在这里，我们使用非默认值实例化模式视图。

运行 playground。我们的三角形现在是绿色的并指向右边：

![](https://koenig-media.raywenderlich.com/uploads/2018/12/Playground_TrianglePattern_Init.png)

我们已经使用 Playgrounds 对模式进行了原型设计。是时候在 Recall 中使用这种模式了。

## 更新 App 的模式视图

打开 **Recall** 启动项目。转到 **PatternView.swift** 并将 `UIBezierPath` 扩展从 playground 复制到文件末尾。

接下来，将 **PatternView.swift** 中的 `PatternView` 类替换为 playground 中的类。

> **注意**：通过使用 Xcode 的代码折叠功能，可以大大简化此过程。 在 playground 中，将光标放在 `class PatternView: UIView {` 后面并选择菜单中的 **Editor ▸ Code Folding ▸ Fold**。双击生成的折叠线以选择整个类，然后按 **Command-C**。在项目中，重复该过程以折叠并选择该类。按 **Command-V** 替换它。

构建并运行应用程序。应该看到如下界面：

![](https://koenig-media.raywenderlich.com/uploads/2018/12/Recall_after_prototype_copied.png)

有点不对劲。模式似乎卡在默认模式下了。看起来新游戏视图没有刷新模式视图。

转到 **GameViewController.swift** 并将以下内容添加到 `setupPatternView(_:towards:havingColor:)` 的末尾：

```objc
patternView.setNeedsDisplay()
```

这会提示系统重绘模式，以便它获取新的模式信息。

构建并运行应用程序。 现在应该看到混合的颜色和方向：

![](https://koenig-media.raywenderlich.com/uploads/2018/12/Recall_all_done.png)

点击其中一个答案按钮并玩通游戏以检查一切是否按预期工作。

恭喜完成 Recall！我们已经从简单的绘图工作的麻烦日子走了很长一段路。

![](https://koenig-media.raywenderlich.com/uploads/2018/12/RageFace_PaintNinja.png)

## 性能

Core Graphics 模式非常快。以下是可用于绘制模式的几个选项：

1. 使用在本教程中学习的 Core Graphics 模式 API；
2. 使用 UIKit 包装的方法，如 `UIColor(patternImage:)`；
3. 在具有许多 Core Graphics 调用的循环中绘制所需的模式；

如果模式只绘制一次，则 UIKit 包装器方法最简单。它的性能也应该与较低级别的 Core Graphics 调用相当。一个例子是背景模式。

Core Graphics 可以在后台线程中工作，而不像 UIKit，它在主线程上运行。Core Graphics 模式在复杂的绘图或动态模式下有更优的性能。

在循环中绘制图案将是最慢的。Core Graphics 模式使绘制调用一次并缓存结果，使其更有效。

## 下一步

现在我们应该对如何使用 Core Graphics 创建模式有充分的了解。查看 `Patterns and Playgrounds` (https://www.raywenderlich.com/409-core-graphics-tutorial-part-3-patterns-and-playgrounds)教程，了解如何使用 UIKit 构建模式。

您可能还想阅读 `Curves and Layers` (https://www.raywenderlich.com/2741-core-graphics-tutorial-curves-and-layers)教程，该教程侧重于绘制各种曲线并使用图层克隆它们。

