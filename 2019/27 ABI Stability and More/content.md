# ABI Stability and More

在 macOS、iOS、watchOS 和 tvOS 上稳定 Swift 的 `ABI` 一直是一个长期目标。虽然稳定的 ABI 是任何语言成熟的重要里程碑，但 Swift 生态系统的最终收益是为应用程序和库提供二进制兼容性。这篇文章描述了 **Swift 5 中的二进制兼容性意味着什么**以及**它将如何在未来的 Swift 版本中发展**。

您可能会问：其他平台怎么样？ABI 稳定性是针对它编译和运行的每个操作系统实现的。Swift 的 ABI 目前在 Apple 平台上已经是稳定了。随着 Swift 在 Linux、Windows 和其他平台上的开发日趋成熟，Swift 核心团队将评估在这些平台上稳定 ABI 的问题。

Swift 5 为应用程序提供二进制兼容性：保证在将来，使用一个版本的 Swift 编译器构建的应用程序将能够与使用其他 Swift 版本构建的库进行通信。即使使用兼容模式与旧语言版本（-swift-version 4.2）通信，这也一样适用。

![](https://swift.org/assets/images/abi-stability-blog/abi-stability.png)

在此示例中，使用 Swift 5.0 构建的应用程序将在安装了 Swift 5 标准库的系统上运行，同样也可以在后续的安装了 Swift 5.1 或 Swift 6 的系统上运行。

Apple OS 的 ABI 稳定性意味着部署到即将发布的这些操作系统的应用程序将不再需要在应用程序包中嵌入 Swift 标准库和“覆盖”库，从而减小其下载大小；Swift 运行时和标准库将随 OS 一起提供，就像 Objective-C 运行时一样。

有关这如何影响提交到 App Store 的应用程序的更多信息，请参阅 `Xcode 10.2 release notes`。

## 模块稳定性

ABI 稳定性更多的是关于在运行时混合不同版本的 Swift。那编译期呢？现在，Swift 使用一种名为 “`swiftmodule`” 的不透明归档格式来描述库的接口，例如框架 “MagicKit”，而不是手动编写的头文件。但是，“swiftmodule” 格式与编译器的当前版本相关联，这意味着如果 MagicKit 是使用不同版本的 Swift 构建的，则应用程序开发人员无法 `import MagicKit`。也就是说，app 开发人员和库作者必须使用相同版本的编译器。

要移除此限制，库作者需要一个当前正在实现的功能，即 **模块稳定性**（module stability）。这涉及使用模块的文本摘要来增强不透明格式，类似于您在 Xcode 的 “`Generated Interface`” 视图中看到的，这样客户端可以使用模块而无需关心它构建的编译器。可以在 Swift 论坛上阅读更多相关信息。

![](https://swift.org/assets/images/abi-stability-blog/module-stability.png)

作为一个例子，你可以使用 Swift 6 构建一个 framework，Swift 6 和未来的 Swift 7 编译器都可以读取该 framework 的接口。

> 注意，这里的所有 Swift 版本号都是假设的。

## Library Evolution

到目前为止，我们一直在讨论更改编译器，但保持 Swift 代码不变。应用程序正在使用的库的修改又如何呢？目前，当一个 Swift 库发生变化时，任何使用该库的应用程序都必须重新编译。这有一些优点：因为编译器知道应用程序正在使用的库的确切版本，从而可以做出额外的假设，减少代码大小并使应用程序运行得更快。但是对于下一版本的库，这些假设可能就不适用了。

这个特性就是**library evolution 支持**：使用新版本的库而无需重新编译客户端。当 Apple 在操作系统中更新库时会发生这种情况，不过当一家公司的二进制框架依赖于另一家公司的二进制框架时，这也很重要。在这种情况下，更新第二个框架理想情况下不需要重新编译第一个框架。

![](https://swift.org/assets/images/abi-stability-blog/library-evolution.png)

在此示例中，应用程序是根据框架的原始版本构建的，为黄色。由于支持 library evolution，它可以在具有黄色版本的系统上运行，也可以在新的、改进的红色版本上运行。

Swift 已经实现了对 library evolution 的支持，非正式地称之为“`resilience`”。这对于需要的库来说是个可选功能，它使用尚未最终确定的注释来在性能和未来的灵活性之间取得平衡，您可以在标准库的源代码中看到。第一个通过 Swift Evolution Process 的是@inlinable，在 Swift 4.2（SE-0193）中添加。

## 总结

| 当Swift有... | ...然后可以修改... | 状态 |
| --- | --- | --- |
| ABI 稳定 | Swift 标准库 | 在 macOS、iOS、watchOS 和 tvOS 的 Swift 中 |
| 模块稳定性 | 编译器| 积极发展 |
| Library Evolution 支持 | 库的API | 大部分已实施，但需要经历Swift演进过程 |


