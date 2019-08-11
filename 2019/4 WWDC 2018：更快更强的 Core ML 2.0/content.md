
> 本文是 WWDC 2018 Session 708 和 Session 709 的读后感，其视频及配套 PDF 文稿链接如下：What is New in Core ML, Part 1, What is New in Core ML, Part 2 本文会首先回顾 Core ML 的基本背景知识，其后着重介绍 Core ML 在应用上和工具上的更新。

## Core ML 回顾

Core ML 是苹果在 2017 年推出的机器学习框架。主要支持图像分类和文本信息处理两大功能。其基本流程是获取模型、导入并生成接口、使用接口编程3个步骤。我们来详细分析每一步：

1. 获取模型。2017年时主要有两种方式：从官网下载和转化第三方框架生成的模型为 Core ML 模型。为了方便转化，苹果推出了 Python 编写的 Core ML Tools。2018年又推出了原生的 Create ML 框架来直接从数据生成 Core ML 模型。Core ML Tools 的优点在于其语法简单直接，缺点在于支持的第三方框架少、生成的模型尺寸过大且不能定制化。
2. 导入并生成接口。这里直接拖拽模型进入 Xcode，Xcode 会自动生成相应的机器学习模型接口。无需任何手动或其他操作，十分方便友好。美中不足的是生成的接口是固定的、无法增加定制化接口。
3. 使用编程接口。根据生成的 API 进行编程。2017年 Core ML 模型只支持对单一对象进行预测，无法批量预测，运行效率比较低下。

可以说 2017 年推出的 Core ML 框架十分易用，但其功能也十分简陋。开发者们只能在一开始模型生成上做定制化操作，且很有可能要依赖第三方框架。之后只能使用 Core ML 生成的固定模型进行编程，十分局限：无法优化预测效率、无法精简尺寸、无法增加新的层级、无法自定义模型。

针对这些缺陷，苹果在今年的 Core ML 2.0 上做出了如下改进——更小、更快、高度定制化。

![1](http://)

## 新功能

Core ML 这次的新功能主要在于模型的接口生成新增了一个可以批量预测的 API。下面代码展示了原来的 API 和 新的 API：

```objc
// 预测单一输入
public func prediction(from: MLFeatureProvider,
options: MLPredictionOptions) throws -> MLFeatureProvider

// 预测多个输入
public func predictions(from: MLBatchProvider,
options: MLPredictionOptions) throws -> MLBatchProvider
```

以前需要用 for 循环完成的操作现在可以用一个方法完成。不进如此，新的批量预测方法相对于 for 循环嵌套单一预测的实现，还用了 batch 进行优化。

原来的 for 循环单一预测方法需要完整地读入每一个数据，将其预处理后发送给 GPU，GPU 计算完毕后再把结果从 GPU 中取出并在程序中输出结果。新的批量预测方法则是消除了预处理和取出的操作，将所有数据一次性发给 GPU，利用 GPU Pipeline 将其逐个计算的同时依次取出结果。另外因为 GPU 一直在运算状态，GPU 可以对计算进行统一优化，使得相似数据的处理越来越快。这样整体的性能就要快上不少，具体逻辑如下图所示：

![2](http://)

苹果当场展示了两种方法之间的效率之差：处理40张图片，新的批量预测方法比 for 循环的单一预测方法比要快将近5秒钟，效率上几乎提高了一倍。

除此之外，Core ML Tools 增加了第三方机器学习框架数量，从原来的6个增加到了11个，包括了最知名的 TensorFlow、IBM Watson、MXNet，数量和质量都有大幅提升。

![3](http://)

## 性能优化

性能优化是 Core ML 的重头戏，苹果宣称 Core ML 2 的速度提高了 30%。我们来看看苹果做了哪些工作：

* 量化权重。Core ML 的模型可以根据需求对权重进行量化。权重越低，模型尺寸越小，运转速度越快，占用内存资源也就越少，但是处理效果也就越差。
* 多尺寸支持。针对图片处理，Core ML 现在只需一个模型，就能处理不同分辨率的图片。相对于之前单一分辨率图片的模型，该模型更加灵活，且因为在底层大量共享代码，使得模型的体积也远远小于原来多个单独模型体积之和。

我们来重点看下量化权重。2017年时 Core ML 的所有模型权重都是32位，也就是说每个模型可以识别 2^32 个不同的特征值。这虽然带来了非常之高的准确度，但同时也使得 Core ML 模型非常之大（20+M）。对于 App 开发来说，尺寸大小是非常值得注意的因素。借鉴 App Thinning 的思路，苹果针对 Core ML 的模型大小进行了优化。现在开发者可以使用 Core ML Tools 对原来32位权重的模型进行量化，根据需要，苹果支持16位、8位、4位等权重。权重越低。模型尺寸越小，运转速度越快，但是处理效果也就越差。所以还是要根据实际需求进行选择，下图中我们可以看到不同模型尺寸和处理效果的对比。

![4](http://)

在权重量化上我们可以针对需求做出最小体积的模型；同时针对多尺寸图片我们可以合并多个类似功能的模型；同时 Core ML Tools 又提供了自由定制权重的 API。在多重措施的优化之下，Core ML 的模型可以最大限度的缩小尺寸，从而带来更合适的加载和运算效率。

## 定制化

苹果在定制化方面推出了两种方案：定制化神经网络层和定制化模型。我们先来看定制化神经网络层。

很多 Core ML 模型的内部实现是多层神经网络，每一层接受上一层的输入，然后做出相应处理再将结果输出给下一层。例如，识别照片中动物是马的过程如下图所示：

![5](http://)

这个神经网络每一层都是固定的、由 Core ML 框架自动生成并优化，我们不能做任何操作。这使得模型的功能大大受到局限：试想我们如果要基于上述模型生成一个新模型，使得该模型不仅能识别马，还能识别鸟、鱼、猫、狗等各种动物，最简单的做法就是将上述模型中判别动物是马的层级给替换掉。Core ML 2 现在提供了这种功能，具体操作步骤是：

1) 获取生成含有特定的层级的模型。一般获取方法是依靠第三方神经网络库，比如 Keras。

2) 用 Core ML Tools 将含有特定层级的模型转化成对应的 Core ML 模型。这其中我们要自定义特殊层转化方法。具体代码如下：

```objc
# 用 keras 神经网络库生成模型，其中特殊层为 GridSampler
model = keras.model.load_model('spatial_transformer_MNIST.h5', custom_object: {'GridSampler': GridSampler})

# 自定义 Core ML 模型中对应特殊层 GridSampler 的转化方法
def convert_grid_sampler(keras_layer):

  params = NerualNetwork_pb2.customLayerParams()

  # 定义名称和描述
  params.className = 'AAPLGridSampler'
  params.description = 'Custom grid sampler layer for the spatial transformer network'

  # 定义层级参数，这里我们只要处理图片的长和宽
  params.parameters["output_height"].intValue = keras_layer.output_size[0]
  params.parameters["output_width"].intValue = keras_layer.output_size[1]

  return params

# 用 Core ML Tools 将 Keras 模型转化，其中特定层 GridSampler 的转化方法定义为 convert_grid_sampler
coreml_model = coremltools.converters.keras.convert(model, custom_conversion_functions = {'GridSampler': convert_grid_sampler})
```

3) 将 Core ML 模型导入 Xcode 中，自定义特殊层的接口。其对应类务必实现 MLCustomLayer 协议，它是自定义神经网络层的行为协议，每个方法的具体解释可以参照苹果的官方文档：`MLCustomLayer` https://developer.apple.com/documentation/coreml/mlcustomlayer。

```objc
public protocol MLCustomLayer {
  public init(parameters: [String : Any]) throws

  public func setWeightData(_ weights: [Data]) throws

  public func outputShapes(forInputShapes: [[NSNumber]]) throws -> [[NSNumber]]

  public func evaluate(inputs: [MLMultiArray], outputs: [MLMultiArray]) throws
 }
```

同时，上文中提到的GridSampler的具体实现如下图：

![6](http://)

当然并不是所有模型的实现都是神经网络。所以苹果还推出了定制化模型。实现一个定制化模型的方法十分简单，就是实现 MLCustomModel协议：

```obj
public protocol MLCustomModel {
  public init(modelDescription: MLModelDescription, parameters: [String : Any]) throws
  
  public func prediction(from: MLFeatureProvider,
options: MLPredictionOptions) throws -> MLFeatureProvider

 
  optional public func predictions(from: MLBatchProvider,
options: MLPredictionOptions) throws -> MLBatchProvider
}
```

其具体说明亦可以参考 苹果官方文档 https://developer.apple.com/documentation/coreml/mlcustommodel。

## 总结

Core ML 2 在 2017 年的基础上增加了新功能，同时对模型大小和运转效率进行了相应优化。其配套工具 Core ML Tools 也增加了可支持的机器学习框架，同时开发者可以借助工具自定义神经网络层和 Core ML 模型。除此之外，苹果推出的 Create ML 也极大地解决了模型获取方面的局限。目前 Core ML 已经深度应用于 Siri、Photos、QuickType 等原生应用上，采用 Core ML 的第三方应用也多达 182 个，相信在不久的将来，Core ML 会成为所有主流 App 的标配。

