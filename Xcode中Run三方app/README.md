# Xcode中Run第三方app入门

### 本实践只用于学习

* 准备一个砸壳重签的app,[砸壳重签名教程](https://github.com/qixin1106/DevelopmentNotes/blob/master/iOS砸壳实践/README.md)

* Xcode创建一个和砸壳app同名的项目, 同名指的是和砸壳app包中的可执行二进制文件同名.

* Bundle id 两边要保持一致,因为你砸壳时就可以修改bundle id,总之保持一致就行

* `Build Phases`中添加一个`New Run Script`

```shell
#!/bin/sh
cp -f -r [砸壳app路径] [copy到路径]
```
> cp: 使用copy命令.
> -f: 强制覆盖. 
> -r: 递归文件夹下所有文件.
> path1: 你砸壳app包的路径
> path2: copy到你build工程生成的包的位置

这个`New Run Script`的运行的时机是,build完成后,install到手机之前.正好满足要求.

***综上所述: 我们主要是通过脚本在编译之后,使用砸壳app替换项目编译的app,达到转换的效果.***

### 备注

如果该app没有做反调试的话,应该可以run起来了.


