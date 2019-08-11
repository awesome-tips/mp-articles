## 背景

框架自带的 Tab Bar 相信大家已经熟悉得不能再熟悉了，一般使用的时候不过是设置两个图标代表选中和未选中两种状态，难免有一些平淡。后来很多控件就在标签选中时进行一些比较抓眼球的动画，不过我觉得大部分都是为了动画而动画。直到后来我看到 `Outlook` 客户端的动画时，我才意识到原来还可以跟用户的交互结合在一起。

![图1 标签图标跟随手势进行不同的动画](https://user-gold-cdn.xitu.io/2019/3/31/169d2e30faad28fc?imageslim)

有意思吧，不过本文并不是要仿制个一模一样的出来，会有稍微变化：

![图2 本文完成的最终效果](https://user-gold-cdn.xitu.io/2019/4/3/169e07e327bcffb9?imageslim)

## 实现分析

写代码之前，咱先讨论下实现的方法，相信你已经猜到标签页的图标显然已经不是图片，而是一个自定义的UIView。将一个视图挂载到原本图标的位置并不是一件难事，稍微有些复杂的是数字滚轮效果的实现，别看它数字不停地在滚动，仔细看其实最多显示2种数字，也就说只要2个Label就够了。

> 基于篇幅，文章不会涉及右侧的时钟效果，感兴趣请直接参考源码。

## 数字滚轮

打开项目 `TabBarInteraction`，新建文件 `WheelView.swift`，它是 `UIView` 的子类。首先设置好初始化函数：

```objc
class WheelView: UIView {
    required init?(coder aDecoder: NSCoder) {
        super.init(coder: aDecoder)
        setupView()
    }
    override init(frame: CGRect) {
        super.init(frame: frame)
        setupView()
    }
}
```

接着创建两个Label实例，代表滚轮中的上下两个Label：

```objc
private lazy var toplabel: UILabel = {
    return createDefaultLabel()
}()

private lazy var bottomLabel: UILabel = {
    return createDefaultLabel()
}()

private func createDefaultLabel() -> UILabel {
    let label = UILabel() 
    label.textAlignment = NSTextAlignment.center
    label.adjustsFontSizeToFitWidth = true
    label.translatesAutoresizingMaskIntoConstraints = false
    return label
}
```

现在来完成 `setupView()` 方法，在这方法中将上述两个 Label 添加到视图中，然后设置约束将它们的四边都与 `layoutMarginsGuide` 对齐。

```objc
private func setupView() {
    translatesAutoresizingMaskIntoConstraints = false
    for label in [toplabel, bottomLabel] {
        addSubview(label)
        NSLayoutConstraint.activate([
            label.topAnchor.constraint(equalTo: layoutMarginsGuide.topAnchor),
            label.bottomAnchor.constraint(equalTo: layoutMarginsGuide.bottomAnchor),
            label.leftAnchor.constraint(equalTo: layoutMarginsGuide.leftAnchor),
            label.rightAnchor.constraint(equalTo: layoutMarginsGuide.rightAnchor)
        ])
    }
}
```

有人可能会问现在这样两个 Label 不是重叠的状态吗？不着急，接下来我们会根据参数动态地调整它们的大小和位置。添加两个实例变量 `progress` 和 `contents`，分别表示滚动的总体进度和显示的全部内容。

```objc
var progress: Float = 0.0
var contents = [String]()
```

我们接下来要根据这两个变量计算出当前两个Label显示的内容以及它们的缩放位置。这些计算都在 `progress` 的 `didSet` 里完成：

```objc
var progress: Float = 0.0 {
    didSet {
        progress = min(max(progress, 0.0), 1.0) 
        guard contents.count > 0 else { return }
        
        /** 根据 progress 和 contents 计算出上下两个 label 显示的内容以及 label 的压缩程度和位置
         *
         *  Example: 
         *  progress = 0.4, contents = ["A","B","C","D"]
         *
         *  1）计算两个label显示的内容
         *  topIndex = 4 * 0.4 = 1.6, topLabel.text = contents[1] = "B"
         *  bottomIndex = 1.6 + 1 = 2.6, bottomLabel.text = contents[2] = "C" 
         *  
         *  2） 计算两个label如何压缩和位置调整，这是实现滚轮效果的原理
         *  indexOffset = 1.6 % 1 = 0.6
         *  halfHeight = bounds.height / 2
         *  ┌─────────────┐             ┌─────────────┐                                 
         *  |┌───────────┐|   scaleY    |             |                            
         *  ||           || 1-0.6=0.4   |             | translationY       
         *  ||  topLabel || ----------> |┌─ topLabel─┐| ------------------   
         *  ||           ||             |└───────────┘| -halfHeight * 0.6 ⎞    ┌─────────────┐
         *  |└───────────┘|             |             |                   ⎥    |┌─ toplabel─┐|
         *  └─────────────┘             └─────────────┘                   ⎟    |└───────────┘|
         *                                                                 ❯   |┌───────────┐|
         *  ┌─────────────┐             ┌─────────────┐                   ⎟    ||bottomLabel||
         *  |┌───────────┐|   scaleY    |             |                   ⎟    |└───────────┘|
         *  ||           ||    0.6      |┌───────────┐| translationY      ⎠    └─────────────┘
         *  ||bottomLabel|| ----------> ||bottomLabel|| -----------------    
         *  ||           ||             |└───────────┘| halfHeight * 0.4     
         *  |└───────────┘|             |             |                      
         *  └─────────────┘             └─────────────┘                      
         *
         * 可以想象出，当 indexOffset 从 0.0 递增到 0.999 过程中，
         * topLabel 从满视图越缩越小至0，而 bottomLabel刚好相反越变越大至满视图，即形成一次完整的滚动
         */
        let topIndex = min(max(0.0, Float(contents.count) * progress), Float(contents.count - 1))
        let bottomIndex = min(topIndex + 1, Float(contents.count - 1))
        let indexOffset =  topIndex.truncatingRemainder(dividingBy: 1)
        
        toplabel.text = contents[Int(topIndex)]
        toplabel.transform = CGAffineTransform(scaleX: 1.0, y: CGFloat(1 - indexOffset))
            .concatenating(CGAffineTransform(translationX: 0, y: -(toplabel.bounds.height / 2) * CGFloat(indexOffset)))
            
        bottomLabel.text = contents[Int(bottomIndex)]
        bottomLabel.transform = CGAffineTransform(scaleX: 1.0, y: CGFloat(indexOffset))
            .concatenating(CGAffineTransform(translationX: 0, y: (bottomLabel.bounds.height / 2) * (1 - CGFloat(indexOffset))))
    }
}
```

最后我们还要向外公开一些样式进行自定义：

```objc
extension WheelView {
    /// 前景色变化事件
    override func tintColorDidChange() {
        [toplabel, bottomLabel].forEach { $0.textColor = tintColor }
        layer.borderColor = tintColor.cgColor
    }
    /// 背景色
    override var backgroundColor: UIColor? {
        get { return toplabel.backgroundColor }
        set { [toplabel, bottomLabel].forEach { $0.backgroundColor = newValue } }
    }
    /// 边框宽度
    var borderWidth: CGFloat {
        get { return layer.borderWidth }
        set {
            layoutMargins = UIEdgeInsets(top: newValue, left: newValue, bottom: newValue, right: newValue)
            layer.borderWidth = newValue
        }
    }
    /// 字体
    var font: UIFont {
        get { return toplabel.font }
        set { [toplabel, bottomLabel].forEach { $0.font = newValue } }
    }
}
```

至此，整个滚轮效果已经完成。

## 挂载视图

在 `FirstViewController` 中实例化刚才自定义的视图，设置好字体、边框、背景色、Contents 等内容，别忘了 `isUserInteractionEnabled` 设置为 `false`，这样就不会影响原先的事件响应。

```objc
 override func viewDidLoad() {
        super.viewDidLoad()
        // Do any additional setup after loading the view.
        
        tableView.delegate = self
        tableView.dataSource = self
        tableView.register(UITableViewCell.self, forCellReuseIdentifier: "DefaultCell")
        tableView.rowHeight = 44

        wheelView = WheelView(frame: CGRect.zero)
        wheelView.font = UIFont.systemFont(ofSize: 15, weight: .bold)
        wheelView.borderWidth = 1
        wheelView.backgroundColor = UIColor.white
        wheelView.contents = data
        wheelView.isUserInteractionEnabled = false
}
```

然后要把视图挂载到原先的图标上，`viewDidLoad()` 方法底部新增代码：

```objc
 override func viewDidLoad() {
    ...
    guard let parentController = self.parent as? UITabBarController else { return }
    let controllerIndex = parentController.children.firstIndex(of: self)!
    var tabBarButtons = parentController.tabBar.subviews.filter({
        type(of: $0).description().isEqual("UITabBarButton")
    })
    guard !tabBarButtons.isEmpty else { return }
    let tabBarButton = tabBarButtons[controllerIndex]
    let swappableImageViews = tabBarButton.subviews.filter({
        type(of: $0).description().isEqual("UITabBarSwappableImageView")
    })
    guard !swappableImageViews.isEmpty else { return }
    let swappableImageView = swappableImageViews.first!
    tabBarButton.addSubview(wheelView)
    swappableImageView.isHidden = true
    NSLayoutConstraint.activate([
        wheelView.widthAnchor.constraint(equalToConstant: 25),
        wheelView.heightAnchor.constraint(equalToConstant: 25),
        wheelView.centerXAnchor.constraint(equalTo: swappableImageView.centerXAnchor),
        wheelView.centerYAnchor.constraint(equalTo: swappableImageView.centerYAnchor)
    ])
 }
```

上述代码的目的是最终找到对应标签 `UITabBarButton` 内类型为 `UITabBarSwappableImageView` 的视图并替换它。看上去相当复杂，但是它尽可能地避免出现意外情况导致程序异常。只要以后 UIkit 不更改类型 `UITabBarButton` 和 `UITabBarSwappableImageView`，以及他们的包含关系，程序基本不会出现意外，最多导致自定义的视图挂载不上去而已。另外一个好处是 `FirstViewController` 不用去担心它被添加到 `TabBarController` 中的第几个标签上。总体来说这个方法并不完美，但目前似乎也没有更好的方法？

> 实际上还可以将上面的代码剥离出来，放到名为TabbarInteractable的protocol的默认实现上。有需要的ViewController只要宣布遵守该协议，然后在viewDidLoad方法中调用一个方法即可实现整个替换过程。

只剩下最后一步了，我们知道 `UITableView` 是 `UIScrollView` 的子类。在它滚动的时候，`FirsViewController` 作为 `UITableView` 的delegate，同样会收到 `scrollViewDidScroll` 方法的调用，所以在这个方法里更新滚动的进度再合适不过了:

```objc
// MARK: UITableViewDelegate
extension FirstViewController: UITableViewDelegate {
    func scrollViewDidScroll(_ scrollView: UIScrollView) {
        //`progress`怎么计算取决于你需求，这里的是为了把`tableview`当前可见区域最底部的2个数字给显示出来。
        let progress = Float((scrollView.contentOffset.y + tableView.bounds.height - tableView.rowHeight) / scrollView.contentSize.height)
        wheelView.progress = progress
    }
}
```

把项目跑起来看看吧，你会得到文章开头的效果。


