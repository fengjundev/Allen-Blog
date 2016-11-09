---
title: React Native for Android 接入实践
date: 2016-10-29 22:58:55
tags:
- react native
- android
categories: 
- android
- react native
---

![](http://ww3.sinaimg.cn/large/006tNbRwgw1f9acb627x1j30ce09baau.jpg)

公司团队从今年5月份开始尝试接入**React Native for Android**并实现业务落地。经过两个多月的努力，目前已形成配合插件化形成的一整套RN容器框架，并完成了金融旗舰店、财经资讯和安金购等业务模块的产品化落地。本文主要记录在进行RN接入的过程中的一些实践和经验。

<!-- more -->

> 出处： *[Allen's Zone](http://allenfeng.com/)*
> 作者： *Allen Feng*

## 前言

FaceBook于2015年9月份开始推出 React Native For Android 版本，加上此前已经开源的iOS版本，React Native真正成为跨平台的客户端框架。这套框架可以让开发者只使用JavaScript代码就可以构建一个跨平台的App。Facebook的官方说法是**Learn once, write anywhere**，即在Android、iOS和Web这几个平台上画UI和写业务逻辑的方式都大致相同，当然，要达到代码复用，还需要额外的适配工作。

相对于目前团队所使用的Native和H5开发方案，React Native具有调试方便、代码复用度高的特点，并且能在维持Web开发节奏的同时，具有和Native版本一致的用户体验。

基于此，一账通团队从今年5月份开始尝试接入React Native for Android并实现业务落地。经过两个多月的努力，目前已形成配合插件化形成的一整套RN容器框架，并完成了金融旗舰店、财经资讯和安金购等业务模块的产品化落地。

本文主要记录在进行RN接入的过程中的一些实践和经验，供大家参考。

## 接入方案

接入RN的常规姿势是直接在工程中添加`compile "com.facebook.react:react-native"`依赖进行接入，但是俗话说，掌握源码才能掌握主动性，对RN进行一些定制化的修改，所以我们需要直接集成RN4A源码。

出于轮子复用、特殊需求以及追求更高性能表现的原因，RN开发过程中，常常会不可避免的需要导出自己的·Native Modules·或者·Native UI Components·来满足业务需求，此时的js层代码就对Native层代码形成了强依赖，影响热更新的部署，为了达到js业务代码和Native能力支持层可以同时动态下发，我们决定将RN源码以及定制化NativeModule以插件化的形式接入一账通。

这样可以满足js业务代码和RN framework的同时热更新。避免了js层因为引用了新的Natve Module而无法进行热更新的尴尬处境。

基于这样的思路，最终实现了以下RN容器：

![RN容器](http://ww3.sinaimg.cn/large/006tNbRwgw1f99jgtht6uj30zu1427c5.jpg)

如上所示，一账通Android RN容器整体分为三层。

### RN框架层
这一层主要是集成了RN4A的源码，并解决了与宿主一些编译期依赖冲突的问题，为了满足一账通现有的开发需求，同时定制了部分的NativeModule和NativeView导出至js层使用。最终给宿主暴露的是`ReactRootView`的一个包装类`ReactViewWrapper`，里面主要包含了RN页面的生命周期回调和获取ReactRootView的方法。

### 插件桥接层
RN插件与一账通主工程在编译期是完全独立，不相互依赖。这一层主要用于桥接一账通主工程（宿主）与RN插件之间的通讯。主要通讯原理是：宿主和RN插件持有同一个`PluginMethodRegistry`注册表，里面声明了所有需要桥接的方法的名称，方法的具体实现者为宿主，插件在需要调用宿主能力的时候，通过反射获取到宿主内部一个名为的`PluginInvoker`的实例，通过`PluginInvoker.invoke(methodName, paramsJsonStr, callback)`来调用宿主里相应的方法实现类，从而达到调用宿主能力的目的。

![插件桥接层](http://ww2.sinaimg.cn/large/006tNbRwgw1f99jic944lj31ai0u4tdc.jpg)

以上两层最后会整体编译成为一个RN插件APK输出给一账通使用。

### APP层
一账通启动后，会先使用插件框架去加载RN插件，加载完成后会调用插件中暴露的接口预加载ReactContext和JsBundle，减少进入RN页面时的白屏时间。
当用户手动点击进入RN页面时，由RN插件初始化并返回一个`ReactViewWrapper`，一账通里的Activity或者Fragment直接使用`ReactViewWrapper.getView()`方法获取到的View作为rootView，待ReactRootView渲染完成后，RN页面就呈现在屏幕上了。

整体运作流程如下：

![运作流程](http://ww4.sinaimg.cn/large/006tNbRwgw1f99jjdsfj8j30xi0n0wi8.jpg)

## 一些问题和解决方案

### 版本兼容问题
一账通目前所支持的minSdkVersion是14，而这与RN所支持的最小版本16是有冲突的，一账通接入RN必须处理这一版本兼容问题。
经过权衡，目前版本的兼容方案是：

- 修改RN源码编译脚本中的minSdkVersion为14
- 然后在一账通内部执行**动态判断**，若SDK版本为16及以上，加载新的RN页面，否则，加载的仍然是该业务上一版本的Native页面

当然，从长远来看，这样的兼容方案会增加以后的开发工作量，例如，当一个业务是全新接入的时候，为了达到兼容的目的，就需要同时开发一个RN页面和H5（或者Native）页面，这无疑是对开发资源的一个浪费，所以我们预计在下一版本直接提升minSdkVersion的版本至16。

### 安全性问题
由于JSBundle存在被拦截替换的风险，一旦被恶意替换，会导致Crash等严重问题。针对此问题，我们实现了一套JSBundle的打包方案（见[Android React Native 打包与签名实践](http://allenfeng.com/2016/10/29/android-react-native-packaging/)），在打包过程中对bundle文件进行加签处理，App在Bundle安装期间会对该文件验签，只有通过验证的JSBundle才会继续被加载。

### 包大小问题
一开始，打包好的RN插件APK大小为8M，这显然是不可接受的。所以，我们对RN插件进行了一些精简的工作：
- 移除x86的支持，只保留armeabi的so
- 删除多语言支持等无用的res文件
- 对于宿主原有的已引入的依赖包，在编译RN插件后移除classes.jar中相关的类，与宿主共用一份
- 改造并移除RN中Fresco的依赖，转为使用宿主已有的图片加载框架
- 改造并移除RN对OkHttp的依赖，转为使用一账通已有的网络框架
- 改造并移除RN中的OkHttp框架，采用一账通现有网络框架
经过前三项的工作，现在的RN插件包减小到了1M左右。

### 稳定性问题
接入过程中，遇到由RN引起的Crash主要有两种：

- so不兼容导致的Crash(如`Caused by: java.lang.UnsatisfiedLinkError: dlopen failed..`)：移除x86的so包后，Release包在x86机型上存在兼容性问题，针对这个问题我们通过修改RN源码中的SoLoader得到了解决。
- JavaScript层由于view property更新导致的Crash：某些时候，在确认js层写法无误的情况下，js属性类型转换还是会莫名其妙的出错，出错后会交由Native Module层来处理。官方原有的处理方式是由一个`DefaultNativeModuleCallExceptionHandler`来接管，该Handler直接抛出了**RuntimeException**！所以我们直接实现自己的`NativeModuleCallExceptionHandler`，将收到的js层错误进行静默处理，并将异常数据采集上报，避免了因为js层一言不合引起的Crash。

### 首屏加载白屏问题：
实际体验时发现，RN页面在进行初次加载的时候，会有短暂的白屏问题，平均为1s左右，严重时可达到3s，非常影响用户体验。经过定位，我们发现RN加载耗时主要集中在ReactContext的初始化和JsBundle的加载上，对此，我们做了下面的优化工作：

- 全局共用一个`ReactInstanceManager`
- 打开App后马上对ReactContext进行预加载操作

经过优化，目前的Release包中的RN页面基本可以达到秒开的程度。当然，还可以进行ReactRootView层级的预加载，但是实际操作下来，这样对内存消耗非常大，而且目前一账通内使用RN的几个页面并不是第一个展现在用户眼前的页面，进行View层级的预加载，可能会得不偿失，为了保证内存与首屏加载速度之间的平衡，我们最终只做了ReactContext的初始化和JsBundle的预加载工作。

## 总结

RN容器的开发过程中，我们同步开展了业务开发工作--进行金融旗舰店和财经资讯模块的RN改造，整体实践下来，并没有出现此前担忧的Android和iOS代码会出现两套的情况，恰恰相反，两个平台维护的js代码是同一套。适配的主要工作量主要在于平台特有的API和定制化的Native Module补齐上，此外，业务层次的代码可达到完全复用。

所以，这对于提高整个移动开发团队的效率和节约开发资源上是有着巨大意义的。

RN提供了热更新的另一种思路，虽然仍然会依赖部分NativeModule的实现，但是我们将这两部分整体进行插件化接入，可以实现js业务能力和Native能力的动态下发，是对现有大多数RN接入方案的一个拓展。