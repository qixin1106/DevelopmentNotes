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

### ObjectiveC.classes

打印所有类

### *#地址/*对象

打印该对象的所有成员变量

`*UIApp` 或 `*#0x14ee9e5a0`

```c
{isa:UIApplication,_hasAlternateNextResponder:false,_hasInputAssistantItem:false,_delegate:(typedef void*)(0x14eface20),_exclusiveTouchWindows:[NSSet setWithArray:@[]]],_event:#"<UIEvent: 0x14efc6b30>",_touchesEvent:#"<UITouchesEvent: 0x14efaa770> timestamp: 184630 touches: {(\n)}",......
```

## 关于.cy脚本

我们可以将一些常用的方法写到脚本里,这样使用时会比较方便.


### 一个简单的测试脚本示例

* QXTool.cy 

```js
(function(exports) {

	// 测试一个加法方法
	// exports.sum() 等价于 QXTool.sum()
	exports.sum = function(a, b) {
		return a + b
	}

	// 获取appid
	exports.appId = [NSBundle mainBundle].bundleIdentifier

	// 获取document路径
	exports.documentPath = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES)[0]

	// 获取当前rootVC
	// 使用函数调用可以是实时获取rootViewController的值
	// 如果不使用,则在导入脚本之后一次性赋值,后面将不会在改变
	exports.rootVC = function() {
		return UIApp.keyWindow.rootViewController
	}
	
})(exports)
```

##### 如`exports.appId = xxxx` 直接赋值的形式来写,则导入脚本之后一次性赋值,后面将不会在改变,类似appId这种值,基本不会改变,所以我们可以直接赋值.
##### 如`exports.rootVC = function() {}` 这种因为rootViewController在不同时期,可能会产生变化,因此我们需要使用函数调用方式,这样每次调用时,可获得当前状态下的值.


### .cy使用方法

当我们写完一个脚本之后,首先要复制到手机中,我们通过`iFunBox`或者`scp`命令来复制文件

手机文件路径:
#### `/usr/lib/cycript0.9`

![-w273](media/16842350287283/16848350353363.jpg)

你可以将脚本直接放在根目录, 也可以选择放在`com`目录中

放在`com`目录中, 需要按照域名倒序的方式命名(参考appID)

如: 

#### `/com/qixin/QXTool.cy`

这样的话,导入脚本时,需要指定位置.

### 导入脚本

```shell
cy# @import QXTool
```
> {sum:function (t,e){return t+e},appId:@"com.qixin.douyin.re",documentPath:@"/var/mobile/Containers/Data/Application/0FCBC2B4-00CE-4EB4-A936-0A26D8487CA8/Documents",rootVC:function (){return UIApp.keyWindow.rootViewController}}

看到反馈出你的注册成功,没有报错.

如果你放置脚本在`com/qixin/`这种形式,那么导入时需要注意:

```shell
cy# @import com.qixin.QXTool2
```

### 使用脚本

![-w325](media/16842350287283/16848358496469.jpg)
![-w348](media/16842350287283/16848358696169.jpg)
![-w571](media/16842350287283/16848359168228.jpg)

### 自动补全

我们可以通过按 `tab` 来激活联想(自动补全), 如果有选个选择, 会贴心的帮你列出所有建议

![-w282](media/16842350287283/16848360279082.jpg)

> 需要多按几次,第一次可能会帮你匹配一个最接近的.然后再按会帮你列出可能匹配结果.

### 固定框架

```js
(function(exports) {
    // 你的代码
})(exports)
```
> 这个是固定写法,你需要关注的是中间代码实现的部分


### 关于全局变量

我们刚才介绍了通过` exports.xxxx `方式来实现, 这种相当于成员方法,调用时,需要`QXTool.xxxx()`来调用,如果你觉得这个太麻烦了,也可以改成全局函数方式,如下:

```js
/*
exports.sum = function(a, b) {
    return a + b
}
*/

// 修改为:

QXTSum = function(a, b) {
    return a + b
}
```

那么当你调用时,可以直接通过 `QXTSum(1, 2)` 的方式来调用.
##### 为了不引起冲突,你可以参考OC命名的方式,加上你的特殊前缀即可,建议使用三个大写字母来做区分