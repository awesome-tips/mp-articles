几天前 *RxSwift 5*<sup>[1]</sup> 终于发布了，这个版本在源码层面没有太大的变动，更多的是舍弃了一些东西，同时重命名了一些东西，另外也有一些底层的改进，让我们来一起看看。

## 将 Relay 拆分成一个独立的框架 -- RxRelay

Relay 是 Subject 上的一个很好的抽象层，可以让您在不关心 errors 或 completion events 的情况下中继元素。从它们被添加到 RxSwift 后，一直是作为 RxCocoa 工程的一部分而存在的。

一些开发人员对此并不满意，因为这意味着必须导入 RxCocoa 才能使用 Relay。 另外由于 Linux 下没有 RxCocoa，所以在 Linux 环境下也无法使用 Relay。

由于上述原因，在 Swift 5 中将 Relays 独立成 RxRelay <sup>[2]</sup>，并调整了 RxSwift 的依赖图，如下：

![1. 左侧是 RxSwift 4 的依赖图，右侧是 RxSwift 5 的依赖图](http://)

这让您可以只使用 RxSwift 和 RxRelay，而不依赖于 RxCocoa（如果您不需要它），并且还与 RxJava 保持一致，在 RxJava 中，Relay 是一个单独的框架。

> 注意：这是向后兼容的更改，因为 RxCocoa 直接导入了 RxRelay。意思是，您可以继续导入 RxCocoa 而无需导入 RxRelay，一切都会像以前一样工作。

## 弃用 TimeInterval，改用 DispatchTimeInterval

RxSwift 5 对 Schedulers 进行了重构，弃用了 TimeInterval，转而使用 DispatchTimeInterval。当需要亚秒级时序时，DispatchTimeInterval 可以获得更好的事件调度粒度和更高的稳定性。

这会影响所有基于时间的操作符，例如 throttle、timeout、delay、take 等。一个很有意思的副作用是，这消除了 take 的歧义。

**RxSwift 4.x:**

![2. RxSwift 4 中使用 TimeInterval](http://)

**RxSwift 5.x:**

![3. RxSwift 5 中使用 DispatchTImeInterval](http://)

## Variable 最终被弃用

Variable 是早期添加到 RxSwift 中的一个概念，它允许您通过“设置”和“获取”当前值来创建一个命令式的 bridge。这是一个看似有用的措施，可以让开发人员更容易上手使用 RxSwift，直到他们完全适应“响应性思维”。

不过这种结构被证明是有问题的，因为开发人员严重滥用这种结构来创建高度命令式的系统，而不是使用 Rx 的声明式性质。这对于响应式编程的初学者来说尤其常见，并且在概念上阻止了许多人理解这是一种不好的做法和代码味道。这就是为什么在 RxSwift 4.x 中已经使用运行时警告来告知将弃用 Variable 的原因。

在 RxSwift 5 中，它现在已被正式完全弃用，如果您需要这种行为，建议的方法是使用 BehaviorRelay（或 BehaviorSubject）。

**RxSwift 4.x:**

![4. RxSwift 4.x 中弃用 Variable 的警告](http://)

**RxSwift 5.x:**

![5. RxSwift 5.x 中完全弃用 Variable](http://)

## 新增 do(on:) 重载

do 是一个很棒的操作符，当你想要执行某些副作用（如日志记录）或只是“监听”流的中间过程。

为了与 RxJava 保持一致，RxSwift 现在不仅提供 do(onNext:)，还提供如 do(afterNext:) 这样的重载。onNext 表示元素发出的时刻，而 afterNext 表示发出并向下游推送的时刻。

**RxSwift 4.x:**

![6. RxSwift 4.x 提供了 do(onNext:onError:onCompleted:)](http://)

**RxSwift 5.x:**

![7. RxSwift 5.x 同时提供了 do(afterNext:afterError:afterCompleted:)](http://)

## bind(to:) 现在支持多个观察者

在某些情况下，您必须将流绑定到多个观察者。在 RxSwift 4 中，您通常只是复制绑定代码：

![8. RxSwift 4一次只允许绑定到一个观察者](http://)

RxSwift 5 现在支持绑定到多个观察者：

![9. RxSwift 5允许绑定到可变参数列表](http://)

这仍然解析到单个 Disposable，这意味着它向后兼容单观察者变体。

## 一个新的 compactMap 操作符

作为开发人员，您可能经常处理 Optional 值的流。为了解包这些值，社区已经有了自己的解决方案，例如来自 RxSwiftExt 的解包运算符或来自 RxOptional 的 filterNil。

RxSwift 5 添加了一个新的 compactMap 操作符，以与 Swift 标准库保持一致，将此功能引入核心库。

**RxSwift 4.x:**

![10. RxSwift 4没有内置解包可选流](http://)

**RxSwift 5.x:**

![11. RxSwift 5.x提供了compactMap来解包可选流](http://)

## toArray() 现在返回 Single<T>

toArray() 是一个操作符，它在流完成后将整个流作为数组发出。

一直以来这个操作符总是返回一个 Observable<T>，但是由于 Traits 的引入 - 特别是 Single，将返回类型更改为 Single<T> 以提供该类型的安全性和仅保证从此操作符获取单个发射值是很有意义的。

**RxSwift 4.x:**

![12. toArray() 在 RxSwift 4.x 中返回 Observable<T>](http://)

**RxSwift 5.x:**

![13. toArray() 在 RxSwift 5.x 中返回 Single<T>](http://)

## 泛型约束命名整理

RxSwift 是泛型约束的重度使用者。从早期开始，库就使用单字母约束来描述某些类型。 例如，ObservableType.E 表示 Observable 流的泛型类型。

这样可以正常工作，但会引起一些混淆，例如 O 可以代表不同场景中的 Observable 和 Observer，或 S 可以代表 Subject 和 Sequence.

此外，这些单字母约束并没有提供良好的自我描述，并使非代码贡献者很难理解文档。

出于这些原因，我们对私有和公共接口的大多数泛型约束进行了大修，使其具有更多信息以便理解。

The most widely impacting rename is E and ElementType to simply Element.

影响最大的重命名将 E 和 ElementType 改为 Element。

**RxSwift 4.x:**

![14. 在 RxSwift 4 中扩展 Observable 使用 E 泛型约束](http://)

**RxSwift 5.x:**

![15. 在 RxSwift 5 中扩展 Observable 使用 Element 泛型约束](http://)

泛型重命名非常广泛。下面是一个大致完整的列表。这些更改中的大多数都与 RxSwift 的内部 API 有关，其中只有少数会影响到开发人员：

* E 和 ElementType 重命名为 Element；
* TraitType 重命名为 Trait；
* SharedSequence.S 重命名为 SharedSequence.SharingStrategy；
* 依情况 O 被重命名为 Observer 和 Source；
* C 和 S 分别重命名为 Collection 和 Sequence；
* 依情况 S 也被重命名为 Subject；
* R 被重命名为 Result；
* ReactiveCompatible.CompatibleType 重命名为ReactiveCompatible.ReactiveBase。

## 社区项目

许多 RxSwift 社区项目已经迁移到 RxSwift 5 并发布了相应的版本，因此迁移过程应该相对顺利。已迁移的一些项目包括：RxSwiftExt，RxDataSources，RxAlamofire，RxOptional 等。

## 总结

上面列出的更改是开发人员会遇到的主要更改，但是还有更多小的修补程序超出我们帖子的范围，例如在 Linux 下完全修复与 Swift 5 的兼容性、轻微异常等。

请随时查看完整的更改日志并参与官方存储库中的讨论：https://github.com/ReactiveX/RxSwift


### 参考

* [1]https://github.com/ReactiveX/RxSwift/releases/tag/5.0.0
* [2]https://github.com/ReactiveX/RxSwift/pull/1924


