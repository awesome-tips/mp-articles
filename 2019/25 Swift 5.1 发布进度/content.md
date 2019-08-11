> Swift 5.0 的 ABI 刚稳定不久，Swift 团队就开始着手 Swift 5.1 的发布了，并在官方网站发了博文来公布 Swift 5.1 的发布进度。以下是官宣的译文，借此也可以管窥一下苹果对发布流程的控制。

这篇文章描述了 Swift 5.1 的目标，发布过程和预计时间表。

## 动机和目标

Swift 5.1 的主要目标是使语言实现模块稳定性。

## 二进制兼容性

在 Apple 平台上，由于 ABI 现在已经稳定，Swift 5.1 与 Swift 5.0 是二进制兼容，并且与未来版本的 Swift 二进制兼容。

在非 Apple 平台（例如Linux）上，ABI 尚未完全稳定，需要进行更多审查。基于 Linux 的新平台将尤其需要这样的审查。

## 源码兼容性

与 Swift 5.0 一样，我们希望使用 Swift 5.0 编译器构建的大多数源代码都可以使用 Swift 5.1 编译器进行编译。但是，Swift 5.1 中的 Bug 修复可能会导致它检测到以前未检测到的代码中的错误。

## Swift 5.1 的快照

作为持续集成测试的一部分，将定期发布 Swift 5.1 版本分支的可下载快照。

一旦发布了 Swift 5.1，除了快照之外，还将发布官方最终版本。

## Swift 5.1 中的更新

Swift 5.0 的开发需要在其融合的整个过程中获得不同寻常的关注，因为每个问题都必须针对其持久的 ABI 影响进行评估。受此影响，Swift 5.1 的开发窗口比以前的版本短得多。需要这种更严格的时间限制来确保提供成熟稳定的 5.1 版本，并为破坏性更改提供更严格的截止日期。

`swift-5.1-branch` 包含将在 Swift 5.1 中发布的更改。该分支将按如下方式进行管理：

* swift-5.1-branch 是从 master 主分支拉取。
* master 分支将定期合并到 swift-5.1-branch 中，直到最终分支日期为止。
* `2019年3月18日（最终分支）`：swift-5.1-branch 将最后一次从 master 合并。在最后一个分支日期之后，将有一个“bake”期，其中只选择关键修复作为发布项（通过拉取请求）。

该计划的一些值得注意的例外情况见下表。每一项每天都会从 master 合并到 swift-5.1-branch。每个例外变更的最终截止日期将延长至 3 月 18 日以后，并将在稍后公布。

![1](http://)

## 将更改放入 Swift 5.1 的指导思想

* Swift 5.1 的所有语言和 API 更改都将通过 Swift Evolution 过程完成。Evolution 提案的目标应该是在分支日期之前完成，以增加影响 Swift 5.1 版本的机会。将根据具体情况考虑例外情况，特别是如果它们与发布的核心目标相关联。
* 其他更改（例如，错误修复、诊断改进、SourceKit 界面改进）将根据其风险和影响被接受。
* 如果有助于提高发布的质量，那么低风险的测试调整也将在发布分支的后期被接受。
* 随着发布的临近，接受的变更的标准将变得越来越严格。

## 受影响的 Repositories

以下 repositories 将有一个 swift-5.1-branch 分支来跟踪源代码，作为 Swift 5.1 版本的一部分：

* indexstore-db
* sourcekit-lsp
* swift
* swift-clang
* swift-cmark
* swift-compiler-rt
* swift-corelibs-foundation
* swift-corelibs-libdispatch
* swift-corelibs-xctest
* swift-integration-tests
* swift-llbuild
* swift-lldb
* swift-llvm
* swift-package-manager
* swift-stress-tester
* swift-syntax
* swift-xcode-playground-support

## 发布分支的 Pull Requests

为了将 Pull Requests 包含在发布分支中，它必须包含以下信息：

* 说明：正在修复或增强的问题的说明。这可以是简短的，但应该很清楚。
* 范围：评估变更的影响/重要性。例如，更改是破坏源语言的变化等。
* SR 问题：如果更改修复或实现了在 bugs.swift.org 上的问题或增强特性，则为SR。
* 风险：采取此更改的发布的（特定）风险是什么？
* 测试：为了进一步验证此更改的影响，已完成或需要进行哪些特定测试？
* 审阅者：受影响组件的一个或多个代码所有者应审核更改。技术审核可由代码所有者委派，或在认为适当或有用时另行要求。

进入 swift-5.1-branch 的所有更改（外部更改将自动从master中合并）必须通过相应发布管理人员接受的 Pull Requests。

