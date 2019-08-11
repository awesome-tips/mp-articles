# 关于 Xcode 10 New Build System，你需要了解的 5 件事

作者 | Shashikant Jagtap
翻译 | 高飞，目前在京东 iOS 组件组，主要做客户端组件相关的工作，业余时间捣鼓新东西，打打球，玩玩游戏。

Apple 在 Xcode 9 中以预览模式发布了 **Xcode New Build System**<sup>[1]</sup>（以下统一使用 New Build System），不过不是默认开启的。而到了 Xcode 10，则默认开启了该选项，在现有工程中使用 New Build System 可能会有一些问题。Apple 也意识到了这些问题，并为 New Build System 创建了一份单独的发布说明。他们还为其中的一些问题给出了可行的解决方法。您可以在*这里*<sup>[1]</sup>阅读 New Build System 的完整发布说明。我们在之前的*博客*<sup>[2]</sup>中详细说明了了 New Build System 的内部细节，在这里我们将说明开发者可能遇到的，而 Apple 在发布说明中没有详细介绍的 5 个问题，例如第三方工具在 New Build System 中如何运作。

## Xcode 10: New Build System

先来简单回顾一下 New Build System，从 Xcode 10 开始，可以从 **<u>Xcode-> Files-> Project/Workspace Settings</u>** 激活新的构建设置，我们可以在 Legacy Build System 和 New Build System 之间切换。

![1](http://)

查看之前关于 Xcode new build system 的*博客*<sup>[2]</sup> 以了解更详细的信息。如果你用 `xcodebuild` 在命令行里编译 iOS 工程，则需要传入额外的参数 `-UseModernBuildSystem=YES` 来强制使用 New Build System。New Build System 对应的二进制文件是 `xcbuild`，路径如下：

```
/Applications/Xcode.app/Contents/SharedFrameworks/XCBuild.framework/Versions/A/Support/xcbuild
```

New Build System 并行运行目标和构建阶段以加速整个 Swift 编译。一旦在 Xcode 中激活，我们将会从 New Build System 中获益，同时也会遇到相关问题。我们将介绍 Xcode New Build System 带来的常见问题和目前解决这些问题的方法。我们将根据 iOS 应用中受影响的范围来对这些问题进行分类。

## 1] info.plist

当一个 iOS 工程使用 New Build System 编译时，一些与 info.plist 文件相关的问题开始出现。我们需要了解一些基于 New Build System 和 Info.plist 文件的规则。

- 任何 target 的 Copy Bundle Resources 编译阶段不应该有任何 plist 文件。否则 New Build System 不会编译 app。另外，如果 bundle 里的文件被复制多次，编译也无法通过。
- New Build System 在 clean 后和增量编译中运行 info.plist 具有不同的优先级。clean 后编译，info.plist 在处理资源文件之后，链接 storyboard 之前，而增量编译时则位于签名之前。
- 如果 target 仅有 Info.plist，而没有任何 Xcode 的引用文件目录，Xcode 编译系统编译会失败。

## 2] CocoaPods

使用 CocoaPods 的 iOS 工程会有一些问题。一些常见问题是：

- 除非我们执行 clean build，否则 Pod 不会更新。内嵌的 pods frameworks 构建阶段无法稳定运行。Github 有一个相关 *issue*<sup>[3]</sup>
- 由于 Cocoapods 相关的一些编译脚本可能无法稳定运行，所以打包 app 可能会失败，或者 app 可能会有一些奇怪的行为。

简言之，目前 Cocoapods 和 New Build System 不能很好的合作。

## 3] Run Script Phase

使用 New Build System，在 Run Script 阶段可能会失败或出现一些奇怪的行为。

在 Xcode 10 中，run script 编译阶段已经改进了很多，然而，我们需要给 run script 阶段指定一些输入文件来辅助构建进程。在 run script 阶段指定输入文件，对编译系统做出正确决定是重要的，就像对于依赖的目标编译，run scripts 是否需要执行。Xcode build system 尝试并行执行一些任务，而如果 run script 的输入没有正确生成，那么 build system 会不知所措甚至失败。在适当的地方给 run scripts 提供输入文件始终是个好主意。随着输入文件的增多，Xcode 10 提供了 .xcfilelist 文件让我们指定所有的输入文件，然后将文件添加到 build phase 的 File List 输入中。Xcode 编译系统始终会在没有输入文件，改变输入文件和丢失输出文件的时候运行 build phase。添加这些文件是重要的，避免在不必要的时候为所有增量编译系统运行这个 phase。

## 4] Clean Build Folder Action

使用 New Build System，Xcode 的 clean 操作被废弃了，取而代之的是使用 `Clean Build Folder` 操作。新的操作移除所有 iOS 应用的派生数据（derived data），并且从零开始一次干净的编译。这意味着如果你使用 Cocoapods，会从零开始重新编译所有的 frameworks，在编译 iOS 工程时造成巨大的延时。如果你使用 Carthage 预编译 frameworks，就不会有那么大的影响。如果你有 clean 编译的习惯，需要特别关注这个操作。同时你也会面临缓慢的 Xcode 索引问题。

## 5] XCCONFIG Files

开发人员大多会使用 `.xcconfig` 文件在某处为特殊的 targets 保存 Xcode 构建设置。这里有一些问题，在 xcconfig 文件里设置的一些条件变量可能不会按预期生效，从而导致编译失败。为了检查你的 xcconfig 文件，Apple 推荐运行下面的命令：

```sh
defaults write com.apple.dt.XCBuild EnableCompatibilityWarningsForXCBuildTransition -bool YES
```

如果这个命令报任何警告和错误，则需要修复它，以获得稳定的编译。

## 结论

New Build System 主要是为了提高 Swift 编译的性能，稳定性和可靠性。它会在应用开发早期阶段捕获配置错误。Xcode 10 已经默认开启，不久之后我们就不得不更新我们的编译过程以适应 New Build System。可以肯定的是，它会为 app 中带来很多提升。你已经开始使用 New Build System 了吗？你有什么经验吗？你遇到过文中未提及的其他问题吗？

### 参考链接

1. https://developer.apple.com/documentation/xcode_release_notes/xcode_10_release_notes/build_system_release_notes_for_xcode_10
2. https://medium.com/xcblog/xcode-new-build-system-for-speedy-swift-builds-c39ea6596e17
3. https://github.com/CocoaPods/CocoaPods/issues/8073

[Five Things You Must Know About Xcode 10 New Build System](https://medium.com/xcblog/five-things-you-must-know-about-xcode-10-new-build-system-41676cd5fd6c)





