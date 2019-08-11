

Dart 团队于上周发布了 Dart 2.3.0 版本，后续又发布了几个小版本迭代，这些版本的重点是新的“UI-as-code”语言功能，增强了集合的表现力。这里我们来简略看看这些新特性。

## 语言

集合在 2.3.0 中得到进一步增强。

### 扩展运算符

在集合字面量内，在表达式前面加上 `...` 可以扩展表达式的结果（类似于 Javascript ES6 中的扩展运算符），并将结果的所有元素直接插入到新的集合中。而如果表达式结果可能为 null，则可以使用 `...?`

```dart
// before 2.3.0
CupertinoPageScaffold(
  child: ListView(children: [
    Tab2Header()
  ]..addAll(buildTab2Conversation())
    ..add(buildFooter())),
);

// 2.3.0
CupertinoPageScaffold(
  child: ListView(children: [
    Tab2Header(),
    ...buildTab2Conversation(),
    buildFooter()
  ]),
);
```

### Collection if

可以在集合字面量中直接使用 if 表达式来根据条件将元素插入集合中。同时 if 表达式还可以结合上述的扩展运算符一起使用。

```dart
// before 2.3.0
Widget build(BuildContext context) {
  var children = [
    IconButton(icon: Icon(Icons.menu)),
    Expanded(child: title)
  ];

  if (isAndroid) {
    children.add(IconButton(icon: Icon(Icons.search)));
  }

  return Row(children: children);
}

// 2.3.0
Widget build(BuildContext context) {
  return Row(
    children: [
      IconButton(icon: Icon(Icons.menu)),
      Expanded(child: title),
      if (isAndroid)
        IconButton(icon: Icon(Icons.search)),
    ],
  );
}
```

### Collection for

可以在集合字面量中使用 for 表达式，将循环迭代生成的元素直接插入到集合中。

```dart
// before 2.3.0
var command = [
  engineDartPath,
  frontendServer,
  ...fileSystemRoots.map((root) => "--filesystem-root=$root"),
  ...entryPoints
      .where((entryPoint) => fileExists("lib/$entryPoint.json"))
      .map((entryPoint) => "lib/$entryPoint"),
  mainPath
];

// 2.3.0
var command = [
  engineDartPath,
  frontendServer,
  for (var root in fileSystemRoots) "--filesystem-root=$root",
  for (var entryPoint in entryPoints)
    if (fileExists("lib/$entryPoint.json")) "lib/$entryPoint",
  mainPath
];
```

需要注意的两点是

* 这些新特性可以组合使用
* 当前不支持在 const 集合字面量中使用这些特性

## 核心库

### dart:isolate

* Isolate 新增 debugName 属性
* Isolate.spawn 和 Isolate.spawnUri 中新增 debugName 可选属性

### dart:core

* RegExp 模式现在可以使用 lookbehind 断言。
* RegExp 模式现在可以使用命名捕获组和命名反向引用。

## Dart VM

* VM 服务现在默认需要身份验证代码。可以通过提供 `--disable-service-auth-codes` 标志来禁用此行为。
* 已删除对已弃用标志'-c'和'--checked'的支持。

## 对 Web 的支持

### dart2js

`dump-info` 中添加了二进制格式。旧的 JSON 格式仍然可用并默认提供，但准备弃用。新的二进制格式更紧凑，生成成本更低。在测试的一些大型应用程序上，序列化速度提高了4倍，内存使用量减少了6倍。

要使用二进制格式，请使用 `--dump-info=binary`，而不是 `--dump-info`。

## 工具

### dartfmt

* 调整 set 字面量格式，以和其他集合字面量保持一致。
* 添加对“UI as code”功能的支持。
* 在断言中正确格式化尾随逗号。
* 改进参数列表中相邻字符串的缩进。

### Linter

Linter 更新到 `0.1.86`，其中包括以下更改：

* 添加了以下 `lints：prefer_inlined_adds`，`prefer_for_elements_to_map_fromIterable`，`prefer_if_elements_to_conditional_expressions`，`diagnostic_describe_all_properties`。
* 更新了 `file_names` 以跳过 `prefixed-extension` Dart文件（.css.dart，.g.dart等）。
* 修复了 `unwanted_parenthesis` 中的误报。

### Pub 客户端

* 添加了一个 CHANGELOG 验证器，如果在不提及当前版本的情况下 `pub publish`，则会给出警告。
* 移除了在进行 pub publish 时对库名的验证。
* 从自定义 pub URL 添加了对 `pub global activate` 的支持。
* 添加了子命令：`pub logout`，以退出当前会话。


