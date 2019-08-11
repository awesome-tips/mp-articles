| 作者：Ryan Edge
| https://medium.com/flutter-community/wishlist-for-flutter-in-2019-65009e2d1ca8

我之前去了 `DartConf 2018`，最初并没有对 Flutter 的有过多的期待，只是想去了解一下。但在离开时，我非常期望看到它在移动平台之外能蓬勃发展并更加成熟。看看 React Native 在过去几年里的发展，我坚信 Flutter 可以达到同样的成果，并坚信 Dart 成为最好的客户端语言的梦想将会实现。

今年，我的热情更是大大提升。我清楚地了解了 Flutter 计划如何从移动平台迁移到桌面和 Web 端，而我的期望也是如此。这里列一下我希望 2019 年 Flutter 生态系统和框架的在哪些方面有所改进。

## 1. RFC (Request For Comments) 流程

从技术角度或开发人员的角度来看，这可能不是最重要的特性。但是我以 RFC 开篇，是因为我认为它为设计和采用此列表中的一些其他项目提供了明确的途径。

如果你不熟悉 RFC 的流程，那么可以参考一下 React 的博客（见参考 1），那上面有一篇很好的文章，描述了这个流程以及为什么 React 会采用它。以下是基本步骤：

* 创建一个 RFC 文档，详细说明提案；
* 将 PR 提交到 RFC 仓库；
* 将反馈纳入提案；
* 经过讨论，核心团队可能会也可能不会接受 RFC；
* 如果 RFC 被接受，则会合并 PR；
* 合并后，任何贡献者都可以提交 PR 实现以供审核。

React 使用该流程的特定功能突出显示了其主要用例：

* 创建新的 API 服务区域的提案；
* 删除已发布功能的提案；
* 不是 BUG 修复的语义或语法更改

在某些情况下，例如主要的 API 更改或受益于深入审核的新约定，简单的 `GitHub issue` 并不是最佳的协作方式。除此之外，对比一下将 GitHub issues 页面的正常模板与 Ember 的非常简单的 RFC 站点，可以看到每个 RFC 都有一个非常深入的总结，动机以及带有示例的详细设计。

目前有几个值得注目的开源项目正在使用 RFC 流程，并取得了巨大成功：

* Rust
* Ember
* Yarn
* React & React Native
* npm

值得注意的是每个流程之间的差异很小。如果您熟悉一个流程，那么其它的基本上都不是问题。它们以公开和可见的方式引入一致性。虽然这些项目中的每一个都有自己的方法来管理 issue，但 RFC 的流程很容易识别，因为它更关注目标。

有趣的是，在 `Influencing the Flutter SDK (The Boring Flutter Development Show, Ep. 13`(参考2) 视频播出之前，我一直在思考这个问题。值得注意的是，此提案旨在扩展开源项目已经遵循的流程，而不是替换它们。 GitHub issues 仍然有助于标识 BUG 和工作项，如同视频中描述的。此列表中的以下所有三个项目都是 RFC 的理想候选者。

## 2. 更简单的继承 Widget/Context 模式

我必须阅读相当多的文章和示例来全面了解 `InheritedWidget` 在 Flutter 中的工作原理。这个概念本身对使用过 React 的我来说并不陌生，它只是不那么直接和有点冗长。我们来看一个我能找到的最简单的例子。

![1](https://cdn-images-1.medium.com/max/1600/1*-glemHdoH29lSNTkiI-bYQ.png)

我相信在其它地方可能会有更简单的例子，但我读过的大部分文章都与我们上面的差不多，所以让我们继续吧。现在让我们将它与 React 中的类似实现进行比较。

![](https://cdn-images-1.medium.com/max/1600/1*IfNy8pknWrS_ZtjISlSEjg.png)

这两个示例基本上做的是同样的事情 - 定义一些共享的状态并将其传递给 Provider 但我更喜欢 React 版本，因为它抽离了复杂性。

目前，如果我必须在使用以下两者之间做出选择

* InheritedWidget；
* Brian Egan 的 `Scoped Model` 库或 Remi Rousselet 的 `Provider` 库；

那么我会选择后者。这是因为这些库抽象出了开发人员可能不关心的大部分复杂性。对于在应用程序中的多个位置共享某个状态的简单用例，我认为如果官方有一个更简单，更简洁的 API，对这些库的需求就会显着减少。

## 3. 函数式 Widget

这可能是这个列表中最自私的条目，这是出于我对函数式编程的倾向以及在绝对必要之前避免使用 OOP 的考虑。函数式编程的好处有的很多论据，但这里我不会介绍。在使用了 Vue 和 React 后，我确信可以使用普通函数来更优雅地表示 UI。让我们看一下 widget 的一个子类，并将其与函数变体进行比较。

![](https://cdn-images-1.medium.com/max/1600/1*8Hk3mHBOyw-IcNtL6PsbDw.png)

我更喜欢后者，因为它简单。它易于阅读且没有那么多视觉干扰（我承认这是一个客观的意见，但大多数函数式编程爱好者都会同意这一点）。

我知道，Flutter 中的类很重要，因为它们可以进行优化。我认为 Flutter＆Dart 团队可以轻松给出解决方案，就像他们为依赖注入和 JSON 序列化提供生成器解决方案一样。

实际上，`Remi Rousselet` 的 `functional_widget` 为这个概念提供了很好的 POC。

## 4. Hooks

比起函数式 Widget 来，我可能更喜欢 Flutter 中的 Hooks，这样可以采用 React 中的一些流行的新旧特性。

Hook 是 React 中用于处理状态逻辑的新的原语。Hook 允许您重用`有状态逻辑`而不改变组件层次结构，这样可以避免“`wrapper hell`”。 Flutter 中存在的与 Hook 最接近的是静态方法 `Theme.of`，它允许您从上下文中检索应用程序主题。以下 widget 是我的想法。

![](https://cdn-images-1.medium.com/max/1600/1*iCW4GhtcsaXPRaPiYHNfeQ.png)

我们在终端中运行 flutter create 创建的示例与上面的示例有很大不同。一个是 HomePage 是一个 Function widget，并且可以更新计数器的状态，而无需区分 `Stateless/Stateful widgets`。我只是调用没有参数的 count 函数来获取值，当我希望更新计数时，我用新计数为参数调用它。Hooks 提供的最大好处是代码托管。我不再需要在单个或多个 Widget 和函数之间频繁切换以理解实现，因为它位于我的 Function widget 中。

在经历过一个非常大而复杂的 React 项目后，我看到了可以使用 Hooks 的应用程序以及它们如何简化应用程序中一些最复杂的问题。我希望在 Flutter 中也会有它们的身影。

这个概念的一些好的 POC 非常活跃，如 Remi Rousselet 的 `flutter_hooks` 和 Alfredo Salzillo 的 `flhooks`

## 社区挑战：更多贡献

JavaScript 生态系统因其社区以及语言的演变而蓬勃发展。Flutter 社区将继续增长，它依赖于作为开源软件贡献者和消费者的我们。

我对社区和我自己的挑战是，我们以任何方式构建彼此，无论是通过开发、捐款，还是仅仅是简单的鼓励。

## 结论

Flutter 也有一个非常积极的期望清单，我认为其中一些可以帮助他们进一步发展。 至少我坚信 RFC 流程将激发协作，而 Function widget 和 Hooks 将极大地改善开发人员体验。我希望在 2019 年结束时，我将在明年提出一个全新的愿望清单。

### 参考链接

1. [Introducing the React RFC Process](https://reactjs.org/blog/2017/12/07/introducing-the-react-rfc-process.html)
2. [Influencing the Flutter SDK (The Boring Flutter Development Show, Ep. 13)](https://youtu.be/nGlh4SVrsFg)
3. [Scoped Model](https://pub.dartlang.org/packages/scoped_model)
4. [Provider](https://pub.dartlang.org/packages/provider)
5. [functional_widget](https://pub.dartlang.org/packages/functional_widget)
6. [flutter_hooks](https://pub.dartlang.org/packages/flutter_hooks)
7. [flhooks](https://pub.dartlang.org/packages/flhooks)



