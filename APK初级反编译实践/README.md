# apk初级反编译实践

### 为啥要反编译

之前某群网友给我安利一款国产gpt产品, 据说是阿里的, 还有ios和android的app可以使用.我先去网站看了下,应该是使用了uni-app开发的,所以移动端估计也是一样的.我先去appstore找了一下,没有找到, 感觉可能是inhouse安装的,所以还是去了网站找下载, 但是ios的下载看起来是个描述文件, 我一下就慌了, 如果是inhouse安装那么应该会跳出提示安装的alert, 于是查看一下描述文件,发现了web clip的字样, 原来不是一个真正的app, 只是在手机桌面上生成一个浏览器地址的快捷方式.还想尝试看下ipa的内容呢,没办法只有apk可以下载.

### uni-app
uni-app是一个使用Vue.js开发跨平台应用的前端框架。开发者编写一套代码，可编译到iOS、Android、微信小程序等多个平台。uni-app具有跨端数量更多、性能体验更优秀、学习成本低、开发成本低等优势。
https://zh.uniapp.dcloud.io

### webclip

webclip是苹果官方推出的一种快捷的网页桌面书签工具，简单的来说就是将您的网站生成一个桌面快捷方式图标，并添加到苹果移动设备的桌面显示。 
webclip只是一种快捷的桌面图标，虽然一眼看上去非常类似APP，但是点击图标打开之后访问的是网站，webclip是一种书签工具，并非APP。

### 准备工具

* [jdk 1.8](https://www.oracle.com/java/technologies/downloads/#java8)

> 因为我的老mac电脑本地只有1.7,要求是必须1.8及以上版本,所以需要升级一下.
> 下载的话,需要注册一下Orecle的账号.
> 可以通过`java -version`来查看本地的版本.
> 安装之后可以到`/Library/Java/JavaVirtualMachines`位置查看

* [apktool](https://ibotpeaches.github.io/Apktool/documentation/)

> apktool是一个反编译Android Apk的第三方工具。 它可以反编译资源，并在进行修改之后重新打包Apk。

> macOS可以使用homebrew安装`brew install apktool`

* [dex2jar](https://sourceforge.net/projects/dex2jar/)

> dex2jar是一个反编译apk的工具，可以将.dex文件转换成.jar文件，去查看源代码（混淆），反之也能将.jar文件转换成.dex文件。dex2jar是一个能操作Android的dalvik (.dex)文件格式和Java的 (.class)的工具集合。它可以将.dex文件转换成Java的.class文件的转换工具。反编译以后的.jar文件可以直接通过JD-GUI查看源代码（源码是混淆的）。

> macOS可以使用homebrew安装`brew install dex2jar`

* [jd-gui](http://java-decompiler.github.io)

> JD-GUI是一个独立的图形化工具，它可以显示“.class”文件的Java源代码。您可以使用JD-GUI浏览重构后的源代码，以便立即访问方法和字段。

### apktool反编译

```shell
apktool d your/path/xxx.apk
```

整完之后,会在该apk目录下生成一个同名文件夹,此时`AndroidManifest.xml`文件可以正常查看

> 其实你也可以直接将apk扩展名改为zip,然后直接解压,与iOS的ipa类似,但是这样`AndroidManifest.xml`人类无法直观查看内容.

### apktool回编译

* 略(无需求)

### dex2jar

* 首先使用zip解压方式解一下apk的包,这是解出来的文件夹中的内容和akptool解出来的不太一样, apktool解出来的都是.smali文件,而zip解出来的可以看到.dex文件

> 在Android平台上，代码一般是用Java编写的，执行Java程序一般需要用到Java虚拟机。但是出于性能上的考虑，并没有使用标准的JVM，而是使用专门的Android虚拟机（5.0以下为Dalvik，5.0以上为ART）。Android虚拟机的可执行文件并不是普通的class文件，而是再重新整合打包后生成的dex文件。dex文件反编译之后就是Smali代码，所以说，Smali语言是Android虚拟机的反汇编语言。smali和baksmali分别是指安卓系统里的Java虚拟机（Dalvik）所使用的一种.dex格式文件的汇编器，反汇编器。而jar文件则是Java Archive，java二进制归档文件，多个.class文件打包的文件。

```shell
d2j-dex2jar your/path/xxx.dex
```

解完之后,会在该.dex目录下,找到同名并加了后缀(xxx-dex2jar.jar)的jar包


### jd-gui

打开jd-gui.app(我这里是macOS平台)...提示我要扔垃圾桶,不慌,按住control键+鼠标右键.选择打开,会提示安全性授权之类的东西(基操).确定要打开.

OMG....

```
ERROR launching 'JD-GUI'

No suitable Java version found on your system!
This program requires Java 1.8+
Make sure you install the required Java version.
```

我傻傻的听从了他的描述,然后去更新JDK11,结果依然是不行.

将该问题贴到网上去查询,很容易找到了[解决办法](https://www.jianshu.com/p/ee2932b46d80)

***解决方案:***

* 在JD-GUI.app上右键选择"显示包内容", 找到/Contents/MacOS/universalJavaApplicationStub.sh,替换成下面提供的sh

* [github Issues](https://github.com/java-decompiler/jd-gui/pull/336)

* [替换的sh](https://github.com/tofi86/universalJavaApplicationStub/blob/v3.0.6/src/universalJavaApplicationStub)


-------

正常打开之后,将之前转出的jar包直接扔进去.就可以查看.

目前我看到的这个app是可以看到源码的,但是其他很多app都是经过加固混淆过的,基本直接看不到什么东西.

