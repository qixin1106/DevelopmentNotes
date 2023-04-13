# iOS第三方app分析

### 练习的目的

* 了解一些其他app用到的技术(学习)
* 找一些好玩的东西(彩蛋)

### 准备工作

* 下载app到越狱手机.通过appstore下载即可.
* 通过砸壳技术得到砸壳后的ipa,参考[iOS砸壳实践学习](https://github.com/qixin1106/DevelopmentNotes/blob/master/iOS砸壳实践/README.md)
* 准备电脑软件MachOView请自行下载.

### 查看app目录下的内容

* 将ipa,改为zip解压
* 进入Payload中,点击discover.app,右键选择"显示包内容"

#### Frameworks

* [Agoraffmpeg.framework](https://docs.agora.io/cn/live-streaming-premium-legacy/landing-page?platform=iOS)
    
> Agoraffmpeg.framework是声网Agora的一个音视频插件，它是基于FFmpeg的音视频处理库，可以用于音视频采集、编码、解码、混音、播放等。

* [AgoraRtcKit.framework](https://docs.agora.io/cn/live-streaming-premium-legacy/landing-page?platform=iOS)

> AgoraRtcKit.framework是Agora RTC SDK的一部分，它是一个iOS平台上的实时音视频通信引擎，可以为iOS应用程序提供实时音视频通信功能。

* [KasaSDK.framework](https://www.kasasmart.com/us)

> KasaSDK是TP-Link公司的智能家居设备控制SDK。它提供了一组API，使开发人员可以轻松地与TP-Link智能家居设备进行通信和控制。
> 嗯❓😳,也许搞错了,我也不太清楚.哈哈

* [TXFFmpeg.framework](https://cloud.tencent.com/document/product/881/20205)

> TXFFmpeg.framework是腾讯云的播放器SDK中的一个框架，用于在iOS平台上进行音视频播放。

* [TXSoundTouch.framework](https://cloud.tencent.com/document/product/881/81204)

> TXSoundTouch.framework是腾讯云的播放器SDK中的一个框架，用于在iOS平台上进行音频变速、变调、变声等处理。

* [ImSDK.framework](https://cloud.tencent.com/document/product/269)

> 腾讯云的即时通讯IM SDK, 即时通信（Instant Messaging，IM）基于 QQ 底层 IM 能力开发，仅需植入 SDK 即可轻松集成聊天、会话、群组、资料管理能力，帮助您实现文字、图片、短语音、短视频等富媒体消息收发，全面满足通信需要。
> 老版本中存在,新版中没有了.

* A.framework

> 和OCSP相关吗?不太清楚

#### PlugIns

* BroadcastUploadExtension.appex
* NotificationServiceExtension.appex
* ShareExtension.appex
* Siri.appex
* TodayExtension.appex
* WidgetExtension.appex

#### Bundle

* 略

> 一大坨

#### info.plist

能看到申请了一堆权限,比较奇怪的是,有个蓝牙权限,这个app我还真不知道在哪使用蓝牙?
目前能看到是,蓝牙,相机相册,通讯录,日程,定位,麦克风,Tracking,Siri,Local Network,最新的包指定是iOS10最小版本.

* LSApplicationQueriesSchemes

> 这里面是URLSchemes白名单,第一个就是Cydia,可能是探测越狱用的,里面大概有50来个吧,很多应该都是和分享有关的吧.

* URL types

> 这里面配置了7个,有一个xhsdiscover,我感觉这个可能是app自己用的跳转头
> 在MachOView中可以扫描到一些使用痕迹,比如
```js
xhsdiscover://user/%@
xhsdiscover://home
......
```
> 感觉应该是使用的路由方式跳转页面
> 里面出现了许多rn相关的页面.
> className中包含许多ViewModel结尾的类,应该是MVVM架构的产物


#### 彩蛋

* fuck的痕迹

> 1. TXVodDownloadManager中可能包含一个"fuck"
> 2. fengkong.com附近的找到一个"what_the_fuck_is_this"


#### 沙盒

### Document

* [Alpha]目录中看到了Lottie库的影子
* 其中里面有个Gifts目录,应该是一些动画特效,都是mp4的动画,但是分成左右2个,左面是原版视频,右边是黑白蒙版,这样播放时应该可以做到镂空的效果.
* [.beacon]目录,里面是一些配置,还要用beacon吗?
* [beautyConfig]目录,里面似乎是一些美颜的配置
* [PrivateMessage.db]看起来似乎是聊天消息的库

> XYPMMessageChatModel这张表看起似乎是储存你的聊天列表
> 里面记录了比如名字,关系,是否屏蔽,未读消息数,chat_id,是否免打扰,最后一条消息等等
> XYPMMessageMessageModel这张表应该记录的是具体聊天的内容
> 不过我这个是老版本的,我看到新版本中似乎没有使用IMSDK了,所以也许这个app已经修改了缓存,目前不太确定.
 
* [default.realm] Realm

### Library

#### Cache

* YYKit可以看出该app使用的图片库是YYkit而不是SDWebImage.
* cache目录中是大多是一些配置和统计相关的东西.

#### Preferences

* 这个目录应该是一些配置文件,其中包含NSUserDefaults的一些配置信息,其中看到了bugly的字样,还有rn_resource

