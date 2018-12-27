---
title: "响应式编程：从Streams到BLoC"
date: 2018-12-27T12:17:51+08:00
draft: false
banner: "https://www.didierboelens.com/images/streams_bloc.png"
author: "kelvinji2009"
authorlink: "https://kelvinji2009.github.io"
summary: "这篇文章主要介绍Streams，BLoC和Reactive Programming的概念、理论和实践范例。"
tags: ["flutter", "BLoC"]
categories: ["flutter"]
---

# 响应式编程：从 Streams 到 BLoC

[原文](https://www.didierboelens.com/2018/08/reactive-programming---streams---bloc/)

这篇文章主要介绍 Streams，BLoC 和 Reactive Programming 的概念、理论和实践范例。

难度：适中

## 介绍

我花了很长时间才找到介绍 Reactive Programming，BLoC 和 Streams 概念的方法。

由于这可以对构建应用程序的方式做出重大改变，我想要一个实际示例来说明：

- 很可能如果你不使用它们，有时可能会难以编码，且性能更低
- 当然包括使用它们的好处
- 使用它们的影响（正面和/或负面）

我做了一个实际例子是一个虚构的应用程序。简而言之，它允许用户查看在线目录中的电影列表，按流派和发布日期过滤它们，标记/取消收藏。 当然，一切都是互动的，用户动作可以在不同的页面中或在同一个页面内发生，并且对视觉方面有实时的影响。

这是一个显示此应用程序的动画。

![streams_app_1.gif](https://www.didierboelens.com/images/streams_app_1.gif)

当您进入此页面以获取有关 Reactive Programming，BLoC 和 Streams 的信息时，我将首先介绍它们的理论基础， 此后，我将向您展示如何在实践中运用它们。

本文的补充内容提供了一些实际用例，可以在此[链接](https://www.didierboelens.com/2018/12/reactive-programming---streams---bloc---practical-use-cases/)找到。

## 什么是 Stream?

### 介绍

为了便于想象 Stream 的概念，只需要想象一个有两端的管道，只能从一端插入一些东西。 当您将某物插入管道时，它会在管道内流动并从另一端流出。

在 Flutter 中，

- 这个管道被叫做 Stream
- 为了控制 Stream，我们通常<1>使用 StreamController
- 为了在 Stream 中插入一些东西，StreamController 公开了一个名为 StreamSink 的“入口”，可以通过 sink 属性访问
- 流出 Stream 的方式由 StreamController 通过 stream 属性公开

<1>我专门使用了“通常”这个词，因为很可能不使用任何 StreamController。 但是，正如您将在本文中看到的那样，我将一定使用 StreamControllers。

### 一个 Stream 可以传递什么？

什么都可以。 从值，事件，对象，集合，映射，错误或甚至另一个流，可以由流传递任何类型的数据。

### 我怎么知道 Stream 传递的是什么东西？

当您需要通知 Stream 传递了某些内容时，您只需要监听 StreamController 的 stream 属性。

定义一个 Listener，您会收到 StreamSubscription 对象。 通过 StreamSubscription 对象，您将收到有关在 Stream 级别发生了某些事情的通知。

只要至少有一个 active 的 listener，Stream 就会开始生成事件，以便每次都通知 active 的 StreamSubscription 对象：

- 一些数据从 stream 流出
- 当一些错误发送到流时
- 当流关闭时

StreamSubscription 对象还允许您：

- 停止监听流
- 暂停流
- 恢复流

### 一个 Stream 是否只是一个简单的 pipeline?

不，流还允许在流出之前处理流入其中的数据。

为了控制 Stream 内部数据的处理，我们使用 StreamTransformer，它是:

- 一个“捕获”Stream 内部流动数据的函数
- 对数据做些处理
- 这种转换的结果也是一个流

从以上的描述中可以了解到，我们可以按顺序使用多个 StreamTransformer。

StreamTransformer 可用于进行任何类型的处理，例如：

- 过滤：根据任何类型的条件过滤数据
- 重组：重新组合数据
- 修改：对数据应用任何类型的修改
- 将数据注入其他流
- 缓冲
- 处理：根据数据进行任何类型的动作/操作
- ...

### Stream 的类型

#### Single-subscription Streams：单订阅流

这种类型的 Stream 只允许在该 Stream 的整个生命周期内使用单个 listener。

> 即使在第一个订阅被取消后，也无法在此类流上收听两次。

#### Broadcast Streams：广播流

第二种类型的 Stream 允许任意数量的 listener。

> 可以随时向广播流添加 listener。 新的 listener 将在它开始监听 Stream 时收到事件。

## 简单例子

### 任意类型的数据

这第一个示例显示了“单订阅”流，它只是打印输入的数据。 您可能会看到数据类型无关紧要。

```dart
import 'dart:async';

void main() {
  //
  // Initialize a "Single-Subscription" Stream controller
  //
  final StreamController ctrl = StreamController();

  //
  // Initialize a single listener which simply prints the data
  // as soon as it receives it
  //
  final StreamSubscription subscription = ctrl.stream.listen((data) => print('$data'));

  //
  // We here add the data that will flow inside the stream
  //
  ctrl.sink.add('my name');
  ctrl.sink.add(1234);
  ctrl.sink.add({'a': 'element A', 'b': 'element B'});
  ctrl.sink.add(123.45);

  //
  // We release the StreamController
  //
  ctrl.close();
}
```

### StreamTransformer

第二个示例显示“广播”流，它传递整数值并仅打印偶数。 为此，我们应用 StreamTransformer 来过滤（第 14 行）值，只让偶数通过。

```dart
import 'dart:async';

void main() {
  //
  // Initialize a "Broadcast" Stream controller of integers
  //
  final StreamController<int> ctrl = StreamController<int>.broadcast();

  //
  // Initialize a single listener which filters out the odd numbers and
  // only prints the even numbers
  //
  final StreamSubscription subscription = ctrl.stream
					      .where((value) => (value % 2 == 0))
					      .listen((value) => print('$value'));

  //
  // We here add the data that will flow inside the stream
  //
  for(int i=1; i<11; i++){
  	ctrl.sink.add(i);
  }

  //
  // We release the StreamController
  //
  ctrl.close();
}
```

## RxDart

如今，如果我不提及 RxDart 包，那么 Streams 的介绍将不再完整。

RxDart 包是 ReactiveX API 的 Dart 实现，它扩展了原始的 Dart Streams API 以符合 ReactiveX 标准。

由于它最初并未由 Google 定义，因此它使用不同的词汇表。 下表给出了 Dart 和 RxDart 之间的相关性。

| Dart             | RxDart     |
| ---------------- | ---------- |
| Stream           | Observable |
| StreamController | Subject    |

正如我刚才所说，RxDart 扩展了原始的 Dart Streams API 并提供了 StreamController 的 3 个主要变体：

### PublishSubject

PublishSubject 是一个普通的广播 StreamController，有一个例外：stream 返回一个 Observable 而不是一个 Stream。

![S.PublishSubject.png](https://www.didierboelens.com/images/S.PublishSubject.png)

如您所见，PublishSubject 仅向 Listener 发送在订阅之后添加到 Stream 的事件。

### BehaviorSubject

BehaviorSubject 也是一个广播 StreamController，它返回一个 Observable 而不是一个 Stream。

![S.BehaviorSubject.png](https://www.didierboelens.com/images/S.BehaviorSubject.png)

与 PublishSubject 的主要区别在于 BehaviorSubject 还将最后发送的事件发送给刚刚订阅的 listener。

### ReplaySubject

ReplaySubject 也是一个广播 StreamController，它返回一个 Observable 而不是一个 Stream。

![S.ReplaySubject.png](https://www.didierboelens.com/images/S.ReplaySubject.png)

默认情况下，ReplaySubject 将 Stream 已经发出的所有事件作为第一个事件发送到任何新的 listener。

### 关于资源的重要说明

> 始终释放不再需要的资源是一种非常好的做法。

以上说明适用于：

- StreamSubscription - 当您不再需要收听流时，取消订阅;
- StreamController - 当你不再需要 StreamController 时，关闭它;
- 这同样适用于 RxDart 的 Subject，当您不再需要 BehaviourSubject，PublishSubject ...时，请将其关闭。

## 如何基于流出的数据构建 Widget？

Flutter 提供了一个非常方便的 StatefulWidget，称为 StreamBuilder。

StreamBuilder 监听 Stream，每当某些数据输出 Stream 时，它会自动重建，调用其 builder 回调。

如何使用 StreamBuilder?

```dart
StreamBuilder<T>(
    key: ...optional, the unique ID of this Widget...
    stream: ...the stream to listen to...
    initialData: ...any initial data, in case the stream would initially be empty...
    builder: (BuildContext context, AsyncSnapshot<T> snapshot){
        if (snapshot.hasData){
            return ...the Widget to be built based on snapshot.data
        }
        return ...the Widget to be built if no data is available
    },
)
```

以下示例模仿默认的“计数器”应用程序，但使用 Stream 而不再使用任何 setState。

```dart
import 'dart:async';
import 'package:flutter/material.dart';

class CounterPage extends StatefulWidget {
  @override
  _CounterPageState createState() => _CounterPageState();
}

class _CounterPageState extends State<CounterPage> {
  int _counter = 0;
  final StreamController<int> _streamController = StreamController<int>();

  @override
  void dispose(){
    _streamController.close();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Stream version of the Counter App')),
      body: Center(
        child: StreamBuilder<int>(
          stream: _streamController.stream,
          initialData: _counter,
          builder: (BuildContext context, AsyncSnapshot<int> snapshot){
            return Text('You hit me: ${snapshot.data} times');
          }
        ),
      ),
      floatingActionButton: FloatingActionButton(
        child: const Icon(Icons.add),
        onPressed: (){
          _streamController.sink.add(++_counter);
        },
      ),
    );
  }
}
```

说明和注释：

- 第 24-30 行：我们正在收听一个流，每当一个新值输出这个流时，我们用该值更新 Text;
- 第 35 行：当我们点击 FloatingActionButton 时，我们递增计数器并通过接收器将其发送到 Stream; 在流中注入值的事实导致侦听它的 StreamBuilder 重建并“刷新”计数器;
- 我们不再需要 state 的概念，所有内容都通过 Stream 接收;
- 这是一个很大的改进，因为调用 setState()方法，强制整个 Widget（和任何子窗口小部件）重建。 在这里，只重建 StreamBuilder（当然还有子窗口小部件）;
- 我们仍然在为页面使用 StatefulWidget 的唯一原因，仅仅是因为我们需要通过 dispose 方法释放 StreamController，第 15 行;

## 什么是 Reactive Programming?

> 响应式编程是使用异步数据流进行编程。

> 换句话说，从事件（例如，点击），变量的变化，消息，......到构建请求，可能改变或发生的一切的所有内容将被传送，由数据流触发。

很明显，所有这些意味着，通过响应式编程，应用程序：

- 变得异步
- 围绕 Streams 和 listener 的概念进行架构
- 当某些事情发生在某个地方（事件，变量的变化......）时，会向 Stream 发送通知
- 如果“某人”收听该流，它将被通知并将采取适当的行动，无论其在应用程序中的位置如何

> 组件之间不再存在紧耦合。

简而言之，当 Widget 向 Stream 发送内容时，该 Widget 不再需要知道：

- 接下来会发生什么，
- 谁可能使用这些信息（一个都没有，一个或几个小部件...）
- 可能使用此信息的地方（一个地方都没有，同一屏，另一屏，几屏......），
- 当这些信息可能被使用时（几乎是直接使用，几秒钟之后使用，永远不会使用......）。

> ...... Widget 只关心自己的业务，就是这样！

乍一看，读到这个，这似乎可能导致应用程序的“无法控制”，但正如我们将看到的，情况恰恰相反。 它给你：

- 构建仅负责特定活动的部分应用程序的机会，
- 轻松模拟一些组件的行为，以允许更完整的测试覆盖，
- 轻松重用组件（应用程序的其他地方或其他应用程序中），
- 重新设计应用程序，并能够在不进行太多重构的情况下将组件从一个地方移动到另一个地方
- ...

我们将很快看到优势......但是这要在我需要介绍完最后一个主题之前：BLoC 模式。

## BLoC 模式

BLoC 模式由来自 Google 的 Paolo Soares 和 Cong Hui 设计，并在 2018 年 DartConf 期间（2018 年 1 月 23 日至 24 日）首次展示。 请观看 YouTube 上的视频[YouTube](https://www.youtube.com/watch?v=PLHln7wHgPE)。

BLoC 代表业务逻辑组件，全称是：Business Logic Component。

简而言之，Business Logic 需要：

- 转移到一个或几个 BLoC
- 尽可能从表示层中删除。 换句话说，UI 组件应该只关心 UI，而不关心业务
- 依赖 Streams 使用：输入（Sink）和输出（Stream）
- 保持平台独立
- 保持环境独立

事实上，BLoC 模式最初被设想为允许独立于平台而重用相同的代码：Web 应用程序，移动应用程序，后端。

### BLoC 到底意味着什么？

BLoC 模式利用了我们刚才讨论过的概念：Streams。

![streams_bloc.png](https://www.didierboelens.com/images/streams_bloc.png)

- 小部件 widget 通过 Sinks 向 BLoC 发送事件，
- BLoC 通过流通知小部件 widget，
- 由 BLoC 实现的业务逻辑不是他们关注的问题。

从这个陈述中，我们可以直接看到一个巨大的好处。

> 感谢业务逻辑与 UI 的分离：
>
> - 我们可以随时更改业务逻辑，对应用程序的影响最小，
> - 我们可能会更改 UI 而不会对业务逻辑产生任何影响，
> - 现在，测试业务逻辑变得更加容易。

### 如何将此 BLoC 模式应用于计数器应用示例？

将 BLoC 模式应用于此计数器应用程序可能看起来有点矫枉过正，但让我先向您展示......

```dart
void main() => runApp(new MyApp());

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return new MaterialApp(
        title: 'Streams Demo',
        theme: new ThemeData(
          primarySwatch: Colors.blue,
        ),
        home: BlocProvider<IncrementBloc>(
          bloc: IncrementBloc(),
          child: CounterPage(),
        ),
    );
  }
}

class CounterPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final IncrementBloc bloc = BlocProvider.of<IncrementBloc>(context);

    return Scaffold(
      appBar: AppBar(title: Text('Stream version of the Counter App')),
      body: Center(
        child: StreamBuilder<int>(
          stream: bloc.outCounter,
          initialData: 0,
          builder: (BuildContext context, AsyncSnapshot<int> snapshot){
            return Text('You hit me: ${snapshot.data} times');
          }
        ),
      ),
      floatingActionButton: FloatingActionButton(
        child: const Icon(Icons.add),
        onPressed: (){
          bloc.incrementCounter.add(null);
        },
      ),
    );
  }
}

class IncrementBloc implements BlocBase {
  int _counter;

  //
  // Stream to handle the counter
  //
  StreamController<int> _counterController = StreamController<int>();
  StreamSink<int> get _inAdd => _counterController.sink;
  Stream<int> get outCounter => _counterController.stream;

  //
  // Stream to handle the action on the counter
  //
  StreamController _actionController = StreamController();
  StreamSink get incrementCounter => _actionController.sink;

  //
  // Constructor
  //
  IncrementBloc(){
    _counter = 0;
    _actionController.stream
                     .listen(_handleLogic);
  }

  void dispose(){
    _actionController.close();
    _counterController.close();
  }

  void _handleLogic(data){
    _counter = _counter + 1;
    _inAdd.add(_counter);
  }
}
```

我已经听到你说“哇......为什么这样写？ 这一切都是必要的吗？”

#### 第一：责任分离

如果您检查 CounterPage（第 21-45 行），其中完全没有任何业务逻辑。

此页面现在仅负责：

- 显示计数器，现在只在必要时刷新（即使页面不必知道）
- 提供按钮，当按下时，请求在计数器上执行动作

此外，整个业务逻辑集中在一个单独的类“IncrementBloc”中。

如果现在您需要更改业务逻辑，您只需更新方法\_handleLogic（第 77-80 行）。 也许新的业务逻辑会要求做非常复杂的事情...... CounterPage 永远不会知道它，这非常好！

#### 第二：可测试性

现在，测试业务逻辑变得更加容易。

无需再通过用户界面测试业务逻辑。 只需要测试 IncrementBloc 类。

#### 第三：自由组织布局

由于使用了 Streams，您现在可以独立于业务逻辑组织布局。

可以从应用程序中的任何位置执行任何操作：只需调用.incrementCounter sink 即可。

您可以在任何页面的任何位置显示计数器，只需监听.outCounter stream。

#### 第四：减少"build"数量

不使用 setState()而是使用 StreamBuilder 这一事实大大减少了“build”的数量，当然只是减少了所需的数量。

从性能角度来看，这是一个巨大的进步。

### 只有一个约束...... BLoC 的可访问性

为了使所有这些工作，BLoC 需要可访问。

有几种方法可以访问它：

- 通过全局单实例

这种方式很有可能，但不是真的推荐。 此外，由于 Dart 中没有类析构函数，因此您永远无法正确释放资源。

- 作为一个本地实例

您可以本地实例化 BLoC。 在某些情况下，此解决方案完全符合某些需求。 在这种情况下，您应该始终考虑在 StatefulWidget 中初始化，以便您可以利用 dispose（）方法来释放它。

- 由祖先提供

使其可访问的最常见方式是通过祖先 Widget 来访问，实现为一个 StatefulWidget。

以下代码显示了通用 BlocProvider 的示例。

```dart
// Generic Interface for all BLoCs
abstract class BlocBase {
  void dispose();
}

// Generic BLoC provider
class BlocProvider<T extends BlocBase> extends StatefulWidget {
  BlocProvider({
    Key key,
    @required this.child,
    @required this.bloc,
  }): super(key: key);

  final T bloc;
  final Widget child;

  @override
  _BlocProviderState<T> createState() => _BlocProviderState<T>();

  static T of<T extends BlocBase>(BuildContext context){
    final type = _typeOf<BlocProvider<T>>();
    BlocProvider<T> provider = context.ancestorWidgetOfExactType(type);
    return provider.bloc;
  }

  static Type _typeOf<T>() => T;
}

class _BlocProviderState<T> extends State<BlocProvider<BlocBase>>{
  @override
  void dispose(){
    widget.bloc.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context){
    return widget.child;
  }
}
```

#### 关于这种通用 BlocProvider 的一些解释

首先，如何将其用作`provider`？

如果您查看示例代码“streams_4.dart”，您将看到以下代码行（第 12-15 行）:

```dart
home: BlocProvider<IncrementBloc>(
          bloc: IncrementBloc(),
          child: CounterPage(),
        ),
```

类似如上代码的用法，我们只需实例化一个新的 BlocProvider，它将包含一个 IncrementBloc，并将 CounterPage 作为 child 渲染。

从那一刻开始，从 BlocProvider 开始的子树的任何小部件部分都将能够通过以下行访问 IncrementBloc：

```dart
IncrementBloc bloc = BlocProvider.of<IncrementBloc>(context);
```

### 我们可以有多个 BLoC 吗？

当然，这是非常可取的。 建议是：

- （如果有任何业务逻辑）每页顶层有一个 BLoC，
- 为什么不是 ApplicationBloc 来处理应用程序状态？
- 每个“足够复杂的组件”都有相应的 BLoC。

以下示例代码在整个应用程序的顶层使用 ApplicationBloc，然后在 CounterPage 顶层使用 IncrementBloc。
同时也展示了，如何获取这两个 BLoCs：

```dart
void main() => runApp(
  BlocProvider<ApplicationBloc>(
    bloc: ApplicationBloc(),
    child: MyApp(),
  )
);

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context){
    return MaterialApp(
      title: 'Streams Demo',
      home: BlocProvider<IncrementBloc>(
        bloc: IncrementBloc(),
        child: CounterPage(),
      ),
    );
  }
}

class CounterPage extends StatelessWidget {
  @override
  Widget build(BuildContext context){
    final IncrementBloc counterBloc = BlocProvider.of<IncrementBloc>(context);
    final ApplicationBloc appBloc = BlocProvider.of<ApplicationBloc>(context);

    ...
  }
}
```

### 为什么不使用 InheritedWidget？

在与 BLoC 相关的大多数文章中，您将看到 Provider 被实现为一个 InheritedWidget。

当然，没有什么能阻止这种类型的实现。 然而，

- 一个 InheritedWidget 没有提供任何 dispose 方法，请记住，在不再需要资源时总是释放资源是一种很好的做法。
- 当然，没有什么能阻止你将 InheritedWidget 包装在另一个 StatefulWidget 中，但是，使用 InheritedWidget 增加了什么呢？
- 最后，如果不受控制，使用 InheritedWidget 经常会导致副作用（请参阅下面的 InheritedWidget 上的 Reminder）。

这 3 个原因解释了我为将选择将通用 BlocProvider 实现为 StatefulWidget，这样我就可以在小部件被销毁时释放资源。

> **Flutter 无法实例化泛型类型**
> 不幸的是，Flutter 无法实例化泛型类型，我们必须将 BLoC 的实例传递给 BlocProvider。 为了在每个 BLoC 中强制执行 dispose（）方法，所有 BLoC 都必须实现> BlocBase 接口。

#### InheritedWidget 备忘录

当我们使用 InheritedWidget 并调用 context.inheritFromWidgetOfExactType（...）方法来获取给定类型的最近窗口小部件时，此方法调用会自动将此“上下文”（= BuildContext）注册到将要重建的窗口中，每当一些改变被引用到 InheritedWidget 子类或其祖先之一的时候，就会触发重建。

> 请注意，为了完全正确，我刚才解释的与 InheritedWidget 相关的问题只发生在我们将 InheritedWidget 与 StatefulWidget 结合使用时。 当您只使用没有 State 的 InheritedWidget 时，问题就不会发生。 但是......我将在下一篇文章中回顾这句话。

链接到 BuildContext 的 Widget（Stateful 或 Stateless）的类型无关紧要。

### 关于 BLoC 的个人笔记

与 BLoC 相关的第三条规则是：“依赖于 Streams 对输入（Sink）和输出（Stream）的独占使用”。

我的个人经历对于这个说法稍微有点不同看法......让我解释一下。

起初，BLoC 模式被设想为跨平台共享相同的代码（AngularDart，...），并且从这个角度来看，该规则非常有意义。

但是，如果您只打算开发一个 Flutter 应用程序，那么根据我的浅薄经验，这有点矫枉过正。

如果我们坚持这个规则，那么就没有 getter 或 setter，只有 sinks 和 streams。 缺点是“所有这些都是异步的”。

我们来看两个例子来说明缺点：

- 你需要从 BLoC 中获取一些数据，以便使用这些数据作为一个页面的输入，这些参数应该立即显示（例如，想像一个参数页面），如果我们不得不依赖 Streams，这会使构建异步页面（很复杂）。 通过 Streams 使其工作的示例代码可能类似于以下内容......很丑陋不是吗？

```dart
class FiltersPage extends StatefulWidget {
  @override
  FiltersPageState createState() => FiltersPageState();
}

class FiltersPageState extends State<FiltersPage> {
  MovieCatalogBloc _movieBloc;
  double _minReleaseDate;
  double _maxReleaseDate;
  MovieGenre _movieGenre;
  bool _isInit = false;

  @override
  void didChangeDependencies() {
    super.didChangeDependencies();

    // As the context of not yet available at initState() level,
    // if not yet initialized, we get the list of the
    // filter parameters
    if (_isInit == false){
      _movieBloc = BlocProvider.of<MovieCatalogBloc>(context);
      _getFilterParameters();
    }
  }

  @override
  Widget build(BuildContext context) {
    return _isInit == false
      ? Container()
      : Scaffold(
    ...
    );
  }

  ///
  /// Very tricky.
  ///
  /// As we want to be 100% BLoC compliant, we need to retrieve
  /// everything from the BLoCs, using Streams...
  ///
  /// This is ugly but to be considered as a study case.
  ///
  void _getFilterParameters() {
    StreamSubscription subscriptionFilters;

    subscriptionFilters = _movieBloc.outFilters.listen((MovieFilters filters) {
        _minReleaseDate = filters.minReleaseDate.toDouble();
        _maxReleaseDate = filters.maxReleaseDate.toDouble();

        // Simply to make sure the subscriptions are released
        subscriptionFilters.cancel();

        // Now that we have all parameters, we may build the actual page
        if (mounted){
          setState((){
            _isInit = true;
          });
        }
      });
    });
  }
}
```

- 在 BLoC 级别，您还可能需要转换某些数据的“假”注入，以触发提供您希望通过流接收的数据。 示例代码可以是：

```dart
class ApplicationBloc implements BlocBase {
  ///
  /// Synchronous Stream to handle the provision of the movie genres
  ///
  StreamController<List<MovieGenre>> _syncController = StreamController<List<MovieGenre>>.broadcast();
  Stream<List<MovieGenre>> get outMovieGenres => _syncController.stream;

  ///
  /// Stream to handle a fake command to trigger the provision of the list of MovieGenres via a Stream
  ///
  StreamController<List<MovieGenre>> _cmdController = StreamController<List<MovieGenre>>.broadcast();
  StreamSink get getMovieGenres => _cmdController.sink;

  ApplicationBloc() {
    //
    // If we receive any data via this sink, we simply provide the list of MovieGenre to the output stream
    //
    _cmdController.stream.listen((_){
      _syncController.sink.add(UnmodifiableListView<MovieGenre>(_genresList.genres));
    });
  }

  void dispose(){
    _syncController.close();
    _cmdController.close();
  }

  MovieGenresList _genresList;
}

// Example of external call
BlocProvider.of<ApplicationBloc>(context).getMovieGenres.add(null);
```

我不知道您的意见，但就个人而言，如果没有任何与代码移植/共享相关的限制，我发现这太重了，我宁愿在需要时使用常规的 getter/setter 并使用 Streams/Sinks 来保持责任分离并在需要的地方广播消息，这样做其实很棒。

## 现在是时候在实践中看看上面讲到的这一切了......

正如本文开头所提到的，我构建了一个示例应用程序来展示如何使用所有这些概念。 完整的源代码可以在[GitHub - boeledi/Streams-Block-Reactive-Programming-in-Flutter: Sample application to illustrate the notions of Streams, BLoC and Reactive Programming in Flutter](https://github.com/boeledi/Streams-Block-Reactive-Programming-in-Flutter)上找到。

请宽容一点，因为这段代码远非完美，可能更好和/或更好的架构，但唯一的目标只是告诉你这一切是如何工作的。

由于源代码记录很多，我只会解释主要原则。

### 数据来源

我使用免费的 TMDB API[API Overview — The Movie Database (TMDb)](https://www.themoviedb.org/documentation/api)
来获取所有电影的列表，以及海报，评级和描述。

为了能够运行此示例应用程序，您需要注册并获取 API 密钥（完全免费），然后将您的 API 密钥放在文件“/api/tmdb_api.dart”第 15 行。

### 应用架构

3 个主要的 BLoC：

- ApplicationBloc（在所有内容之上），负责提供所有电影类型的列表;
- FavoriteBloc（下一层），负责处理“收藏夹”的概念;
- MovieCatalogBloc（在 2 个主要页面之上），负责根据过滤器提供电影列表;

6 个页面：

- 主页：登陆页面，允许导航到 3 个子页面;
- ListPage：将电影列为 GridView 的页面，允许过滤，收藏夹选择，访问收藏夹以及在后续页面中显示电影详细信息;
- ListOnePage：类似于 ListPage，但电影列表显示为水平列表，下面是详细信息;
- 收藏页面：列出收藏夹的页面，允许取消选择任何收藏夹;
- 过滤器：允许定义过滤器的 EndDrawer：流派和最小/最大发布日期。从 ListPage 或 ListOnePage 调用此页面;
- 详细信息：页面仅由 ListPage 调用以显示电影的详细信息，但也允许选择/取消选择电影作为收藏;

1 个子 BLoC：

- FavoriteMovieBloc，链接到 MovieCardWidget 或 MovieDetailsWidget 以处理作为收藏的电影的选择/取消选择

5 个主要小部件 widget：

- FavoriteButton：小部件，负责显示收藏夹的数量，实时，并在按下时重定向到 FavoritesPage;
- FavoriteWidget：小部件，负责显示一个喜欢的电影的细节并允许其取消选择;
- FiltersSummary：小部件负责显示当前定义的过滤器;
- MovieCardWidget：小部件，负责将一部电影显示为卡片，电影海报，评级和名称，以及一个图标，表示该特定电影的选择是已收藏的;
- MovieDetailsWidget：小部件，负责显示与特定电影相关的详细信息，并允许其选择/取消选择是否收藏。

### 不同 BLoCs/Streams 的编排

下图显示了如何使用主要 3 个 BLoC：

- 在 BLoC 的左侧，展示了哪些组件调用 Sink
- 在右侧，展示了哪些组件监听 Stream

例如，当 MovieDetailsWidget 调用 inAddFavorite Sink 时，会触发 2 个流：

- outTotalFavorites 流强制重建 FavoriteButton，和
- outFavorites 流
  - 强制重建 MovieDetailsWidget（“收藏”图标）
  - 强制重建\_buildMovieCard（“收藏”图标）
  - 用于构建每个 MovieDetailsWidget

![streams_flows.png](https://www.didierboelens.com/images/streams_flows.png)

### Observations

大多数小部件和页面都是 StatelessWidgets，这意味着：

- 强制重建的 setState（）几乎从未使用过。 例外情况是：
  - 当用户点击 MovieCard 时，在 ListOnePage 中刷新 MovieDetailsWidget。 这也可能是由一个流驱动的......
  - 在 FiltersPage 中允许用户在接受过滤器之前通过 Sink 更改过滤器。
- 应用程序不使用任何 InheritedWidget

- 应用程序几乎是 100％BLoCs/Streams 驱动，这意味着大多数小部件彼此独立，包括它们在应用程序中的位置
  - 一个实际的例子是 FavoriteButton，显示收藏夹的数量徽章。 应用程序计算此 FavoriteButton 的 3 个实例，每个实例显示在 3 个不同的页面中。

### 显示电影列表（显示无限列表的技巧说明）

要显示符合过滤条件的电影列表，我们使用 GridView.builder（ListPage）或 ListView.builder（ListOnePage）作为无限滚动列表。

使用 TMDB API 一次 20 个电影的页面提取电影。

提醒一下，GridView.builder 和 ListView.builder 都将 itemCount 作为输入，如果提供了，则表示要显示的项目数。调用 itemBuilder，索引从 0 到 itemCount-1（不等于）。

正如您将在代码中看到的那样，我随意为 GridView.builder 添加了 30 多个。理由是，在这个例子中，我们正在操纵假定的无限数量的条目（这不是完全正确但是谁在乎呢，这只是个例子）。这将强制 GridView.builder 请求显示“最多 30 个”条目。

此外，GridView.builder 和 ListView.builder 只在认为必须在视口中呈现某个条目（索引）时才调用 itemBuilder。

此 MovieCatalogBloc.outMoviesList 返回 List<MovieCard>，迭代以构建每个电影卡片。第一次，这个 List<MovieCard>是空的，但是由于 itemCount：... + 30，我们欺骗系统，它将要求通过\_buildMovieCard（...）呈现 30 个不存在的条目。

正如您将在代码中看到的，此例程对 Sink 进行了一次奇怪的调用：

```dart
// Notify the MovieCatalogBloc that we are rendering the MovieCard[index]
    movieBloc.inMovieIndex.add(index);
```

此调用告诉 MovieCatalogBloc 我们要渲染 MovieCard [index]。

然后\_buildMovieCard（...）继续验证与 MovieCard [index]相关的数据是否存在。 如果是，则渲染后者，否则显示 CircularProgressIndicator。

对 StreamCatalogBloc.inMovieIndex.add（index）的调用由 StreamSubscription 监听，StreamSubscription 将索引转换为某个 pageIndex 数字（一页最多可计 20 部电影）。 如果尚未从 TMDB API 获取相应页面，则会调用 API。 获取页面后，所有已获取电影的新列表将发送到\_moviesController。 当 GridView.builder 监听该流（= movieBloc.outMoviesList）时，后者请求重建相应的 MovieCard。 由于我们现在拥有了数据，可以渲染它了。

## 参考和链接

描述 PublishSubject，BehaviorSubject 和 ReplaySubject 的图片由[ReactiveX](http://reactivex.io/)发布。

其他一些有趣值得一读的文章：

- [Fundamentals of Dart Streams](https://www.burkharts.net/apps/blog/)
- [rx_command package](https://pub.dartlang.org/packages/rx_command)
- [Build reactive mobile apps in Flutter — companion article](https://medium.com/flutter-io/build-reactive-mobile-apps-in-flutter-companion-article-13950959e381)
- [Flutter with Streams and RxDart \| SkillsCast | 9th July 2018](https://skillsmatter.com/skillscasts/12254-flutter-with-streams-and-rxdart)

## 结论

很长的文章，但还有更多的话要说，因为对我而言，这是展开 Flutter 应用程序的方法。 它提供了很大的灵活性。

Stay tuned for new articles, soon. Happy coding.
