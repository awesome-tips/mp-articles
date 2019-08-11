直入主题吧。在本教程中，我们将使用 Flutter 来开发适用于 Android 和 iOS 的谷歌翻译应用程序。下面是程序的基本界面。

![](https://cdn-images-1.medium.com/max/800/1*cubKKFKieuXjxPRPFe1M5Q.png)

## 创建工程

要创建项目，我们必须运行 Android Studio 并单击 `Start a new Flutter project`，然后选择 `Flutt2
er application`。我们在表单中填写项目名称，`flutter SDK` 的路径，项目位置和描述，如下所示：

![](https://cdn-images-1.medium.com/max/800/1*5FSaXaq6u2WMV4OSbJnaKA.png)

## 清空代码

创建项目后，我们首先清理 `main.dart` 文件中生成的代码。原本有一些代码，但我们希望尽可能简单。

```dart
import 'package:flutter/material.dart';
void main() => runApp(MyApp());
class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Google Translate',
      theme: ThemeData(
        primarySwatch: Colors.blue,
        primaryColor: Colors.blue[600],
      ),
      home: Scaffold(
        appBar: AppBar(
          title: Text('Google Translate'),
          elevation: 0.0,
        ),
        body: Center(
          child: Text("We are going to translate everything !"),
        ),
      ),
    );
  }
}
```

这段代码更干净且更容易理解。`MaterialApp` 类为我们的应用程序定义了设计外观。然后在 `ThemeData` 类中添加蓝色主题，以便更接近真实的 `Google Translate` 应用程序。`Scaffold` 创建应用程序的全局结构，它包含一个 `AppBar` 和一个 `body`。`body` 部分是我们在 `AppBar` 下显示应用程序内容的地方。

## 组织代码

![](https://cdn-images-1.medium.com/max/800/1*5T09ck3kPrUJmtaqf5HOXg.png)

我的代码以这种方式组织，包含 components、screens、models 和 services。这只是一种组织方式，还有其它的方式，就看自己是怎么考虑。test 文件夹也以相同方式组织。

## 创建第一个组件

我们将创建第一个组件，来显示我们输入的文本的语言，以及要翻译成的语言。

```dart
import 'package:flutter/material.dart';
class ChooseLanguage extends StatefulWidget {
  ChooseLanguage({Key key}) : super(key: key);
  @override
  _ChooseLanguageState createState() => _ChooseLanguageState();
}
class _ChooseLanguageState extends State<ChooseLanguage> {
  String _firstLanguage = "English";
  String _secondLanguage = "French";
  @override
  Widget build(BuildContext context) {
    return Container();
  }
}
```

这段代码创建了一个 `ChooseLanguage` 组件，该组件定义了两个变量来用于表示选择的语言。

```dart
@override
Widget build(BuildContext context) {
  return Container(
    height: 55.0,
    decoration: BoxDecoration(
      color: Colors.white,
      border: Border(
        bottom: BorderSide(
          width: 0.5,
          color: Colors.grey[500],
        ),
      ),
    ),
  );
}
```

我们为容器定义特定高度和装饰以对组件进行样式设置。在 `BoxDecoration` 中，我们定义背景颜色和容器边框，以在组件底部显示一条分隔线。

接着我们将添加一个 `Row` 元素，它将帮助我们在一行上显示我们的 `widgets`。然后，我们可以确定如何对齐此行中的元素。实际上，row 有两个轴，主轴与 row 方向相同，横轴与其方向交叉。操纵这两个轴，我们可以很容易地以我们想要的方式显示我们的组件。以下模式显示了 `widget` 如何使用不同属性进行显示。

![](https://cdn-images-1.medium.com/max/800/1*cjbJWO4e5tplEOAgHilu0A@2x.png)

在 Flutter 中，可以很容易重建这个显示，我们所需要的只是一个 `Row` 元素，并定义主轴和横轴对齐方式。`Expanded` 元素将被添加到 `Row` 的子 `widget` 中。

```dart
@override
Widget build(BuildContext context) {
  return Container(
    ...
    child: Row(
      mainAxisAlignment: MainAxisAlignment.start,
      crossAxisAlignment: CrossAxisAlignment.center,
      children: <Widget>[
        ...
      ],
    ),
  );
}
```

现在是时候在我们的 `Row` 中添加 `widget` 了。我们需要在 `InkWell` 组件内简单地创建一个 `Text` 组件。`InkWell` 组件将在文本周围创建一个可单击的区域，并显示 `splash` 效果。

```dart
child: Row(
  mainAxisAlignment: MainAxisAlignment.start,
  crossAxisAlignment: CrossAxisAlignment.center,
  children: <Widget>[
    Expanded(
      child: Material(
        color: Colors.white,
        child: InkWell(
          onTap: () {},
          child: Center(
            child: Text(
              this._firstLanguage,
              style: TextStyle(
                color: Colors.blue[600],
                fontSize: 15.0,
              ),
            ),
          ),
        ),
      ),
    ),
    ...
    Expanded(
      child: Material(
        color: Colors.white,
        child: InkWell(
          onTap: () {},
          child: Center(
            child: Text(
              this._secondLanguage,
              style: TextStyle(
                color: Colors.blue[600],
                fontSize: 15.0,
              ),
            ),
          ),
        ),
      ),
    ),
  ],
),
```

我们使用 `Expanded` 元素来构建包含白色背景颜色的 `Text`。`InkWell` 元素需要子树中的 `Material` 来显示 splash 效果。`onTap` 事件暂时不使用，我们稍后会在点击 `Text` 时更改语言。我们将 `Text` 显示在 `Center` 中，因为 `Expanded` 将占用所有空间，而我们希望将文本居中。最后，我们添加了变量 `firstLanguage` 和 `secondLanguage` 来显示语言。

最后一步是创建一个图标按钮，以像谷歌翻译应用程序中那样切换语言。这很简单，我们将创建一个带有白色背景的 `Material`，来包含我们的 `IconButton`。我们在 Flutter 库中已经存在的图标列表中选择我们的图标。我选择使用 `Icons.compare_arrows`，它与实际应用程序中的不完全相同。

```dart
<Widget>[
  Expanded(
    ...
  ),
  Material(
    color: Colors.white,
    child: IconButton(
      icon: Icon(
        Icons.compare_arrows,
        color: Colors.grey[700],
      ),
      onPressed: () {},
    ),
  ),
  Expanded(
    ...
  ),
]
```

现在我们的组件终于完成了，我们需要将它添加到页面中。然后我们应该能看到下面的效果。

```dart
import 'package:flutter/material.dart';

import '../components/choose-language.dart';

class HomePage extends StatefulWidget {
  HomePage({Key key, this.title}) : super(key: key);

  final String title;

  @override
  _HomePageState createState() => _HomePageState();
}

class _HomePageState extends State<HomePage> {

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text(widget.title),
        elevation: 0.0,
      ),
      body: Column(
        children: <Widget>[
          ChooseLanguage(),
      ),
    );
  }
}
```

![](https://cdn-images-1.medium.com/max/800/1*E4G-AEjkGpVYHONtQUN5nA.gif)

## 翻译操作的 UI

我们现在需要创建组件来选择我们想要翻译的文本。在 Google 应用程序中，我们必须输入文本或使用其它的方式，例如拍照。我们遵循与 iOS 应用程序相同的设计。因此，我们将在本教程的下一部分中创建一个可点击的 widget。

![](https://cdn-images-1.medium.com/max/800/1*QRxQLSiX0SVo5CxvW6SJAQ.png)

我们将使用 `Column` 和 `Row` 创建具有此结构的组件以显示我们的 UI。`Column` 的工作方式与 `Row` 的工作方式相同，唯一的区别是它显示元素的方向。我们现在将在一个新组件中编写我们想要实现的结构。

```dart
import 'package:flutter/material.dart';

class TranslateText extends StatefulWidget {
  TranslateText({Key key}) : super(key: key);

  @override
  _TranslateTextState createState() => _TranslateTextState();
}

class _TranslateTextState extends State<TranslateText> {
  @override
  Widget build(BuildContext context) {
    return Card(
      color: Colors.white,
      margin: EdgeInsets.all(0.0),
      elevation: 2.0,
      child: Container(
        height: 150.0,
        child: Column(
          mainAxisAlignment: MainAxisAlignment.spaceBetween,
          crossAxisAlignment: CrossAxisAlignment.start,
          children: <Widget>[
            Expanded(...),
            Row(
              mainAxisAlignment: MainAxisAlignment.spaceBetween,
              children: <Widget>[
                Material(
                  color: Colors.white,
                  child: Column(
                     children: <Widget>[...],
                   ),
                ),
              ],
            ),
          ],
        ),
      ),
    );
  }
}
```

我们还为 `Column` 和 `Row` 定义了主轴和交叉轴的对齐方式。在这里，我们选择了 `MainAxisAlignment.spaceBetween`，因为我们希望在可点击的图标之间留出一些空间。

由于基本结构已完成，现在可以专注于输入部分。输入部分不是我们将在其中输入文本的 widget。在实际的应用中，当我们点击这部分时，会出现一个输入框来输入文本。

```dart
class _TranslateTextState extends State<TranslateText> {
  @override
  Widget build(BuildContext context) {
    return Card(
      ...,
      child: Container(
        height: 150.0,
        child: Column(
          ...,
          children: <Widget>[
            Expanded(
              child: InkWell(
                onTap: () {},
                child: Container(
                  width: double.infinity,
                  padding: EdgeInsets.only(
                    left: 16.0, 
                    right: 16.0, 
                    top: 16.0
                  ),
                  child: Text(
                    "Enter text",
                    style: TextStyle(
                      color: Colors.grey[700],
                    ),
                  ),
                ),
              ),
            ),
            Row(...),
          ],
        ),
      ),
    );
  }
}
```

这段代码与我们创建的第一个组件非常相似。是一个简单的 `InkWell` 内部包含一个 `Text`。我们将保留 `onTap` 函数，就像我们在创建的第一个组件中所做的那样。我们现在要在 `Row` 中创建可点击的图标。

我们将使用相同的代码来显示带有描述性文本的 4 个图标。我们可以在 Row 中编写相同 4 份代码，但是如果我们这样做，就会有很多重复代码。

使用此方法，在更改代码必须重复修改很多处。最好的解决方案是创建另一个组件，我们将使用不同的参数调用四次。

```dart
import 'package:flutter/material.dart';

class ActionButton extends StatefulWidget {
  ActionButton({Key key, this.icon, this.text, this.imageIcon}) : super(key: key);

  final IconData icon;
  final AssetImage imageIcon;
  final String text;

  @override
  _ActionButtonState createState() => _ActionButtonState();
}

class _ActionButtonState extends State<ActionButton> {

  @override
  Widget build(BuildContext context) {
    return Material(
      color: Colors.white,
      child: FlatButton(
        padding: EdgeInsets.only(
          left: 8.0,
          right: 8.0,
          top: 2.0,
          bottom: 2.0,
        ),
        onPressed: () {},
        child: Column(
          children: <Widget>[...],
        ),
      ),
    );
  }
}
```

**创建可点击图标**

我们使用了新的组件 `ActionButton`，与其他组件不同，我们添加了参数 icon，imageIcon 和 text。 我使用的一些图标不在谷歌的库中，因此我创建了自己的图标。

这就是我区分图标和 `ImageIcon` 的原因。我们将创建一个函数来显示 `IconData` 或 `AssetImage` 中的图标。

```dart
Widget _displayIcon() {
  if (this.widget.icon != null) {
    return Icon(
      this.widget.icon,
      size: 23.0,
      color: Colors.blue[800],
    );
  } else if (this.widget.imageIcon != null) {
    return ImageIcon(
      this.widget.imageIcon,
      size: 23.0,
      color: Colors.blue[800],
    );
  } else {
    return Container();
  }
}
```

此函数检查 `icon` 或 `imageIcon` 变量是否为 null，当其中一个变量不为 null 时，我们将使用右侧组件显示图像。事实上，我们通过 `Icon` 组件显示 `IconData`。 `AssetImage` 基于 `ImageIcon` 组件。最后如果两者都为 null，则显示一个空容器。

```dart
@override
Widget build(BuildContext context) {
  return Material(
    color: Colors.white,
    child: FlatButton(
      ...,
      child: Column(
        children: <Widget>[
          _displayIcon(),
          Text(
            this.widget.text,
            style: TextStyle(fontSize: 12),
          ),
        ],
      ),
    ),
  );
}
```

我们在 `Column` 中添加了 `_displayIcon` 函数。它将使用我们要传递给组件的图标或图像图标中显示正确的 widget。我们现在可以在之前创建的 `TranslateText` 中调用我们的组件了。

```dart
import 'package:flutter/material.dart';

import 'ActionButton.dart';

class _TranslateTextState extends State<TranslateText> {
  @override
  Widget build(BuildContext context) {
    return Card(
      ...,
      child: Container(
        height: 150.0,
        child: Column(
          ...,
          children: <Widget>[
            Expanded(
              ...,
            ),
            Row(
              mainAxisAlignment: MainAxisAlignment.spaceBetween,
              children: <Widget>[
                ActionButton(
                  icon: Icons.camera_alt,
                  text: "Camera",
                ),
                ActionButton(
                  imageIcon: AssetImage("assets/pen.png"),
                  text: "Handwriting",
                ),
                ActionButton(
                  imageIcon: AssetImage("assets/conversation.png"),
                  text: "Conversation",
                ),
                ActionButton(
                  icon: Icons.keyboard_voice,
                  text: "Voice",
                ),
              ],
            ),
          ],
        ),
      ),
    );
  }
}
```

我们导入组件，使用不同的参数调用 ActionButton 4 次，以便为每个 `ActionButton` 提供唯一的渲染。

**添加图片资源(可选)**

我在我们刚制作的一些按钮中添加了自己的图像，但我们需要更改根文件夹中的文件 `pubspec.yaml` 来引入这些图像。

```yaml
flutter:

  # The following line ensures that the Material Icons font is
  # included with your application, so that you can use the icons in
  # the material Icons class.
  uses-material-design: true

  # To add assets to your application, add an assets section, like this:
  assets:
    - assets/
```

我在 `pubspec.yaml` 文件中添加了 `- assets /`，并在我的图像的根文件夹中创建了一个 `assets` 文件夹。完成此操作后，可能需要重新运行该应用程序。最后，你可以像我之前那样调用你的图像，例如：`AssetImage("assets/pen.png")`。

如果您想了解有关 assert 的更多信息以及我们如何使用它们，请您阅读 Flutter 文档。

**看看成果**

第二个组件完成后，应用程序的 UI 现在几乎完成了。

![](https://cdn-images-1.medium.com/max/800/1*IwaFjrzRdb-LUWBiZMyjEA.gif)

## 显示最近翻译的列表

我们需要再创建一个新组件来显示最近的翻译列表。仍然使用相同的原理，我们将开发一个组合行和列的 widget。`List` 可以像 Column 一样显示我们的项目，不过更加自动化，而不是添加 n 次组件。此外，可以在 List 上滑动。

![](https://cdn-images-1.medium.com/max/800/1*V6VRI81NrmF3bgd0zmzJ6g.png)

首先，我定义了一个 `Translate` 类，它由我们在列表中显示项目所需的元素组成。

```dart
class Translate {
  String text;
  String translatedText;
  bool isStarred;

  Translate(String text, String translated, bool isStarred) {
    this.text = text;
    this.translatedText = translated;
    this.isStarred = isStarred;
  }
}
```

然后，我们可以创建一个名为 `ListTranslate` 的新组件。 我们在 `itemCount` 中定义 `ListView` 将显示的行数。在 `itemBuilder` 中，我们使用数组列表显示行。

```dart
import 'package:flutter/material.dart';
import '../models/translate.dart';

class ListTranslate extends StatefulWidget {
  ListTranslate({Key key}) : super(key: key);

  @override
  _ListTranslateState createState() => _ListTranslateState();
}

class _ListTranslateState extends State<ListTranslate> {
  List<Translate> _items = [];
  Widget _displayCard(int index) {
    return Card(
      child: Container(
      ),
    );
  }
  @override
  Widget build(BuildContext context) {
    return Expanded(
      child: ListView.builder(
        itemCount: _items.length,
        itemBuilder: (BuildContext ctxt, int index) {
          return _displayCard(index);
        },
      ),
    );
  }
}
```

下一步是创建列表的项目，就像我们之前制作的草图一样。为此，我们将使用 Row 和 Column，就像我们为最后的组件所做的那样。不过，我们将添加一些样式，如边框半径，边距和填充，让它更好看。

```dart
Widget _displayCard(int index) {
  return Card(
    shape: RoundedRectangleBorder(
      borderRadius: BorderRadius.all(Radius.circular(0.0)),
    ),
    margin: EdgeInsets.only(left: 8.0, right: 8.0, top: 0.5),
    child: Container(
      height: 80.0,
      padding: EdgeInsets.only(left: 16.0, top: 16.0, bottom: 16.0),
      child: Row(
        mainAxisAlignment: MainAxisAlignment.spaceBetween,
        children: <Widget>[
          Flexible(
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              mainAxisAlignment: MainAxisAlignment.spaceAround,
              children: <Widget>[
                Text(
                  _items[index].text,
                  style: TextStyle(
                    fontWeight: FontWeight.w600,
                  ),
                  maxLines: 1,
                  overflow: TextOverflow.ellipsis,
                ),
                Text(
                  _items[index].translatedText,
                  style: TextStyle(
                    fontWeight: FontWeight.w400,
                  ),
                  maxLines: 1,
                  overflow: TextOverflow.ellipsis,
                ),
              ],
            ),
          ),
          IconButton(
            onPressed: () {},
            icon: Icon(
              Icons.star_border,
              size: 23.0,
              color: Colors.grey[700],
            ),
          ),
        ],
      ),
    ),
  );
}
```

这将显示列表中的每一行，以展示搜索的信息。`Flexible` 小部件将其高度和宽度扩展到最大值。由于我们在 `Text` 中添加了 `maxLine` 和 `overflow`，如果我们的文本太长，则会被截断。

```dart
IconButton(
  onPressed: () {},
  icon: Icon(
    _items[index].isStarred ? Icons.star : Icons.star_border,
    size: 23.0,
    color: _items[index].isStarred ? Colors.blue[600] : Colors.grey[700],
  ),
),
```

如果行的元素已加星标，则我们想要更改颜色和图标。我们使用三元运算符直接从列表信息中更改它。

```dart
List<Translate> _items = [
  Translate(
    "yellowish",
    "jaunâtre",
    true,
  ),
  ...,
];
```

我们暂时使用一些信息填充列表项，以查看它的外观。现在，设计终于完成了，我们可以观察我们在整个教程期间所做的工作。

![](https://cdn-images-1.medium.com/max/800/1*2dxZ6ziTaO4xeIT1vlYHJQ.gif)

## 结论

这篇教程展示了使用 Flutter 为 Android 和 iOS 创建移动应用程序有多简单快捷。我们还发现一些部分看起来像 Web 开发中 `flexbox`。即使这是一个移动 App 的开发，Web 开发人员也可以快速地了解所做的事情。我们目前只做了一个没有任何功能的基本应用程序，但是编写下一个部分并不困难！



