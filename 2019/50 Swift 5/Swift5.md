# Swift 5 新特性补充

今年 1 月底，苹果发布了 iOS 12.2 Beta 和 Xcode 10.2 Beta，同时给我们带来了 Swift 5，其中最重要的特性莫过于 **ABI 稳定**，具体可以参考知识小集之前翻译的[《Swift 5 新特性一览》](https://mp.weixin.qq.com/s/3zEwQug4xrQSYa6FCJQh7g)，也可以参考前端之巅的这篇文章[《Swift 5 新特性详解：ABI 稳定终于来了！》](https://mp.weixin.qq.com/s/kmCuSt-1LT6uazURNo6z5Q)。

昨天（2019 年 3 月 26 日），Swift 5 终于随 Xcode 10.2 正式发布，本文将在上述文章 Beta 版的基础上，**补充**正式版新增的一些特性，具体可以参考官方原文：

* [Swift 5 Release Notes for Xcode 10.2](https://developer.apple.com/documentation/xcode_release_notes/xcode_10_2_release_notes/swift_5_release_notes_for_xcode_10_2)

## 通用

### Swift 5 运行时支持命令行工具包

从 Xcode 10.2 开始，Swift 命令行工具需要依赖于 macOS 中的 Swift 库。这些库从 macOS Mojave 10.14.4 起将默认被包含在系统中。但在 macOS Mojave 10.14.3 及更早版本中，你可以从 [More Downloads for Apple Developers](https://developer.apple.com/download/more/) 中下载安装一个可选包以为 Swift 命令行工具提供运行时所需的库。如果你已经安装了此软件包的测试版，请将其替换为正式版。此外，只有 Swift 命令行工具才需依赖此可选包，而具有图形用户界面的应用程序则不需要。

## Swift 语言

### 新特性

1、现在可以使用增强分隔符来表示字符串文本（string literals）。对于一个字符串文本，如果其左引号之前带有一个或多个井号（`#`），此时它会将反斜杠和双引号视为字符而不转义，除非该字符串的末尾跟有相同数量的井号。使用增强分隔符可以避免将包含多个双引号或反斜杠字符的字符串文本与额外的转义符混淆。[SE-0200]

```swift
print(#"<a href="\#(url)" title="Apple Developer">"#)
// Equivalent to:
print("<a href=\"\(url)\" title=\"Apple Developer\">")
```

2、如果你声明了一个类型与标准库中的类型同名，则你的类型现在会覆盖标准库中的类型。

例如，在名为 `Foo` 的模块中声明了一个 `Result` 类型：

```swift
// Module `Foo`.
public enum Result<T> {
    case value(T)
    case error(Error)
}
```

那么，在任何导入 `Foo` 模块的代码中对 `Result` 类型的非限定引用将默认被解析为 `Foo.Result`：

```swift
import Foo
func doSomething() -> Result<Int> { /* … */ }
```

此时，要引用 Swift 标准库中的 `Result` 类型，则需要使用 `Swift.` 前缀显式声明，例如：

```swift
import Foo
func useStandardLibraryResult() -> Swift.Result<Int, Error> { /* … */ }
```

## Swift 标准库

### 新特性

1、标准库现在包含的 `Result` 枚举，带有 `Result.success(_:)` 和 `Result.failure(_:)` 两种情况。可以使用 `Result` 来手动传递和处理错误，在 `do-catch` 语句和 `try` 表达式无法使用的情况下，例如当你使用可能导致失败的异步 API 时。

作为此添加的一部分，`Error` 协议现在也遵循自身，这使得在通用上下文中处理错误更加容易。[SE-0235]

2、标准库定义了 `SIMD` 类型和基本运算符。在 `simd` framework 中提供的类型，如 `float2` 和 `float3` 等，现在它们是新标准库类型的类型别名。

`SIMD` 类型是标量元素类型的泛型。例如，旧的 `float3` 类型是 `SIMD3<Float>` 的类型别名。任何遵循 `SIMDScalar` 协议的类型都可以用作 `SIMD` 向量的标量类型，但是，有效的向量化取决于为相关的 `SIMDStorage` 类型选择一个良好的数据布局以及进行有效的下标操作。

大多数使用 `SIMD` 类型的现有代码将可以继续使用新的通用 `SIMD` 类型，但需要注意一些更改。

新类型增加了一些新的一致性，`SIMD` 向量现在是 `Hashable`，`Equatable` 和 `Codable`。这将允许你删除自己现有代码中实现这些一致性的扩展。

对运算符集的重载现已被得到大量的扩展，可提供向量-标量算法。这使得编写一些东西变得更加容易，但在某些情况下会给类型检查器带来歧义，并且可能需要拆分某些表达式或使用显式类型进行注释。

由于类型现在是通用的泛型而不是具体的，如果您已经在 `simd` framework 类型上定义了自己的协议，那么可能需要重构一致性，因为 Swift 泛型不能对协议具有多个条件一致性。这种情况比较少见，你通常需要做的是重构代码，如下所示：

```swift
protocol MyVectorProtocol { /* ... */ }
extension float2: MyVectorProtocol { /* ... */ }
extension double2: MyVectorProtocol { /* ... */ }
```

现要改为使用以下结构：

```swift
protocol MySIMDScalarProtocol: SIMDScalar { /* ... */ }
extension SIMD2 where Scalar: MySIMDScalarProtocol { /* ... */ }
// Or even:
protocol MySIMDScalarProtocol: SIMDScalar { /* ... */ }
extension SIMD where Scalar: MySIMDScalarProtocol { /* ... */ }
```

此更改可以允许你删除一些冗余的实现，但它要求您定义任何必要的实现钩子（hooks），这些 hooks
引用 Darwin 系统中的标量类型的 C 头文件里的具体函数。[SE-0229]

3、`Set` 和 `Dictionary` 现在为每个新创建的实例使用不同的哈希种子。因此，在相同 sets 和 dictionaries 中元素的顺序与以前版本中的顺序相比差别更大：

```swift
let a: Set<Int> = [1, 2, 3, 4, 5]
let b: Set<Int> = [1, 2, 3, 4, 5]
a == b  // true
print(a) // [1, 4, 3, 2, 5]
print(b) // [4, 2, 5, 1, 3]
```

现有代码中，如果错误地假定两个不相关但相等的集合或字典包含相同顺序的元素，这在 Swift 5 中将更容易产生错误的结果。尽管元素排序在不同的 `Set` 或 `Dictionary` 实例之间是不稳定的，但是在同一实例上的多次迭代之间，顺序不会发生变化。除了强调这些集合不能保证一致的元素排序之外，此更改还修复了一些批量操作的情况，例如 `union(_:)` 表现出二次性能。

4、为了防止 Cocoa 对象出现不一致的哈希，`NSObject` 的 `hashValue` 属性将不再是可重写的。重写 `hashValue` 在 Swift 4.2 中已被废弃。可以在 `NSObject` 的子类中重载 `hash` 属性以实现自定义哈希值。举个例子：

```swift
class Person: NSObject {
    let name: String

    init(name: String) {
        self.name = name
        super.init()
    }

    override func isEqual(_ other: Any?) -> Bool {
        guard let other = other as? Person else { return false }
        return other.name == self.name
    }

    override var hash: Int {
        var hasher = Hasher()
        hasher.combine(name)
        return hasher.finalize()
    }
}
```

此外，哈希和相等判断是紧密相关的。如果重写了 `hash`，则也需要重写 `isEqual(_:)`，反之亦然。

### 已知的问题

在 Xcode 10.2 Beta 版中，`Sequence` 协议里增加的 `count(where:)` 方法现已被移除。

**解决办法：**使用 `reduce(_:_:)` 可以有效地计算出与断言匹配的出现次数：

```swift
let occurrences = sequence.reduce(0) { predicate($1) ? $0 + 1 : $0 }
```

### 已解决的问题

1、在字符串上设置 `utf8` 属性现可以正常工作。

2、传递一个 `UnsafeBufferPointer<UInt8>` null 值给 String 结构体的 `init(decoding:as:)` 方法现在可以正常地返回空字符串。

## Swift 编译器

### 新特性

1、为了减小 Swift 元数据占用的大小，Swift 中定义的便捷初始化方法（convenience initializer）现在只会在调用了 Objective-C 中定义的 `designated initializer` 才会提前分配对象的内存空间。在大多数情况下，这不会对你的应用产生影响，但如果你的 `convenience initializer` 被 Objective-C 调用，且没有调用自身暴露给 Objective-C 的 `self.init` 方法，那么最初通过 `alloc` 分配的内存空间会在没有调用任何 `initializer` 的情况下被释放。这对于使用 `initializer` 的人来说容易产生困扰，因为他们并没有意识到任何对象替换（object replacement）的发生。其中一个例子是 `init(coder:):` 方法：如果 `NSKeyedUnarchiver` 调用了在 Swift 中定义的 `init(coder:)`，并且存档的对象图包含循环，则 `NSKeyedUnarchiver` 的实现可能会出错。

为了避免这种情况，你需要确保不支持对象替换的 `convenience initializer` 调用了自身暴露给 Objective-C 的 `initializers`，这些 `initializers` 可以是在 Objective-C 中定义的，也可以是因为它们用 `@objc` 标记，或者是因为它们覆盖了暴露给 Objective-C 的 `initializers`，也可以是它们满足 `@objc` 协议的要求。

2、Swfit 不再支持超过 16 字节对齐的 C 类型。事实上，Swift 编译器也从未正确处理过这些类型。(31411216)

3、在 Swift 5 中，非 `final class` 的便捷初始化方法中的 `self` 的类型现在是动态的 `Self` 类型，而不是具体的类型了。(47323459)

## 结语

以上的总结是在我们之前翻译的 Beta 版[《Swift 5 新特性一览》](https://mp.weixin.qq.com/s/3zEwQug4xrQSYa6FCJQh7g)基础上新增的一些特性，完整的内容请查看之前的文章和官方 Release Note。

最后，你也可以直接阅读老司机周报刚刚翻译出炉的完整版：

* [Swift 5 终于来了，快来看看有什么更新！！](https://mp.weixin.qq.com/s/-fLVdoTz3lT5Kxnea0-Avg)
