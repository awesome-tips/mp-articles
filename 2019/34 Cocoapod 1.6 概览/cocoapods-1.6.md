
# CocoaPods 1.6 概览

前几天，CocoaPods 官方博客发布了一篇新博文[《CocoaPods 1.7.0 Beta!》](http://blog.cocoapods.org/CocoaPods-1.7.0-beta/)，看到标题你可能会有点慌，1.6 正式版不是刚发布吗，这么快 1.7.0 就开始 Beta 版了？！

而且，根据上述博文介绍，1.7.0 的新版本特性似乎还很酷炫：**多 Swift 版本支持、生成多个 Xcodeproj、增量安装、对 Master Specs Repo 提供 CDN 支持、允许在 Windows 系统上使用 CocoaPods 等。**

不过，1.7.0 仅是刚刚开始开发和测试，距离正式版的发布应该还有一段时间。就如 1.6.0 从去年的 8 月份就发布了 Beta 版，而经历大半年直到上个月才发布正式版：

* 2018-08-16: **1.6.0.beta.1**
* 2018-10-17: **1.6.0.beta.2**
* 2019-01-25: **1.6.0.rc.1**
* 2019-01-29: **1.6.0.rc.2**
* 2019-02-07: **1.6.0**
* 2019-02-22: **1.6.1**

CocoaPods 1.6.0 版本的更新主要集中在针对大型工程的**稳定性、性能和可扩展性方面**。本文将简要介绍一下 1.6.0 的几个重要更新，如：重写 Build Settings 的生成过程，使其速度更快，且更容易维护；以及增强了在同一 pod 库中集成多个 test specs 等。

其完整的更新内容可以查阅 Changelog：https://github.com/CocoaPods/CocoaPods/releases/tag/1.6.0

## 重写 Build Settings 的生成

在 1.6.0 中，**完全重写**了 Build Settings（位于 CocoaPods 生成的 `.xcconfig` 文件中）的生成过程。这次重写的主要目的是清理旧的代码，并使其在大型工程上可以更好地扩展。

我们知道，当 CocoaPods 的贡献者对 Build Settings 进行更改时，通常会导致一定程度的编译性能下降，并引发无法预料的边缘情况（edge cases）组合。这使得测试这些更改以及验证每个 Bug 修复是否正确执行变得非常困难。

所以，从用户的角度看，这次重写应该是无侵入的，对代码的编译不会造成影响。如果你在使用过程中遇到问题，可以在 GitHub 上提一个 issue，并提供复现问题的步骤和示例程序。

为了验证这次重写生成 Build Settings 的正确性，我们与 LinkedIn 的朋友密切合作，他们在一个大型项目工程中使用了 CocoaPods。通过这次重写，以及其他一些性能改进，使得 `pod install` 的执行时间从最初的 3 分钟减少到大约 40 秒，提升了将近 77%。

虽然这次更改对用户来说是不可见的，但它有助于随着 App 工程复杂度的增加，保证 CocoaPods 的可扩展性。

## 增强 Test Spec 的集成

对于 pod 库作者来说，Test Specs 是 CocoaPods 非常基础的一部分。在一个 pod 库的 `podspec` 配置文件中，作者可以声明不同的测试源（test sources），并通过 `lint` 命令来执行，进行不同的单元测试。从 1.6.0 开始，多个 `test_spec` 条目将不再生成单个测试 bundle target，而是分别为每个测试用例生成各自的 bundle。

举个例子，如下：

```ruby
Pod::Spec.new do |s|
  # ... rest of root spec entries go here

  # Unit Test Sources - Those do not require an app host to run. 
  # They also require 'OCMock' dependency.
  s.test_spec 'Tests' do |test_spec|
    test_spec.source_files = 'Tests/**/*.{h,m}'
    test_spec.dependency 'OCMock'
  end

  # SnapShot Tests Sources - Those *do* require an app host to run.
  s.test_spec 'SnapshotTests' do |test_spec|
    test_spec.requires_app_host = true
    test_spec.source_files = 'SnapshotTests/**/*.{h,m}'
  end
```

根据上面描述，运行 `Tests` 测试用例是不需要依赖 app 宿主工程的，而对于 `SnapshotTests` 则需要。但在之前的 CocoaPods 版本中，此时 `Tests` 和 `SnapshotTests` 这两个 test specs 会被合并到同一个单元测试 bundle target 中，导致 `Tests` 的执行也需要在一个 app 宿主工程下，因此可能会带来不必要的影响。

如下图所示：

![](https://file.kangzubin.com/blog/static/20190306/test_specs_before.png)

这意味着，同一 pod 库所有的 test specs 都会被合并到同一个测试 bundle target 中，pod 库的作者无法分离它们并以不同的方式进行分类，例如，分别在不同的 test specs 中配置不同的依赖项、资源或编译器选项。

在 1.6.0 版本中，将生成如下内容：

![](https://file.kangzubin.com/blog/static/20190306/test_specs_after.png)

现在看起来好多了，每个 test spec 都会单独创建一个对应的 bundle target，互不影响。

此外，最近 CocoaPods 还发布了一个 **[cocoapods-generate](https://github.com/square/cocoapods-generate)** 插件，让开发者创建 pod 库的流程得到很大的改善和简化。

## 更长的测试周期

由于 CocoaPods 是由世界各地的开发者在业余空闲时间一起维护和改进的，所以接下来我们将不再对 CocoaPods 每个版本的发布日期做出承若。

1.6.0 版本将会经历很长的测试周期，并在稳定版发布之前一直保持 `beta` 名称（译者注：正如开头所说的，大概花了大半年时间才发布了 1.6.0 的正式版）。

目前 1.7.0 版本已经开始测试了，欢迎使用，你可以通过以下方式安装 beta 版，并在 GitHub 上提交 issue 反馈问题：

```sh
gem install cocoapods --pre
```

## 扩展阅读

* https://github.com/CocoaPods/CocoaPods/releases/tag/1.6.0
* http://blog.cocoapods.org/CocoaPods-1.6.0-beta/
* http://blog.cocoapods.org/CocoaPods-1.7.0-beta/
