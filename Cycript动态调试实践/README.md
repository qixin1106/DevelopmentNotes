# Cycript动态调试

[官方网站](http://www.cycript.org)

**你需要一部越狱手机**

## 1.SSH到iPhone

* `ssh root@192.168.0.xxx`

> ip是你局域网手机地址
> 参考: [SSH连接iPhone](https://github.com/qixin1106/DevelopmentNotes/blob/master/OpenSSH连越狱iPhone/README.md)

## 2.查看进程

* `ps -A`

> ps -A 选项用于显示所有正在运行的进程信息，包括系统进程和用户进程。

或者

* `ps aux`

> ps aux 选项用于显示所有正在运行的进程信息，包括进程的详细信息，例如进程的 CPU 占用率、内存使用情况等。

找到你需要调用的app进程,记录下`pid`或者`进程名字`

## 3.启动Cycript

* cycript -p `pid` 或者 `进程名字`

> 显示`cy#`则说明启动成功

## 常用命令

### 打印UI层级

* `[[UIApp keyWindow] recursiveDescription].toString()`

> 不加`toString()`的话,不会显示换行

```objc
<UIWindow: 0x13f6e88d0; frame = (0 0; 414 736); gestureRecognizers = <NSArray: 0x13f6ec2b0>; layer = <UIWindowLayer: 0x13f6e5dc0>>
   | <UILayoutContainerView: 0x13f6f86b0; frame = (0 0; 414 736); autoresize = W+H; gestureRecognizers = <NSArray: 0x14080c6e0>; layer = <CALayer: 0x13f6efba0>>
   |    | <UINavigationTransitionView: 0x1408050e0; frame = (0 0; 414 736); clipsToBounds = YES; autoresize = W+H; layer = <CALayer: 0x13f6ddca0>>
   |    |    | <UIViewControllerWrapperView: 0x140b15b20; frame = (0 0; 414 736); autoresize = W+H; layer = <CALayer: 0x140b15e50>>
   |    |    |    | <UIView: 0x13f6a6100; frame = (0 0; 414 736); autoresize = W+H; gestureRecognizers = <NSArray: 0x140a4f050>; layer = <CALayer: 0x13f674630>>
   |    |    |    |    | <WelcomeVideoView: 0x13f69f280; frame = (0 0; 414 736); layer = <CALayer: 0x13f6c0320>>
   |    |    |    |    |    | <AVPlayerView: 0x140863710; frame = (0 0; 414 736); autoresize = W+H; layer = <CALayer: 0x13f6a2ec0>>
   ......
```

比如在上面列表中随便找一个对象,` <KEPLoginTextField: 0x1408a2f40; baseClass = UITextField; frame = (83 14; 215 22);`

它看起来似乎是一个登录的输入框,继承自UITextField,那么我们可以通过以下方式动态修改该对象的属性.

比如修改背景颜色: 

```objc
#0x13f6e9db0.backgroundColor = [UIColor redColor]
```
> 使用`#` + `对象地址` 用于获取对象, 然后和iOS开发一致. 很好, 很强大


### NSUserDefaults

* `[[NSUserDefaults standardUserDefaults] dictionaryRepresentation].toString()`

> 读取配置

```objc
`{
    AKDeviceUnlockState = 0;
    AddingEmojiKeybordHandled = 1;
    AntialiasedFontDilationEnabled = 0;
    AppleICUForce24HourTime = 1;
    AppleITunesStoreItemKinds =     (
        podcast,
        artist,
        "itunes-u",
        booklet,
        document,
        movie,
        eBook,
        software,
        "software-update",
        "podcast-episode"
    );
    ......
}`
```



### nextResponder

* `[#0x13f69f280 nextResponder]`

```objc
cy# [#0x13f69f280 nextResponder]
#"<UIView: 0x13f6a6100; frame = (0 0; 414 736); autoresize = W+H; gestureRecognizers = <NSArray: 0x140a4f050>; layer = <CALayer: 0x13f674630>>"
cy# [#0x13f6a6100 nextResponder]
#"<KEPVideoLoginViewController: 0x13f864c00>"
cy# [#0x13f864c00 nextResponder]
#"<UIViewControllerWrapperView: 0x140b15b20; frame = (0 0; 414 736); autoresize = W+H; layer = <CALayer: 0x140b15e50>>"
cy# [#0x140b15b20 nextResponder]
#"<UINavigationTransitionView: 0x1408050e0; frame = (0 0; 414 736); clipsToBounds = YES; autoresize = W+H; layer = <CALayer: 0x13f6ddca0>>"
cy# [#0x1408050e0 nextResponder]
#"<UILayoutContainerView: 0x13f6f86b0; frame = (0 0; 414 736); autoresize = W+H; gestureRecognizers = <NSArray: 0x14080c6e0>; layer = <CALayer: 0x13f6efba0>>"
cy# [#0x13f6f86b0 nextResponder]
#"<BaseUINavigationController: 0x13f88fa00>"
cy# [#0x13f88fa00 nextResponder]
#"<UIWindow: 0x13f6e88d0; frame = (0 0; 414 736); gestureRecognizers = <NSArray: 0x13f6ec2b0>; layer = <UIWindowLayer: 0x13f6e5dc0>>"
cy# [#0x13f6e88d0 nextResponder]
#"<UIApplication: 0x13f580e10>"
cy# [#0x13f580e10 nextResponder]
#"<KAppDelegate: 0x13f676250>"
cy# [#0x13f676250 nextResponder]
null
```

> 如上所示,当我们对一个UIView对象使用`nextResponder`方法时,则按照响应链,返回对象


### UIControl action

`<KEPActionButton: 0x140963260; baseClass = UIButton; frame = (46 271.667; 322 50); clipsToBounds = YES; opaque = NO; layer = <CALayer: 0x140942090>>`

假设有这样一个按钮, 我们通过如下命令获取它的selector

* `[#0x140963260 allTargets]` 获取target

```objc
[NSSet setWithArray:@[#"<RACPassthroughSubscriber: 0x140a4b040>"]]]
```
`@property(nonatomic,readonly) NSSet *allTargets;                                           // set may include NSNull to indicate at least one nil target`


-------


* `[#0x140963260 allControlEvents]` 获取ControlEvents

```objc
64
```
> UIControlEventTouchUpInside  = 1 <<  6,

`@property(nonatomic,readonly) UIControlEvents allControlEvents;                            // list of all events that have at least one action`

-------

* `[#0x140963260 actionsForTarget:#0x140a4b040 forControlEvent:64]` 获取 selector

```objc
@["sendNext:"]
```

`- (nullable NSArray<NSString *> *)actionsForTarget:(nullable id)target forControlEvent:(UIControlEvents)controlEvent;`

-------


### choose(class)

在当前页面中查找某个类的所有实例.

* `choose(KEPActionButton)`

```objc
[#"<KEPActionButton: 0x140963260; baseClass = UIButton; frame = (46 271.667; 322 50); clipsToBounds = YES; opaque = NO; layer = <CALayer: 0x140942090>>"]
```