| 作者：Andrea Bizzotto
| 原文链接：https://medium.com/coding-with-flutter/dart-vs-swift-a-comparison-6491e945dc17

Dart 和 Swift 是我最喜欢的编程语言。我在商业和开源代码中广泛使用它们。

本文提供了 Dart 和 Swift 之间的比较，旨在：

* 突出显示两者之间的差异；
* 作为开发人员从一种语言转移到另一种语言（或使用两者）的参考。

一些背景：

* Dart 支持 Flutter，这是 Google 用于从单一代码库构建漂亮的本机应用程序的框架。
* Swift 通过 iOS，macOS，tvOS 和 watchOS 为 Apple 的 SDK 提供支持。

以下是两种语言的主要特征（`Dart 2.1` 和 `Swift 4.2`）的比较。由于深入讨论每个功能超出了本文的范围，因此更多的信息可以参考各自的文档。

## 目录

* 对照表
* 变量
* 类型推断
* 可变/不可变变量
* 函数
* 命名和未命名参数
* 可选和默认参数
* 闭包
* 元组
* 控制流
* 集合
* Nullability & Optionals
* 类
* 继承
* 属性
* 协议/抽象类
* Mixins
* 扩展
* 枚举
* 结构体
* 错误处理
* 泛型
* 访问控制
* 异步编程：Future
* 异步编程：Stream
* 内存管理
* 编译和执行
* 其它未涵盖功能

## 对照表

![](https://cdn-images-1.medium.com/max/800/1*16HAMTB-9z7pQ1X7iZoI8g.png)

## 变量

Dart 中变量**声明**语法如下：

```objc
String name;
int age;
double height;
```

Swift 中是如下：

```objc
var name: String
var age: Int
var height: Double
```

Dart 中变量**初始化**语法如下：

```objc
var name = 'Andrea';
var age = 34;
var height = 1.84;
```

Swift 中是如下：

```objc
var name = "Andrea"
var age = 34
var height = 1.84
```

在此示例中，不需要类型注释。这是因为两种语言都可以从赋值右侧的表达式推断出类型。

## 类型推断

类型推断意味着我们可以在 Dart 中编写以下代码：

```dart
var arguments = {'argA': 'hello', 'argB': 42}; // Map<String, Object>
```

编译器会自动解析 `arguments` 的类型。

在 Swift 中，同样可以写成：

```swift
var arguments = [ "argA": "hello", "argB": 42 ] // [ String : Any ]
```

### 更多细节

Dart 文档有如下描述：

> 分析器可以推断字段、方法、局部变量和大多数泛型类型参数的类型。当分析器没有足够的信息来推断特定类型时，将使用动态类型。

Swift 文档中有如下描述：

Swift 广泛使用类型推断，允许您省略代码中许多变量和表达式的类型或部分类型。例如，不是写 `var x:Int = 0`，而是可以写 `var x = 0`，完全省略类型 - 编译器正确地推断出 x 为 `Int` 类型的值。

#### 动态类型

可以使用 Dart 中的 `dynamic` 关键字和 Swift 中的 `Any` 关键字声明可以是任何类型的变量。

在读取 `JSON` 等数据时，通常会使用动态类型。

## 可变/不可变变量

变量可以声明为**可变**或**不可变**。

为了声明**可变**变量，两种语言都使用 `var` 关键字。

```swift
var a = 10; // int (Dart)
a = 20; // ok

var a = 10 // Int (Swift)
a = 20 // ok
```

为了声明**不可变**变量，Dart 使用 `final`，Swift 使用 `let`。

```swift
final a = 10;
a = 20; // 'a': a final variable, can only be set once.

let a = 10
a = 20 // Cannot assign to value: 'a' is a 'let' constant
```

注意：Dart 文档定义了两个关键字 `final` 和 `const`，其工作方式如下：

> 如果您不打算更改变量值，请使用 `final` 或 `const`，而不是 `var` 或类型。`final` 变量只能设置一次；`const` 变量是编译时常量。（`Const` 变量是隐式 `final`。）final 顶层类型变量或类变量在第一次使用时被初始化。

在 Dart 网站上的[这篇文章](https://news.dartlang.org/2012/06/const-static-final-oh-my.html)中可以找到进一步的解释：

> final 意味着一次赋值。final 变量或字段必须具有 `initializer`。 一旦赋值，就不能改变 final 变量的值。 

在 Swift 中，我们用 `let` 声明常量。

> 常量声明会在程序中引入常量命名值。使用 `let` 关键字声明常量，并具有以下形式：

```swift
let constant name: type = expression
```

常量声明定义常量名称和初始化表达式值之间的不可变绑定；设置常量值后，无法更改。

## 函数

函数在 Swift 和 Dart 中都是一等公民。

这意味着就像对象一样，函数可以作为参数传递，保存为属性或作为结果返回。

作为初始比较，我们可以看到如何声明不带参数的函数。

在 Dart 中，返回类型在方法名称之前：

```dart
void foo();
int bar();
```

在 Swift 中，我们使用 `-> T` 表示法作为后缀。如果没有返回值（Void），则不需要这样做：

```swift
func foo()
func bar() -> Int
```

## 命名及未命名(un-named)参数

两种语言都支持命名和未命名的参数。

在 Swift 中，参数默认为**命名参数**：

```swift
func foo(name: String, age: Int, height: Double)
foo(name: "Andrea", age: 34, height: 1.84)
```

在 Dart 中，我们使用花括号（{}）定义命名参数：

```dart
void foo({String name, int age, double height});
foo(name: 'Andrea', age: 34, height: 1.84);
```

在 Swift 中，我们使用`下划线(_)` 作为外部参数来定义未命名的参数：

```swift
func foo(_ name: String, _ age: Int, _ height: Double)
foo("Andrea", 34, 1.84)
```

在 Dart 中，我们通过省略花括号（{}）来定义未命名的参数：

```dart
void foo(String name, int age, double height);
foo('Andrea', 34, 1.84);
```

## 可选和默认参数

两种语言都支持默认参数。

> 在 Swift 中，您可以通过在该参数的类型之后为参数赋值来为函数中的任何参数定义默认值。如果定义了默认值，则可以在调用函数时省略该参数。

```swift
func foo(name: String, age: Int = 0, height: Double = 0.0) 
foo(name: "Andrea", age: 34) // name: "Andrea", age: 34, height: 0.0
```

在 Dart 中，可选参数可以是位置参数，也可以是命名参数，但不能同时。

```dart
// positional optional parameters
void foo(String name, [int age = 0, double height = 0.0]);
foo('Andrea', 34); // name: 'Andrea', age: 34, height: 0.0
// named optional parameters
void foo({String name, int age = 0, double height = 0.0});
foo(name: 'Andrea', age: 34); // name: 'Andrea', age: 34, height: 0.0
```

## 闭包

作为顶层(first-class)对象，函数可以作为参数传递给其他函数，或者分配给变量。

在此上下文中，函数也称为**闭包**。

这是一个函数的 Dart 示例，它迭代一个 item 列表，使用闭包来打印每个项目的索引和内容：

```dart
final list = ['apples', 'bananas', 'oranges'];
list.forEach((item) => print('${list.indexOf(item)}: $item'));
```

闭包带有一个参数（`item`），打印该项的索引和值，并且不返回任何值。

注意使用`箭头符号(=>)`。这可以代替花括号内的单个 `return` 语句：

```dart
list.forEach((item) { print('${list.indexOf(item)}: $item'); });
```

Swift 中的相同代码如下所示：

```swift
let list = ["apples", "bananas", "oranges"]
list.forEach({print("\(String(describing: list.firstIndex(of: $0))) \($0)")})
```

在这种情况下，我们不为传递给闭包的参数指定名称，而使用 `$0` 代替第一个参数。这完全是可选的，我们仍然可以使用命名参数：

```swift
list.forEach({ item in print("\(String(describing: list.firstIndex(of: item))) \(item)")})
```

闭包通常用作 Swift 中异步代码的完成块（请参阅下面有关`异步编程`的部分）。

## 元组

Swift 文档的描述如下：

> 元组将多个值分组为单个复合值。元组中的值可以是任何类型，并且不必具有相同的类型。

这些可以用作小型轻量级类型，在定义具有多个返回值的函数时非常有用。

以下是如何在 Swift 中使用元组：

```swift
let t = ("Andrea", 34, 1.84)
print(t.0) // prints "Andrea"
print(t.1) // prints 34
print(t.2) // prints 1.84
```

Dart 中有一个单独[三方包](https://pub.dartlang.org/packages/tuple)支持元组：

```dart
const t = const Tuple3<String, int, double>('Andrea', 34, 1.84);
print(t.item1); // prints 'Andrea'
print(t.item2); // prints 34
print(t.item3); // prints 1.84
```

## 控制流

两种语言都提供多种控制流语句。

例如，if、for、while、switch 语句。

在这里介绍这些将是相当冗长的，所以请参考官方文档。

## 集合(arrays, sets, maps)

### Arrays / Lists

数组是有序的对象组。

在 Dart 中，使用 `List` 对象来表示数组：

```dart
var emptyList = <int>[]; // empty list
var list = [1, 2, 3]; // list literal
list.length; // 3
list[1]; // 2
```

Swift 中数组是内置类型：

```swift
var emptyArray = [Int]() // empty array
var array = [1, 2, 3] // array literal
array.count // 3
array[1] // 2
```

### Sets

Swift 文档中的描述：

> Set 在集合中存储相同类型的不同值，没有定义的顺序。当项目的顺序不重要时，或者当您需要确保元素仅出现一次时，您可以使用集合而不是数组。

Dart 中 Set 类的定义：

```dart
var emptyFruits = Set<String>();
var fruits = Set<String>.from(['apple', 'banana']); // set from Iterable
```

Swift 中的示例：

```swift
var emptyFruits = Set<String>()
var fruits = Set<String>(["apple", "banana"])
```

### Maps / Dictionaries

Swift 文档对 `map/dictionary` 有一个很好的定义：

> 字典存储相同类型的键与集合中相同类型的值之间的关联，而没有特定的排序。每个值都与唯一键相关联，该唯一键充当字典中该值的标识符。

Dart 中的 map 定义如下：

```dart
var namesOfIntegers = Map<Int,String>(); // empty map
var airports = { 'YYZ': 'Toronto Pearson', 'DUB': 'Dublin' }; // map literal
```

Swift 中 `map` 称为字典：

```swift
var namesOfIntegers = [Int: String]() // empty dictionary
var airports = ["YYZ": "Toronto Pearson", "DUB": "Dublin"] // dictionary literal
```

## Nullability & Optionals

在Dart中，任何对象都可以为 `null`。并且尝试访问 `null` 对象的方法或变量会导致空指针异常。这是计算机程序中最常见的错误来源。

从一开始，`Swift` 就多了一个选择，一个内置的语言功能，用于声明对象是否可以有值。看看文档：

> 您可以在可能缺少值的情况下使用 Optional。`Optional` 表示两种可能性：要么存在值，您可以解开可选项以访问该值，或者根本没有值。

与此相反，我们可以使用非 Optional 变量来保证它们始终具有值：

```swift
var x: Int? // optional
var y: Int = 1 // non-optional, must be initialized
```

注意：说 Swift 变量是可选的与 Dart 变量可以为 null 是大致相同。

如果没有对选项的语言级支持，我们只能在运行时检查变量是否为 `null`。

使用 Optional，我们在编译时对这些信息进行编码。我们可以解开 Optional 以安全地检查它们是否包含值：

```swift
func showOptional(x: Int?) {
  // use `guard let` rather than `if let` as best practice
  if let x = x { // unwrap optional
    print(x)
  } else {
    print("no value")
  }
}

showOptional(x: nil) // prints "no value"
showOptional(x: 5) // prints "5"
```

如果我们知道变量必须有值，我们可以使用 `non-optional` 的值：

```swift
func showNonOptional(x: Int) {
  print(x)
}
showNonOptional(x: nil) // [compile error] Nil is not compatible with expected argument type 'Int'
showNonOptional(x: 5) // prints "5"
```

上面的第一个例子在 Dart 中的实现如下：

```dart
void showOptional(int x) {
  if (x != null) {
    print(x);
  } else {
    print('no value');
  }
}
showOptional(null) // prints "no value"
showOptional(5) // prints "5"
```

第二个如下实现：

```dart
void showNonOptional(int x) {
  assert(x != null);
  print(x); 	
}
showNonOptional(null) // [runtime error] Uncaught exception: Assertion failed
showNonOptional(5) // prints "5"
```

有 optional 意味着我们可以在**编译时**而不是在**运行时**捕获错误。及早捕获错误会让代码更安全，错误更少。

Dart 缺乏对 optional 的支持在某种程度上通过使用断言（以及用于命名参数的 [@required 注释](https://pub.dartlang.org/documentation/meta/latest/meta/required-constant.html)）得到缓解。

这些在 `Flutter SDK` 中广泛使用，但会产生额外的样板代码。

## 类

类是用面向对象语言编写程序的主要构建块。

Dart 和 Swift 都支持类，但有一些差异。

### 语法

这里有一个带有 `initializer` 和三个成员变量的 `Swift` 类：

```swift
class Person {
  let name: String
  let age: Int
  let height: Double
  init(name: String, age: Int, height: Double) {
    self.name = name
    self.age = age
    self.height = height
  }
}
```

在 Dart 中：

```dart
class Person {
  Person({this.name, this.age, this.height});
  final String name;
  final int age;
  final double height;
}
```

请注意在 Dart 构造函数中使用的 `this.[propertyName]`。这是用于在构造函数运行之前设置实例成员变量的语法糖。

### 工厂构造函数

在 Dart 中，可以使用工厂构造函数。

> 在实现并不总是创建其类的新实例的构造函数时，请使用 `factory` 关键字。

工厂构造函数的一个实际用例是从 JSON 创建模型类时：

```dart
class Person {
  Person({this.name, this.age, this.height});
  final String name;
  final int age;
  final double height;
  factory Person.fromJSON(Map<dynamic, dynamic> json) {
    String name = json['name'];
    int age = json['age'];
    double height = json['height'];
    return Person(name: name, age: age, height: height);
  }
}
var p = Person.fromJSON({
  'name': 'Andrea',
  'age': 34,
  'height': 1.84,
});
```

## 继承

Swift 使用单继承模型，这意味着任何类只能有一个超类。Swift类可以实现多个接口（也称为协议）。

Dart 类具有基于 `mixin` 的继承。如文档描述：

> 每个对象都是一个类的实例，所有类都来自 `Object`。基于 `Mixin` 的继承意味着虽然每个类（除了Object）只有一个超类，但是类体可以在多个类层次结构中重用。

以下是 Swift 中的单继承：

```swift
class Vehicle {
  let wheelCount: Int
  init(wheelCount: Int) {
    self.wheelCount = wheelCount
  }
}
class Bicycle: Vehicle {
  init() {
    super.init(wheelCount: 2)
  }
}
```

在 Dart 中：

```dart
class Vehicle {
  Vehicle({this.wheelCount});
  final int wheelCount;
}
class Bicycle extends Vehicle {
  Bicycle() : super(wheelCount: 2);
}
```

## 属性

这些在 Dart 中称为实例变量，在 Swift 中只是属性。

在 Swift 中，存储和计算属性之间存在区别：

```swift
class Circle {
  init(radius: Double) {
    self.radius = radius
  }
  let radius: Double // stored property
  var diameter: Double { // read-only computed property
    return radius * 2.0
  }
}
```

在 Dart 中，我们有相同的区分：

```dart
class Circle {
  Circle({this.radius});
  final double radius; // stored property
  double get diameter => radius * 2.0; // computed property
}
```

除了计算属性的 `getter` 之外，我们还可以定义 `setter`。

使用上面的例子，我们可以重写 `diameter` 属性以包含一个 `setter`：

```swift
var diameter: Double { // computed property
  get {
    return radius * 2.0
  }
  set {
    radius = newValue / 2.0
  }
}
```

在 Dart 中，我们可以像这样添加一个单独的 `setter`：

```dart
set diameter(double value) => radius = value / 2.0;
```

### 属性观察者

这是 Swift 的一个特有功能。如文档描述：

> 属性观察者负责观察并响应属性值的变化。每次设置属性值时都会调用属性观察者，即使新值与属性的当前值相同。

这是他们的使用方式：

```swift
var diameter: Double { // read-only computed property
  willSet(newDiameter) {
    print("old value: \(diameter), new value: \(newDiameter)")  
  }
  didSet {
    print("old value: \(oldValue), new value: \(diameter)")  
  }
}
```

## 协议/抽象类

这里我们讨论用于定义方法和属性，而不指定它们的实现方式的结构。这在其他语言中称为接口。

在 Swift 中，接口称为协议。

```swift
protocol Shape {
  func area() -> Double
}
class Square: Shape {
  let side: Double
  init(side: Double) {
    self.side = side
  }
  func area() -> Double {
    return side * side
  }
}
```

Dart有一个类似的结构，称为抽象类。抽象类无法实例化。但是，他们可以定义具有实现的方法。

上面的例子在 Dart 中可以这样写：

```dart
abstract class Shape {
  double area();
}
class Square extends Shape {
  Square({this.side});
  final double side;
  double area() => side * side;
}
```

## Mixins

在 Dart 中，`mixin` 只是一个常规类，可以在多个类层次结构中重用。

以下代码演示了我们使用 `NameExtension mixin` 扩展我们之前定义的 `Person` 类：

```dart
abstract class NameExtension {
  String get name;
  String get uppercaseName => name.toUpperCase();
  String get lowercaseName => name.toLowerCase();
}
class Person with NameExtension {
  Person({this.name, this.age, this.height});
  final String name;
  final int age;
  final double height;	
}
var person = Person(name: 'Andrea', age: 34, height: 1.84);
print(person.uppercaseName); // 'ANDREA'
```

## 扩展

扩展是 Swift 语言的一个特性。如文档描述：

> 扩展为现有的类，结构，枚举或协议类型添加新功能。这包括扩展那些无法访问原始源代码的类型的能力（称为追溯建模）。

在 Dart 中使用 `mixins` 是无法实现这一点的。

借用上面的例子，我们可以像这样扩展 `Person` 类：

```swift
extension Person {
  var uppercaseName: String {
    return name.uppercased()
  }
  var lowercaseName: String {
    return name.lowercased()
  }
}
var person = Person(name: "Andrea", age: 34, height: 1.84)
print(person.uppercaseName) // "ANDREA"
```

扩展的内容比我在这里介绍的要多得多，特别是当它们与协议和泛型一起使用时。

扩展的一个非常常见的用例是为现有类型添加协议一致性。例如，我们可以使用扩展来为现有模型类添加序列化功能。

## 枚举

Dart 对枚举有一些非常基本的支持。

而 Swift 中的枚举非常强大，因为它们支持关联类型：

```swift
enum NetworkResponse {
  case success(body: Data) 
  case failure(error: Error)
}
```

这使得编写这样的逻辑成为可能：

```swift
switch (response) {
  case .success(let data):
    // do something with (non-optional) data
  case .failure(let error):
    // do something with (non-optional) error
}
```

请注意 `data` 和 `error` 参数是如何互斥的。

在 Dart 中，我们无法将其他值与枚举相关联，上面的代码可以按以下方式实现：

```dart
class NetworkResponse {
  NetworkResponse({this.data, this.error})
  // assertion to make data and error mutually exclusive
  : assert(data != null && error == null || data == null && error != null);
  final Uint8List data;
  final String error;
}
var response = NetworkResponse(data: Uint8List(0), error: null);
if (response.data != null) {
  // use data
} else {
  // use error
}
```

几个注意事项：

* 在这里，我们使用断言来弥补我们没有 optional 的事实。
* 编译器无法帮助我们检查所有可能的情况。这是因为我们不使用 `switch` 来处理响应。

总之，Swift 枚举比 Dart 强大且富有表现力。

像 [Dart Sealed Unions](https://github.com/fluttercommunity/dart_sealed_unions) 这样的第三方库提供了类似于 Swift 枚举的功能，可以帮助填补空白。

## 结构体

在 Swift 中，我们可以定义结构和类。

这两种结构都有许多共同点，也有一些不同之处。

主要区别在于：

> 类是引用类型，结构体是值类型

文档中的描述如下：

> 值类型是一种类型，其值在被赋值给变量或常量时被复制，或者在传递给函数时被复制。
> Swift 中所有结构和枚举都是值类型。这意味着您创建的任何结构和枚举实例 - 以及它们所有的值类型的属性 - 在代码中传递时始终会被复制。
> 与值类型不同，引用类型在分配给变量或常量时或者传递给函数时不会被复制。而是使用对同一现有实例的引用。

要了解这意味着什么，请考虑以下示例，其中我们重新使用 Person 类使其变为可变：

```swift
class Person {
  var name: String
  var age: Int
  var height: Double
  init(name: String, age: Int, height: Double) {
    self.name = name
    self.age = age
    self.height = height
  }
}
var a = Person(name: "Andrea", age: 34, height: 1.84)
var b = a
b.age = 35
print(a.age) // prints 35
```

如果我们将 Person 重新定义为 `struct`，我们有：

```swift
struct Person {
  var name: String
  var age: Int
  var height: Double
  init(name: String, age: Int, height: Double) {
    self.name = name
    self.age = age
    self.height = height
  }
}
var a = Person(name: "Andrea", age: 34, height: 1.84)
var b = a
b.age = 35
print(a.age) // prints 34
```

结构体的内容比我在这里介绍的要多得多。

结构体可用于处理 Swift 中的数据和模型，从而产生具有更少错误的强大代码。

## 错误处理

使用 Swift 文档中的定义：

> 错误处理是响应程序中的错误条件并从中恢复的过程。

Dart 和 Swift 都使用 `try/catch` 作为处理错误的技术，但存在一些差异。

在 Dart 中，任何方法都可以抛出任何类型的异常。

```dart
class BankAccount {
  BankAccount({this.balance});
  double balance;
  void withdraw(double amount) {
    if (amount > balance) {
      throw Exception('Insufficient funds');
    }
    balance -= amount;
  }
}
```

可以使用 `try/catch` 块捕获异常：

```swift
var account = BankAccount(balance: 100);
try {
  account.withdraw(50); // ok
  account.withdraw(200); // throws
} catch (e) {
  print(e); // prints 'Exception: Insufficient funds'
}
```

在 Swift 中，我们显式声明方法何时可以抛出异常。这是通过 `throws` 关键字完成的，并且任何错误都必须符合错误协议：

```swift
enum AccountError: Error {
  case insufficientFunds
}
class BankAccount {
  var balance: Double
  init(balance: Double) {
    self.balance = balance
  }
  func withdraw(amount: Double) throws {
    if amount > balance {
      throw AccountError.insufficientFunds
    }
    balance -= amount
  }
}
```

在处理错误时，我们在 `do/catch` 块内使用 `try` 关键字。

```swift
var account = BankAccount(balance: 100)
do {
  try account.withdraw(amount: 50) // ok
  try account.withdraw(amount: 200) // throws
} catch AccountError.insufficientFunds {
  print("Insufficient Funds")
}
```

请注意，当调用抛出异常的方法时，`try` 关键字是如何使用的。

错误本身是强类型的，所以我们可以有多个 `catch` 块来覆盖所有可能的情况。

### try, try?, try!

Swift 提供了一种处理错误的不那么繁琐的方法。

我们可以使用不带 `do/catch` 块的 `try?`。这将会忽略任何异常：

```swift
var account = BankAccount(balance: 100)
try? account.withdraw(amount: 50) // ok
try? account.withdraw(amount: 200) // fails silently
```

或者，如果我们确定某个方法不会抛出异常，我们可以使用 `try!`：

```swift
var account = BankAccount(balance: 100)
try! account.withdraw(amount: 50) // ok
try! account.withdraw(amount: 200) // crash
```

上面的示例将导致程序崩溃。所以，在生产代码中不建议使用 `try!`，它更适合编写测试。

总之，Swift 中错误处理的显式性质在 API 设计中非常有益，因为它可以很容易地知道方法是否可以抛出。

同样，在方法调用时使用 `try` 让我们能关注到可能抛出错误的代码，迫使我们考虑错误情况。

在这方面，错误处理让 Swift 比 Dart 更安全、更可靠。

## 泛型

Swift 文档描述：

> 泛型代码使您能够根据需求编写可以使用任何类型的灵活的可重用的函数和类型。您可以编写避免重复的代码，并以清晰、抽象的方式表达其意图。

两种语言都支持泛型。

泛型的最常见用例之一是集合，例如数组、集合和映射。

我们可以使用它们来定义我们自己的类型。以下是我们如何在 Swift 中定义通用 Stack 类型：

```swift
struct Stack<Element> {
  var items = [Element]()
  mutating func push(_ item: Element) {
    items.append(item)
  }
  mutating func pop() -> Element {
    return items.removeLast()
  }
}
```

类似的，在 Dart 中可以这样写：

```dart
class Stack<Element> {
  var items = <Element>[]
  void push(Element item) {
    items.add(item)
  }
  void pop() -> Element {
    return items.removeLast()
  }
}
```

泛型在 Swift 中非常有用非常强大，它们可用于在协议中定义类型约束和相关类型。

## 访问控制

Swift 文档描述如下：

> 访问控制限制从其他源文件和模块中的代码访问你的代码。此功能可以隐藏代码的实现细节，并指定一个首选接口，通过该接口可以访问和使用该代码。

Swift 有五个访问级别：`open`, `public`, `internal`, `file-private` 和 `private`。

这些关键字用于处理模块和源文件的上下文中。文档描述如下：

> 模块是一个代码分发单元 - 一个框架或应用程序，它作为一个单元构建和发布，可以在另一个模块中使用 Swift 的 `import` 关键字导入。

`open` 和 `public` 访问级别可让代码在模块外部访问。

`private` 和 `file-private` 访问级别可让代码无法在其定义的文件之外访问。

例如：

```swift
public class SomePublicClass {}
internal class SomeInternalClass {}
fileprivate class SomeFilePrivateClass {}
private class SomePrivateClass {}
```

Dart 中的访问级别更简单，仅限于 `public` 和 `private`。文档描述如下：

> 与 Java 不同，Dart 没有关键字 `public`，`protected` 和 `private`。如果标识符以下划线 `_` 开头，则它私有的。

例如：

```dart
class HomePage extends StatefulWidget { // public
  @override
  _HomePageState createState() => _HomePageState();
}
class _HomePageState extends State<HomePage> { ... } // private
```

Dart 和 Swift 中访问控制的设计目标不同。因此，访问级别非常不同。

## 异步编程：Future

异步编程是 Dart 中真正闪耀的地方。

在处理任务时需要某种形式的异步编程，例如：

* 从 Web 下载内容
* 与后端服务通信
* 执行长时间运行的操作

在这些情况下，最好不要阻塞执行的主线程，这可能会使我们的程序卡住。

Dart 文档描述如下：

> 异步操作可让您的程序在等待某个任务完成时去执行其它操作。Dart 使用 Future 对象来表示异步操作的结果。要使用 Future，可以使用 `async/await` 或 `Future API`。

作为一个例子，让我们看看我们如何使用异步编程：

* 使用服务器验证用户
* 存储访问令牌以保护存储
* 获取用户个人资料信息

在 Dart 中，这可以通过结合使用 `Future` 和 `async/await` 来完成：

```dart
Future<UserProfile> getUserProfile(UserCredentials credentials) async {
  final accessToken = await networkService.signIn(credentials);
  await secureStorage.storeToken(accessToken, forUserCredentials: credentials);
  return await networkService.getProfile(accessToken);
}
```

在 Swift 中，不支持 `async/await`，我们只能通过闭包来实现这一点：

```swift
func getUserProfile(credentials: UserCredentials, completion: (_ result: UserProfile) -> Void) {
  networkService.signIn(credentials) { accessToken in
    secureStorage.storeToken(accessToken) {
      networkService.getProfile(accessToken, completion: completion)
    }
  }
}
```

由于嵌套的 `completion` 块，这导致了“**厄运金字塔(pyramid of doom)**”。在这种情况下，错误处理变得非常困难。

在 Dart 中，上面代码中的处理错误只需在代码周围添加一个 `try/catch` 块到 `getUserProfile` 方法即可。

作为参考，有人建议将来向 Swift 中添加 `async/await`。在下面这个 proposal 中有详细描述：

* [Async/Await proposal for Swift](https://gist.github.com/lattner/429b9070918248274f25b714dcfc7619)

在实现之前，开发人员可以使用第三方库，例如 Google 的 `Promises` 库。

## 异步编程：Stream

Dart 将 `Stream` 作为核心库的一部分来实现，但 Swift 没有。

Dart 文档描述如下：

> Stream 是一个异步事件序列。

Stream 是响应式程序的基础，它们在状态管理中发挥着重要作用。

例如，Stream 是搜索内容的绝佳选择，每次用户更新搜索字段中的文本时，都会发出一组新结果。

Stream 不包含在 Swift 核心库中。不过第三方库（如 `RxSwift`）提供了对流的支持。

Stream 是一个广泛的主题，这里不详细讨论。

## 内存管理

Dart 使用高级垃圾回收(garbage collection)方案管理内存。

Swift 通过自动引用计数（ARC）管理内存。

这可以保证良好的性能，因为内存在不再使用时会立即释放。

然而，它确实将部分负担地从编译器转移到开发人员。

在 Swift 中，我们需要考虑对象的生命周期和所有权，并正确使用适当的关键字（`weak`, `strong`, `unowned`）以避免循环引用。

## 编译和执行

首先来看看 `JIT` 和 `AOT` 编译器之间的重要区别：

### JIT

`JIT` 编译器在程序执行期间运行，也就是**即时编译**。

`JIT` 编译器通常与动态语言一起使用，其中类型不是提前确定的。`JIT` 程序通过解释器或虚拟机（VM）运行。

### AOT

在运行之前，AOT 编译器在创建程序期间运行。

AOT 编译器通常与静态语言一起使用，后者知道数据的类型。AOT 程序被编译为本机机器代码，在运行时由硬件直接执行。

下面引用了 `Wm Leler` 的这篇文章：

> 当在开发期间完成 AOT 编译时，它总是导致更长的开发周期（对程序进行更改和能够执行程序以查看更改结果之间的时间）。 但 AOT 编译让程序的运行更可预测，而不会在运行时暂停进行分析和编译。AOT 编译的程序也可以快速启动（因为它们已经被编译）。
> 相反，JIT 编译提供了更快的开发周期，但可能导致执行速度变慢或更加笨拙。特别是，JIT 编译器的启动时间较慢，因为当程序开始运行时，JIT 编译器必须在执行代码之前进行分析和编译。研究表明，如果开始执行的时间超过几秒钟，很多人都会放弃。

作为一种静态语言，Swift 是提前编译的。

Dart 则同时支持 `AOT` 和 `JIT`。与 Flutter 一起使用时，这提供了显著的优势。看看下面的描述：

> 在开发过程中使用 JIT 编译，使用更快的编译器。然后，当应用程序准备好发布时，将它编译为 AOT。因此，借助先进的工具和编译器，Dart 可以提供两全其美的优势：极快的开发周期，快速的执行和启动时间。 - Wm Leler

使用 Dart，可以两全其美。

Swift 有 AOT 编译的主要缺点。即编译时间随着代码库的大小而增加。

对于中型应用程序（10K 到 100K 行之间），编译应用程序很容易花费几分钟。

对于 Flutter 应用程序来说并非如此，无论代码库的大小如何，我们都会不断进行亚秒级热加载。

## 其它未涵盖功能

本文未涵盖以下功能，因为它们在 Dart 和 Swift 中非常相似：

* 运算符
* 字符串
* Swift 中的可选链（在 Dart 中称为条件成员访问）。

### 并发

* 并发编程在 Dart 中通过 `isolate` 来提供。
* Swift 使用 `Grand Central Dispatch（GCD）`和分发队列。

## Dart 中缺失的那些我喜欢的 Swift 特性

* `Structs`
* 带关联类型的 Enums
* `Optionals`

## Swift 中缺失的那些我喜欢的 Dart 特性

* JIT 编译器
* Future 和 `await/async`
* Stream 和 `yield/async*`

## 结论

Dart 和 Swift 都是出色的语言，非常适合构建现代移动应用程序及其他应用程序。

这两种语言都有自己独特的优点。

在比较过移动应用程序开发和两种语言的工具时，我觉得 Dart 占了上风。这是由于 JIT 编译器，它是 Flutter 中有状态热加载的基础。

在构建应用程序时，热加载可以大大提高生产力，因为它可以将开发周期从几秒或几分钟加速到不到一秒钟。

开发时间比计算时间更耗费资源。

因此，优化开发人员的时间是一个非常明智的举措。

另一方面，我觉得 Swift 有一个非常强大的类型系统。类型安全性融入 Swift 的所有语言功能，能更自然地开发出健壮的程序。

一旦我们抛开个人偏好，编程语言就是工具。作为开发人员，我们的任务是为工作选择最合适的工具。

无论如何，我们可以希望两种语言在发展过程中互相借鉴最好的想法。


