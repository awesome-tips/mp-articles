

**一句代码自动生成 Model 文件，拖入工程既能使用。**


## 前言

当一个网络数据比较复杂时，往往需要一些功夫来创建对应的数据模型，笔者正是苦于手动创建 Model 痛苦，决定做一个工具来自动创建 Model 文件。

为了降低工具开发成本，直接基于 iOS 系统库来做。如果是做 Mac 上的工具，会存在一些技术问题，比如不便于使用 iOS 程序的动态链接库，处理 iOS 中的一些类型时会比较乏力，并且工具不知道目标工程的信息，在判断类名重复、读取工程信息等情况时会很不方便。

本文讲解 **YBModelFile** 的设计思路和技术细节。

## 一、示例

为了便于理解，先放上一个 json：

```json
{
	"name":"jack", 
	"address":{"city":"北京", "location":"x,x"},
	"orderList":[{"id":1, "goods":"手机"}, {"id":2, "goods":"电脑"}]
}
```

可以构建为如下的一些类：

```objc
@interface PersonAddressModel : NSObject
@property (nonatomic, copy) NSString *city;
@property (nonatomic, copy) NSString *location;
@end

@interface PersonOrderListModel : NSObject
@property (nonatomic, copy) NSString *goods;
@property (nonatomic, assign) NSInteger *id;
@end

@interface PersonModel : NSObject
@property (nonatomic, copy) NSString *name;
@property (nonatomic, strong) PersonAddressModel *address;
@property (nonatomic, copy) NSArray<PersonOrderListModel *> *orderList;
@end
```

下面就来讲述，工具如何来自动构建这些东西（当然还包括.m文件的一些实现）。


## 二、构建多叉树

工具需要通过 json 数据构建一些自定义的类，那么如示例所示，构建一个类必须要知道它的类名和所有属性，而一个类的属性可能是另一个类，也可能是一个包裹类的数组...

很容易想到分治法，将数据局部化处理，于是可以将它们构建为一个树形结构：

![1](https://upload-images.jianshu.io/upload_images/2909132-dce7ad2e4511d960.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在这个树形结构中，`Custom Class`即是工具需要构建的类，而诸如`NSString, NSNumber, NSInteger`等本身就存在无需构建，值得注意的是`NSArray`类型的属性里面会包含一个类，这个类可能是系统类也可能是自定义类。

可以明确的是，构建类时子节点需要作为父节点的属性，那么`Custom Class`是有子节点的，而系统类型`NSString`等是不需要子节点的。

由此，通过遍历 json 转换的字典就能构建出这样一个树形结构。工具中如下所示表示一个节点：

```objc
@interface YBMFNode : NSObject
/** 节点类型 */
@property (nonatomic, assign) YBMFNodeType type;
/** 子节点 */
@property (nonatomic, strong) NSMutableDictionary<NSString *, YBMFNode *> *children;
...
@end
```

子节点通过一个字典来存储，`key`表示对应节点在 json 中的字段名，构建类时要作为属性名。

## 三、类名和属性名的处理

在构建树的过程中，同时需要处理类名和属性名。从上面的示例可以看出，属性名直接就可以取 json 中的 key；类名可以通过父节点的类名加上 key (比如`PersonModel + address = PersonAddressModel`) 。

看起来问题是处理了，实际上还需要做一系列的判断保证类名和属性名的合法性。

### 类名处理：

- 过滤掉 key 中非法字符。
- 将 key 蛇形命名转换为驼峰命名。
- 父节点类名 + key + 后缀 = 当前类名
- 类名判重：一是工程目录和系统库中已经存在的类；二是一次程序生命周期将要创建的类。什么叫将要创建的类？其实就是在一次程序运行中，通过工具要创建的新类，由于在本次程序运行中这些新类还未加入工程，所以无法通过代码获取。笔者通过申请一个静态的 hash 容器把将要创建的新类放进去，就能轻易的判断重复。
- 类名重复处理：当知道类名重复时，处理方案就很多了，笔者是在类名末尾加上数字，循环累加这个数字直到不重名为止。
- 为什么不做类复用：首先个人对规范的理解，数据模型类最好不好复用；其次从技术上说，由于自动创建的类名每一次都不可预估，判断类是否可以复用只有通过遍历所有的属性来比较，而已知的类不好规定可复用的范围，对于时间和空间复杂度来说也是不小的挑战，所以笔者认为做类复用没有什么太大意义。
- 为什么不过滤保留字：通常情况来说，工具需要使用者传入一个主 Model 的名字，这个名字通常是大写字母开头，之后的类会拓展类名，并且还要拼接后缀，所以理论上直接规避了和保留字的冲突 (大写的保留字比如 YES 和 NO)。
- **TODO :** json 嵌套过深，类名过长问题：考虑两种处理方法，一是限制类名长度，二是使用与 key 无关的类名拓展策略，不过显而易见每种方式都有缺陷。

### 属性名处理：

- 过滤掉 key 中非法字符。
- 如果和保留字重名，全大写。
- 如果前缀有特殊字符 (比如`init、new`)，把特殊字符部分大写。
- 如果前缀包含数字，在前面加上"_"。
- 属性名判重：情况一一个类中有两个相同的属性，这种情况可能有人说 json 对象也不会有重名字段吧，那是因为笔者前面对属性名做了处理可能会出现重名 (比如`order>list`和`order?list`处理后都是`order_list`)；情况二是与父类的属性重名，所以笔者遍历了父类的所有实例变量，同时加上了`NSObject`协议的属性 (若数据模型的基类不是`NSObject`情况可能会出现协议名未包含的情况)。
- 属性名重复处理：同类名重复处理一样。

## 四、算法逻辑分离

由于需要实现动态的类文件分布，当两个类放在一起和两个类分开，它们所涉及的代码是不一样的 (比如两个类并在一起只需要一个文件头注解、不需要进行另一个头文件的导入)。

所以工具将`.h`和`.m`中的代码分块处理，比如文件顶部注解、导入文件依赖、实际业务代码等划分为不同的处理单元。基于多叉树的模型，可以灵活的通过深搜或广搜等来进行动态的代码插入，实现灵活控制，为已有功能或者将来要做的功能提供一个有力的数据结构支撑。

同时，为了拓展性和定制性，笔者创建数个协议并提供默认实现，使用者可以进行灵活的局部算法替换：

```objc
/** 名字处理器 */
@property (nonatomic, strong) id<YBMFNameHandler> nameHander;
/** 文件头部注解处理器 */
@property (nonatomic, strong) id<YBMFFileNoteHandler> fileNoteHander;
/** .h文件代码处理器 */
@property (nonatomic, strong) id<YBMFFileHHandler> fileHHandler;
/** .m文件代码处理器 */
@property (nonatomic, strong) id<YBMFFileMHandler> fileMHandler;
/** 节点作为父节点的属性时 Code 格式处理器 */
@property (nonatomic, strong) id<YBMFCodeForParentHandler> codeForParentHandler;
```

## 五、类拆分策略

有了上面提到的东西，构建`.h`或`.m`文件的代码字符串就是一个轻而易举的事情。

### 类分离为多个文件

实现一个类对应一组`.h/.m`文件策略，直接通过一个深度优先搜索，在过程中组装文件代码并且创建文件，不过处理逻辑是后序的，也就是说树的层级越深越先创建，这样是为了一个类依赖的类总是先于这个类创建，当中途出现异常时，已经创建的文件能有效运行。

### 类集中在一个文件

很多时候我们希望一个 json 下的数据模型类放到一个文件中，得益于算法逻辑模块分离，可以很轻松的使用深度优先搜索来动态构建需要的代码，组装为合理的结构。这种情况笔者仍然采用后序处理，目的是为了让一个类依赖的类总是处于它的上方，这样在一个文件中就不需要使用`@class AnyClass;`来声明了。

### TODO : 类分离粒度控制

考虑在复杂场景下，可能需要按需拆分文件，比如 100 个类需要划分为 10 组`.h/.m`文件。目前能想到的是三种方式：

- 对多叉树按照层级划分文件，在一层的类划分到一个文件，这种处理方式的缺点是一个文件的所有类是兄弟节点没有什么逻辑关联，不便于管理。
- 通过设置一个最大层级来控制，比如设置的层级是 3，那么第 3 层之后的子节点类都合并到第 3 层的类文件中。
- 三是在深搜过程中记录文件中的类数量，一个文件达到数量限制就创建新的文件来写入类。

## 后语

该工具是笔者为开发效率所做的努力，通常情况下能为大家节约不少时间，希望能对大家有所帮助。

在发现需求、设计方案、遇到问题、解决问题的过程中，笔者似乎感受到了技术之外的东西。对于一个优秀的工程师来说，能随时高效全面解决问题的能力比技术本身重要，希望大家共勉，有朝一日能用优秀二字来标榜自己。

✨✨😁✨✨


GitHub 地址：[YBModelFile](https://github.com/indulgeIn/YBModelFile)

