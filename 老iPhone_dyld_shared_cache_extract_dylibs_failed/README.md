# 老iPhone调试dyld_shared_cache_extract_dylibs failed

* macOS (11.7.6 (20G1231))
* Xcode (13.2.1 (13C100))
* iPhone6 plus (9.0.2越狱)

我有一个越狱的iPhone6 plus, 最近插上usb,xcode会提示`dyld_shared_cache_extract_dylibs failed`

> dyld_shared_cache_extract_dylibs failed这个错误通常是由于Xcode缓存的问题导致的。您可以尝试退出Xcode，删除~/Library/Developer/Xcode/iOS DeviceSupport/文件夹中对应设备版本的文件夹，然后重新启动Xcode。

以上是网上普遍的解决方案

Finder中 `cmd+shift+g` 前往 `~/资源库/Developer/Xcode/iOS DeviceSupport/`找到你手机系统版本对应的文件夹删掉.

> Symbols文件夹是用于符号化崩溃日志的。如果您删除了该文件夹，下次连接设备时，Xcode会从设备重新下载符号数据。


不过显然对我的手机无效....

## 对我有效解决办法

替换大法,即用高版本的"com.apple.dyld"覆盖你报错版本

Finder中 `cmd+shift+g` 前往 `~/资源库/Developer/Xcode/iOS DeviceSupport/版本/Symbols/System/Library/Caches/com.apple.dyld`

