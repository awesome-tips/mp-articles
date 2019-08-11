# Swift 5 新特性预览(最低支持Xcode 10.2 beta版)

> 万众期待的 Swift 5 终于来了，苹果爸爸答应的 ABI 稳定也终于来了。

## App 瘦身

### 新特性

Swift apps no longer include dynamically linked libraries for the Swift standard library and Swift SDK overlays in build variants for devices running iOS 12.2, watchOS 5.2, and tvOS 12.2. As a result, Swift apps can be smaller when deployed for testing using TestFlight, or when thinning an app archive for local development distribution.

Swift 应用程序不再包含用于 Swift 标准库的`动态链接库`和用于运行 `iOS 12.2`，`watchOS 5.2` 和 `tvOS 12.2` 的设备的构建变体中的 `Swift SDK overlays`。因此，当为 `TestFlight` 进行测试部署时，或者在为本地开发分发瘦身应用的 `archive` 包时，Swift 应用程序可以更小。

To see the difference in file sizes between an app that’s thinned for iOS 12.2 and an app that’s thinned for iOS 12.1 or earlier, set your app’s deployment target to iOS 12.1 or earlier, then create an archive of your app with the scheme set to Generic iOS Device. After building the archive, select Distribute App from the Archives organizer, then select Development distribution. Be sure to select a specific device—such as iPhone XS—in the App Thinning pull-down menu. When the distribution process completes, open the App Thinning Size Report in the newly created folder. The variant for iOS 12.2 will be smaller than the variant for iOS 12.1 and earlier. The exact size difference depends on the number of system frameworks your app uses.

要对比 iOS 12.2 和 iOS 12.1(或更早版本) 瘦身后 App 的文件大小差异，可以设置 App 的 `deployment target` 为 `iOS 12.1` 或更早版本，设置 `scheme set` 为 `Generic iOS Device` 并生成一个 App 的归档。在构建完成后，在 `Archives organizer` 选择中 `Distribute App`，然后选择 `Development distribution`。确保在 `App Thinning` 下拉菜单中选择一个特定设备，如 `iPhone XS`。当分发完成后，在新创建的文件夹下打开 `App Thinning Size Report`。iOS 12.2 系统的变体将小于 iOS 12.1 及更早的系统的变体。确切的大小差异取决于您的 App 使用的系统框架的数量。

For information about app thinning, see What is app thinning? in Xcode Help. For information about app file sizes, see View builds and file sizes in App Store Connect Help.

关于 App 瘦身更多的信息，可以查看 [Xcode Help](https://help.apple.com/xcode/mac/current/) 中的 [What is app thinning?](https://help.apple.com/xcode/mac/current/#/devbbdc5ce4f)有关应用程序文件大小的信息，请参考 [App Store Connect Help](https://help.apple.com/app-store-connect/) 中的 [View builds and file sizes](https://help.apple.com/app-store-connect/#/dev3b56ce97c)

## Swift 语言

### 新特性

* The @dynamicCallable attribute lets you call named types like you call functions using a simple syntactic sugar. The primary use case is dynamic language interoperability. (SE-0216)

* `@dynamicCallable` 允许您使用一个简单的语法糖像调用函数一样来调用命名类型。主要的应用场景是动态语言互操作。([SE-0216](https://github.com/apple/swift-evolution/blob/master/proposals/0216-dynamic-callable.md))

例如：

```swift
@dynamicCallable struct ToyCallable {
    func dynamicCall（withArguments：[Int]）{}
    func dynamicCall（withKeywordArguments：KeyValuePairs <String，Int>）{}
}

let x = ToyCallable（）

x（1,2,3）
// 等价于`x.dynamicallyCall（withArguments：[1,2,3]）`

x(label: 1, 2)
// 等价于`x.dynamicallyCall(withKeywordArguments: ["label": 1, "": 2])`
```

* Key paths now support the identity keypath (\.self), a WritableKeyPath that refers to its entire input value (SE-0227):

* Key path 现在支持特性(identity) keypath (`\.self`)，这是一个引用自身完整输入值的 [WritableKeyPath](https://developer.apple.com/documentation/swift/writablekeypath?language=objc)。([SE-0227](https://github.com/apple/swift-evolution/blob/master/proposals/0227-identity-keypath.md))

```swift
let id = \Int.self
var x = 2
print(x[keyPath: id]) // Prints "2"
x[keyPath: id] = 3
print(x[keyPath: id]) // Prints "3"
```

* Prior to Swift 5, you could write an enumeration case that took variadic arguments:  
*  在 Swift 5 之前，您可以编写一个带有可变参数的枚举 case：

```objc
enum X {
    case foo(bar: Int...) 
}
func baz() -> X {
    return .foo(bar: 0, 1, 2, 3) 
} 
```

 This wasn’t intentionally supported and now generates an error. (46821582)

Instead, change the enumeration case to take an array and explicitly pass an array in: 

之前不是特意要支持这个特性，而且现在这样写会报错了。

取而代之的是，让枚举的 `case` 携带一个数组，并显式传递一个数组

```objc
enum X {
    case foo(bar: [Int]) 
} 
 func baz() -> X {
    return .foo(bar: [0, 1, 2, 3]) 
} 
```

* In Swift 5 mode, try? with an expression of an Optional type flattens the resulting optional, instead of returning a nested optional. (SE-0230)

* 在 Swift 5 中，带有一个可选类型的表达式的 `try?` 将会展平生成的可选项，而不是返回嵌套的可选项。（[SE-0230](https://github.com/apple/swift-evolution/blob/master/proposals/0230-flatten-optional-try.md))

* If a type T conforms to one of the protocols in Initialization with Literals—such as ExpressibleByIntegerLiteral—and literal is a literal expression, then T(literal) creates a literal of type T using the corresponding protocol, rather than calling an initializer of T with a value of the protocol’s default literal type.

For example, expressions like UInt64(0xffff_ffff_ffff_ffff) are now valid, where previously they would overflow the default integer literal type of Int. (SE-0213)

* 如果类型` T` 符合[Initialized with Literals](https://developer.apple.com/documentation/swift/swift_standard_library/initialization_with_literals?language=objc)中的其中一个协议（如 [ExpressibleByIntegerLiteral](https://developer.apple.com/documentation/swift/expressiblebyintegerliteral?language=objc)），且 `literal` 是一个字面量表达示时，则 `T(literal)` 会使用相应的协议创建一个类型 `T` 的字面量，而不是使用一个协议的默认字面量类型的值来调用 `T` 的 `initializer`。

  如，类似于 `UInt64(0xffff_ffff_ffff_ffff)` 这样的表达式现在是有效的，则之前会由于整型字面量的默认类型是 `Int`，而导致溢出。([SE-0213](https://github.com/apple/swift-evolution/blob/master/proposals/0213-literal-init-via-coercion.md))
  
  * String interpolation has improved performance, clarity, and efficiency. (SE-0228)

The older _ExpressibleByStringInterpolation protocol is removed; if you have code that makes use of this protocol, you need to update it for the new design. You can use #if to conditionalize code between Swift 4.2 and Swift 5. For example:

* 提高了`字符串插值`操作的性能、清晰性和效率。([SE-0228](https://github.com/apple/swift-evolution/blob/master/proposals/0228-fix-expressiblebystringinterpolation.md))

  旧的 `_ExpressibleByStringInterpolation` 协议被删除；如果您有使用此协议的代码，则需要做相应更新。您可以使用 `#if` 条件判断来区分 `Swift 4.2` 和 `Swift 5` 的代码。例如：

```objc
#if compiler(<5)
extension MyType: _ExpressibleByStringInterpolation { /*...*/ }
#else
extension MyType: ExpressibleByStringInterpolation { /*...*/ }
#endif 
```

## Swift 标准库

### 新功能

* [DictionaryLiteral](https://developer.apple.com/documentation/swift/dictionaryliteral?language=objc) 类型重命名为 [KeyValuePairs](https://developer.apple.com/documentation/swift/keyvaluepairs?language=objc)。([SE-0214](https://github.com/apple/swift-evolution/blob/master/proposals/0214-DictionaryLiteral.md))

* Swift strings bridged into Objective-C code may now return a non-nil value from CFStringGetCStringPtr when appropriate, and the pointer returned from -UTF8String is tied to the string’s lifetime rather than the innermost autorelease pool. Correct programs shouldn’t have any problems and may see significant speed-ups. However, it may cause previously untested code to run, which could expose latent bugs; for example, if there’s a check for a non-nil value, the branch may never have been taken prior to Swift 5. (26236614)

* 桥接到 `Objective-C` 代码的 `Swift` 字符串现在可以在适当的时候从 [CFStringGetCStringPtr](https://developer.apple.com/documentation/corefoundation/1542133-cfstringgetcstringptr?language=objc)返回一个 `non-nil` 值，同时从 `-UTF8String` 返回的指针与字符串的生命周期相关联，而不是最相近的那个 `autorelease pool`。如果程序正确，那应该没有任何问题，并且会发现性能显著提高。但是，这也可能会让之前一些未经测试的代码运行，从而暴露一些潜在的问题；例如，如果有一个对 `non-nil` 值的判断，而相应分支在 Swift 5 之前却从未被执行过。(26236614)

* The Sequence protocol no longer has a SubSequence associated type. Methods on Sequence that previously returned SubSequence now return concrete types. For example, suffix(_:) now returns an Array. (47323459)

Extensions on Sequence that use SubSequence should be amended to either similarly use a concrete type, or be amended to be extensions on Collection instead, where SubSequence remains available. (45761817)

* [Sequence](https://developer.apple.com/documentation/swift/sequence?language=objc) 协议不再具有 `SubSequence` 关联类型。先前返回 `SubSequence` 的 `Sequence` 方法现在会返回具体类型。例如，[suffix(_:)](https://developer.apple.com/documentation/swift/sequence/3128822-suffix?language=objc)现在会返回一个 `Array`。(47323459)

  使用 `SubSequence` 的 `Sequence` 扩展应该修改为类似地使用具体类型，或者修改为 [Collection](https://developer.apple.com/documentation/swift/collection?language=objc) 的扩展，在 `Collection` 中 `SubSequence` 仍然可用。(45761817)

例如：

```swift
extension Sequence {
    func dropTwo() -> SubSequence {
        return self.dropFirst(2)
    }
}
```

需要改为：

```objc
extension Sequence {
    func dropTwo() -> DropFirstSequence<Self> { 
        return self.dropFirst(2)
    }
}
```

或者是：

```
extension Collection {
    func dropTwo() -> SubSequence {
        return self.dropFirst(2)
    }
}
```

* The String structure’s native encoding switched from UTF-16 to UTF-8, which may improve the relative performance of String.UTF8View compared to String.UTF16View. Consider re-evaluating any code specifically tuned to use String.UTF16View for performance. (42339222)

* [String](https://developer.apple.com/documentation/swift/string?language=objc)结构的原生编码将从 `UTF-16` 切换到 `UTF-8`，与 [String.UTF16View](https://developer.apple.com/documentation/swift/string/utf16view?language=objc) 相比，这会提高相关联的 [String.UTF8View](https://developer.apple.com/documentation/swift/string/utf8view?language=objc) 的性能。重新对所有代码进行评审以提高性能，尤其是使用了 `String.UTF16View` 的代码。

## Swift 包管理器

### 新功能

* Targets can now declare some commonly used, target-specific build settings when using the Swift 5 Package.swift tools-version. The new settings can also be conditionalized based on platform and build configuration. The included build settings support Swift- and C-language defines, C language header search paths, linked libraries, and linked frameworks. (SE-0238) (23270646)

* 现在，在使用 Swift 5 软件包管理器时，`Targets` 可以声明一些常用的针对特定目标的 `build settings` 设置。新设置也可以基于平台和构建配置进行条件化处理。包含的构建设置支持 `Swift` 和 `C` 语言定义，`C` 语言头文件搜索路径，链接库和链接框架。([SE-0238](https://github.com/apple/swift-evolution/blob/master/proposals/0238-package-manager-build-settings.md))(23270646)

* Packages can now customize the minimum deployment target setting for Apple platforms when using the Swift 5 Package.swift tools-version. Building a package emits an error if any of the package dependencies of the package specify a minimum deployment target greater than the package’s own minimum deployment target. (SE-0236) (28253354)

* 在使用 Swift 5 软件包管理器时，`package` 现在可以自定义 Apple 平台的最低 `deployment target`。而如果 `package A` 依赖于 `package B`，但 `package B` 指定的最小 `deployment target` 高于 package A 的最小 `deployment target`，则构建 package A 时会抛出错误。([SE-0236](https://github.com/apple/swift-evolution/blob/master/proposals/0236-package-manager-platform-deployment-settings.md))(28253354)

* A new dependency mirroring feature allows top-level packages to override dependency URLs. (SE-0219) (42511642)

* 新的依赖镜像功能允许顶层包覆盖依赖 URL。([SE-0219](https://github.com/apple/swift-evolution/blob/master/proposals/0219-package-manager-dependency-mirroring.md))(42511642)

  使用以下命令设置镜像：
  
```objc
$ swift package config set-mirror \
--package-url <original URL> --mirror-url <mirror URL>
```

* The swift test command can generate code coverage data in a standard format suitable for consumption by other code coverage tools using the flag --enable-code-coverage. The generated code coverage data is available inside <build-dir>/<configuration>/codecov. (44567442)

* `swift` 测试命令可以使用标志 `--enable-code-coverage`，来生成标准格式的代码覆盖率数据，以便其它代码覆盖工具使用。生成的代码覆盖率数据存储在 `<build-dir>/<configuration>/codecov` 目录中。

* Swift 5 no longer supports the Swift 3 Package.swift tools-version. Packages still on the Swift 3 Package.swift tools-version should update to a newer tools-version. (41974124)

* Swift 5 不再支持 `Swift 3` 版本的软件包管理器。仍然在使用 Swift 3 Package.swift 工具版本（`tool-version`）上的软件包应该更新到新的工具版本上。

* Package manager operations for larger packages are now significantly faster. (35596212)

* 对体积较大的包进行包管理器操作现在明显更快了。 

* The Swift Package Manager has a new --disable-automatic-resolution flag that forces the package resolution to fail when the Package.resolved entries are no longer compatible with the dependency versions specified in the Package.swift manifest file. This feature can be useful for a continuous integration system to check if a package’s Package.resolved is out of date. (45822895)

* Swift 包管理器有一个新的 `--disable-automatic-resolution` 标志项，当 `Package.resolved` 条目不再与 `Package.swift` 清单文件中指定的依赖项版本兼容时，该标志项强制包解析失败。此功能对于持续集成系统非常有用，可以检查包的 `Package.resolved` 是否已过期。 

* The swift run command has a new --repl option that launches the Swift REPL with support for importing library targets of a package. This allows you to easily experiment with API from a package target without needing to build an executable that calls that API. (44889181)

* `swift run` 命令有一个新的 `--repl` 选项，它会启动 `Swift REPL`，支持导入包的库目标。这使您可以轻松地从包目标中试用 API，而无需构建调用该 API 的可执行文件。

* For more information about using the Swift Package Manager, visit Using the Package Manager on swift.org.

* 有关使用 Swift 包管理器的更多信息，请访问 [swift.org](https://swift.org/) 上的 [Using the Package Manager](https://swift.org/getting-started/#using-the-package-manager)。

## Swift 编译器

### 新特性

* Exclusive memory access is now enforced at runtime by default in optimized (-O and -Osize) builds. Programs that violate exclusivity will trap at runtime with an “overlapping access” diagnostic message. You can disable this using a command line flag: -enforce-exclusivity=unchecked, but doing so may result in undefined behavior. Runtime violations of exclusivity typically result from simultaneous access of class properties, global variables—including variables in top-level code—or variables captured by escaping closures. (SR-7139) (37830912)

* 现在，在优化（`-O` 和 `-Osize`）构建中，默认情况下在运行时强制执行独占内存访问。违反排他性的程序将在运行时抛出带有“重叠访问”诊断消息错误。您可以使用命令行标志禁用此命令：`-enforce-exclusivity = unchecked`，但这样做可能会导致未定义的行为。运行时违反排他性通常是由于同时访问类属性，全局变量（包括顶层代码中的变量）或通过 `eacaping` 闭包捕获的变量。（[SR-7139](https://bugs.swift.org/browse/SR-7139)）

* Swift 3 mode has been removed. Supported values for the -swift-version flag are 4, 4.2, and 5. (43101816)

* Swift 3 运行模式已被删除。`-swift-version` 标志支持的值为 `4`、`4.2` 和 `5`。

* In Swift 5 mode, switches over enumerations that are declared in Objective-C or that come from system frameworks are required to handle unknown cases—cases that might be added in the future, or that may be defined privately in an Objective-C implementation file. Formally, Objective-C allows storing any value in an enumeration as long as it fits in the underlying type. These unknown cases can be handled by using the new @unknown default case, which still provides warnings if any known cases are omitted from the switch. They can also be handled using a normal default case.

* 在 Swift 5 中，在 `switch` 语句中使用 `Objective-C` 中声明的或来自系统框架的枚举时，必须处理未知的 `case`，这些 `case` 可能将来会添加，也可能是在 `Objective-C `实现文件中私下定义。形式上，`Objective-C` 允许在枚举中存储任何值，只要它匹配底层类型即可。这些未知的 `case` 可以使用新的 `@unknown default case` 来处理，当然如果 `switch` 中省略了任何已知的 `case`，编译器仍然会给出警告。它们也可以使用普通的 `default case` 来处理。

If you’ve defined your own enumeration in Objective-C and you don’t need clients to handle unknown cases, you can use the NS_CLOSED_ENUM macro instead of NS_ENUM. The Swift compiler recognizes this and doesn’t require switches to have a default case.

如果您已在 `Objective-C` 中定义了自己的枚举，并且不需要客户端来处理 `unknown case`，则可以使用 `NS_CLOSED_ENUM` 宏而不是 `NS_ENUM`。Swift 编译器识别出这一点，并且不需求 `switch` 语句必须带有 `default case`。

In Swift 4 and 4.2 modes, you can still use @unknown default. If you omit it and an unknown value is passed into the switch, the program traps at runtime, which is the same behavior as Swift 4.2 in Xcode 10.1. (SE-0192) (39367045)

在 `Swift 4` 和 `4.2` 模式下，您仍然可以使用 `@unknown default`。如果省略 `@unknown default`，而又传递了一个未知的值，则程序在运行时抛出异常，这与 `Xcode 10.1` 中的 `Swift 4.2` 上的行为是一致的。([SE-0192](https://github.com/apple/swift-evolution/blob/master/proposals/0192-non-exhaustive-enums.md))(39367045)

* Default arguments are now printed in SourceKit-generated interfaces for Swift modules, instead of just using a placeholder default. (18675831)

* 现在在 `SourceKit` 生成的 Swift 模块接口中会打印默认参数，而不仅仅是使用占位符。

* unowned and unowned(unsafe) variables now support Optional types. (47326769)

* `unowned` 和 `unowned(unsafe)` 类型的变量现在支持可选类型。

### 已知的问题

The Swift compiler crashes in the “Merge swiftmodule” build step if any members of the UIAccessibility structure are referenced. The build log includes a message that says:

如果引用了 `UIAccessibility` 结构的任何成员，则 Swift 编译器会在 “`Merge swiftmodule`” 构建步骤中崩溃。构建日志包含一条消息：

```
Cross-reference to module 'UIKit'
... UIAccessibility
... in an extension in module 'UIKit'
... GuidedAccessError 
```

This issue may also occur with other types that contain NS_ERROR_ENUM enumerations, but UIAccessibility is the most common one. (47152185)

包含 `NS_ERROR_ENUM` 枚举的其他类型也可能出现此问题，但 `UIAccessibility` 是最常见的。(47152185)

Workaround: Build that target using the Whole Module option for the Compilation Mode build setting under “Swift Compiler - Code Generation”. This is the default for most Release configurations.

**解决方法**：在 `target` 的 `Build Setting -> Swift Compiler -> Code Generation` 下，设置 `Compilation Mode` 的值为 `Whole Module`。这是大多数 `Release` 配置的默认设置。

* To reduce the size taken up by Swift metadata, convenience initializers defined in Swift now only allocate an object ahead of time if they’re calling a designated initializer defined in Objective-C. In most cases, this has no effect on your program, but if your convenience initializer is called from Objective-C, the initial allocation from +alloc is released without any initializer being called. This can be problematic for users of the initializer that don’t expect any sort of object replacement to happen. One instance of this is with initWithCoder:: the implementation of NSKeyedUnarchiver may behave incorrectly if it calls into Swift implementations of init(coder:) and the archived object graph contains cycles.

* 为了减小 `Swift` 元数据的大小，Swift 中定义的 `convenience initializers` 如果调用了 `Objective-C` 中定义的一个 `designated initializer`，那只会提前分配一个对象。在大多数情况下，这对您的程序没有影响，但如果从 `Objective-C` 调用 `convenience initializers`，那么 `+alloc` 分配的初始内存会被释放，而不会调用任何 `initializer`。对于不希望发生任何类型的对象替换的调用者来说，这可能是有问题的。其中一个例子是 [initWithCoder:](https://developer.apple.com/documentation/foundation/nscoding/1416145-initwithcoder?language=objc) ：如果 [NSKeyedUnarchiver](https://developer.apple.com/documentation/foundation/nskeyedunarchiver?language=objc) 调用 `Swift` 实现 `init(coder:)` 并且存档对象存在循环时，则 `NSKeyedUnarchiver` 的实现可能会出错。
  
  In a future release, the compiler will guarantee that a convenience initializer never discards the object it was invoked on as long as the initializer it delegates to via self.init is also exposed to Objective-C, either because it’s defined in Objective-C, or because it’s marked with @objc, or because it overrides an initializer exposed to Objective-C, or because it satisfies a requirement of an @objc protocol. (46823518)
  
  在将来的版本中，编译器将保证一个 `convenience initializer` 永远不会丢弃它所调用的对象，只要它通过 `self.init` 委托给它的初始化程序也暴露给 `Objective-C`，或者是它在 `Objective-C` 中定义了，或者是使用 `@objc` 标记的，或者是重写了一个暴露给 `Objective-C` 的 `initializer`，或者是它满足 `@objc` 协议的要求。(46823518)

* If a key path literal refers to a property that’s defined in Objective-C or is defined with the @objc and dynamic modifiers in Swift, then compilation may fail with an “unsupported relocation of local symbol 'L_selector'” error, or the key path may not generate the correct hash value or equality comparison with other key paths at runtime. (47184763)

* 如果一个 `keypath` 字面量引用了 `Objective-C` 中定义的属性，或者是在 `Swift` 中使用 `@objc` 和 `dynamic` 修饰符定义的属性，则编译可能会失败，并且报 “`unsupported relocation of local symbol 'L_selector'`” 错误，或者 key path 字面量无法在运行时生成正确的哈希值或处理相等比较。

Workaround: You can define a wrapper property that isn’t @objc, and refer to that key path instead. The resulting key path isn’t equal to a key path that would have referred to the original Objective-C property, but applying the key path has the same effect.

**解决方法**：您可以定义一个不是 `@objc` 修饰的包装属性，来引用这个 `key path`。得到的 `key path` 与引用原始 `Objective-C` 属性的 `key path` 不相等，但使用包装属性效果是相同的。

* Some projects might experience compile time regressions from previous releases. (47304789)

* 某些项目可能会遇到以前版本的编译时回归。

* Swift command line projects crash on launch with “dyld: Library not loaded” errors. (46824656)

* Swift 命令行项目在启动时因抛出 “`dyld：Library not loaded`” 错误而崩溃。

Workaround: Add a user-defined build setting SWIFT_FORCE_STATIC_LINK_STDLIB=YES.

**解决方法**：添加自定义的构建设置 `SWIFT_FORCE_STATIC_LINK_STDLIB=YES`。

### 已解决的问题

* Extension binding now supports extensions of nested types which themselves are defined inside extensions. Previously this could fail with some declaration orders, producing spurious “undeclared type” errors. (SR-631) (20337822)

* 扩展绑定现在支持嵌套类型的扩展，这些嵌套类型本身是在扩展内定义的。之前可能会因为一些声明顺序而失败，并产生 “未声明类型” 错误。([(SR-631)](https://bugs.swift.org/browse/SR-631))

In Swift 5 mode, a class method that returns Self can no longer be overridden with a method that returns a concrete class type that isn’t final. Such code isn’t type safe and needs to be updated. (SR-695) (47322892)

* 在 Swift 5 中，返回 `Self` 的类方法不能再被使用返回非 `final` 的具体类类型的方法来覆盖。此类代码不是类型安全的，需要更新。([SR-695](https://bugs.swift.org/browse/SR-695))

例如：

```
class Base { 
    class func factory() -> Self { /*...*/ }
} 

class Derived: Base {
    class override func factory() -> Derived { /*...*/ } 
} 
```

* In Swift 5 mode, attempting to declare a static property with the same name as a nested type is now always correctly rejected. Previously, it was possible to perform such a redeclaration in an extension of a generic type. (SR-7251) (47325738)

* 在 Swift 5 模式下，现在会明确禁止声明与嵌套类型同名的静态属性。以前，可以在泛型类型的扩展中执行这样的声明。([SR-7251](https://bugs.swift.org/browse/SR-7251))

例如：

```
struct Foo<T> {}

extension Foo { 
    struct i {}

    // Error: Invalid redeclaration of 'i'.
    // (Prior to Swift 5, this didn’t produce an error.) 
    static var i: Int { return 0 }
}
```

* Designated initializers with variadic parameters are now correctly inherited in subclasses. (16331406)

* 现在可以在子类里继承父类中具有可变参数的初始化方法。 

* In Swift 5 mode, @autoclosure parameters can no longer be forwarded to @autoclosure arguments in another function call. Instead, you must explicitly call the function value with parentheses: (); the call itself is wrapped inside an implicit closure, guaranteeing the same behavior as in Swift 4 mode. (SR-5719) (37321597)

* 在 Swift 5 中，函数中的 `@autoclosure` 参数不能再作为 `@autoclosure` 参数传递到另一个函数中调用。相反，您必须使用括号显式调用函数值:`()`；调用本身包含在一个隐式闭包中，保证了与 `Swift 4` 相同的行为。（[SR-5719](https://bugs.swift.org/browse/SR-5719)）

  例如：

```
func foo(_ fn: @autoclosure () -> Int) {}
func bar(_ fn: @autoclosure () -> Int) {
    foo(fn) // Incorrect, `fn` can’t be forwarded and has to be called.
    foo(fn()) // OK
} 
```

* Complex recursive type definitions involving classes and generics that would previously cause deadlocks at runtime are now fully supported. (38890298)

* 现在完全支持在类和泛型中定义复杂的递归类型，而此前可能会导致死锁。

* In Swift 5 mode, when casting an optional value to a generic placeholder type, the compiler will be more conservative with the unwrapping of the value. The result of such a cast now more closely matches the result you would get in a nongeneric context. (SR-4248) (47326318)

* 在 Swift 5 中，当将可选值转换为泛型占位符类型时，编译器在解包值时会更加谨慎。这种转换的结果现在更接近于非泛型上下文中的结果。([SR-4248](https://bugs.swift.org/browse/SR-4248))

  例如：

```
func forceCast<U>(_ value: Any?, to type: U.Type) -> U {
    return value as! U 
} 

let value: Any? = 42
print(forceCast(value, to: Any.self))
// Prints "Optional(42)"
// (Prior to Swift 5, this would print "42".)

print(value as! Any)
// Prints "Optional(42)"
```

* Protocols can now constrain their conforming types to those that subclass a given class. Two equivalent forms are supported:

* 协议现在可以将它们的实现类型限定为指定类型的子类。支持两种等效形式：

```
protocol MyView: UIView { /*...*/ }
protocol MyView where Self: UIView { /*...*/ } 
```
 Swift 4.2 accepted the second form, but it wasn’t fully implemented and could sometimes crash at compile time or runtime. (SR-5581) (38077232)

Swift 4.2 接受了第二种形式，但没有完全实现，有时可能在编译时或运行时崩溃。([SR-5581](https://bugs.swift.org/browse/SR-5581))

* In Swift 5 mode, when setting a property from within its own didSet or willSet observer, the observer will now only avoid being recursively called if the property is set on self (either implicitly or explicitly). (SR-419) (32334826)

* 在 Swift 5 中，当在属性自身的 `didSet` 或 `willSet` 中设置属性本身时，会避免递归调用（不论是隐式或显式地设置自身的属性）。([SR-419](https://bugs.swift.org/browse/SR-419))

  例如：

```objc
class Node {
    var children = [Node]() 
    var depth: Int = 0 {
        didSet { 
            if depth < 0 {
                // Won’t recursively call didSet, because this is setting depth on self. 
                depth = 0
            } 

            // Will call didSet for each of the children,
            // as this isn’t setting the property on self.
            // (Prior to Swift 5, this didn’t trigger property
            // observers to be called again.)
            for child in children { 
                child.depth = depth + 1
            } 
        }
    }
}
```

* `#sourceLocation` is now honored when diagnostics are emitted in Xcode. That is, if your source uses #sourceLocation to map lines in a generated file back to the original source, diagnostics now show up in the original source file rather than in the generated file. (43647151)

* Xcode中 的 `diagnostics` 对 `#sourceLocation` 进行了支持。也就是说，如果您使用`#sourceLocation` 将生成的文件中的行映射回源代码时，`diagnostics` 会显示在原始源文件中的行数而不是生成的文件中的。 

* Using a generic type alias as the parameter or return type of an @objc method no longer results in the generation of an invalid Objective-C header. (SR-8697) (43347303)

* 使用泛型类型别名作为参数或 `@objc` 方法的返回类型，不再导致生成无效的 `Objective-C header`。([SR-8697](https://bugs.swift.org/browse/SR-8697))

