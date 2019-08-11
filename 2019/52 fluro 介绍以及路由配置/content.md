## 前言

无论在 react 还是在 vue 中，都会有路由的配置，当然，他们一般都是单页面应用，但是对于开发而言，将路由统一管理，也无非是一个非常简洁方便的形式，甚至在不使用 `fluro` 的时候，Material 也提供了 `onGenerateRoute` 来配置路由，只不过那样会使入口页面非常的臃杂。所以这里的路由管理，我们使用 fluro。

## Fluro

我们在 `flutter package` 中搜索 fluro，然后查看他的包详情

笔者喜欢直接转到 GitHub 去查看相关文档，因为里面会有 example 可以查看

在 example 中，我们可以看到他的使用规范为在 lib 目录下新建一个 config 目录，在 `application.dart` 文件中，配置总 Router，同级目录下注册 route，并且注册相关 handle 函数。在 example 下，我们可以查看到 url 传参，包括一般字段、link 以及函数的传递。

具体的文档，大家可以自行到 GitHub 或者 flutter package 中自行查看。这里我们直接看在项目中的使用

## 配置项目 Route

### 项目注入新 package

```sh
pubspec.yaml
  dependencies:
    flutter:
      sdk: flutter
  
    # The following adds the Cupertino Icons font to your application.
    # Use with the CupertinoIcons class for iOS style icons.
    cupertino_icons: ^0.1.2
    dio: ^1.0.6
    fluro: "^1.3.7"
```

添加 fluro 依赖。

### 修改文件的目录名

观察 flutter 官网 app 或者 example 中，官方的命名规范，文件名都是以小写字母以及下划线组成，迎合官方标准，我们将项目中，我们自己的文件，曾经的驼峰命名法全部修改为小写字母+下划线的标准

![1](http://)

### 配置路由

这里我们在 lib 目录下新建一个 routers 目录，仿着官方 example 的样子，配置我们的路由

**lib/routers/application.dart**

```dart
  import 'package:fluro/fluro.dart';
  
  class Application{
    static Router router;
  }
```

application 中我们就注册一个总的 Router，方便后面给 Material 中 onGenerateRoute 注册使用。

**lib/routers/router_handler.dart**

```dart
  import 'package:flutter/material.dart';
  import 'package:fluro/fluro.dart';
  import '../pages/article_detail.dart';
  
  Handler articleDetailHandler = Handler(
      handlerFunc: (
        BuildContext context, Map<String, List<String>> params) {
          String articleId = params['id']?.first;
          String title = params['title']?.first;
          print('index>,articleDetail id is $articleId');
          return ArticleDetail(articleId, title);
        }
  );
```

handle 就是对于路由的处理函数，这里我们先注册一个对于详情页的处理函数，handlerFunc 里面的 params 就是我们的 url 的查询参数。通过 Dart 的 `?.` 运算符可以“安全”的获取其参数值。最后 return 我们需要跳转的页面。

对于我们不使用 fluro 的情况下 ，跳转页面就是 通过 Material 的 Navigator 跳转的，而传参呢，也就非常的代码"语义化"了,通过实例化对象而已。

```dart
  Navigator.of(context).push(new MaterialPageRoute(builder: 
      (BuildContext context) => new SidebarPage('First Page')));    //在new方法时调用控件的构造函数传入参数值
```

**lib/routers/routes.dart**

```dart
  import './router_handler.dart';
  import 'package:fluro/fluro.dart';
  import 'package:flutter/material.dart';
  
  class Routes {
    static String root = '/';
    static String articleDetail = "/detail";
  
    static void configureRoutes(Router router) {
      router.notFoundHandler = new Handler(
          handlerFunc: (BuildContext context, Map<String, List<String>> params) {
        print("ROUTE WAS NOT FOUND !!!");
      });
  
      router.define(articleDetail, handler: articleDetailHandler);
    }
  }
```

routes 页面主要是讲路由以及 handle 函数组合起来，也是我们页面路由配置的入口文件，如上，我们暂时，只配置了 notFoundHandler 以及 detail 的页面。

**lib/pages/my_app.dart**

回到我们的项目入口文件，将我们写好的路由配置注册进去。

首先引入我们需要的配置文件

```dart
  import '../routers/routes.dart';
  import '../routers/application.dart';
```

在构造函数中去初始化我们的 Routers

```dart
    final router = new Router();
    Routes.configureRoutes(router);
    Application.router = router;
```

最后在我们的 MaterialApp 中的 onGenerateRoute 中注入进入即可

```dart
   onGenerateRoute: Application.router.generator,
```

### 使用Router

首先，我们需要先写好我们的路由跳转目标页面，其实这个页面应该之前就写好，不然配置路由的时候怎么实例化呢是吧，莫方，现在补上也是么得问题的

**lib/pages/article_detail.dart**

```dart
  import 'package:flutter/material.dart';
  
  class ArticleDetail extends StatelessWidget {
    final String articleId;
    final String title;
  
    ArticleDetail(@required this.articleId, @required this.title);
  
    @override
    Widget build(BuildContext context) {
      return Scaffold(
        appBar: AppBar(
          title: Text(title),
        ),
        body: Center(
          child: Text("这篇文章的id是$articleId"),
        ),
      );
    }
  }
```

代码不做太多解释了，比较简单。接下来我们看下我们如何使用路由吧

跳转 detail 页面，当然是在点击首页 list cell
的时候进行跳转，所以这里，我们需要给每一个 cell 添加一个点击监听

**lib/wudgets/index_list_cell.dart**

首先引入我们的application路由，以及dart的core包，因为涉及到url的传递，所以我们需要进行一次加密，否则会报错。

```dart
  import '../routers/application.dart';
  import 'dart:core'; 
```

由于需要加入点击监听，所以这里我们使用 InkWell widget 来包装下，关于 InkWell的更多介绍，大家可以查看相关文档:文档地址

```dart
    InkWell(
      onTap: () {
        print('跳转到详情页');
        Application.router.navigateTo(context, "/detail?id=${Uri.encodeComponent(cellInfo.detailUrl)}&title=${Uri.encodeComponent(cellInfo.title)}");
      },
      child:...
    )
```

对的，组合新的一行代码就是`Application.router.navigateTo(context, "/detail?id=${Uri.encodeComponent(cellInfo.detailUrl)}&title=${Uri.encodeComponent(cellInfo.title)}");` 因为涉及中文以及 url 所以这里我们进行了 `Uri.encodeComponent`

最终，我们即可看到我们的app已经可以跳转并且传参啦。

![2](http://)

**完整代码地址** https://github.com/Nealyang/flutter/blob/9080a67ff7/flutter_juejin/lib/widgets/index_list_cell.dart?1541387732552

## 总结

如上我们完成了页面路由的配置、跳转和传参，以及命名规范。至此，你应该学会

* 查找 Flutter package
* 路由配置以及 Material 的路由配置
* 不配置路由情况的跳转和传参
* 官方 demo 命名规范（视团队规范而定）

