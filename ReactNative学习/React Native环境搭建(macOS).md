# React Native环境搭建(macOS)

[参考网站](https://reactnative.dev/docs/environment-setup?guide=native)

## Expo Go和React Native CLI的区别

- Expo Go是Expo平台提供的一种开发环境，它基于React Native构建，并提供了一系列的开发工具和服务。适合快速原型开发和初学者玩一下。

- React Native CLI是React Native官方提供的一种开发环境配置方式。需要使用React Native CLI命令行工具来创建、编译和打包React Native应用，并且需要使用原生代码进行开发和调试。比较灵活性，适用于对原生开发有更多需求的开发者。

## Expo Go安装方式

### 安装Nodejs (https://nodejs.org/zh-cn)

安装完成后，打开终端应用程序（Terminal），运行以下命令验证 Node.js 和 npm 的安装是否成功：
```shell
node -v
# v18.16.1
npm -v
# 9.5.1
```

您可以选择更新 npm 到最新版本。在终端中运行以下命令来进行 npm 更新：

```shell
npm install -g npm
```

这将全局安装最新版本的 npm

### 通过Expo Go快速开始

```shell
npx create-expo-app AwesomeProject

cd AwesomeProject

npx expo start
```

## React Native CLI安装

### 安装Nodejs和watchman

如果您已经在系统上安装了Node，请确保它是Node 16或更新版本。
```shell
brew install node
brew install watchman
```
Watchman是Facebook用于观察文件系统变化的工具。

### 安装Xcode

通过`Mac App Store`安装Xcode

### 安装Xcode命令行工具

Xcode菜单 > 设置 > Locations > Command Line Tools

### 安装iOS模拟器

Xcode菜单 > 设置 > Components 

### 安装CocoaPods

参考(https://guides.cocoapods.org/using/getting-started.html)

### 构建一个新应用程序

卸载之前的全局react-native-cli软件包(如果之前安装过的话,因为可能会导致意外问题)
```shell
npm uninstall -g react-native-cli @react-native-community/cli
```

创建一个应用
```shell
npx react-native@latest init AwesomeProject
```
