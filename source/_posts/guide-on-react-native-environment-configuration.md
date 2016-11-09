---
title: React Native 开发环境配置指南
date: 2015-11-05 21:49:13
tags: react native
categories: react native
---

![](http://ww3.sinaimg.cn/large/006y8mN6gw1f9hkos04udj30dv0dr40f.jpg)

<!-- more -->

> 出处： *[Allen's Zone](http://allenfeng.com/)*
> 作者： *Allen Feng*

## 安装Homebrew
`Homebrew`是Mac OSX上的一个软件包管理工具，能在Mac中方便的安装或者卸载软件.

在终端执行以下命令：

```shell
➜  ~  ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```
之前安装过的同学可执行以下命令检查下更新
```shell
➜  ~  brew update && brew upgrade
```

## 安装nvm
`Node Version Manager`：Node.js版本管理器

先确保`curl`已安装：
```shell
➜  ~  brew install curl
```

安装`nvm`
```shell
➜  ~  curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.29.0/install.sh | bash
```
  
  
打开文件`/etc/profile`，在末尾添加以下环境变量：
```shell
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh" # This loads nvm
```

修改好后，注意使用
```shell
➜  ~  source /etc/profile
```
使配置文件生效



在终端直接输入`nvm --version`查看是否成功
```shell
➜  ~  nvm --version
0.29.0
```

然后，使用`nvm`安装最新版本的`node.js` (这一步会很久....如果失败了，可以直接[自行下载Node.js安装](https://nodejs.org/en/download/))

```shell
➜  ~  nvm install node && nvm alias default node
```

安装好后，测试下是否成功
```shell
➜  ~  node -v
v4.2.1
➜  ~  npm -v
2.14.7
```


## 安装watchman
`watchman`是Facebook 的一个用于监控文件变更并触发指定操作的工具
```shell
➜  ~  brew install watchman
```

## 安装flow
`flow`是一个 JavaScript 的静态类型检查器，建议安装它，以方便找出代码中可能存在的类型错误：
```shell
➜  ~  brew install flow
```

## 安装react-native
先自行翻墙，或者执行以下命令加淘宝镜像(推荐后者)：
```shell
➜  ~  npm --registry https://registry.npm.taobao.org info underscore
```

安装`react-native`
```shell
➜  ~  npm install -g react-native-cli
```

## 配置Android开发环境
主要是SDK之类的配置，教程很多，此处不再赘述，这里主要提一点：
在`/etc/profile`配置一个名为`ANDROID_HOME`的环境变量，指向SDK根目录，因为在build RN工程时会用到

在`/etc/profile`加入的语句如下：
```shell
export ANDROID_HOME="/Users/fengjun/Developement/Environment/sdk/"
```



#### 至此，环境搭建全部完成



## 创建Hello World

执行`react-native init HelloWorld`，这将会创建一个名为`HelloWorld`的`React Native`工程

等待下载依赖完成后（大约一两分钟），将会看到以下输出：

```shell
This will walk you through creating a new React Native project in /Users/fengjun/Developement/react-learn/HelloWorld
Installing react-native package from npm...
Setting up new React Native app in /Users/fengjun/Developement/react-learn/HelloWorld
To run your app on iOS:
   Open /Users/fengjun/Developement/react-learn/HelloWorld/ios/HelloWorld.xcodeproj in Xcode
   Hit the Run button
To run your app on Android:
   Have an Android emulator running (quickest way to get started), or a device connected
   cd /Users/fengjun/Developement/react-learn/HelloWorld
   react-native run-android
```


工程创建成功


接下来`cd`进`HelloWorld`，执行`react-native run-android`，就可以看到你的第一个RN工程了，Have fun !

## 另附：

### Demo
[使用React Native实现的豆瓣电影](https://github.com/fengjundev/DoubanMovie-React-Native)
[基于React Native的动态部署实践](https://github.com/fengjundev/React-Native-Remote-Update)

> demo clone下来后，在工程目录下执行 `npm install` 安装依赖，再执行 `react-native run-android` 即可

### 其他教程

[Javascript教程](http://www.liaoxuefeng.com/wiki/001434446689867b27157e896e74d51a89c25cc8b43bdb3000)
[Flexbox教程](https://css-tricks.com/snippets/css/a-guide-to-flexbox/)
