# Flutter异步编程笔记

以下内容来自Flutter的youtube频道,在学习的同时,加上自己查阅的资料的,以及在VSCode中的代码实践总结

* [Isolate](##Isolate)
* Futures
* Stream
* Async/Await
* [Generators](##Generators)


## Isolate

在Flutter中，`Isolate`是一种Dart语言的并发机制，它可以让你在单个Dart程序中同时运行多个代码块，每个代码块都在自己的隔离环境中运行。这意味着每个`Isolate`都有自己的内存堆栈和消息队列，它们之间不能直接共享内存。`Isolate`之间可以通过消息传递进行通信。

许多Flutter程序通常都在单一的`Isolate`中运行所有代码,但是如果你需要的话,你可以使用多个`Isolate`

Flutter应用程序启动时，会启动一个主线程，也就是Root Isolate，它在内部运行一个EventLoop事件循环。在这个Root Isolate中，你可以创建其他Isolate，并在它们中运行代码。

| root isolate | isolate | isolate | isolate |
| --- | --- | --- | --- |
| 🔁 | 🔁 | 🔁 | 🔁 |

每个isolate都拥有自己的`Event Loop`和内存, 即便是原本的Isolate是这个新的Isolate的母体,它也无法访问.

**它们之间唯一的合作方式,就是来回传递消息.**

| isolate | <-- message --> | isolate |
| --- | --- | --- |


> 我感觉flutter中的root isolate有点像iOS的主线程,其他isolate有点像iOS中的子线程,并且他们在不同的线程中都有类似Runloop之类的东西,Flutter中称之为Event loop

需要注意的是,iOS中的线程是共享内存的,而Flutter中的isolate是不共享内存的,猛一听感觉很严格,特别是之前如果使用Java或者C++这样的语言,但是他对Dart的编码器是有一些好处的,比如Isolate的内存分配和垃圾回收, 不需要锁定,它只有单一线程,因此,如果它不忙,知道内存没有被改变,这对Flutter程序非常有用,它有时需要快速构建和拆除一堆小组件

> Isolate是Dart/Flutter中的一个并发模型，它提供了一个独立的内存空间，用于执行代码。每个Isolate都有自己的堆内存和垃圾回收器。当你创建一个新的Isolate时，它会分配一定数量的内存，用于存储代码和数据。当Isolate不再需要时，它会被垃圾回收器回收。


#### Isolate.spawn
```dart
import 'dart:isolate';

void main() async {
  /**
   * 这里是原本的isolate
   */

  // 创建一个用于通讯的对象,用于监听回调消息
  final receivePort = ReceivePort();
  receivePort.listen((message) {
    print('收到: $message');
    // flutter: 收到: {id: 123, name: Chris}
  });

  // 通过Isolate.spawn创建一个新的isolate(隔离环境)
  // 并指定调用的方法及所需传参
  await Isolate.spawn(_getUserName, receivePort.sendPort);

}

void _getUserName(SendPort sendPort) {
  /**
   * 这里是新的isolate
   */
  print('欢迎来到新的isolate');
  print("正在996的为您处理工作...请稍后...");
  // 假装处理耗时操作(其实在摸鱼)
  Future.delayed(const Duration(seconds: 3)).then((value) {
    // 假装处理完成了,通知原本的isolate
    sendPort.send({"id": 123, "name": "Chris"});
  });
}
// 欢迎来到新的isolate
// 正在996的为您处理工作...请稍后...
// 收到: {"id": 123, "name": "Chris"}
```


## Event Loop

Event Loop是一个循环，它会不断地从事件队列中取出事件并处理它们。在Flutter中，Event Loop是由Dart运行时提供的。当你创建一个Flutter应用程序时，Dart运行时会自动创建一个Event Loop。Event Loop会处理所有的事件，包括用户输入、网络请求、定时器等等。

> 这不跟iOS的Runloop差不多嘛😅

## Futures

在Flutter中，Future是一个异步操作，它表示一个可能在未来完成的任务。当您需要进行一些耗时的操作时，例如从服务器获取数据或者读取本地存储的文件，这些任务可能需要一些时间才能完成。如果在同步模式下执行这些操作，那么应用程序将会被阻塞，直到操作完成。而使用Future可以让您在等待操作完成的同时，继续执行其他任务。

> 大佬说这个Future就类似一个"小盒子",它里面装着的是你未来完成任务的结果.只是你暂时还不能打开它,等到合适的时机,你才能打开,看到你的"礼物"

```dart
import 'package:http/http.dart' as http;

final myFuture = http.get(Uri.parse('https://jsonplaceholder.typicode.com/albums/1'));

// 这里then方法就是通知你可以打开"盒子"的时机
myFuture.then((resp) {
    // 我看到了我的"礼物"
    print(resp.body);
});
```
至于这个礼物是什么时候放进去的,这个就要问Event Loop了,当它等到了返回的结果时,就会用放到"盒子"中,并通知你可以打开使用了.

你也可以通过构造函数创建

```dart
final myFuture = Future( () {
  print("创建一个Future");
    return 12;
  },
);
print("完成");
// 打印顺序:
// 1.完成
// 2.创建一个Future
```
这个顺序说明,,你确实获得了一个"盒子"(myFuture),但是它是一个未完成的Future

> 大佬说: 你先拿着这个,稍后我会继续运行你的函数,并给你在盒子里提供一些数据.


如果你知道了Future的值,也可以直接使用`Future.value()`来直接给构造函数命名,尽管如此,Future依然是已异步方法完成,相当于你原地打开了"盒子"😅

```dart
final myFuture = Future.value(12);
```

还有一个原地抛异常的方式

```dart
final myFuture = Future.error(Exception());
```

#### Future.delayed

指定一段时间之后回调
这是一个比较常用的构造函数,类似iOS的,GCD after.

比如你需要模拟某个耗时操作,可以先写个假的等待之类的.

或者延迟多久然后干什么之类的

```dart
print("111");
Future.delayed(const Duration(seconds: 2), () {
  print("222");
});
// 111
// 2秒后
// 222
```
总之Future和三种状态
* 未完成
* 完成有值
* 完成有异常

如果我想带一个返回值,那么可以这么做

```dart
print("111");
Future.delayed(const Duration(seconds: 2), () {
  print("222");
  return "Hello";
});
```
***但是这么做的话,我该如何获得这个返回值呢?***

你可以使用then()函数来获取.

```dart
print("111");
Future.delayed(const Duration(seconds: 2), () {
    print("222");
    return "333";
}).then((value) {
    print(value);
});
// 111
// 222
// 333
```

***如果你的延迟处理遇到了异常咋办?***

这里要用到catchError

```dart
  print("111");
  Future.delayed(const Duration(seconds: 2), () {
    print("222");
    // return "333";
    // 写死一个异常
    throw "出错啦";
  }).then((value) {
    // 这里不会进入
    print(value);
  }).catchError((error) {
    print(error);
  });
  // 111
  // 222
  // 出错啦
```
catchError和then的工作原理都一样.

由此可见,这符合Future的三个状态

***最后还有一个回调适用成功之后,无论有值还是有异常都会进入***

**whenComplete()**

```dart
  print("111");
  Future.delayed(const Duration(seconds: 2), () {
    print("222");
    // return "333";
    throw "出错啦";
  }).then((value) {
    print(value);
  }).catchError((error) {
    print(error);
  }).whenComplete(() {
    print("完成了");
  });
```

这有点像 try-catch-finally吧.


#### FutureBuilder

FutureBuilder是Flutter中的一个Widget，它可以帮助您在等待异步操作完成时，显示一些占位符或者加载指示器。当异步操作完成后，FutureBuilder会自动更新UI，以显示异步操作的结果。

这玩意很好用

```dart
class MyWidget extends StatefulWidget {
  const MyWidget({super.key});

  @override
  State<MyWidget> createState() => _MyWidgetState();
}

class _MyWidgetState extends State<MyWidget> {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
        appBar: AppBar(
          title: const Text('Title'),
        ),
        body: Center(
          child: FutureBuilder(
            // 设置一个请求
            future: _fetchNetworkData(),
            // builder来构建组件
            // 注意该builder会多次调用
            // 你需要根据snapshot.connectionState来获得当前步骤
            builder: (context, snapshot) {
              print(snapshot.connectionState);
              //flutter: ConnectionState.waiting
              //flutter: ConnectionState.done

              // 当waiting状态时,我们返回一个占位组件
              if (snapshot.connectionState == ConnectionState.waiting) {
                return const Text("正在加载...");
              }
              
              // 如果有error,这进行error小组件展示
              if (snapshot.hasError) {
                return Text("错误: ${snapshot.error}");
              }

              // 如果有数据
              if (snapshot.hasData) {
                return Text(snapshot.data as String);
              }

              // done之后如果没数据,显示这个组件
              return const Text("无数据");
            },
          ),
        ));
  }

  // 请求一个接口,获取数据
  Future<String> _fetchNetworkData() async {
    final resp = await http
        .get(Uri.parse('https://jsonplaceholder.typicode.com/albums/11'));
    if (resp.statusCode == HttpStatus.ok) {
      return resp.body;
    } else {
      throw "请求失败 ${resp.statusCode}";
    }
  }
}
```

## Streams

在Flutter中，Stream是一个抽象类，用于表示一序列异步数据的源。它是一种产生连续事件的方式，可以生成数据事件或者错误事件，以及流结束时的完成事件。Stream通常用于处理连续的异步操作，例如从服务器获取数据或者读取本地存储的文件。
  
```dart
// 在Dart中，async*是一种特殊的函数类型，它可以用来创建一个异步生成器。
// 异步生成器是一种可以生成多个值的异步操作，这些值可以通过yield语句来产生。
// 在异步生成器中，async关键字表示这是一个异步函数，而*表示这是一个异步生成器。
// 与普通的异步函数不同，异步生成器可以使用yield语句来产生多个值，而不是只产生一个值。
Stream<int> countStream() async* {
  for (int i = 1; i <= 10; i++) {
    await Future.delayed(const Duration(seconds: 1));
    yield i;
  }
}
```
```dart
final myStream = countStream();
// 订阅
final subscription = myStream.listen(
    (event) {
        print("数据 = $event");
    },
);
/*
flutter: 数据 = 1
flutter: 数据 = 2
flutter: 数据 = 3
flutter: 数据 = 4
flutter: 数据 = 5
flutter: 数据 = 6
flutter: 数据 = 7
flutter: 数据 = 8
flutter: 数据 = 9
flutter: 数据 = 10
*/
```

**注意看**

通常`myStream.listen`只能被单一订阅,如果你同时进行多次订阅,后面就会报错.

```dart
final myStream = countStream();

  final subscription = myStream.listen(
    (event) {
      print("数据 = $event");
    },
  );
  
  
  /*
  出现异常。
StateError (Bad state: Stream has already been listened to.)
  */
  final subscription2 = myStream.listen(
    (event) {
      print("数据2 = $event");
    },
  );
```

**但是如果非要对一个Stream进行多次监听,需要将Stream转化为广播Stream**

#### asBroadcastStream()

```dart
// 转为广播Stream
  final myStream = countStream().asBroadcastStream();

  final subscription = myStream.listen(
    (event) {
      print("数据 = $event");
    },
  );
  final subscription2 = myStream.listen(
    (event) {
      print("数据2 = $event");
    },
  );
/*
flutter: 数据 = 1
flutter: 数据2 = 1
flutter: 数据 = 2
flutter: 数据2 = 2
flutter: 数据 = 3
flutter: 数据2 = 3
flutter: 数据 = 4
flutter: 数据2 = 4
flutter: 数据 = 5
flutter: 数据2 = 5
flutter: 数据 = 6
flutter: 数据2 = 6
flutter: 数据 = 7
flutter: 数据2 = 7
flutter: 数据 = 8
flutter: 数据2 = 8
flutter: 数据 = 9
flutter: 数据2 = 9
flutter: 数据 = 10
flutter: 数据2 = 10
*/
```

和之前的Future一样,你也可以通过添加onError方法,来捕获异常.

#### onError

```dart
Stream<int> countStream() async* {
  for (int i = 1; i <= 10; i++) {
    await Future.delayed(const Duration(seconds: 1));
    // 到5时,报一个异常
    if (i == 5) {
      yield* Stream.error("出现异常了!!");
    }
    yield i;
  }
}
```
```dart
  final myStream = countStream();
  final subscription = myStream.listen(
    (event) {
      print("数据 = $event");
    },
    onError: (error) {
      print("错误: $error");
    },
  );
/*
flutter: 数据 = 1
flutter: 数据 = 2
flutter: 数据 = 3
flutter: 数据 = 4
flutter: 错误: 出现异常了!!
*/
```

我们看到当遇到异常时,Stream也停掉了.
这是因为有一个**cancelOnError**的属性
该属性,默认是true.

还有一个**onDone**属性,可以在Stream完成时,调用
如果我们没有将cancelOnError设置为false,那么遇到异常停止Stream之后也不会进入onDone

```dart
Stream<int> countStream() async* {
  for (int i = 1; i <= 10; i++) {
    await Future.delayed(const Duration(milliseconds: 200));
    if (i == 5) {
      yield* Stream.error("出现异常了!!");
    }
    yield i;
  }
}
```
```dart
  final myStream = countStream();
  final subscription = myStream.listen(
    (event) {
      print("数据 = $event");
    },
    onError: (error) {
      print("错误: $error");
    },
    cancelOnError: true,
    onDone: () {
      print("完了");
    },
  );
/*
flutter: 数据 = 1
flutter: 数据 = 2
flutter: 数据 = 3
flutter: 数据 = 4
flutter: 错误: 出现异常了!!
flutter: 数据 = 5 ❓❓❓
flutter: 数据 = 6 
flutter: 数据 = 7
flutter: 数据 = 8
flutter: 数据 = 9
flutter: 数据 = 10
flutter: 完了
*/
```

这里发现我们的数据在出现异常时,依然把数据5给返回了,这是因为yield没有跳出的效果,因此要调整一下代码

```dart
Stream<int> countStream() async* {
  for (int i = 1; i <= 10; i++) {
    await Future.delayed(const Duration(milliseconds: 200));
    if (i == 5) {
      yield* Stream.error("出现异常了!!");
    } else { // 使用else来分离
      yield i;
    }
  }
}
/*
flutter: 数据 = 1
flutter: 数据 = 2
flutter: 数据 = 3
flutter: 数据 = 4
flutter: 错误: 出现异常了!!
flutter: 数据 = 6
flutter: 数据 = 7
flutter: 数据 = 8
flutter: 数据 = 9
flutter: 数据 = 10
flutter: 完了
*/
```

#### Stream 的灵活使用

* 可以将Stream中的元素进行转换为String

```dart
  final myStream = countStream().map((event) {
    return "String $event";
  });
  final subscription = myStream.listen(
    (event) {
      print("数据 = $event");
    },
    onError: (error) {
      print("错误: $error");
    },
    cancelOnError: false,
    onDone: () {
      print("完了");
    },
  );
  /*
  flutter: 数据 = String 1
flutter: 数据 = String 2
flutter: 数据 = String 3
flutter: 数据 = String 4
flutter: 错误: 出现异常了!!
flutter: 数据 = String 6
flutter: 数据 = String 7
flutter: 数据 = String 8
flutter: 数据 = String 9
flutter: 数据 = String 10
flutter: 完了
  */
```

* 只打印偶数,并转换字符串

```dart
  final myStream = countStream().where((event) {
    return event % 2 == 0;
  }).map((event) {
    return "String $event";
  });
  final subscription = myStream.listen(
    (event) {
      print("数据 = $event");
    },
    onError: (error) {
      print("错误: $error");
    },
    cancelOnError: false,
    onDone: () {
      print("完了");
    },
  );
/*
flutter: 数据 = String 2
flutter: 数据 = String 4
flutter: 错误: 出现异常了!!
flutter: 数据 = String 6
flutter: 数据 = String 8
flutter: 数据 = String 10
flutter: 完了
*/
```

#### 创建自己的Stream

大多数情况,你都是使用系统或者第三方的库给你封装好的Stream供你使用.但是你也可以自己创建属于自己的Stream

```dart
class MyStream {
  int _count = 1;
  final _controller = StreamController<int>();
  
  // 一个公共属性,以便其他对象可以订阅它
  Stream<int> get stream {
    return _controller.stream;
  }

  MyStream() {
    Timer.periodic(const Duration(milliseconds: 100), (timer) {
      // 向stream中添加数据,并对外显示
      _controller.sink.add(_count);
      // 计数增加
      _count++;
    });
  }
}
```


#### 如何在Flutter中配合组件使用 
#### StreamBuilder

还记的之前的FutureBuilder吗?

这次我们要使用的是StreamBuilder, 用法类似

```dart
class _MyWidgetState extends State<MyWidget> {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
        appBar: AppBar(
          title: const Text('Title'),
        ),
        body: Center(
          child: StreamBuilder<int>(
            stream: countStream(),
            builder: (BuildContext context, AsyncSnapshot<int> snapshot) {
              if (snapshot.hasError) {
                return Text('Error: ${snapshot.error}');
              } else if (!snapshot.hasData) {
                return const CircularProgressIndicator();
              } else {
                return Text('Count: ${snapshot.data}');
              }
            },
          ),
        ));
  }

  // 返回Stream
  static Stream<int> countStream() async* {
    for (int i = 1; i <= 10; i++) {
      await Future.delayed(const Duration(seconds: 1));
      yield i;
    }
  }
}
```

通过这段代码,可以看出:
* 首先当第一次进入countStream的循环时,会有await的1秒等待时间,这个恰好对应`else if (!snapshot.hasData)`,此时我们显示一个Loading组件
* 当第一次yield回数据时,我们使用Text组件来显示返回结果

UI显示效果是Count: 数字,从1变到10

**假设我想做一个倒计时的显示效果,是不是可以用这个Stream来实现呢?**


## Async / Await

在Flutter中，async和await是用于异步编程的关键字。async用于标记一个函数是异步的，而await用于等待异步操作完成。

当您使用async关键字标记一个函数时，您可以在函数中使用await关键字来等待异步操作完成。这使得您可以使用同步代码结构来编写异步代码，从而使代码更易于阅读和理解。

第一次看到这个标记时,我记得还是在swift中.当时一脸懵.感到很困惑,这和我们在iOS中使用GCD+block的方式有很大的不同.后来做golang的同事说这叫协程.

不知道你们是否见过block的回调地狱,各种嵌套.给阅读造成了很大的困惑.这里说一下,如果flutter或者swiftUI这种描述性构建UI的方式,不做合理拆分的话,也是一种阅读的灾难.

**async/await,实际上就是Futures和Streams的替代语法**这可以帮助你编写更清晰,更易读的代码



```dart
/// 使用Future的方式来操作

// 从磁盘读取数据
Future<int> _loadFromDisk() {}
// 请求后台数据
Future<String> _fetchNetworkData(int id) {}

Future<ProcessedData> createData() {
    return _loadFromDisk().then((id) {
        return _fetchNetworkData(id);
    }).then((data) {
        return ProcessedData(data);
    });
}
```

尽管使用链式调用看起来还不错吧.但是感觉还是不太直接.
最好能像同步那样,从上到下一条龙服务.读起来最简单,也是最符合人类阅读习惯的

**如果同同步方式写的话, 是不是阅读起来更加清晰了呢?**
```dart
ProcessedData createData() {
    final id = _loadFromDisk();
    final data = _fetchNetworkData(id);
    return ProcessedData(data);
}
```

**关键在于你使用Async/Await就真的可以这么写🤑**

* 首先先标记createData方法为一个异步方法, 也是说明这里我要使用await

```dart
ProcessedData createData() async {
    final id = _loadFromDisk();
    final data = _fetchNetworkData(id);
    return ProcessedData(data);
}
```

* 接下来开始加入await,代表我要异步等待方法执行结果.

```dart
ProcessedData createData() async {
    // 等待_loadFromDisk返回值
    final id = await _loadFromDisk();
    // 等待_fetchNetworkData返回值
    final data = await _fetchNetworkData(id);
    // 最终创建ProcessedData对象返回
    return ProcessedData(data);
}
```

怎么样这样是不是看起来更清爽了.它看起来就像是我们把一件一件的事情,按顺序做下来.

如果您要try-catch代码,也比较方便

```dart
// 和同步代码的try-catch处理方式一致
ProcessedData createData() async {
    try {
        final id = await _loadFromDisk();
        final data = await _fetchNetworkData(id);
        return ProcessedData(data);
    } on HttpException catch (error) {
        print("error : $error");
        return ProcessedData.empty();
    } finally {
        print("done");
    }
}
```


#### 不太常见的,但你可以这么用

```dart
// 一个简单的把数字都加起来的方法
int getTotal(List<int> numbers) {
    int total = 0;
    for (final value in numbers) {
        total += value;
    }
    return total;
}
```

但如果这组数字是从Stream中获取的呢?按照他们异步到达的顺序做求和,当这个Stream结束的时候,返回求和值

```dart
int getTotal(Stream<int> numbers) {
    int total = 0;
    for (final value in numbers) {
        total += value;
    }
    return total;
}
```

修改如下:

```dart
Future<int> getTotal(Stream<int> numbers) async {
    int total = 0;
    // 等待数据到达
    // 第一个数据来了循环一次
    // 然后等待下一个数据来
    await for (final value in numbers) {
        total += value;
    }
    // 直到Stream结束,await也结束了,这里返回求和值
    return total;
}
```

好了!!


# Generators

在Flutter中，Generator functions是一种特殊的函数，它们可以在需要时生成一个序列。Generator functions使用yield关键字来生成值，而不是return关键字。当函数执行到yield语句时，它会返回一个值并暂停执行，直到下一次调用它时继续执行。这使得它们非常适合于处理大量数据或需要延迟计算的数据。

**它们可以进行同步或者异步操作**

|  | Single Value | Zero or more values |
| --- | --- | --- |
| sync | int | Iterable\<int> |
| async | Future\<int> | Stream\<int> |


### 同步生成器


```dart
abstract class Iterator<E> {
    bool moveNext();
    E get current;
}

class MyString extends Iterator<String> {
    final List<String> strings;
    MyString(this.strings);
    
    Iterator<String> get iterator {
        return strings.iterator;
    }
}

void main() {
    final myStrings = MyStrings([
        "1",
        "2",
        "3",
    ]);
    // 你可以使用for循环遍历
    // dart会自动为我调用moveNext和current
    for (final str in myStrings) {
        print(str);
    }
    
    // 可以配合map,where等使用,如:
    final lengths = myStrings.map( (s) {
        return s.length;
    });
    for (final length in lengths) {
        print(length);
    }
}
```

#### 如何创建一个Iterables的生成器呢?


假设有这个一个方法, 它会生成一系列数字,在你给定的范围内
```dart
// sync* 是标记为同步生成多个值,
Iterable<int> getRange(int start, int finish) sync* {
    for (int i = start; i <= finish; i++) {
        // yield标示生成该值
        // yield有点像return,但是它并没有结束的能力(之前Stream的例子中已经踩坑了)
        // 它仅提供单个值,并等待调用者请求下一个值,如此重复.
        yield i;
    }
}
```

由于返回值是Iterable,所以你可以使用for/where循环等常规操作

```dart
Iterable<int> getRange(int start, int finish) sync* {
    for (int i = start; i <= finish; i++) {
        yield i;
    }
}

void mian() {
    final numbers = getRange(1, 10).where((num) {
        return num % 2 == 0;
    });
    
    // numbers.forEach(print);
    for (int val in numbers) {
        print(val);
    }
}
```

#### yield 和 yield* 和 function*


```dart
/*
当你在函数声明中使用 * 时，如"function*"，它会将函数声明为生成器函数。
生成器函数是一种特殊类型的函数，它可以在执行期间暂停和恢复。
当你调用生成器函数时，它会返回一个迭代器对象，
该对象可以用于迭代生成器函数中的值。
*/
function* generator() {
  /*
  当你使用 yield 时，你只能返回一个值。
  例如，你可以使用 yield 返回一个数字或字符串。
  */
  yield 1;
  /*
  但是，当你使用 yield* 时，你可以返回多个值。
  例如，你可以使用 yield* 返回一个数组或另一个生成器函数的值。 
  */
  yield* [2, 3, 4];
  yield 5;
}

const iterator = generator();

console.log(iterator.next().value); // expected output: 1

console.log(iterator.next().value); // expected output: 2

console.log(iterator.next().value); // expected output: 3

console.log(iterator.next().value); // expected output: 4

console.log(iterator.next().value); // expected output: 5

```

### 异步生成器

#### 返回Future\<int> 

假设有我们有一个网络请求方法,我们传入一个值,然后服务器会将该值翻倍并返回

```dart
// 从服务器获取翻倍值
Future<int> fetchDouble(int val) async {
  // 假装在请求
  await Future.delayed(const Duration(seconds: 2));
  return val * 2;
}

void main() {
    fetchDouble(1).then(print);
    // flutter: 2
}
```

#### 返回Stream\<int>

假设我们要返回多个值呢?
这看起来和之前`sync`很像,我们只需要将`async`改为`async*`

```dart
// 从服务器获取翻倍值
Future<int> fetchDouble(int val) async {
  // 假装在请求
  await Future.delayed(const Duration(milliseconds: 200));
  return val * 2;
}

// 连续请求,并按需依次返回所需的值
Stream<int> fetchDoubles(int start, int finish) async* {
  for (int i = start; i <= finish; i++) {
    yield await fetchDouble(i);
  }
}

void main() {
  fetchDoubles(1, 10).listen(print);
}
/*
flutter: 2
flutter: 4
flutter: 6
flutter: 8
flutter: 10
flutter: 12
flutter: 14
flutter: 16
flutter: 18
flutter: 20
*/
```

