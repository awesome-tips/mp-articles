
Note: This tutorial requires Xcode 9 Beta 1 or later, Swift 4 and iOS 11.



机器学习现在可谓风靡一时。许多人都听说过，但很少有人知道它是什么。

这个iOS机器学习教程将向你介绍 Core ML 和 Vision，这是 iOS 11 中引入的两个全新框架。

具体来说，你将学习如何使用新的 API 与 Places205-GoogLeNet 模型，来对图像进行分类。

## 开始

下载 [starter 工程](https://koenig-media.raywenderlich.com/uploads/2017/06/SceneDetector_Starter-1.zip)。工程已经包含了用于显示图像的用户界面，并允许用户从照片库中选择图像。因此，你可以专注于实现应用程序的机器学习和视觉处理功能。

构建并运行工程，你会看到一张夜晚城市的图像，还有一个按钮：

![](https://koenig-media.raywenderlich.com/uploads/2017/06/Screen-Shot-2017-06-09-at-12.43.49-PM.png)

从照片库中选择其他图片。这个初始工程的 Info.plist 已经有一个 **Privacy – Photo Library Usage Description**，因此可能会提示你获取使用权限。

图像和按钮之间有一个标签，可以在其中显示模型的图像场景分类。

## iOS 机器学习

机器学习是一种人工智能，计算机在没有明确编程的情况下进行“学习”。机器学习工具并不是编码一个算法，而是通过在大量数据中查找模式，使计算机能够开发和优化算法。

### 深度学习

自 20 世纪 50 年代以来，人工智能研究人员开发了许多机器学习方法。Apple 的 Core ML 框架支持神经网络、树集合、支持向量机、广义线性模型、特征工程和管道模型。从 2012 年谷歌使用 YouTube 视频来训练 AI 以识别猫和人开始，神经网络已取得了很多的成功。5 年之后，谷歌又发起了一次比赛，来确定 5000 种动植物。Siri 和 Alexa 等应用也归功于神经网络。

神经网络试图用不同方式链接在一起的节点层对人脑处理过程进行建模。每个附加层都需要大幅提高计算能力：例如 Inception v3 这种物体识别模型，具有 48 层和大约 2000 万个参数。但计算都是基本的矩阵乘法，GPU 可以非常高效地处理。GPU 的成本下降使人们能够创建多层深度神经网络，因此称为深度学习。

![](https://koenig-media.raywenderlich.com/uploads/2017/06/neural-network-650x198.png)

神经网络需要大量的训练数据，理想情况下需要涵盖各种可能性。现今用户生成的数据量爆炸性增长也促进了机器学习的发展。

训练模型是指向神经网络提供训练数据，并让它计算公式，这个公式用于组合输入参数以产生输出。训练模型可以是离线的，通常是在具有大量 GPU 的计算机上进行。

要使用该模型，你可以为其提供新输入，并计算输出：这称为**推理**(inferencing)。推理仍需要大量计算，以计算新输入对应的输出。由于有 Metal 这样的框架，现在可以在手持设备上进行这些计算。

正如你将在本教程结尾处看到的那样，深度学习远非完美。构建一组真正具有代表性的训练数据真的很难，而且很容易过度训练，对一些古怪的特征给予过多的权重。

### Apple 提供了什么？

Apple 在 iOS 5 中引入了 NSLinguisticTagger 来分析自然语言。 在 iOS 8 中，引入了 Metal，提供了对设备 GPU 底层访问接口。

2016年，Apple 在其 Accelerate 框架中添加了 BNNS，使开发人员能够构建用于推理的神经网络（而不是训练）。

2017 年，Apple 又推出了 Core ML 和 Vision！

* Core ML 使你可以更轻松地在应用中使用训练的模型。
* Vision：使您可以轻松访问 Apple 的模型，以检测面部、面部标记、文本、矩形、条形码和对象。

您还可以将任何基于图像分析的 Core ML 模型封装在 Vision 模型中，这是这篇教程要实现的。由于这两个框架都是基于 Metal 构建的，因此它们可以在设备上高效运行，你也无需将用户的数据发送到服务器。

## 将 Core ML 模型集成到应用程序中

这篇教程使用 Places205-GoogLeNet 模型，你可以从 Apple 的 [Machine Learning](https://developer.apple.com/machine-learning/) 页面下载该模型。向下滑动到**Working with Models**，然后下载第一个。在这个页面中，还请注意其他三个模型，它们都在图像中检测物体 - 树木，动物，人等等。

> 注意：如果你的训练模型是使用诸如 Caffe, Keras 或 scikit-learn 等机器学习工具创建，则需要将模型转换为 Core ML 支持的模型，可以参考 [Converting Trained Models to Core ML](https://developer.apple.com/documentation/coreml/converting_trained_models_to_core_ml)。

### 在工程中添加模型

下载完 `GoogLeNetPlaces.mlmodel` 后，将其拖到项目的 `Project Navigator -> Resources` 组中：

![](https://koenig-media.raywenderlich.com/uploads/2017/06/add-model.png)

选择此文件，稍等一会。Xcode 生成模型类时会出现一个箭头：

![](https://koenig-media.raywenderlich.com/uploads/2017/06/model.png)

点击箭头查看生成的类：

![](https://koenig-media.raywenderlich.com/uploads/2017/06/generated-model-class.png)

Xcode 生成了输入和输出类，以及主类 `GoogLeNetPlaces`，这个类有一个 `model` 属性和两个 `prediction` 方法。

`GoogLeNetPlacesInput` 有一个 `CVPixelBuffer` 类型的 `sceneImage` 属性。Vision 框架会将我们熟悉的图像格式转换为正确的输入类型。

Vision 框架还会将 `GoogLeNetPlacesOutput` 属性转换为自己的结果类型，并管理对 `prediction` 方法的调用，因此在所有这些生成的代码中，你的代码将只使用 model 属性。

### 在 Vision 模型中封装 Core ML 模型

最后，你可以写一些代码！打开 `ViewController.swift`，在 `import UIKit` 下面导入两个框架：

```objc
import CoreML
import Vision
```

接着，在 IBActions extension 下面添加以下 extension：

```objc
// MARK: - Methods
extension ViewController {

  func detectScene(image: CIImage) {
    answerLabel.text = "detecting scene..."
  
    // Load the ML model through its generated class
    guard let model = try? VNCoreMLModel(for: GoogLeNetPlaces().model) else {
      fatalError("can't load Places ML model")
    }
  }
}
```

以下是你所做的：

首先需要显示一条消息，以便用户知道正在发生的事情。

`GoogLeNetPlaces` 的指定初始化方法会抛出一个异常，因此在创建时必须使用 try 语句。

`VNCoreMLModel` 只是与 Vision 请求一起使用的 Core ML 模型的容器。

标准 Vision 工作流程是：

* 创建模型
* 创建一个或多个请求
* 创建并运行请求处理程序。

刚刚创建了模型，因此下一步是创建请求。

在 `detectScene(image:)` 的末尾添加以下代码：

```objc
// Create a Vision request with completion handler
let request = VNCoreMLRequest(model: model) { [weak self] request, error in
  guard let results = request.results as? [VNClassificationObservation],
    let topResult = results.first else {
      fatalError("unexpected result type from VNCoreMLRequest")
  }

  // Update UI on main queue
  let article = (self?.vowels.contains(topResult.identifier.first!))! ? "an" : "a"
  DispatchQueue.main.async { [weak self] in
    self?.answerLabel.text = "\(Int(topResult.confidence * 100))% it's \(article) \(topResult.identifier)"
  }
}
```

`VNCoreMLRequest` 是一个使用 Core ML 模型完成工作的图像分析请求。它的完成处理程序接收请求和错误对象作为参数。

检查 `request.results` 是否是一个 `VNClassificationObservation` 对象的数组，这是当 Core ML 模型是分类器而不是预测器或图像处理器时 Vision 框架返回的对象。而 `GoogLeNetPlaces` 是一个分类器，因为它只预测一个特征：图像的场景分类。

`VNClassificationObservation` 有两个属性：

* identifier：一个 String
* confidence：一个 [0, 1] 之间的数字，用于描述分类正确的概率

使用基于对象检测的模型时，你可能只想查看 confidence 大于某个阈值的对象，例如 30％。

然后，获取具有最高 confidence 值的第一个结果，并将不定冠词设置为 “a” 或 “an”，具体取决于标识符的首字母。最后，回到主队列以更新标签。在下面将会看到分类工作发生在主队列之外，因为它可能很慢。

现在，进入第三步：创建并运行请求处理程序。

将以下行添加到 detectScene(image:) 的末尾：

```objc
// Run the Core ML GoogLeNetPlaces classifier on global dispatch queue
let handler = VNImageRequestHandler(ciImage: image)
DispatchQueue.global(qos: .userInteractive).async {
  do {
    try handler.perform([request])
  } catch {
    print(error)
  }
}
```

`VNImageRequestHandler` 是标准的 Vision 框架请求处理程序；它不是 Core ML 模型特有的。将 detectScene(image:) 的参数表示的图像传递给它。然后通过调用其 perform 方法，并传递一组请求对象，来运行处理程序。在这个工程中，只有一个请求。

`perform` 方法抛出一个错误，所以你将它包装在 try-catch 中。

### 使用模型对场景进行分类

上面有很多代码，但现在你只需要在两个地方调用 `detectScene(image:)`。

在 `viewDidLoad()` 的末尾和 `imagePickerController(_:didFinishPickingMediaWithInfo:)` 的末尾添加以下代码：

```objc
guard let ciImage = CIImage(image: image) else {
  fatalError("couldn't convert UIImage to CIImage")
}

detectScene(image: ciImage)
```

构建并运行，一段时间后就可以看到分类的结果：

![](https://koenig-media.raywenderlich.com/uploads/2017/06/Screen-Shot-2017-06-09-at-12.47.28-PM.png)

嗯，是的，图像中有摩天大楼，还有一列火车。

点击按钮，然后选择照片库中的第一张图片：一张光照下的树叶的特写：

![](https://koenig-media.raywenderlich.com/uploads/2017/06/Screen-Shot-2017-06-09-at-12.42.16-PM.png)

嗯，也许如果你眯着眼睛，可以想象 Nemo 或 Dory 在游泳吗？但至少你知道“a”和“an”的分类是能工作的。

## 看看 Apple 的 Core ML 示例应用程序

这篇教程的项目类似于 `WWDC 2017 Session 506 Vision Framework` 的示例项目：基于 Core ML 构建。Vision + ML 示例应用程序使用 `MNIST` 分类器，它用于识别手写数字 - 对邮政分拣的自动化非常有用。它还使用原生 Vision 框架的方法 `VNDetectRectanglesRequest`，并包含 Core Image 代码以更正检测到的矩形的透视图。

您还可以从 Core ML 文档页面下载不同的示例工程。 `MarsHabitatPricePredictor` 模型的输入只是数字，因此代码直接使用生成的 `MarsHabitatPricer` 的方法和属性，而不是将模型包装在 Vision 模型中。通过一次更改一个参数，很容易看出模型只是一个线性回归：

```
137 * solarPanels + 653.50 * greenHouses + 5854 * acres
```

## 结论

您可以在[此处](https://koenig-media.raywenderlich.com/uploads/2017/06/SceneDetector_Final-1.zip)下载这篇教程的完整项目。如果模型缺失了，请将其替换为你下载的模型。

现在你已经准备好将现有模型集成到你的应用中。以下是一些更详细的资源：

* [Apple’s Core ML Framework documentation](https://developer.apple.com/documentation/coreml)
* [WWDC 2017 Session 703 Introducing Core ML](https://developer.apple.com/videos/play/wwdc2017/703/)
* [WWDC 2017 Session 710 Core ML in depth](https://developer.apple.com/videos/play/wwdc2017/710/)

从 2016 年起：

* [WWDC 2016 Session 605 What’s New in Metal, Part 2](https://developer.apple.com/videos/play/wwdc2016/605)：demo 展示了应用程序进行 Inception 模型分类计算的速度，这要归功于 Metal。
* [Apple’s Basic Neural Network Subroutines documentation](https://developer.apple.com/documentation/accelerate/bnns)

考虑建立自己的模型？ 我想这超出了本教程（以及我的专业知识）的范围。这些资源可能会帮助你入门：

* [RWDevCon 2017 Session 3 Machine Learning in iOS](https://videos.raywenderlich.com/courses/81-rwdevcon-2017-vault-tutorials/lessons/3)：Alexis Gallagher 做了很多出色的工作，指导我们完成为神经网络收集训练数据（微笑或皱眉的视频）的过程，训练它，然后检查模型是否好用。他的结论是：“你可以建立有用的模型，而不需要成为数学家或在一个大公司。”
* [Quartz article on Apple’s AI research paper](https://qz.com/873043/apples-first-research-paper-tries-to-solve-a-problem-facing-every-company-working-on-ai/)：Dave Gershgorn 关于人工智能的文章非常清晰且内容丰富。本文非常出色地总结了 Apple 的第一篇 AI 研究论文：研究人员使用经过真实图像训练的神经网络来优化合成图像，从而有效地生成大量高质量的新训练数据，不存在个人数据隐私问题。
* [Awesome CoreML Models](https://github.com/likedan/Awesome-CoreML-Models)，是一系列与 Core ML 一起使用的机器学习模型：试用它们，并贡献自己的模型！

最后但并非最不重要的是，我从 Andreessen Horowitz 的 Frank Chen 的 AI 简明历史中学到了很多东西：[AI and Deep Learning a16z podcast](http://a16z.com/2016/06/10/ai-deep-learning-machines/)。大家要以参考一下。

