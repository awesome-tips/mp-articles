# CocoaPods 1.7.0 新特性

上个月，CocoaPods 在发布了 1.6.0 正式版不久后，就马上开始了 **1.7.0 Beta** 版的公测，它在先前版本重写底层架构的基础上进行了大量的扩展，是一次巨大更新。

本文将介绍 1.7.0 的几个新特性，主要总结自 CocoaPods 官方博文《CocoaPods 1.7.0 Beta!》，如有描述不当的地方，请查阅原文：

*http://blog.cocoapods.org/CocoaPods-1.7.0-beta/*

## 支持多个 Swift 版本

在 CocoaPods 1.4.0 中首次引入的 `swift_version` 字段（用于指定 pod 库支持的 Swift 版本），现已被扩展成支持多个 Swift 版本。这对于 pod 库的作者来说很有帮助，同时也给 pod 库的使用者在选择使用哪一 Swift 版本时提供了更大的灵活性。

现在，pod 库的作者只需在其 `podspec` 配置文件中通过 `swift_versions` 字段指定其支持的 Swift 版本，即可实现该功能。

下面是一个 `podspec` 示例，指定了它支持的不同 Swift 版本：

```ruby
Pod::Spec.new do |spec|
  spec.name = 'CoconutLib'
  spec.version = '1.0'
  # ... rest of root spec entries go here
  spec.swift_versions = ['3.2', '4.0', '4.2']
end
```

此时，除非 `CoconutLib` pod 库的使用者另有指定，否则 CocoaPods 将在安装过程中自动选择**最新版本**的 Swift，在上述例子中为 `4.2`。

但是，在很多情况下，pod 的使用者并无法使用最新的 Swift 版本，因为他们的开发工具链可能还不支持它。例如，在不支持 Swift 4 的旧版 Xcode 中打开的项目，将无法集成上述 `CoconutLib` pod 库，并最终会导致编译错误。为了解决这个问题，CocoaPods 1.7.0 提供了新的能力，即：可通过 `supports_swift_versions` 字段为 `Podfile` 文件中集成的每个 target 指定所需的 Swift 版本。

例如：

```ruby
target 'MyApp' do
  supports_swift_versions '>= 3.0', '< 4.0'
  pod 'CoconutLib', '~> 1.0'
end
```

通过上述配置就可以成功集成 `CoconutLib` pod 库，并将使用 Swift `3.2` 作为编译版本，因为它是唯一一个版本同时满足了 “`MyApp` target 定义中指定的版本要求” 和 “`CoconutLib podspect` 中声明的可支持的 Swift 版本列表”。

**注意：**我们也可以在 `Podfile` 文件的根级别声明 `supports_swift_versions` 字段，此时它将作用于 `Podfile` 中声明的所有 targets。此外，对于嵌套的 targets（例如 test targets），其 Swift 版本要求将从其父级 targets 中继承，除非它们自己明确指定了。

在选择要使用的 Swift 版本时，可能会出现许多其他边缘情况（edge cases）。我们建议你仔细阅读有关此更改的提案，以了解更多信息：

*https://github.com/CocoaPods/CocoaPods/issues/8191*

#### lint 时验证不同 Swift 版本

Pod 库的作者通常很难维护他们的项目基础设施以支持 Swift 的多个版本（主要是对于较旧的版本），因此，在 `lint` CocoaPods 库时，默认将会选择最新的 Swift 版本来验证。

但对于那些拥有像 CI 系统等基础设施的 pod 库作者来说，为了确保他们的 pod 库可以在旧版本的 Swift 下正常工作，在 `lint` 验证期间可以使用 `--swift-version` 参数来指定 Swift 版本以覆盖默认值。

#### 弃用 `.swift-version` 文件

到目前为止，大多数 pod 库作者都依赖于在其 repo 的根目录中指定一个 `.swift-version` 文件，以便于在成功发布 pod 库时声明他们所正式支持的 Swift 版本。但是，此信息不会在已发布的 `podspec` 中被转录，因此在集成期间，在选择要使用的 Swift 版本时，并不会考虑它。

这可能会导致许多问题，尤其是当新版本的 Swift 发布时，因为使用者可能会自动选择使用最新版本的 Swift，而此时 pod 作者还没来得及正式声明支持它。

**我们强烈建议 pod 库作者在其 `podspec` 文件中改成使用 `swift_version` 字段来声明他们官方支持的 Swift 版本。**

我们还建议删除 repo 中的 `.swift-version` 文件，除非您将其用于其他工具，例如 `swiftenv`。 `swift_version` 字段将始终优先于 `.swift-version` 文件。

最后，在 `lint` 时将会显示警告，提醒 pod 库的作者不再使用 `.swift-version` 文件，并且在未来 CocoaPods 的更新版本中，我们计划完全取消对它的支持。

## 引入 App Specs

通过在 CocoaPods 1.3.0 中引入的 `test specs`，我们可以在此基础上构建一个扩展平台，引入可由 pod 库作者来提供的不同类型的配置说明（specifications）。在 1.7.0 版本中，我们引入了 `app apecs`，允许 pod 库作者在其 `podspec` 中描述一个应用程序。

App specs 可以给作者带来很多帮助，例如，他们可以用于将一个示例应用与其 pod 库一起发布，作为使用者如何将其集成到各自应用程序的教程。

App specs 推进了“独立开发”的概念，其中，`podspec` 可以用作生成整个（可丢弃的）开发工作空间所需的唯一信息。

以下是 app specs 声明的示例：

```ruby
Pod::Spec.new do |s|
  s.name         = 'CoconutLib'
  s.version      = '1.0'
  s.authors      = 'Coconut Corp', { 'Monkey Boy' => 'monkey@coconut-corp.local' }
  s.homepage     = 'http://coconut-corp.local/coconut-lib.html'
  s.summary      = 'Coconuts For the Win.'
  s.description  = 'All the Coconuts'
  s.source       = { :git => 'http://coconut-corp.local/coconut-lib.git', :tag => 'v1.0' }
  s.license      = {
    :type => 'MIT',
    :file => 'LICENSE',
    :text => 'Permission is hereby granted ...'
  }

  s.source_files = 'Sources/*.swift'

  s.app_spec 'SampleApp' do |app_spec|
    app_spec.source_files = 'Sample/*.swift'
  end  
end
```

App specs 可以利用大多数 CocoaPods 支持的 spec DSL（特定领域语言，这里指 `podspec` 文件的语法）来声明依赖，编译配置项等。

默认情况下，app specs 不会自动集成到使用当前 pod 库的项目中。如果你希望这样做，你可以在 `Podfile` 文件中指定，类似于 test specs：

```ruby
target 'MyApp' do
  use_frameworks!
  pod 'CoconutLib', '~> 1.0', :appspecs => ['SampleApp']
end
```

我们希望在未来能够将 app specs 提升到一个顶级（top-level）概念，在这个概念中，开发者的应用程序可以通过一个 app spec 来描述，这样仅通过 CocoaPods 集成就好，完全不需要维护 `.xcodeproj` 了。

## 生成多个 Xcodeproj

在历史版本中，CocoaPods 总是生成一个 `Pods.xcodeproj`，它包含了编译项目所需的所有 targets 和 build settings。对于这种仅使用一个工程来集成整个 `Podfile`，一般比较适用于较小的项目。但是，随着项目的增长，`Pods.xcodeproj` 文件的大小也会随之增加。

`Pods.xcodeproj` 文件越大，Xcode 用于解析其内容的时间越长，这会降低 Xcode 的使用体验。我们注意到，通过将每个 pod 库集成为其自己单独的 Xcode project，并嵌套在顶级 `Pods.xcodeproj` 下，可以显著提高大型 CocoaPods 项目的性能，而不是把所有的 targets 都放到同一个 Xcode project 中。

此外，在大型代码库中，这项功能特别有用，因为开发人员可以选择仅打开他们需要处理的特定 `.xcodeproj`（位于 `Pods/` 目录下），而不是打开整个工作空间（workspace），那样可能会减慢其开发过程。

无论是因为性能问题，还是你仅是喜欢使用多个 Xcode projects 来设置 workspace，CocoaPods 现在支持使用 `generate_multiple_pod_projects` 安装选项来进行此设置。

你可以在 `Podfile` 文件中开启此功能，如下所示：

```ruby
install! 'cocoapods', :generate_multiple_pod_projects => true
```

默认情况下，此选项是**关闭的**，但我们建议您尝试开启使用它，并向我们报告您遇到的任何问题。我们期望在将来对 CocoaPods 进行主要版本更新时，这将成为生成 workspace 时的默认选项。

下面我们看一下它长什么样：

Without multi-xcodeproj (default):

![](https://file.kangzubin.com/blog/static/20190315/no-multi-xcodeproj.png)

With multi-xcodeproj:

![](https://file.kangzubin.com/blog/static/20190315/multi-xcodeproj.png)

**警告：**如果你项目中的 pods 依赖于使用引号语法来导入头文件，则将工程切换为使用多个 `.xcodeproj` 可能会导致一些编译错误。例如 `#import "PDDebugger.h"` 将不再适用于依赖 `PonyDebugger` 的 pod 库，而应该改为：`#import <PonyDebugger/PDDebugger.h>`，以正确导入依赖的 framework 及其相关头文件。

## 增量安装

当执行 `pod install` 时，CocoaPods 现在支持**仅**重新生成自上次 install 以来发生更改的 pod  库，而不是像之前那样重新生成整个 workspace。根据项目的大小，这样做对于每个 pod 库的 install 可以节省几秒到几分钟的时间。

你可以在 `Podfile` 文件中通过 `incremental_installation` 选项开启此功能，如下所示：

```ruby
install! 'cocoapods',
         :generate_multiple_pod_projects => true,
         :incremental_installation => true
```

**注意：**目前，`incremental_installation` 选项需要启用 `generate_multiple_pod_projects` 安装选项才能使其正常运行。

## 新增 `scheme` 字段

Pod 库的作者现在可以为他们的 specs，test specs 以及 app specs 自定义 `scheme` 的生成。目前支持指定环境变量和启动参数，并可以在将来轻松扩展。

举个例子：

```ruby
Pod::Spec.new do |spec|
  spec.name = 'CoconutLib'
  spec.version = '1.0'
  # ... rest of root spec entries go here
  spec.test_spec 'Tests' do |test_spec|
    test_spec.source_files = 'Tests/**/*.swift'
    test_spec.scheme = { 
      :launch_arguments => ['Arg1', 'Arg2'], 
      :environment_variables => { 'Key1' => 'Val1'}
    }
  end
end
```

以上示例将为 `Tests` test spec 生成如下 `scheme`：

![](https://file.kangzubin.com/blog/static/20190315/scheme_config.png)

**注意：**您可以选择为某一特定平台配置 `scheme`，例如，`test_spec.ios.scheme`  将仅为 iOS target 配置 `scheme`。

## 支持 `.xcfilelist`

在 CocoaPods 1.7.0 中，`script phases` 现在支持使用 `.xcfilelist` 来指定脚本的输入和输出路径。CocoaPods 将自动检测正在集成的 Xcode 项目是否支持 `.xcfilelist`，且其优先级高于单独设置的 `input/output` 路径条目。

这不但减少了 CocoaPods 在用户项目中占用的空间，而且也利用了在每种配置（例如 'Debug' 和 'Release'）中使用不同 `input/output` 路径的能力。

## 实验特性

在 1.7.0 版本中还提供了一些令人兴奋且重要的实验特性。

#### 为 Master Specs Repo 提供 CDN 支持

Master specs repo 对于 CocoaPods 的正常运行至关重要，但是，多年来，它的规模已经急剧增长，使其成为 CocoaPods 的第一大难题。 

*https://github.com/CocoaPods/Specs*

这对于那些网络连接速度较慢的人来说尤其如此，因为这种情况下克隆整个 repo 及其完整提交历史几乎是不可能的。此外，由于克隆需要很长时间，CI 系统在首次设置时也会显著变慢。

在 1.7.0 中，我们正在启动 CDN 支持，以避免在本地机器或 CI 系统上克隆 `master specs repo`，让使用 CocoaPods 更加方便。这可以通过将 `Podfile` 中声明的 `master specs repo` 的`source` 替换为如下内容来实现：

```ruby
# source 'https://github.com/CocoaPods/Specs' comment or remove this line.
source 'https://cdn.jsdelivr.net/cocoa/'
```

此外，你可以通过执行 `pod repo remove master` 来删除现有基于 `git` 的 repo。

我们要感谢 **[jsDelivr](https://www.jsdelivr.com/)** 对这项工作的支持和帮助！目前，我们将继续保持这两种 `master specs repo` 的使用方式，但我们强烈建议你切换成上述最新的 `source`，特别是对于首次使用 CocoaPods 来说可以节省很多时间。

我们将根据测试结果和服务的稳定性，希望从 1.7.0 开始，CocoaPods 不再需要用户克隆 `master specs repo` 才能正常使用。

#### 为 Windows 系统提供支持

从 1.7.0 开始，我们添加了对 Windows 系统的支持！我们鼓励 Windows 用户使用 CocoaPods 并在 GitHub Issues 中向我们报告任何问题：

*https://github.com/cocoapods/cocoapods/issues*

## 如何更新

CocoaPods 1.7.0 是一个非常令人兴奋的巨大版本更新，我们建议您通过如下方式升级并参与测试：

```sh
$ gem install cocoapods --pre
```
