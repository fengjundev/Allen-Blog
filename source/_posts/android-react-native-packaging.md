---
title: Android React Native 打包与签名实践
date: 2016-10-29 14:18:17
tags: 
- react native
- android
- 打包
- 签名
categories: 
- android
- react native
---

![](http://ww3.sinaimg.cn/large/006tNbRwgw1f9ac5adg60j30dw09b0um.jpg)

我们都知道，RN开发完成后，不管是集成到app中一起发布还是热更新发布，都会涉及到js代码的打包。如果代码中引用了静态图片资源，还需要连同图片一起打包。除此以外，还要想办法保证我们所执行的js代码的安全性和完整性，基于这些需求，我们团队形成了一套RN打包签名的方案，本文主要对这套方案记录，以供参考。

<!-- more -->

> 出处： *[Allen's Zone](http://allenfeng.com/)*
> 作者： *Allen Feng*

### 静态图片加载方式的选择
根据[官方文档](https://facebook.github.io/react-native/docs/images.html)，加载**静态**图片资源一共有两种方式。第一种是将图片放置在原生app的`res/drawable`目录中，然后在js代码中使用下面方式加载：

```javascript
<Image source={{uri: 'app_icon'}} style={{width: 40, height: 40}} />
```

但是这样的方式并不优雅，不仅会增大宿主APP的体积，还会导致js层的实现与原生层耦合在一起，当热更新时无法做到图片的更新。

第二种方式是采用相对路径的方式进行加载，例如，我们有以下的目录结构

```bash
src
├── button.js
└── img
    ├── check.png
    └── check_pressed.png
```

然后在`button.js`这样引用：

```javascript
<Image source={require('./img/check.png')} />
```

这样的方式就比较灵活，只要将js代码和使用到的图片打包在一起，就可以在热更新的时候做到业务代码和图片的同步更新。

所以我们在项目中统一采用的第二种方案去加载静态图片。


### 对图片路径的定制
我们希望打包时，整个RN包内部是下面的结构，将所有图片放置在`images`目录下，这样的目录结构比较清晰明了.

```bash
├── images
│   ├── drawable-hdpi
│   ├── drawable-mdpi
│   ├── drawable-xhdpi
│   ├── drawable-xxhdpi
│   └── drawable-xxxhdpi
└── index.android.bundle
```

但是这里有个问题，release版本中，RN采用`require`方式加载图片时，生成的路径是下面的形式：

```bash
<bundle当前路径>/drawable-mdpi/foldername_imagename.png
```

根据上面打包的结构，需要定制成为以下的形式才能正确访问到图片：

```bash
<bundle当前路径>/images/drawable-mdpi/foldername_imagename.png
```

所以就涉及到了对图片路径定制的需求，经过RN源码的探索，发现对于图片路径的解析是由`node_modules/react-native/Libraries/Image/resolveAssetSource.js`来完成的，解析生成图片路径主要代码如下：

```javascript
/**
 * `source` is either a number (opaque type returned by require('./foo.png'))
 * or an `ImageSource` like { uri: '<http location || file path>' }
 */
function resolveAssetSource(source: any): ?ResolvedAssetSource {
  if (typeof source === 'object') {
    return source;
  }

  var asset = AssetRegistry.getAssetByID(source);
  if (!asset) {
    return null;
  }

  const resolver = new AssetSourceResolver(getDevServerURL(), getBundleSourcePath(), asset);
  if (_customSourceTransformer) {
    return _customSourceTransformer(resolver);
  }
  return resolver.defaultAsset();
}

function setCustomSourceTransformer(
  transformer: (resolver: AssetSourceResolver) => ResolvedAssetSource,
): void {
  _customSourceTransformer = transformer;
}
```

可以看到，我们可以通过调用`setCustomSourceTransformer`来设置自己的`AssetSourceResolver`来完成自定义的路径解析生成器，加上需要的`images`这一部分路径即可。


### Bundle打包和图片导出
主要是使用官方提供的`react-native bundle`命令实现这个过程，如

```bash
react-native bundle --entry-file index.android.js --bundle-output ./output/my.bundle --dev false --platform android --assets-dest ./output/images/
```

具体的参数可以通过`--help`查看说明，需要注意的是，`assets-dest`指定的就是js代码中引用到的图片资源的导出路径，如果不指定，将不会导出图片。

命令执行完成后，`output`目录下：

```bash
├── images
│   ├── drawable-hdpi
│   ├── drawable-mdpi
│   ├── drawable-xhdpi
│   ├── drawable-xxhdpi
│   └── drawable-xxxhdpi
└── index.android.bundle
```

### 签名与打包
在完成了上面的操作后，我们只是得到了一个文件夹，里面包含了bundle文件和图片文件，如果涉及到中间人攻击，bundle文件可能会被篡改，安全无法得到保证，所以我们需要对这个文件夹下的文件进行加签处理。具体采用的是下面的签名方案：

![打包流程](http://ww1.sinaimg.cn/large/72f96cbajw1f994rs0nznj21kw13k7cb.jpg)

（1）生成MANIFEST.MF文件：这是摘要文件。遍历build目录所有文件（entry），对非文件夹、非签名文件的文件，逐个使用SHA1生成摘要信息，再用Base64编码。并在摘要文件头部写入bundle的版本信息等。

说明：如果有人改变了安装包中的文件，那么在安装校验的时候，改变后的文件摘要信息与MANIFEST.MF的校验信息不同，将无法安装。但是，如果攻击者重新生成了摘要文件，就可以通过验证，所以这只是一个非常简单的验证。需要结合（2）确保安全性。

（2）生成CERT.SF文件：这是摘要文件的签名文件。对前一步生成的MANIFEST.MF，使用SHA1-RSA算法，用私钥进行签名。在安装时，在客户端使用公钥进行解密，解密之后MANIFEST.MF的内容进行比对，如果相符，则表明内容没有被异常修改。

说明：在这一步，即使攻击者修改了内容，并生成了新的摘要文件，但是攻击者没有开发者的私钥，所以不能生成正确的签名文件（CERT.SF）。客户端在对安装包进行验证的时候，用公钥对不正确的签名文件进行解密，得到的结果和摘要文件（MANIFEST.MF）对应不起来，所以不能通过检验，不能成功安装文件。从而确保了安全性。

完成上述流程后，将签名文件和bundle等文件一起压缩，就完成了整个打包的流程。






