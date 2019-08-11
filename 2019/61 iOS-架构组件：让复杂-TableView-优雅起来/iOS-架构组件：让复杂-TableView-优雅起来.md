

## 一、传统方式的弊端

`UITableView`是出场率极高的视图组件，开发者通过实现`<UITableViewDataSource>`和`<UITableViewDelegate>`协议方法来配置布局逻辑，面向协议设计模式在苹果的代码设计中很常见，它能适应大部分的业务场景且足够灵活。

`UITableView`相关的协议方法充分体现了单一职责原则，这种方式优点很多，比如某一时刻组件只需要关心当前需要的数据，避免了多余的计算，同时也可以让数据及时释放减小内存峰值。

然而当某一个界面结构比较复杂多元且展示顺序可动态变化的时候，开发者往往需要写大量的`if/else/else if`或`switch`分支语句来区分不同`section/row`的视图类型及其布局，由于`UITableView`相关协议方法的职责单一性，这种分支语句会重复出现在多个协议方法里面。

显然这种场景下，`UITableView`变得不那么优雅。

## 二、常规优化思路

理所当然的，大家很容易想到使用一个中间类来将布局一个 Cell 分散的数据集中起来管理：

```objc
@interface CellLayout : NSObject
@property (nonatomic, strong) Class cellClass;
@property (nonatomic, assign) CGFloat cellHeight;
@property (nonatomic, strong) AnyModel *cellModel;
...
@end
```

然后在`UITableView`各个协议方法里从`NSArray<CellLayout *> layoutArray;`数组中拿到`CellLayout`对象配置就行了，如此，开发者只需要关心如何构建`layoutArray`数组，避免了在协议方法中写过多的分支语句。

这种思路就做了两件事：
- 提供一个包含某个 Cell 所有布局信息的中间类。
- 在中间类确定的情况下，`<UITableViewDataSource>`和`<UITableViewDelegate>`协议方法返回值只需要依据对应中间类的某个属性，简洁清晰。

笔者思考过后，花了些时间做了一个[小组件](https://github.com/indulgeIn/YBHandyTableView)，它能让你轻松的实现这种优化方案，核心操作就是用数组来替代协议方法为`UITableView`配置数据。

当然，这么做有它的局限性，后文再来分析。

## 三、组件架构设计

![1](https://)

经过前面的分析，组件要做的事情有两个，一个是设计一个中间类，一个是封装`<UITableViewDataSource>`和`<UITableViewDelegate>`协议方法的实现。

### 核心思路

按照常规的思路，可能会想到设计一个通用的中间类，就像之前说的`CellLayout`，然后利用继承的特性来为`CellLayout`添加额外的属性。这样确实能达到目的，不过这样带来了较为严重的耦合，需要开发者一开始就知道他必须写一个类来继承自你的`CellLayout`，若本身业务中需要继承另外一个类就很蛋疼了（毕竟 OC 不支持多继承）；再者，若某一天想要剔除这种方案可能会很麻烦，`CellLayout`设计得越臃肿、包含的业务越多将越难剥离。

并且，一个`CellLayout`是解决不了问题的，因为配置`UITableView`可能需要`UITableViewCell`的一些数据，也需要一些通用的方法来告知`UITableViewCell`何时配置数据刷新UI，也就意味着按照这种逻辑，还需要写一个`BaseTableViewCell `......

显然，这种方式并不优雅，也违背了依赖倒置原则。

笔者的做法是将这个“中间类”抽象出来，作为两个协议：`YBHTCellProtocol`和`YBHTCellConfigProtocol`，这两个协议包含了布局`UITableView`所需的数据，当然可以结合自己的业务扩充这两个协议。`YBHTCellProtocol`由自定义的`UITableViewCell`来实现；`YBHTCellConfigProtocol`随意开发者用什么类来实现，通常情况下，使用包含`UITableViewCell`所需数据的`Model`来实现是最快捷的做法（可看Demo中的使用案例）。


### 保证深度定制性

考虑到一个问题，`UITableView`相关协议方法非常多，若为`YBHTCellProtocol`和`YBHTCellConfigProtocol`拓展所有的配置将会需要大量的代码，可能有些得不偿失。

所以笔者使用多代理 (`YBHandyTableViewProxy`) 来保证组件使用方深度定制的需求，也是为了避免某些特殊情况下，使用该组件的业务模块能快速的拓展之前没有的功能：

```objc
- (void)ybht_addDelegate:(id<UITableViewDelegate>)delegate;
- (void)ybht_addDataSource:(id<UITableViewDataSource>)dataSource;
```

当然这样做会有隐患，所以建议读者朋友若想使用该组件先了解它的原理，该组件的代码不多也不高深，相信只要感兴趣的朋友能很快理解。

## 四、组件的弊端

组件的配置方式很简单：

```objc
NSArray<id<YBHTCellConfigProtocol>> tmpArr = ...;
[anyTableView.ybht_rowArray addObjectsFromArray:tmpArr];
[anyTableView reloadData];
```
正如代码所见，需要传入的是实现`YBHTCellConfigProtocol `协议的实例，同时需要对应的`UITableViewCell `实现`YBHTCellProtocol `协议（可对比 UML 类图）。

取个例子，若你在`UIViewController`里面写了一个`UITableView`，然后使用该组件配置数据，可以明确的是组件将`<UITableViewDataSource>`和`<UITableViewDelegate>`协议封装起来，`UIViewController`和你定制的那些`UITableViewCell `已经没有了耦合，也就意味着，它们之间的交互将不能直接进行。

那么，它们如何间接的交互呢？

1. `YBHandyTableViewIMP`是组件实现`<UITableViewDataSource>`和`<UITableViewDelegate>`协议的类，那么将`UIViewController`对象传入到该类就能实现与`UITableViewCell `的交互，但是由于`YBHandyTableViewIMP`和`UITableViewCell `不直接依赖而是都依赖于`YBHTCellProtocol `协议，这为定制性的交互带来了困难。
2. 从另一个方面思考问题，从组件的使用方法可知，`UIViewController`和`id<YBHTCellConfigProtocol>`之间是有关联的，而`id<YBHTCellConfigProtocol>`与`UITableViewCell<YBHTCellProtocol>`是有关联的，所以可以通过`id<YBHTCellConfigProtocol>`将`UIViewController`传递到`UITableViewCell`中，然后进行交互。
3. 基于响应链的传递路径来拦截事件。这种方式比较巧，但是却始终感觉不是那么稳妥，它的好处是处理`UITableViewCell`的交互事件完全可以不经过该组件就能完成。

最后，笔者建议使用第二种方式。

不过不管哪种方式来说都不太优雅了，在业务开发中应该多考虑一下，`UITableViewCell`中会不会有大量的事件需要传递到最外层的业务，比如跳转界面、网络请求等就可以直接在`UITableViewCell`里面调用。若大量的交互是必然的（或者说是为了满足业务架构规范），那就放弃“偷懒”，专门设计一个适合业务的方式吧。

## 五、结语

本文是笔者做的一个小实践的思路分享，需要明白的是，一个代码设计并非能满足所有的业务，特别是这种和具体业务紧密相连的组件。在一开始笔者还满怀希望，觉得这个组件的场景很大，后来发现还是有局限性。

组件总是会让粒度变大，当你追求更小粒度的时候你会发现：我去，好像这个组件没有了意义😂。

设计有取舍，没有万能的方案。


**GitHub 地址：[YBHandyTableView]( https://github.com/indulgeIn/YBHandyTableView)**

