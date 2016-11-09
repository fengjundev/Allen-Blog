---
title: 谈谈Android的so
date: 2016-11-06 21:29:47
tags: 
- android
- so
- abi
categories: 
- android
---

![](http://ww3.sinaimg.cn/large/006y8mN6jw1f9hkvk9rm1j30sg0izadl.jpg)

一般情况下，我们不需要关心so。但是当APP使用的第三方SDK中包含了so文件，或者自己需要使用NDK开发某些功能，就有必要去好好了解下so的一些知识。

<!-- more -->

> 出处： *[Allen's Zone](http://allenfeng.com/)*
> 作者： *Allen Feng*

## 什么是ABI和so
早期的Android设备只支持ARMv5的CPU架构，随着Android系统的快速发展，搭载Android的硬件平台也早已多样化了，又加入了ARMv7，x86，MIPS，ARMv8，MIPS64和x86_64。
每一种CPU架构，都定义了一种ABI（Application Binary Interface，应用二进制接口），ABI定义了其所对应的CPU架构能够执行的二进制文件（如.so文件）的格式规范，决定了二进制文件如何与系统进行交互。
![ABI及支持的指令集](http://ww3.sinaimg.cn/large/72f96cbagw1f9bfd1lkw2j21gq18g42s.jpg)
每一种ABI的详细介绍可以参见官方的介绍[ABI Management](https://developer.android.com/ndk/guides/abis.html)。

so（*shared object*，共享库）是机器可以直接运行的二进制代码，是Android上的动态链接库，类似于Windows上的dll。每一个Android应用所支持的ABI是由其APK提供的.so文件决定的，这些so文件被打包在apk文件的lib/<abi>目录下，其中*abi*可以是上面表格中的一个或者多个。
例如，解压一个apk文件后，在lib目录下可以看到如下文件：
```bash
lib
|
├── armeabi
│   └── libmath.so
├── armeabi-v7a
│   └── libmath.so
├── mips
│   └── libmath.so
└── x86
    └── libmath.so
```
说明该应用所支持的ABI为armeabi, armeabi-v7a, mips, 和x86。

注：可以使用`aapt`命令快速查看apk支持的abi
```bash
~ aapt dump badging baidutieba.apk | grep abi
native-code: 'armeabi' 'mips' 'x86'
```

## 为什么使用so
- so机制让开发者最大化利用已有的C和C++代码，达到重用的效果，利用软件世界积累了几十年的优秀代码；
- so是二进制，没有解释编译的开消，用so实现的功能比纯java实现的功能要快；
- so内存分配不受Dalivik/ART的单个应用限制，减少OOM；
- 相对于java代码，二进制代码的反编译难度更大，一些核心代码可以考虑放在so中。

## 为指定的ABI生成so
默认情况下，NDK只会为*armeabi*生成.so文件，若需要生成支持其他ABI的.so文件，可以在*Application.mk*文件中指定`APP_ABI`参数：
```
APP_ABI := armeabi-v7a
```

`APP_ABI`参数可以被指定多个值以支持多个ABI：
```
APP_ABI := armeabi armeabi-v7a x86
```

当然，你也可以使用`all`来生成支持所有ABI的so：
```
APP_ABI := all
```

![各种ABI对应的值](http://ww3.sinaimg.cn/large/72f96cbagw1f9bfq62g6tj21gy0h6dhy.jpg)

## 查看Android系统的ABI支持
Android可以在运行期间确定当前系统所支持的ABI，这是由系统编译时的具体参数指定的：
- `primary ABI`（主ABI）：对应当前系统中使用的机器码类型
- `secondary ABI`（副ABI）：表示当前系统支持的其他ABI类型

许多手机支持不止一个ABI，比如，一个基于ARMv7的设备会将armeabi-v7a定义为*primary ABI*，armeabi作为*secondary ABI*，意味着这台机器同时支持armeabi-v7a和armeabi。
许多基于x86的设备也可以运行armeabi-v7a和armeabi的so，对于这些机器，*primary ABI*是x86，*secondary ABI*则是armeabi-v7a.

但是，为了能得到更好的性能表现，我们应该尽可能的直接提供primary ABI所对应的so文件。比如，我们可以为x86手机直接提供x86的so文件，而不是仅提供arm的so让系统通过*houdini*去动态转换arm指令，避免转换过程中的性能损耗。

查看Android系统支持的ABI有以下两种方法：
### 使用adb命令
`/system/build.prop`中指定了支持的ABI类型，在adb中，可使用如下命令查看：

```bash
shell@NX529J:/ $ getprop | grep abilist
[ro.product.cpu.abi]: [arm64-v8a]
[ro.product.cpu.abilist32]: [armeabi-v7a,armeabi]
[ro.product.cpu.abilist64]: [arm64-v8a]
[ro.product.cpu.abilist]: [arm64-v8a,armeabi-v7a,armeabi]
```

### 使用API获取
使用[Build.SUPPORTED_ABIS](https://developer.android.com/reference/android/os/Build.html#SUPPORTED_ABIS)可以获取当前设备支持的ABI列表：

```java
import android.os.Build;
String supportedAbis = Build.SUPPORTED_ABIS;
```

## x86手机对arm的支持
值得注意的是原本x86架构的CPU是不支持运行arm架构的native代码的，但Intel和Google合作在x86机子的系统内核层之上加入了一个名为*houdini*的Binary Translator（二进制转换中间层），这个中间层会在运行期间动态的读取arm指令并将之转换为x86指令去执行。
![Binary Translator](http://ww2.sinaimg.cn/large/65e4f1e6gw1f9g5kj885ij218g0nrq9k.jpg)
所以能看到很多没有提供x86对应so的应用（如新浪微博）也能够运行在x86手机上。

## apk安装过程中对so的选择
在Android上安装应用程序时，Package Manager会扫描整个apk文件，寻找符合下面文件路径格式的动态连接库：

```
lib/<primary-abi>/lib<name>.so
```
在这里，`primary-abi`是上面表中的abi的值，`name`对应的是我们在*Android.mk*中定义的*LOCAL_MODULE*的值，

如果在apk内并没有找到适合当前机器*primary-abi*的so，Package Manager会尝试寻找适合*secondary-abi*的so文件：

```
lib/<secondary-abi>/lib<name>.so
```

即安装应用时，系统会根据当前CPU架构选择最优ABI适配，如果找到了合适的so文件，包管理器会将该ABI文件夹下所有so库全部拷贝至应用的data目录下：`data/data/<package_name>/lib/`

注意：apk安装过程对so选择*是基于整个ABI文件夹的，而非以单个so文件为粒度*，也就是说把lib/armeabi 、lib/armeabi-v7a、lib/x86等等文件夹的其中一个文件夹内所有.so复制到应用的data目录下。

如果我们在代码中调用了某个so的功能，而最终拷贝的ABI文件夹下并没有提供这个文件，apk的安装过程中并不会报错，但是运行时会遇到`java.lang.UnsatisfiedLinkError`。

## so的加载

对于so的加载，Android在`System`类中提供了两种方法：

```java
   /**
     * See {@link Runtime#loadLibrary}.
     */
    public static void loadLibrary(String libName) {
        Runtime.getRuntime().loadLibrary(libName, VMStack.getCallingClassLoader());
    }

   /**
     * See {@link Runtime#load}.
     */
    public static void load(String pathName) {
        Runtime.getRuntime().load(pathName, VMStack.getCallingClassLoader());
    }
```

### System.loadLibrary
这是我们最常用的一个方法，`System.loadLibrary`只需要传入so在*Android.mk*中定义的*LOCAL_MODULE*的值即可，
系统会调用`System.mapLibraryName`把这个libName转化成对应平台的so的全称并去尝试寻找这个so加载。
比如我们的so文件全名为*libmath.so*，加载该动态库只需要传入`math`即可：

```java
System.loadLibrary("math");
```

### System.load
对于`System.load`方法，官方是这样介绍的：
> Loads a code file with the specified filename from the local file system as a dynamic library. 
> The filename argument must be a complete path name.

所以它为动态加载非apk打包期间内置的so文件提供了可能，也就是说可以使用这个方法来指定我们要加载的so文件的路径来动态的加载so文件。
比如我们在打包期间并不打包so文件，而是在应用运行时将当前设备适用的so文件从服务器上下载下来，放在`/data/data/<package-name>/mydir`下，然后在使用so时调用：
```java
System.load("/data/data/<package-name>/mydir/libmath.so");
```
即可成功加载这个so，开始调用本地方法了。

> 其实loadLibrary和load最终都会调用nativeLoad(name, loader, ldLibraryPath)方法，只是因为loadLibrary的参数传入的仅仅是so的文件名，所以，loadLibrary需要首先找到这个文件的路径，然后加载这个so文件。 
而load传入的参数是一个文件路径，所以它不需要去寻找这个文件路径，而是直接通过这个路径来加载so文件。

但是当我们把需要加载的so文件放在SdCard中，会发生什么呢？把上面so的路径改成`/mnt/sdcard/libmath.so`，再尝试加载时，会得到如下错误：
```bash
java.lang.UnsatisfiedLinkError: dlopen failed: couldn't map "/mnt/sdcard/libmath.so" segment 1: Permission denied
```
这是因为SD卡等外部存储路径是一种可拆卸的（mounted）不可执行（noexec）的储存媒介，不能直接用来作为可执行文件的运行目录，使用前应该把可执行文件复制到APP内部存储下再运行。所以使用`System.load`加载so时要注意把so拷贝至`/data/data/<package-name>/`下。

## 通过精简so来减小包大小
现在的apk动辄几十M或者更大，apk包大小的精简成为了开发过程中的重要一环。通过上面的介绍，我们知道x86、x86_64、armeabi-v7a、arm64-v8a设备都支持armeabi架构的so，因此，通过移除不必要的so来减小包大小是一个不错的选择。

### 按照ABI分别单独打包APK
我们可以选择在Google Play上传指定ABI版本的APK，生成不同ABI版本的APK可以在build.gradle中进行如下配置：
```groovy
android {
    // Some other configuration here...

    splits {
        abi {
            enable true
            reset()
            include 'x86', 'armeabi', 'armeabi-v7a', 'mips' //select ABIs to build APKs for
            universalApk false // generate an additional APK that contains all the ABIs
        }
    }
}
```

### 只提供`armabi`的so
上面的方法需要应用市场提供用户设备CPU类型更识别的支持，在国内并不是一个十分适用的方案。常用的处理方式是利用gradle中的abiFilters配置。
首先配置修改主工程`build.gradle`下的`abiFilters`：
```groovy
android {
    // Some other configuration here...

    defaultConfig {
        ndk {
            abiFilters 'armeabi'
        }
    }
}
```
abiFilters后面的ABI类型即为要打包进apk的ABI类型，除此以外都不打包进apk里。
然后在项目的根目录下的`gradle.properties`（没有的话新建一个）中加入下面这行：
```groovy
android.useDeprecatedNdk=true
```
通过上面方法减少的apk体积是十分可观的，也是目前比较主流的处理方案。

### 进阶版方案
如果进一步，会发现上面的方案并不完美。首先是性能问题：使用兼容模式去运行arm架构的so，会丢失专门为当前ABI优化过的性能；其次还有兼容性问题，虽然x86设备能兼容arm类型的函数库，但是并不意味着100%的兼容，某些情况下还是会发生crash，所以x86的arm兼容只是一个*折中方案*，为了最好的利用x86自身的性能和避免兼容性问题，我们最好的做法仍是专为`x86`提供对应的so。
针对这些问题，我们可以采用一个相对更好的方案：让所有so都来自于网路，应用下载服务器上的so库后，利用`System.load`方法动态加载当前设备对应的so.

## 需要注意的问题
### 不要把so放错地方
首先要注意的是不要把另一个ABI下的so文件放在另一个ABI文件夹下（每个ABI文件夹下的so文件名是相同的，有可能会搞错）。

### 尽可能为所有ABI提供so
理想状况下，应该尽可能为所有ABI都提供对应的so，这一点的好处我们已经在上面讨论过了：在可以发挥更好性能的同时，还能减少由于兼容带来的某些crash问题。当然，这一点要结合实际情况（如SDK提供的so不全、芯片市场占有率、apk包大小等）去考量，如果使用的so本身就很小，我们大可为尽可能多的ABI都提供so。
若是局限于包大小等因素，可以结合*通过精简so来减小包大小*一节中提供的第三个方案来调整so的使用策略。

### 所有ABI文件夹提供的so要保持一致
*这是一个十分容易出现的错误。*
如果我们的应用选择了支持多个ABI，要十分注意：对于每个ABI下的so，但要么*全部支持，要么都不支持*。不应该混合着使用，而应该为每个ABI目录提供对应的.so文件。

先举个例子，Bugtags的so支持所有的ABI：
```bash
libs
|
├── arm64-v8a
│   └── libBugtags.so
├── armeabi
│   └── libBugtags.so
├── armeabi-v7a
│   └── libBugtags.so
├── mips
│   └── libBugtags.so
├── mips64
│   └── libBugtags.so
├── x86
│   └── libBugtags.so
└── x86_64
    └── libBugtags.so
```

但不是所有开发者提供的so都支持所有ABI：
```bash
lib
|
├── armeabi
│   └── libImages.so
└── armeabi-v7a
    └── libImages.so
```

如果不做任何设置，最终打出来的apk的lib目录会是这样的：
```bash
lib
|
├── arm64-v8a
│   └── libBugtags.so
├── armeabi
│   ├── libBugtags.so
│   └── libImages.so
├── armeabi-v7a
│   ├── libBugtags.so
│   └── libImages.so
├── mips
│   └── libBugtags.so
├── mips64
│   └── libBugtags.so
├── x86
│   └── libBugtags.so
└── x86_64
    └── libBugtags.so
```

参考上面*apk安装过程中对so的选择*一节，假设当前设备是x86机器，包管理器会先去lib/x86下寻找，发现该文件夹是存在的，所以最终只有lib/x86下的so--即只有libBugtags.so会被安装。当尝试在运行期间加载`libImages.so`时，就会遇上下面常见的*UnsatisfiedLinkError*错误：

```bash
E/xxx   (10674): java.lang.UnsatisfiedLinkError: dalvik.system.PathClassLoader[DexPathList[[zip file "/data/app/xxx-2/base.apk"],nativeLibraryDirectories=[/data/app/xxx-2/lib/x86, /vendor/lib, /system/lib]]] couldn't find "libImages.so"
E/xxx   (10674):     at java.lang.Runtime.loadLibrary(Runtime.java:366)
```

所以，我们需要遵循这样的*准则*：
- 对于so开发者：支持所有的平台，否则将会搞砸你的用户。
- 对于so使用者：要么支持所有平台，要么都不支持。

然而，因为种种原因（遗留so、芯片市场占有率、apk包大小等），并不是所有人都遵循这样的原则。

一种可行的处理方案是：取你所有的so库所支持的ABI的交集，移除其他（可以通过上面介绍的`abiFilters`来实现）。
如上面的例子，最终生成的apk可以是:
```bash
lib
|
├── armeabi
│   ├── libBugtags.so
│   └── libImages.so
└── armeabi-v7a
    ├── libBugtags.so
    └── libImages.so
```
