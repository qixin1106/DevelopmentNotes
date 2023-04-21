# iOS砸壳实践学习

### 为啥要学习呢

* 增加一项新技能
* 加强app功能
* 了解iOS内部原理
* 借鉴他人的app
* 分析恶意程序,进行安全评审
* 从分析点去保护app的安全性

### 为什么要加壳

iOS app加壳是为了保护应用程序的安全性。加壳是指将应用程序的二进制文件进行加密，使其难以被破解。这样可以保护应用程序的代码和数据，防止被黑客攻击和盗取。此外，加壳还可以防止应用程序被反编译，从而保护应用程序的知识产权。

### 静态砸壳

静态脱壳是指在不运行应用程序的情况下，对应用程序进行分析，找到加壳代码并将其去除的过程。

硬脱壳需要知道Apple的私钥解密，要破解苹果加密算法难度还是很大的。

### 动态砸壳

动态脱壳是指在运行应用程序时，通过调试器等工具，对应用程序进行分析，找到加壳代码并将其去除的过程。

动态脱壳是在程序启动时系统会将下载的IPA加载到内存，再由壳程序对可执行文件进行解密。

### 砸壳工具

目前选择是得[Frida](https://frida.re/),因为看网上的讯息说这个比较好使.

### iPhone上的准备工作

* 越狱

* Cydia中添加源：http://build.frida.re, 并下载frida

* usb连接到电脑

### mac电脑上的准备工作

* Python环境

* 克隆 frida-ios-dump 
 ``` shell
 git clone https://github.com/AloneMonkey/frida-ios-dump.git
 ```
* 安装环境 
```shell
sudo pip install -r requirements.txt --upgrade
```
* 安装usbmuxd 
```shell
brew install usbmuxd
```
* 建立端口映射（把 2222 映射到 22） 
```shell
iproxy 2222 22
```
> 执行完"iproxy 2222 22"后,你的窗口处于监听状态.
> 显示"Creating listening port 2222 for device port 22
waiting for connection"
> 此窗口就不用管它了

* 新开一个命令行窗口
```shell
# cd到你下载frida-ios-dump的目录
./dump.py [你要砸的程序名字] 或 [程序的bundle id]
```

> **非常重要:**
> 使用./dump.py脚本之前一定要确保你被砸壳的app启动
> 我理解的砸壳的原理就是,本机下载的app都是有壳的,启动时系统必须先解密,相当于内存中的是解密之后的程序,并且Frida砸壳时,可能还要注入代码等.

* 成功后在你的frida-ios-dump目录下,会多一个ipa(砸完壳)

* 验证
```shell
otool -l 可执行文件路径 | grep crypt
```
> cryptid 0 说明已经砸壳成功, 1 说明有壳

### frida常用命令

来源:https://frida.re/docs/examples/ios/

### 一些常用工具

**iFunBox**
用于查看手机中app的bundle资源或者沙盒文件.
app bundle path: `/var/mobile/Containers/Bundle/Application/`
sandbox path: `/var/mobile/Containers/Data/Application/`
> 注意:
> iPhone中使用`iFile`来查看本地资源
> macOS中使用iFunBox来查看,推荐macOS工具,因为iFile未注册的,查看程序时,无法显示app的名字,是显示的UUID,这样在手机中app较多时不太好找.

### class-dump导出头文件

**class-dump**
* 下载地址:https://github.com/nygard/class-dump

* 我直接使用的master

* Xcode编译target = class-dump这个.
> 其中我遇到一个报错问题CDLCBuildVersion.h文件,
>         case PLATFORM_IOSMAC:           return @"iOS Mac";
> PLATFORM_IOSMAC这个没有定义,我直接注释掉了.
> 通过!

* 编译的class-dump可执行文件移动到
```shell
# 移动到/usr/local/bin 目录中
open /usr/local/bin
# 并且更改权限
sudo chmod 777 /usr/local/bin/class-dump
```
* 导出头文件
```shell
class-dump -H [XXX.app的路径] -o [导出到某路径文件夹]
```

### 重签名

我使用[ios-app-signer](https://github.com/DanTheMan827/ios-app-signer)

* Plugins目录,和Watch目录也可以删掉,如果你不需要Extension和Watch的话.否则这俩个按说也要重签一下.我不需要调试这俩所以就直接删除了.

* 要重签的app包中的info.plist打开,修改bundle id,改成一个你自己的

* Xcode重新创建一个空项目,build之后,去你build的app包里把"embedded.mobileprovision"文件拷贝到被重签的app包中

* 打开ios-app-signer


1. input File:选择你要被重签的xxx.app包
2. Signing Certificate: 选择你本地的证书
3. Provisioning Profile: 选择Re-Sign Only
4. New Application ID: 新的bundle id(因为之间手动改了,因此不需要填)
5. App Display Name: 新的app显示名字,可以写一个,区分正常的版本.
6. App Version:build号,可以不改
7. App Short Version:版本号,可以不改
8. Ignore Plugins folder: 忽略插件目录,因为手动删除了,所以点不点都行.
9. No get-task-allow,默认选上,不用管


下面点start,会弹出一个提示框,问你新的包要放在哪里,找个地方保存就可以了.

### 安装到手机

* 通过Xcode菜单/Window/Device and Simulators/安装到手机中,如果你在上一步改过名字,那么此时手机里会出现你安装的重签名app


### 在Xcode中调试

* 先在手机中打开你重签的app,此时最好后台杀掉那个正版的app(如果存在的话),这样后面好找一些.因为虽然你之前改名了,但是改的是DisplayName,可执行程序的名字并没有修改.这里显示的是可执行程序的名字.
* Debug->Attach to Process->你的app
