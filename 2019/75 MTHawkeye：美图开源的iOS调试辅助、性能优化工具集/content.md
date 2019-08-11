MTHawkeye 是美图 iOS 团队在使用的调试辅助、性能优化辅助工具集，旨在帮助 iOS 开发者提升开发效率、辅助优化性能体验。

在产品开发周期内，我们引入 MTHawkeye 来帮助我们更快的发现、查找、分析、定位、解决问题：

* 开发阶段，侧重于开发调试辅助，及时侦测问题，并在必要时提示开发者及时处理
* 测试阶段，侧重于根据测试场景，收集尽可能多的数据，用于自动化测试分析报告
* 线上阶段，侧重补充传统 APM 组件缺失，但自身业务需要收集的一些性能数据

作为美图内部日常使用的基础工具，现将其开源，期待后续有更多实用的插件以帮助开发者提高效率，更便捷的优化 App 的性能。欢迎 Star，提交 Issue 和 PR，也欢迎外部独立的组件接入到 MTHawkeye 中。

## MTHawkeye 包含了哪些功能

MTHawkeye 简单可分为上中下三层，除了最下面的`基础层`外，中间为`UI 基础层`，最上层的各个插件内部根据不同场景做了职责拆分，应用可根据自己的需要接入。

* 基础层主要提供了插件管理能力，存储能力和一些基础工具类。
* UI基础层 则提供了开发、测试阶段使用的界面层骨架，包含了悬浮窗、主界面框架和设置面板，插件可以集成到其中。

MTHawkeye 上层的功能插件主要以性能侦测插件为主，也引入并改进了 FLEX 作为调试辅助的一个插件，应用接入 MTHawkeye 时可自定义增改自己需要的插件。内置的插件根据关注点分成了 Memory, TimeConsuming, Energy, Network, Graphics, Storage, Utility 几个类别。

### 1. Memory Plugins

**LivingObjectSniffer**

LivingObjectSniffer 主要用于跟踪观察 ViewController 直接或间接持有的对象，以及自定义 View 对象，侦测他们是否异常存活，比如内存泄露、未及时释放或者不必要的内存缓存。

在开发、测试阶段，侦测到的异常情况可以以浮窗警告、Toast 的形式提示开发、测试人员。自动化测试时也可以直接提取记录的存活对象做进一步的分析判断。

**Allocations**

Allocations 类同于 Instrument 的 Allocations 功能，跟踪应用实际分配的内存详情，在应用内存使用异常（异常上升、OOM 退出）时可以通过记录的内存使用详情数据，来排查内存使用问题。

开发、测试阶段使用演示：

![1](http://)

自动化测试、线上阶段接入后，可以持续的跟踪自动化用例、用户场景的实际内存使用情况。

### 2. TimeConsuming Plugins

**UITimeProfiler**

UITimeProfiler 用于辅助主线程耗时任务的优化。内部分成了 VC Life Trace 和 ObjC CallTrace 两部分。VC Life Trace 用于跟踪打开 ViewController 各个阶段的具体时间点，ObjC CallTrace在开启后，则可跟踪耗时大于指定阈值的 Objective-C 方法，类同于 Instrument 的 Time Profiler 功能。

界面层部分将两部分的数据合并展示，便于开发者更便捷的找出关注流程的耗时信息，示例如下：

![2](http://)

自动化测试、线上阶段接入后，无需埋点或插入其他代码，即可持续的跟踪启动耗时、页面打开耗时和其他关键流程耗时。

**ANRTrace**

ANRTrace 用于捕获卡顿事件，同时采样卡顿发生时的主线程调用栈

**FPSTrace**

FPSTrace 用于跟踪界面 FPS 以及 OpenGL 刷新绘制 FPS，并在浮窗上显示当前值

### 3. Energy Plugins

**CPUTrace**

CPUTrace 用于跟踪 CPU 持续高使用率，同时记录高使用率期间主要调用了哪些方法。

### 4. Network Plugins

**Network Monitor**

NetworkMonitor 监听记录 App 内 HTTP(S) 网络请求的各个阶段耗时，并提供内置的记录查看界面，便于开发者排查优化网络问题。

1. 继承自 FLEX 的网络请求记录，过滤搜索。同时优化监听初始化逻辑，大幅减少对启动时间的影响
2. 针对 iOS 9 后的 NSURLSession 的请求，增加记录 URLSessionTaskMetrics 方便查看请求各个阶段的时间
3. 基于 URLSessionTaskMetrics 增加类似 Chrome 网络调试的 waterfall 视图，方便查看网络的先后和同时并发的请求
4. 增加重复网络请求的侦测
5. 增强搜索栏，支持多条件搜索（域名筛选、重复请求、url 过滤、status 过滤）
6. 记录展示完整的网络请求记录（增加 request headers, request body, response body 记录）

开发、测试阶段的演示示例：

![3](http://)

**Network Inspect**

NetworkInspect 插件基于 Network Monitor，根据记录的网络请求实际情况，侦测是否有可改进优化的项，上层可以自定义自己的规则。

### 5. Graphics Plugins

**OpenGLTrace**

OpengGLTrace 用于跟踪 OpenGL 资源内存占用情况，辅助发现 OpenGL API 错误调用、异常参数传递。

OpenGL 渲染引擎在调试上十分不方便，尽管 Xcode 提供了 Frame Capture 的 OpenGL ES 调试工具让我们可以在开发过程中调试每一帧图像绘制流程，但是对于运行过程中的资源泄漏很难排查。

为此 OpenGLTrace 借助 FishHook Hook 了 iOS 平台上 OpenGL ES 和部分 CoreVideo 的函数，实现对 Texture/Program 等资源的监控。

### 6. Storage Plugins

**DirectoryWatcher**

DirectoryWatcher 主要用于沙盒文件夹的大小跟踪，便于开发测试过程中发现异常的文件管理问题。同时也集成了 FLEX 的沙盒文件查看，并扩展支持了文件或文件夹的 AirDrop。

### 7. Utility Plugins

**FLEX**

日常开发中常用的调试辅助工具，MTHawkeye 插件扩展支持了沙盒文件的 AirDrop 功能

## 接入

参考 Readme 的 接入说明 (https://github.com/meitu/MTHawkeye/blob/master/Readme-cn.md#0x01-%E6%8E%A5%E5%85%A5)

## 接入自己的插件

如果有一个模块在开发过程中需要避开很多坑，或者开发过程中调试/优化相关的日志代码很多，可以考虑编写一个调试辅助组件，然后基于 MTHawkeye 基础框架 API，可将这个组件接入到 MTHawkeye 框架中使用，以便统一交互和接口。

详见：MTHawkeye 插件开发说明文档 (https://github.com/meitu/MTHawkeye/blob/master/doc/hawkeye-plugin-dev-guide-cn.md)

## 性能影响说明

各插件的性能影响说明参见项目仓库下的各插件文档。

## 开源地址

https://github.com/meitu/MTHawkeye
https://github.com/meitu/MTGLDebug
https://github.com/meitu/MTAppenderFile

