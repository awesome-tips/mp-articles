## 背景

Swift 5 出了，主要是 ABI 稳定了，从 ABI Dashboard 来看为了解决 ABI 稳定问题，对 type metadata 也有不少改动。众所周知，我们 App 的 JSON 库 HandyJSON 是强依赖 metadata 结构的，如果 metadata 有大规模的改动可能直接导致这个库完全不能用，本着早发现早治疗的心态我赶快下载了 Xcode 10.2 beta，一跑果然编不过了，没办法只好自己着手来解决问题了。

## Metadata 的结构演进

为了便于理解，先画个图看一下 metadata 的具体结构，每一格代码一个指针长度，这是 64 位系统下的 metadata 结构，32 位系统下 nominal type descriptor 的偏移在 11 个指针长度的位置，官方文档里有详细的说明。Swift type metadata 的结构其实并没有明显的变化，而其中的 nominal type descriptor 结构却经历了一系列的变化。

![](http://)

**Swift 4.2 以前**

在 Swift 4.2 （不包括 4.2）以前的结构是这样的：

![](https://user-gold-cdn.xitu.io/2019/2/26/16929595b0015018?w=490&h=734&f=png&s=61393)

```objc
struct _NominalTypeDescriptor {
    var mangledName: Int32
    var numberOfFields: Int32
    var fieldOffsetVector: Int32
    var fieldNames: Int32
    var fieldTypesAccessor: Int32
}
```
复制代码nominal type descriptor 包含了属性的名字和访问属性的类型信息的函数，HandyJSON 最初的原理就是从 nominal type descriptor 中取得属性的类型信息然后把 JSON 字串里的相应值赋进去，由于 fieldTypeAccessor 符合 c 的 calling convention，把指针强转一下就能获得类型信息：

```objc
var fieldTypes: [Any.Type]? {
    guard let nominalTypeDescriptor = self.nominalTypeDescriptor else {
        return nil
    }
    guard let function = nominalTypeDescriptor.fieldTypesAccessor else { return nil }
    return (0..<nominalTypeDescriptor.numberOfFields).map {
        return unsafeBitCast(function(UnsafePointer<Int>(pointer)).advanced(by: $0).pointee, to: Any.Type.self)
    }
}
```

**Swift 4.2**

Swift 4.2 对 nominal type descriptor 做了调整，struct 和 class 结构变得有所不同，乍看没有少什么东西，其实对 fieldTypesAccessor 这个函数做了修改，不再符合 c 的 calling convention，因此不可以再从 nominal type descriptor 获取类型信息。

```objc
struct _StructContextDescriptor: _ContextDescriptorProtocol {
    var flags: Int32
    var parent: Int32
    var mangledName: Int32
    var fieldTypesAccessor: Int32
    var numberOfFields: Int32
    var fieldOffsetVector: Int32
}

struct _ClassContextDescriptor: _ContextDescriptorProtocol {
    var flags: Int32
    var parent: Int32
    var mangledName: Int32
    var fieldTypesAccessor: Int32
    var superClsRef: Int32
    var reservedWord1: Int32
    var reservedWord2: Int32
    var numImmediateMembers: Int32
    var numberOfFields: Int32
    var fieldOffsetVector: Int32
}
```

尽管苹果希望我们用 Mirror 来做反射，但是其实 Mirror 至今为止都不包含属性的类型的信息，因此苹果留了一个临时接口 `swift_getFieldAt` 来帮助我们获取类型信息：

```objc
@_silgen_name("swift_getFieldAt")
func _getFieldAt(
    _ type: Any.Type,
    _ index: Int,
    _ callback: @convention(c) (UnsafePointer<CChar>, UnsafeRawPointer, UnsafeMutableRawPointer) -> Void,
    _ ctx: UnsafeMutableRawPointer
)
```

为什么说是临时的呢，因为 Swift 5 的时候就发现这个接口没了。。。。

**Swift 5.0**

到了 Swift 5.0 的时候，前面已经说过了获取类型的那个接口没了，那么我们只好翻出 Swift 的源码来找找思路了，找到 `TypeContextDescriptorBuilderBase` 类的 `layout()` 方法：

```objc
void layout() {
  asImpl().computeIdentity();

  super::layout();
  asImpl().addName();
  asImpl().addAccessFunction();
  asImpl().addReflectionFieldDescriptor();
  asImpl().addLayoutInfo();
  asImpl().addGenericSignature();
  asImpl().maybeAddResilientSuperclass();
  asImpl().maybeAddMetadataInitialization();
}
```

按源码写出 nominal type descriptor 的结构如下：

```objc
struct _StructContextDescriptor: _ContextDescriptorProtocol {
    var flags: Int32
    var parent: Int32
    var mangledNameOffset: Int32
    var fieldTypesAccessor: Int32
    var reflectionFieldDescriptor: Int32
    var numberOfFields: Int32
    var fieldOffsetVector: Int32
}

struct _ClassContextDescriptor: _ContextDescriptorProtocol {
    var flags: Int32
    var parent: Int32
    var mangledNameOffset: Int32
    var fieldTypesAccessor: Int32
    var reflectionFieldDescriptor: Int32
    var superClsRef: Int32
    var metadataNegativeSizeInWords: Int32
    var metadataPositiveSizeInWords: Int32
    var numImmediateMembers: Int32
    var numberOfFields: Int32
    var fieldOffsetVector: Int32
}
```

虽然 fieldTypesAccessor 还是无法调用，但是我们发现这里多了一个 reflectionFieldDescriptor 指针，直觉告诉我办法应该在这个东西里面，所以先看下这个东西是什么结构：

```objc
void addReflectionFieldDescriptor() {
  ....
    
  B.addRelativeAddress(IGM.getAddrOfReflectionFieldDescriptor(
    getType()->getDeclaredType()->getCanonicalType()));
}
```

逻辑基本就是拿到 ReflectionFieldDescriptor 的地址，然后把地址放到相应的内存里，需要注意的是这里放的是一个相对的地址，RelativePointer 的注释中写道：

```sh
// A reference can be absolute or relative:
//
//   - An absolute reference is a pointer to the object.
//
//   - A relative reference is a (signed) offset from the address of the
//     reference to the address of its direct referent.
```

相对引用指的是相对当前引用指针地址的偏移量，于是我们有了获取 ReflectionFieldDescriptor 地址的方法：

```objc
var reflectionFieldDescriptor: FieldDescriptor? {
    guard let contextDescriptor = self.contextDescriptor else {
        return nil
    }
    let pointer = UnsafePointer<Int>(self.pointer)
    let base = pointer.advanced(by: contextDescriptorOffsetLocation)
    let offset = contextDescriptor.reflectionFieldDescriptor
    let address = base.pointee + 4 * 4 // (4 properties in front) * (sizeof Int32)
    guard let fieldDescriptorPtr = UnsafePointer<_FieldDescriptor>(bitPattern: address + offset) else {
        return nil
    }
    return FieldDescriptor(pointer: fieldDescriptorPtr)
}
```

拿到了地址，我们还需要知道 FieldDescriptor 这个结构是什么样子的，我们找到 FieldDescriptor 这个类：

```objc
// Field descriptors contain a collection of field records for a single
// class, struct or enum declaration.
class FieldDescriptor {
  const FieldRecord *getFieldRecordBuffer() const {
    return reinterpret_cast<const FieldRecord *>(this + 1);
  }

  const RelativeDirectPointer<const char> MangledTypeName;
  const RelativeDirectPointer<const char> Superclass;

public:
  FieldDescriptor() = delete;

  const FieldDescriptorKind Kind;
  const uint16_t FieldRecordSize;
  const uint32_t NumFields;

  using const_iterator = FieldRecordIterator;
  
  ....
}
```

FieldDescriptor 的结构里有一个 FieldRecord 的数组，从名字看里面应该保存了类型信息，我们再翻出 FieldRecord 的源码：

```objc
class FieldRecord {
  const FieldRecordFlags Flags;
  const RelativeDirectPointer<const char> MangledTypeName;
  const RelativeDirectPointer<const char> FieldName;
  ....
}
```

很遗憾 FieldRecord 并没有直接保存类型信息，只有一个 MangledTypeName ，问题不大，我们还有一个叫 swift_getTypeByMangledNameInContext 的函数，这个函数背后调用的 swift_getTypeByMangledName 函数与之前的 getFieldAt 内部调用的是同一个函数，返回是 Any.Type：

```objc
@_silgen_name("swift_getTypeByMangledNameInContext")
public func _getTypeByMangledNameInContext(
    _ name: UnsafePointer<UInt8>,
    _ nameLength: Int,
    genericContext: UnsafeRawPointer?,
    genericArguments: UnsafeRawPointer?)
    -> Any.Type?
```

至此我们解决了由 Swift 5.0 metadata 变动导致的灾难性编译问题，顺便把 metadata 结构梳理了一下，代码已经提交到了 dev_for_swift5.0 分支。

## 后记

Mirror 是官方支持的反射工具，使用 Metadata 这种办法算是一种非主流的做法，但是苹果也意识到 Mirror 里面有部分信息无法提供，据说是技术上有一点困难所以暂时没法把类型信息等放到 Mirror 里面，所以才在 Metadata 里增加了用于反射的信息，ABI Dashboard 里也说 ABI 稳定的优先级高于完整的反射功能，可见 Metadata 这一部分的结构暂时不会大改了，但是远期来看苹果还是会在 Mirror 里面完整支持反射功能。

### 相关资料

* https://swift.org/abi-stability/#data-layout
* https://github.com/apple/swift-evolution
* https://github.com/apple/swift/blob/master/docs/ABI/TypeMetadata.rst#nominal-type-descriptor
* https://github.com/alibaba/HandyJSON

