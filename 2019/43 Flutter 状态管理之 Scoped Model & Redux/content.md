## 前言

可能作为一个前端，在学习 Flutter 的过程中，总感觉非常非常相似 React Native，甚至于，其中还是有state的概念 `setState`，所以在 Flutter 中，也当然会存在非常多的解决方案，比如 redux 、RxDart 还有 Scoped Model等解决方案。今天，我们主要介绍下常用的两种 State 管理解决方案：redux、scoped model。

## Scoped Model

### 介绍

Scoped Model 是 package 上 Dart 的一个第三方库`scoped_model`[1]。Scoped Model 主要是通过数据 model 的概念来实现数据传递，表现上类似于 react 中 context 的概念。它提供了让子代widget轻松获取父级数据model的功能。

从官网中的介绍可以了解到，它直接来自于Google正在开发的新系统Fuchsia核心 Widgets 中对 Model 类的简单提取，作为独立使用的独立 Flutter 插件发布。

在直接上手之前，我们先着重说一下 Scoped Model 中几个重要的概念

* Model 类，通过继承 Model 类来创建自己的数据 model，例如 SearchModel 或者 UserModel ，并且还可以监听 数据model的变化
* ScopedModelDescendant widget , 如果你需要传递数据 model 到很深层级里面的 widget ，那么你就需要用 ScopedModel 来包裹 Model，这样的话，后面所有的子widget 都可以使用该数据 model 了（是不是更有一种 context 的感觉）
* ScopedModelDescendant widget ，使用此 widget 可以在 widget tree 中找到相应的 Scope的Model ，当 数据 model 发生变化的时候，该 widget 会重新构建

当然，在 Scoped Model 的文档中，也介绍了一些 实现原理

* Model类实现了Listenable接口
* AnimationController和TextEditingController也是Listenables
* 使用InheritedWidget将数据 model 传递到Widget树。 重建 InheritedWidget 时，它将手动重建依赖于其数据的所有Widgets。 无需管理订阅！
* 它使用 AnimatedBuilder Widget来监听Model并在模型更改时重建InheritedWidget

### 实操 Demo

demo 地址[2]

![1](http://)

从 gif 上可以看到咱们的需求非常的简单，就是在当前页面更新了count后，在第二个页面也能够传递过去。当然，`new ResultPage(count:count)` 就没意思啦~ 咱不讨论哈

**新建数据 model**

```dart
// lib/model/counter_model.dart

import 'package:scoped_model/scoped_model.dart';
  
class CounterModel extends Model {
	int _counter = 0;
	  
	int get counter => _counter;
	  
	void increment() {
	  
	  _counter++;
	  
	  // 通知所有的 listener
	  notifyListeners();
	}
}
```

* 这一步非常的简单，新建一个类去继承 Model
* 里面定义了一个 get方法，以便于后面取数据model
* 定义了 increment 方法，去改变我们的数据 model ，调用 package 中的 通知方法 notifyListeners


```dart
  // lib/main.dart

  import 'package:flutter/material.dart';
  import './model/counter_model.dart';
  import 'package:scoped_model/scoped_model.dart';
  import './count_page.dart';
  
  void main() {
    runApp(MyApp(
      model: CounterModel(),
    ));
  }
  
  class MyApp extends StatelessWidget {
  
    final CounterModel model;
  
    const MyApp({Key key,@required this.model}):super(key:key);
  
    @override
    Widget build(BuildContext context) {
      return ScopedModel(
        model: model,
        child: MaterialApp(
          title: 'Scoped Model Demo',
          home:CountPage(),
        ),
      );
    }
  }
```

这是 app 的入口文件，划重点

* MyApp 类 中，我们传入一个定义好的数据 model ，方便后面传递给子类
* 将 MaterialApp 用 ScopedModel 包裹一下，作用上面已经介绍了，方便子类可以拿到 ，类似于 redux 中 Provider 包裹一下
* 一定需要将数据 model 传递给 ScopedModel 的 model 属性中

```dart
// lib/count_page.dart

  class CountPage extends StatelessWidget {
    @override
    Widget build(BuildContext context) {
      return Scaffold(
        appBar: AppBar(
          title: Text('Scoped Model'),
          actions: <Widget>[
            IconButton(
              tooltip: 'to result',
              icon: Icon(Icons.home),
              onPressed: (){
                Navigator.push(context,MaterialPageRoute(builder: (context)=>ResultPage()));
              },
            )
          ],
        ),
        body: Center(
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: <Widget>[
              Text('你都点击'),
              ScopedModelDescendant<CounterModel>(
                builder: (context, child, model) {
                  return Text(
                    '${model.counter.toString()} 次了',
                    style: TextStyle(
                      color: Colors.red,
                      fontSize: 33.0,
                    ),
                  );
                },
              )
            ],
          ),
        ),
        floatingActionButton: ScopedModelDescendant<CounterModel>(
          builder: (context,child,model){
            return FloatingActionButton(
              onPressed: model.increment,
              tooltip: 'add',
              child: Icon(Icons.add),
            );
          },
        ),
      );
    }
  }
```

常规布局和widget这里不再重复介绍，我们说下主角：`Scoped Model`

* 简单一句，哪里需要用数据 model ，哪里就需要用 ScopedModelDescendant
* ScopedModelDescendant中的build方法需要返回一个widget，在这个widget中我们可以使用数据 model中的方法、数据等

最后在 `lib/result_page.dart` 中就可以看到我们数据 model 中的 count 值了，注意这里跳转页面，我们并没有通过参数传递的形式传递 `Navigator.push(context,MaterialPageRoute(builder: (context)=>ResultPage()));`

完整项目代码：flutter_scoped_model[2]

## flutter_redux

相信作为一个前端对于 redux 一定不会陌生，而 Flutter 中也同样存在 state 的概念，其实说白了，UI 只是数据（state）的另一种展现形式。study-redux 是笔者之前学习 redux 时候的一些笔记和心得。这里为了防止有新人不太清楚redux，我们再来介绍下redux的一些基本概念

### state

state 我们可以理解为前端UI的状态（数据）库，它存储着这个应用所有需要的数据。

![2](http://)

### action

既然这些state已经有了，那么我们是如何实现管理这些state中的数据的呢，当然，这里就要说到action了。 什么是action？E:action：动作。 是的，就是这么简单。。。

只有当某一个动作发生的时候才能够触发这个state去改变，那么，触发state变化的原因那么多，比如这里的我们的点击事件，还有网络请求，页面进入，鼠标移入。。。所以action的出现，就是为了把这些操作所产生或者改变的数据从应用传到store中的有效载荷。 需要说明的是，action是state的唯一信号来源。

### reducer

**reducer决定了state的最终格式**。 reducer是一个纯函数，也就是说，只要传入参数相同，返回计算得到的下一个 state 就一定相同。没有特殊情况、没有副作用，没有 API 请求、没有变量修改，单纯执行计算。reducer对传入的action进行判断，然后返回一个通过判断后的state，这就是reducer的全部职责。 从代码可以简单地看出：

```dart
  import {INCREMENT_COUNTER,DECREMENT_COUNTER} from '../actions';
  
  export default function counter(state = 0,action) {
      switch (action.type){
          case INCREMENT_COUNTER:
              return state+1;
          case DECREMENT_COUNTER:
              return state-1;
          default:
              return state;
      }
  }
```

对于一个比较大一点的应用来说，我们是需要将reducer拆分的，最后通过redux提供的combineReducers方法组合到一起。 比如：

```dart
  const rootReducer = combineReducers({
      counter
  });
  
  export default rootReducer;
```

这里你要明白：每个 reducer 只负责管理全局 state 中它负责的一部分。每个 reducer 的 state 参数都不同，分别对应它管理的那部分 state 数据。 combineReducers() 所做的只是生成一个函数，这个函数来调用你的一系列 reducer，每个 reducer 根据它们的 key 来筛选出 state 中的一部分数据并处理， 然后这个生成的函数再将所有 reducer 的结果合并成一个大的对象。

### store

store是对之前说到一个联系和管理。具有如下职责

* 维持应用的 state；
* 提供 getState() 方法获取 state
* 提供 dispatch(action) 方法更新 state；
* 通过 subscribe(listener) 注册监听器;
* 通过 subscribe(listener) 返回的函数注销监听器。

再次强调一下 Redux 应用只有一个单一的 store。当需要拆分数据处理逻辑时，你应该使用 reducer 组合 而不是创建多个 store。 store的创建通过redux的createStore方法创建，这个方法还需要传入reducer，很容易理解：毕竟我需要dispatch一个action来改变state嘛。 应用一般会有一个初始化的state，所以可选为第二个参数，这个参数通常是有服务端提供的，传说中的Universal渲染。后面会说。。。 第三个参数一般是需要使用的中间件，通过applyMiddleware传入。

说了这么多，action，store，action creator，reducer关系就是这么如下的简单明了：

![3](http://)

### 结合 flutter_redux

> 一些工具集让你轻松地使用 redux 来轻松构建 Flutter widget，版本要求是 redux.dart 3.0.0+

**Redux Widgets**

* StoreProvider ：基础组件，它将给定的 Redux Store 传递给所欲请求它的的子代组件
* StoreBuilder ： 一个子代组件，它从 StoreProvider 获取 Store 并将其传递给 widget 的 builder 方法中
* StoreConnector ：获取 Store 的一个子代组件

StoreProvider ancestor，使用给定的 converter 函数将 Store 转换为 ViewModel ，并将ViewModel传递给 builder。 只要 Store 发出更改事件（action），Widget就会自动重建。 无需管理订阅！

**注意**

Dart 2 需要更严格的类型！

1、确认你正使用的是 redux 3.0.0+
2、在你的组件树中，将 new StoreProvider(...) 改为 new StoreProvider<StateClass>(...)
3、如果需要从StoreProvider<AppState> 中直接获取 Store<AppState> ，则需要将 `new StoreProvider.of(context)` 改为 `StoreProvider.of<StateClass>` .不需要直接访问 Store 中的字段，因为Dart2可以使用静态函数推断出正确的类型

### 实操演练

官方demo的代码先大概解释一下

```dart
  import 'package:flutter/material.dart';
  import 'package:flutter_redux/flutter_redux.dart';
  import 'package:redux/redux.dart';
  
  //定义一个action: Increment
  enum Actions { Increment }
  
  // 定义一个 reducer，响应传进来的 action
  int counterReducer(int state, dynamic action) {
    if (action == Actions.Increment) {
      return state + 1;
    }
  
    return state;
  }
  
  void main() {
    // 在 基础 widget 中创建一个 store，用final关键字修饰  这比直接在build方法中创建要好很多
    final store = new Store<int>(counterReducer, initialState: 0);
  
    runApp(new FlutterReduxApp(
      title: 'Flutter Redux Demo',
      store: store,
    ));
  }
  
  class FlutterReduxApp extends StatelessWidget {
    final Store<int> store;
    final String title;
  
    FlutterReduxApp({Key key, this.store, this.title}) : super(key: key);
  
    @override
    Widget build(BuildContext context) {
      // 用  StoreProvider 来包裹你的 MaterialApp 或者别的 widget ，这样能够确保下面所有的widget能够获取到store中的数据
      return new StoreProvider<int>(
        // 将 store  传递给 StoreProvider
        // Widgets 将使用 store 变量来使用它
        store: store,
        child: new MaterialApp(
          theme: new ThemeData.dark(),
          title: title,
          home: new Scaffold(
            appBar: new AppBar(
              title: new Text(title),
            ),
            body: new Center(
              child: new Column(
                mainAxisAlignment: MainAxisAlignment.center,
                children: [
                  new Text(
                    'You have pushed the button this many times:',
                  ),
                  // 通过 StoreConnector 将 store 和 Text 连接起来，以便于 Text直接render
                  // store 中的值。类似于 react-redux 中的connect
                  //
                  // 将 Text widget 包裹在 StoreConnector 中，
                  // `StoreConnector`将会在最近的一个祖先元素中找到 StoreProvider
                  // 拿到对应的值，然后传递给build函数
                  //
                  // 每次点击按钮的时候，将会 dispatch 一个 action并且被reducer所接受。
                  // 等reducer处理得出最新结果后， widget将会自动重建
                  new StoreConnector<int, String>(
                    converter: (store) => store.state.toString(),
                    builder: (context, count) {
                      return new Text(
                        count,
                        style: Theme.of(context).textTheme.display1,
                      );
                    },
                  )
                ],
              ),
            ),
            // 同样使用 StoreConnector 来连接Store 和FloatingActionButton
            // 在这个demo中，我们使用store 去构建一个包含dispatch、Increment 
            // action的回调函数
            //
            // 将这个回调函数丢给 onPressed
            floatingActionButton: new StoreConnector<int, VoidCallback>(
              converter: (store) {
                return () => store.dispatch(Actions.Increment);
              },
              builder: (context, callback) {
                return new FloatingActionButton(
                  onPressed: callback,
                  tooltip: 'Increment',
                  child: new Icon(Icons.add),
                );
              },
            ),
          ),
        ),
      );
    }
  }
```

上面的例子比较简单，鉴于 `小册Flutter入门实战：从0到1仿写web版掘金App[3]` 下面有哥们在登陆那块评论了Flutter状态管理，

这里我简单使用redux模拟了一个 `登陆的demo[4]`

![4](http://)

> lib/reducer/reducers.dart

首先我们定义action需要的一些action type

```dart
  enum Actions{
    Login,
    LoginSuccess,
    LogoutSuccess
  }
```

然后定义相应的类来管理登陆状态

```dart
  class AuthState{
    bool isLogin;     //是否登录
    String account;   //用户名
    AuthState({this.isLogin:false,this.account});
  
    @override
    String toString() {
      return "{account:$account,isLogin:$isLogin}";
    }
  }
```

然后我们需要定义一些action，定义个基类，然后定义登陆成功的action

```dart
  class Action{
    final Actions type;
    Action({this.type});
  }
  
  class LoginSuccessAction extends Action{
  
    final String account;
  
    LoginSuccessAction({
      this.account
    }):super( type:Actions.LoginSuccess );
  }
```

最后定义 `AppState` 以及我们自定义的一个中间件。

```dart
  // 应用程序状态
  class AppState {
    AuthState auth; //登录
    MainPageState main; //主页
  
    AppState({this.main, this.auth});
  
    @override
    String toString() {
      return "{auth:$auth,main:$main}";
    }
  }
  
  AppState mainReducer(AppState state, dynamic action) {
  
    if (Actions.LogoutSuccess == action) {
      state.auth.isLogin = false;
      state.auth.account = null;
    }
  
    if (action is LoginSuccessAction) {
      state.auth.isLogin = true;
      state.auth.account = action.account;
    }
  
    print("state changed:$state");
  
    return state;
  }
  
  loggingMiddleware(Store<AppState> store, action, NextDispatcher next) {
    print('${new DateTime.now()}: $action');
  
    next(action);
  }
```

在稍微大一点的项目中，其实就是reducer 、 state 和 action 的组织会比较麻烦，当然，罗马也不是一日建成的， 庞大的state也是一点一点累计起来的。

下面就是在入口文件中使用 redux 的代码了，跟基础demo没有差异。

```dart
  import 'package:flutter/material.dart';
  import 'package:flutter_redux/flutter_redux.dart';
  import 'package:redux/redux.dart';
  import 'dart:async' as Async;
  import './reducer/reducers.dart';
  import './login_page.dart';
  
  void main() {
    Store<AppState> store = Store<AppState>(mainReducer,
        initialState: AppState(
          main: MainPageState(),
          auth: AuthState(),
        ),
        middleware: [loggingMiddleware]);
  
    runApp(new MyApp(
      store: store,
    ));
  }
  
  class MyApp extends StatelessWidget {
    final Store<AppState> store;
  
    MyApp({Key key, this.store}) : super(key: key);
  
    @override
    Widget build(BuildContext context) {
      return new StoreProvider(store: store, child: new MaterialApp(
        title: 'Flutter Demo',
        theme: new ThemeData(
          primarySwatch: Colors.blue,
        ),
        home:  new StoreConnector<AppState,AppState>(builder: (BuildContext context,AppState state){
          print("isLogin:${state.auth.isLogin}");
          return new MyHomePage(title: 'Flutter Demo Home Page',
            counter:state.main.counter,
              isLogin: state.auth.isLogin,
              account:state.auth.account);
        }, converter: (Store<AppState> store){
          return store.state;
        }) ,
        routes: {
          "login":(BuildContext context)=>new StoreConnector(builder: ( BuildContext context,Store<AppState> store ){
  
            return new LoginPage(callLogin: (String account,String pwd) async{
              print("正在登录，账号$account,密码:$pwd");
              // 为了模拟实际登录，这里等待一秒
              await new Async.Future.delayed(new Duration(milliseconds: 1000));
              if(pwd != "123456"){
                throw ("登录失败，密码必须是123456");
              }
              print("登录成功!");
              store.dispatch(new LoginSuccessAction(account: account));
  
            },);
          }, converter: (Store<AppState> store){
            return store;
          }),
  
        },
      ));
    }
  }
  
  class MyHomePage extends StatelessWidget {
    MyHomePage({Key key, this.title, this.counter, this.isLogin, this.account})
        : super(key: key);
    final String title;
    final int counter;
    final bool isLogin;
    final String account;
  
    @override
    Widget build(BuildContext context) {
      print("build:$isLogin");
      Widget loginPane;
      if (isLogin) {
        loginPane = new StoreConnector(
            key: new ValueKey("login"),
            builder: (BuildContext context, VoidCallback logout) {
              return new RaisedButton(
                onPressed: logout, child: new Text("您好:$account,点击退出"),);
            }, converter: (Store<AppState> store) {
          return () =>
              store.dispatch(
                  Actions.LogoutSuccess
              );
        });
      } else {
        loginPane = new RaisedButton(onPressed: () {
          Navigator.of(context).pushNamed("login");
        }, child: new Text("登录"),);
      }
      return new Scaffold(
        appBar: new AppBar(
          title: new Text(title),
        ),
        body: new Center(
          child: new Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: <Widget>[
  
              /// 有登录，展示你好:xxx,没登录，展示登录按钮
              loginPane
            ],
          ),
        ),
  
      );
    }
  }
```

完整项目代码：`Nealyang/Flutter[5]`

## Flutter Go

更多学习 Flutter的小伙伴，欢迎入 QQ 群 Flutter Go :679476515

关于 Flutter 组件以及更多的学习，敬请关注我们正在开发的： `alibaba/flutter-go[6]`

![5](http://)

### 参考

1. scoped_model https://pub.flutter-io.cn/packages/scoped_model#-readme-tab-
2. demo 地址：https://github.com/Nealyang/flutter/tree/master/flutter_scoped_model
3. Flutter入门实战：从0到1仿写web版掘金App https://juejin.im/book/5bff85f3e51d453c6c05fa57
4. 登陆的demo https://github.com/Nealyang/flutter/tree/master/flutter_try_redux
5. Nealyang/Flutter https://github.com/Nealyang/flutter
6. flutter-go https://github.com/alibaba/flutter-go


