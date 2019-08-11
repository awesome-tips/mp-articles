| 作者：Lorenzo Boaro
| 链接：https://www.raywenderlich.com/349664-core-graphics-tutorial-arcs-and-paths

在本教程中，我们将学习如何绘制圆弧和路径。特别是，我们将 Grouped TableView 的每个页脚的底部添加整齐的弧线、线性渐变和适合弧形曲线的阴影，来美化我们的 table view。所有这些都是通过使用 Core Graphics 的强大功能实现的！

## 开始

在本教程中，我们将使用 **LearningAgenda** 示例程序，这是一个 iOS 应用程序，演示了我们要学习的内容。

首先下载初始工程(https://koenig-media.raywenderlich.com/uploads/2019/01/LearningAgenda-2.zip)。下载后，在 Xcode 中打开**LearningAgenda.xcodeproj**。

为了让我们更专注于主要内容，初始项目已设置好了与圆弧和路径无关的所有内容。

构建并运行应用程序，我们将看到以下界面：

![](https://koenig-media.raywenderlich.com/uploads/2019/01/ss1-1.png)

如图所示，有一个分组的 table view，包含两个部分，每个部分都有一个标题和三行内容。我们在这里要做的所有工作就是在每个部分下方创建弧形页脚。

## 增强页脚

在实现功能之前，我们需要创建并设置一个自定义的页脚，这将作为我们后续工作的基础。

要为闪亮的新页脚创建类，可以右键单击 **LearningAgenda** 文件夹，然后选择“**新建文件**”。 接下来，选择 **Swift File** 并将文件命名为 **CustomFooter.swift**。

切换到 **CustomFooter.swift** 文件并使用以下代码替换其内容：

```objc
import UIKit

class CustomFooter: UIView {
  override init(frame: CGRect) {
    super.init(frame: frame)
    
    isOpaque = true
    backgroundColor = .clear
  }
  
  required init?(coder aDecoder: NSCoder) {
    fatalError("init(coder:) has not been implemented")
  }

  override func draw(_ rect: CGRect) {
    let context = UIGraphicsGetCurrentContext()!
    
    UIColor.red.setFill()
    context.fill(bounds)
  }
}
```

这里，我们重写 `init(frame:)` 以设置 `isOpaquetrue`。我们还将背景颜色设置为 `clear`。

> 注意：当视图完全或部分透明时，不应使用 `isOpaque` 属性。否则，结果可能无法预测。

我们同样重写 `init?(coder:)`，因为它是必需的，但是我们不提供任何实现，因为我们不会在 `Interface Builder` 中使用自定义页脚视图。

`draw(_:)` 使用 `Core Graphics` 提供自定义 rect 的内容。我们将红色设置为填充颜色以覆盖页脚本身的整个 bounds。

现在，打开 **TutorialsViewController.swift** 并将以下两个方法添加到文件底部的 `UITableViewDelegate` 扩展中：

```objc
func tableView(_ tableView: UITableView, heightForFooterInSection section: Int) -> CGFloat {
  return 30
}
  
func tableView(_ tableView: UITableView, viewForFooterInSection section: Int) -> UIView? {
  return CustomFooter()
}
```

上述方法组合形成 30 个 point 高度的自定义页脚视图。

构建并运行项目，如果一切正常，我们应该看到以下内容：

![](https://koenig-media.raywenderlich.com/uploads/2019/01/ss2-1-281x500.png)

### 回到业务

好了，既然我们已经有了一个占位符视图，那么现在是时候了。但首先我们来设定一个目标。

![](https://koenig-media.raywenderlich.com/uploads/2018/12/arcs-and-paths-footer-to-build-1.png)

上图有以下几点可以注意一下：

* 页脚视图底部是一个整齐的弧形；
* 从浅灰色渐变到深灰色；
* 阴影与弧形曲线一致

### 圆弧后面的数学

圆弧是表示圆的一部分的曲线。在上面这种情况下，页脚视图底部所需的圆弧是一个非常大的圆的顶部，具有非常大的半径，从某个起始角度到某个结束角度。

![](https://koenig-media.raywenderlich.com/uploads/2018/12/arcs-and-paths-diagram.png)

那我们如何向 Core Graphics 描述这个弧？我们将使用 `CGContext` 的 `addArc(center:radius:startAngle:endAngle:clockwise:)` 方法。该方法需要以下五个输入参数：

* 圆的中心点
* 圆的半径
* 绘制线的起点，也称为起始角度
* 绘制线的终点，也称为结束角度
* 圆弧的方向

但是我们又该如何去设置这些值呢？

![](https://koenig-media.raywenderlich.com/uploads/2018/12/arcs-and-paths-frustrated.png)

我们需要一些简单的数学知识，并计算出所有这些值！

我们知道的第一件事是想要绘制弧的边界框的大小：

![](https://koenig-media.raywenderlich.com/uploads/2018/12/arcs-and-paths-box.png)

我们知道的第二件事是一个有趣的数学定理，称为**相交和弦定理**。基本上，这个定理指出，如果你在一个圆中绘制两个交叉的和弦，第一个和弦的分段的乘积将等于第二个和弦的分段的乘积。请记住，和弦是连接圆中两个点的线。

![](https://koenig-media.raywenderlich.com/uploads/2018/12/arcs-and-paths-chord-thereom-480x500.png)

> 注意：如果我们想了解其原因，请访问 http://www.mathopenref.com/chordsintersecting.html - 它有一个很酷的小型 JavaScript 演示，我们可以直接使用。

有了这两点知识，看看如果我们画出如下两个和弦时会发生什么：

![](https://koenig-media.raywenderlich.com/uploads/2018/12/arcs-and-paths-to-be-cords.png)

因此，绘制一条线连接弧形矩形的底点和从弧形顶部向下到圆形底部的另一条线。

如果我们这样做，知道了 **a**，**b** 和 **c**，就可以得出 **d**。

所以 d 的计算公式是：**(a * b) / c**。用它代替，会是：

```objc
// Just substituting...
let d = ((arcRectWidth / 2) * (arcRectWidth / 2)) / (arcRectHeight);
// Or more simply...
let d = pow(arcRectWidth, 2) / (4 * arcRectHeight);
```

现在我们知道了 **c** 和 **d**，就可以使用以下公式计算半径：**(c + d) / 2**：

```objc
// Just substituting...
let radius = (arcRectHeight + (pow(arcRectWidth, 2) / (4 * arcRectHeight))) / 2;
// Or more simply...
let radius = (arcRectHeight / 2) + (pow(arcRectWidth, 2) / (8 * arcRectHeight));
```

现在我们已经知道了半径，只需从阴影矩形的中心点减去半径即可获得中心：

```objc
let arcCenter = CGPoint(arcRectTopMiddleX, arcRectTopMiddleY - radius)
```

一旦知道了中心点，半径和圆弧矩形，就可以用一些三角函数计算起点和终点角度：

![](https://koenig-media.raywenderlich.com/uploads/2018/12/arcs-and-paths-trig-n-1.png)

我们首先计算出图中所示的角度。如果我们还记得 `SOHCAHTOA`(https://en.wikipedia.org/wiki/Mnemonics_in_trigonometry#SOH-CAH-TOA)，可能会想起角度的余弦等于三角形相邻边缘的长度除以斜边的长度。

换句话说，`cosine(angle) = (arcRectWidth / 2) / radius`。因此，为了得到角度，我们只需要取余弦，它是余弦的倒数：

```objc
let angle = acos((arcRectWidth / 2) / radius)
```

现在我们知道了这个角度，获得起点和终点角度应该相当简单：

![](https://koenig-media.raywenderlich.com/uploads/2018/12/arcs-and-paths-angles-471x500.png)

现在我们了解了如何去做，就可以将它们写到一个函数里去了。

> 注意：顺便说一句，使用 `CGContext` 类型中提供的 `addArc(tangent1End:tangent2End:radius:)` 方法，可以更简单地绘制这样的弧。

### 绘制圆弧和创建路径

我们所做的第一件事是将度数转换为弧度的方法。为此，将使用在 iOS 10 和 macOS 10.12 中引入的 **Foundation Units 和 Measurements** API。

> Foundation 框架提供了一种使用和表示物理量的强大方法。除角度外，它还提供了几种内置单元类型，如速度，持续时间等。

打开 `Extensions.swift` 并将以下代码粘贴到文件末尾：

```objc
typealias Angle = Measurement<UnitAngle>

extension Measurement where UnitType == UnitAngle {  
  init(degrees: Double) {
    self.init(value: degrees, unit: .degrees)
  }

  func toRadians() -> Double {
    return converted(to: .radians).value
  }
}
```

在上面的代码中，我们可以在 `Measurement` 类型上定义一个扩展，将其用法限制为角度单位。 `init(degrees:)` 仅适用于度数角度。`toRadians()` 允许我们将度数转换为弧度。

> 注意：也可以使用公式 `radians = degrees * π / 180` 将度数转换为弧度，反之亦然。

保留在 **Extensions.swift** 文件中，找到 `CGContext` 的扩展块。在最后一个花括号之前，粘贴以下代码：

```objc
static func createArcPathFromBottom(
  of rect: CGRect, 
  arcHeight: CGFloat, 
  startAngle: Angle, 
  endAngle: Angle
) -> CGPath {
  // 1
  let arcRect = CGRect(
    x: rect.origin.x, 
    y: rect.origin.y + rect.height, 
    width: rect.width, 
    height: arcHeight)
  
  // 2
  let arcRadius = (arcRect.height / 2) + pow(arcRect.width, 2) / (8 * arcRect.height)
  let arcCenter = CGPoint(
    x: arcRect.origin.x + arcRect.width / 2, 
    y: arcRect.origin.y + arcRadius)    
  let angle = acos(arcRect.width / (2 * arcRadius))
  let startAngle = CGFloat(startAngle.toRadians()) + angle
  let endAngle = CGFloat(endAngle.toRadians()) - angle
  
  let path = CGMutablePath()
  // 3
  path.addArc(
    center: arcCenter, 
    radius: arcRadius, 
    startAngle: startAngle, 
    endAngle: endAngle, 
    clockwise: false)
  path.addLine(to: CGPoint(x: rect.maxX, y: rect.minY))
  path.addLine(to: CGPoint(x: rect.minX, y: rect.minY))
  path.addLine(to: CGPoint(x: rect.minY, y: rect.maxY))
  // 4
  return path.copy()!
}
```

到这已经进展了不少，以下是具体描述：

* 此函数使用整个区域的矩形和弧度应该有多大的浮点数。请记住，圆弧应位于矩形的底部。我们可以根据这两个值计算 `arcRect`。
* 然后，通过上面讨论的数学公式计算出半径，中心，起点和终点角度。
* 接下来，创建路径。路径将由圆弧和弧上方矩形边缘周围的线组成。
* 最后，返回路径的不可变副本。我们不希望从函数外部修改路径。

> 注意：与 `CGContext` 扩展中可用的其他函数不同，`createArcPathFromBottom(of:arcHeight:startAngle:endAngle:)` 返回 `CGPath`。这是因为路径将被重复使用多次。稍后会详细介绍。

现在我们有了一个辅助方法来绘制弧线，现在是时候用我们的新弧形替换你的矩形页脚视图了。

打开 **CustomFooter.swift** 并使用以下代码替换 `draw(_:)`：

```objc
override func draw(_ rect: CGRect) { 
  let context = UIGraphicsGetCurrentContext()!
  
  let footerRect = CGRect(
    x: bounds.origin.x, 
    y: bounds.origin.y, 
    width: bounds.width, 
    height: bounds.height)
  
  var arcRect = footerRect
  arcRect.size.height = 8
  
  context.saveGState()
  let arcPath = CGContext.createArcPathFromBottom(
    of: arcRect, 
    arcHeight: 4, 
    startAngle: Angle(degrees: 180), 
    endAngle: Angle(degrees: 360))
  context.addPath(arcPath)
  context.clip()

  context.drawLinearGradient(
    rect: footerRect, 
    startColor: .rwLightGray, 
    endColor: .rwDarkGray)
  context.restoreGState()
}
```

在通常的 Core Graphics 设置之后，我们将为整个页脚视图区域和想要圆弧的区域创建一个边界框。

然后，通过调用刚刚编写的 `createArcPathFromBottom(of:arcHeight:startAngle:endAngle:)` 静态方法获得弧形路径。然后，我们可以将路径添加到上下文并剪切到该路径。

后续进一步的绘图将限于该路径。然后，可以使用 **Extensions.swift** 中的 `drawLinearGradient(rect:startColor:endColor:)` 绘制从浅灰色到深灰色的渐变。

构建并运行应用程序。如果一切正常，我们应该看到以下界面：

![](https://koenig-media.raywenderlich.com/uploads/2019/01/ss3-1.png)

看起来不错，但我们需要再完善一下。

## 裁剪，路径和偶数规则

在 **CustomFooter.swift** 中，将以下内容添加到 `draw(_:)` 的底部：

```objc
context.addRect(footerRect)
context.addPath(arcPath)
context.clip(using: .evenOdd)
context.addPath(arcPath)
context.setShadow(
  offset: CGSize(width: 0, height: 2), 
  blur: 3, 
  color: UIColor.rwShadow.cgColor)
context.fillPath()
```

这里有一个新的，非常重要的概念。

要绘制阴影，请启用阴影绘制，然后填充路径。然后 Core Graphics 将填充路径并在下方绘制适当的阴影。

但是我们已经使用渐变填充了路径，因此并不希望用颜色覆盖该区域。

嗯，这听起来像裁剪工作！我们可以设置裁剪，以便 Core Graphics 仅绘制页脚区域外部分。然后，我们可以告诉它填充页脚区域并绘制阴影。由于设置了裁剪，页脚区域填充将被忽略，但阴影将显示。

但是我们没有这样一条路径 - 唯一的路径是页脚区域而不是外部区域。

使用 Core Graphics 的一些功能，我们可以轻松地根据内部获取外部路径。我们只需向上下文添加多个路径，然后使用 Core Graphics 提供的特定规则添加裁剪。

当我们向上下文添加多个路径时，Core Graphics 需要某种方式来确定是否应该填充哪些点。例如，你可以有一个圆圈形状，其中外部是填充但内部是空的，或者是圆环形状，其中内部填充但外部是空的。

我们可以指定不同的算法让 Core Graphics 知道如何处理它。本教程中将使用的算法是 **EO**，甚至是 **even-odd**。

在 EO 中，对于任何给定点，Core Graphics 将从该点绘制一条线到绘图区域的外部。如果该线穿过奇数个点，它将被填充。如果它穿过偶数个点，则不会被填充。

以下是 Quartz2D Programming Guide 中的图示：

![](https://koenig-media.raywenderlich.com/uploads/2018/12/arcs-and-paths-even-odd.png)

因此，通过使用 EO 变体，我们告诉 Core Graphics，即使已经向上下文添加了两条路径，它也应该将其视为遵循 EO 规则的一条路径。因此，外部部分，即整个页脚矩形，应该被填充，但内部部分，即弧形路径则不应该。我们告诉 Core Graphics 剪切到该路径并仅在外部区域绘制。

设置裁剪区域后，添加弧的路径，设置阴影并填充圆弧。当然，由于它被剪裁，实际上什么都没有被填充，但阴影仍将被绘制在外部区域！

构建并运行项目，如果一切顺利，我们现在应该看到页脚下方的阴影：

![](https://koenig-media.raywenderlich.com/uploads/2019/01/ss4-1.png)

恭喜！我们已使用 Core Graphics 创建了自定义 table view 的页脚视图！

## 下一步去哪？

我们可以下载项目的完整版本。

通过本教程，我们已经学习了如何创建圆弧和路径。现在，可以将这些概念直接应用到应用中！

如果您想了解有关 Core Graphics 的更多信息，请查看 Quartz 2D Programming Guide。


