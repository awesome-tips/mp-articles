| 作者：Ryan Turner
| 链接：http://facebook.github.io/react-native/blog/2019/03/12/releasing-react-native-059

Facebook 于本月 12 号发布了 `React Native v0.59`。这次更新来自 88 个贡献者的 644 次提交。让我们一起来看看这个版本有什么新特性。以下是官方通稿的译文。

## 🎣 Hooks

React Hooks 是此版本的一部分，它允许跨组件重用有状态逻辑。下面这些资源介绍了 hook 相关的信息，可以参考一下：

* **Introducing Hooks**：解释了为什么在 React 添加 Hook。(https://reactjs.org/docs/hooks-intro.html)
* **Hooks at a Glance**：对内置 Hooks 的快速预览。(https://reactjs.org/docs/hooks-overview.html)
* **Building Your Own Hooks**：演示了使用自定义 Hooks 重用代码。(https://reactjs.org/docs/hooks-custom.html)
* **Making Sense of React Hooks**：探索了 Hooks 解锁的新可能性。(https://medium.com/@dan_abramov/making-sense-of-react-hooks-fdbde8803889)
* **useHooks.com**：展示社区维护的 Hooks 清单和 demo。

可以在应用中试一试，看看会不会让你感到兴奋。

## 📱 更新的 JSC，以在 Android 上提升性能和支持 64-bit

React Native 使用 JSC（JavaScriptCore）为应用程序提供支持。Android 上的 JSC 已经存在了几年，这意味着很多现代 JavaScript 功能都不受支持。更糟糕的是，与 iOS 的现代 JSC 相比，它表现不佳。随着这个版本的发布，这一切将改变。

在 @DanielZlotin，@ dulmandakh，@ gengjiawen，@kmagiera 和 @kudo 等大神的努力下，Android 的 JSC 已有很大改进，还来了 64 位支持，同时性能也大幅改进。

## 💨 更快的应用程序启动与内联需求

我们希望帮助开发者拥有高性能的 React Native 应用程序，并努力将 Facebook 的优化带入社区。应用程序根据需要加载资源，而不是减慢启动速度。此功能称为“**内联需求**”(`inline requires`)，因为它允许 `Metro` 识别延迟加载的组件。具有深入和多样化组件架构的应用程序将获得最大的改进。

![1](http://)

当升级到 0.59 时，会有一个新的 `metro.config.js` 文件；将选项设置为 true 并向我们提供反馈！更多信息可以参考文档 Performance 一章。(https://facebook.github.io/react-native/docs/0.56/performance#inline-requires)

## 🚅 进行中的精简核心库(Lean core)

React Native 是一个庞大而复杂的项目，具有复杂的 repository。这使得代码库对于贡献者来说并不亲民，难以测试，并且作为开发依赖库来说太大。精简核心库是我们所做的一些努力，通过将代码迁移到单独的库以更好地管理来解决这些问题。过去的几个版本已经看到了这个措施的第一步。

您可能会注意到额外组件现已正式弃用。这是一个好消息，因为现在由这些功能的所有者来积极维护它们。注意警告消息并迁移到新库以获取这些功能，因为它们将在以后的版本中删除。下面的表格显示了组件，其状态以及迁移到的新位置。

![2](http://)

在接下来的几个月里，将会有更多的组件被精简。

## 👩🏽‍💻 CLI改进

React Native 的命令行工具是开发人员进入生态系统的入口点，但它们长期存在问题并且缺乏官方支持。CLI 工具已移至新的 repository，一组专门的维护人员已经做了一些令人兴奋的改进。

现在，日志格式化显得更加友好。命令现在几乎是立即运行 - 可以注意到一些区别：

![3](http://)

## 🚀 升级到0.59

要升级到 0.59，我们建议使用 `rn-diff-purge` 确定当前 React Native 版本到 0.59 之间的更改，然后手动更新这些更改。将项目升级到 0.59 后，您将能够使用新改进的 `react-native upgrade` 升级命令（基于rn-diff-purge！）升级到后续的 0.60 及更高版本，因为新版本将支持这一特性。

## 🔨 破坏性更新

根据谷歌的最新建议，已经梳理了 0.59 中的 Android 支持，这可能会破坏现有应用程序。此问题可能表现为运行时崩溃和消息，"You need to use a Theme.AppCompat theme (or descendant) with this activity"。我们建议更新项目的 `AndroidManifest.xml` 文件，确保 `android:theme` 值是 `AppCompat` 主题（例如 `@style / Theme.AppCompat.Light.NoActionBar`）。

`react-native-git-upgrade` 命令已在 0.59 中删除，支持新改进的 `react-native upgrade` 命令。

## 🤗 小结

据官方声明，0.59 是一个重大的发布，同时在今年剩下的时间里，还会有更多的改进。更多的信息可以查看官方文档。

看来 Facebook 这次是来真的了。

