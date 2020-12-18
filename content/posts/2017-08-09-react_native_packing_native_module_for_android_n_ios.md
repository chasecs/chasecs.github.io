---
layout: post
title:  "React Native 原生模块库打包指南"
date: 2017-08-09 
categories: ["React"]
description: "本文以百度 OCR 的 iOS SDK 和 Android SDK 为例，介绍如何将其打包为一个原生模块库（Native Module library） “react-native-baidu-ocr” ，供 React Navive APP 调用..."
---


**最终项目的 repository：[react-native-baidu-ocr](https://github.com/chasecs/react-native-baidu-ocr)**

## 原生模块（Native Module）的使用方式

React Native（下称RN）的官方文档介绍了[Native Modules](https://facebook.github.io/react-native/docs/native-modules-ios.html) （下称“原生模块”）的使用方法，用于访问原生平台的 API。但是，文档里的方法并不适合直接在项目使用。

以 iOS 为例，如果按照 RN 文档的做法，你需要用 Xcode 项目的 `.xcodeproj` 文件，在里面新建一个 `.m` Objective-C File 和一个 `.h` Header File，然后在这两个文件里实现一个“RCTBridgeModule”协议的 Objective-C 类。

这样做的缺点我想到的有两个，一是，如果要实现的原生模块多了，`.m` 和 `.h` 也会递增，目录容易混乱难以管理；二是，这样的原生模块不容易复用，其它项目没法直接使用这个原生模块。

通行的做法是，将实现后原生模块打包好，然后存放在项目的`node_modules`的文件内，供 APP 调用。

## 打包原生模块的主要工作

打包一个原生模块，最主要的工作有两部分：
- 安装该模块，即 "react-native link" 完成的工作。
- 写代码，对于iOS，是写一个实现了“RCTBridgeModule”协议的Objective-C类；对于Android，是写一个继承了 ReactContextBaseJavaModule的Java类。

所以，要打包 React Native 的原生模块，你最好有以下知识储备：
- 有 RN 开发经验。如 RN 文档所说，原生模块是该框架的高级特性，当然是有一定的经验更易理解。
- 了解 Java 和 Objective-C， 至少能看懂这两种代码。

本文将以百度 OCR 的 [iOS SDK](http://ai.baidu.com/docs#/OCR-iOS-SDK/top) & [Android SDK](http://ai.baidu.com/docs#/OCR-Android-SDK/top) 为例，介绍如何将其打包为一个名为 “react-native-baidu-ocr” 的 原生模块库（library），供 RN APP 调用。

这个例子相比一般的原生模块开发（RN文档示例）会多一项工作，即引入SDK，流程大致如下：
- 创建一个 RN 项目 ExampleApp，用于调试。
- 初始化 react-native-baidu-ocr 包。
- 分别针对 Android 和 iOS 平台配置、引入 SDK、写代码。


## 初始化原生模块库

先创建一个 RN 项目 ExampleApp。

创建RN库有多种方法。

如果只需要支持 iOS 平台，可以直接使用`react-native-cli`命令:
```
react-native new-library --name <yourLibraryName>
```
命令执行后，可以在项目的 ./Libraries 或是 ./node_modules/react-native/Libraries 目录找到该文件夹。

而 Android 库，目前只知道可以通过 Android Studio 来创建，操作起来比较麻烦。

上述这两种方法此处不作赘述。这里使用一个开源的工具 [react-native-create-library](https://github.com/frostney/react-native-create-library)，来帮我们初始化同时支持 Android 和 iOS 的库。

首先，安装 react-native-create-library
```
 npm install -g react-native-create-library
```

进入你要存放library的目录，创建 BaiduOCR
```
react-native-create-library BaiduOCR
```

新建的 BaiduOCR 里，包含 Android 和 iOS 项目，还有不需要的 windows，直接删掉
```
rm -rf windows/
```

*和 react-native-create-library 类似的工具还有 [react-native-create-bridge](https://github.com/peggyrayzis/react-native-create-bridge)，不过没用过*

目录下创建 react-native-baidu-ocr, 把BaiduOCR目录下的文件都转移过去，初始化就算完成了。

## 打包 iOS 部分

先下载 [百度 OCR iOS SDK](http://ai.baidu.com/docs#/OCR-iOS-SDK/top)。下载下来的文件里没有现成的 AipOcrSdk.framework，需要自己生成并导入，过程有点复杂，以后另文专述。

把 AipOcrSdk.framework 和 AipBase.framework 复制粘粘到 ios 目录

![复制粘粘到 ios 目录](/images/react_native_packing_native_module_for_android_n_ios/p3.png)
 <img alt="" src="{{ site.url }}" style="border: 1px solid black" >

进入ios目录，用 Xcode 打开 RNBaiduOcr.xcodeproj。

点击左侧栏蓝色项目logo，中间栏 TARGETS，Build Phases，选择 Link Binary With Libraries，点击‘+’，选 Add Another，把刚刚复制到 ios 目录的 AipOcrSdk.framework 和 AipBase.framework 加入
![Link Binary With Libraries](/images/react_native_packing_native_module_for_android_n_ios/p1.png)

参照 baidu 官方文档，在 `RNBaiduOcr.m` 实现“RCTBridgeModule”协议。代码实现部分可以直接看[源码](https://github.com/chasecs/react-native-baidu-ocr/blob/master/ios/RNBaiduOcr.m)，这里就不细说。

代码实现后，需要将 RNBaiduOcr 项目 build 一下。如果 “'React/RCTDefines.h' file not found”，改为"RCTDefines.h"

用 Xcode 打开 ExampleApp 项目，Libraires 右键 Add Files to “ExampleApp” ,选择 node_module/react-native-baidu-ocr/ios/RNBaiduOcr.xcodeproj

选择 ExampleApp -- TARGETS -- build phases -- Link Binary With Libraries
![Link Binary With Libraries](/images/react_native_packing_native_module_for_android_n_ios/p4.png)

点击“+”，Add “libRNBaiduOcr.a”
![Link Binary With Libraries](/images/react_native_packing_native_module_for_android_n_ios/p5.png)


打开 ExampleApp 项目 Libiraies 里面的 RNBaiduOcr.xcodeproj,将 AipOcrSdk.framework 和 AipBase.framework 复制到 ExampleApp 文件夹（注意此时应该关掉 Xcode 之前打开的 RNBaiduOcr 项目）
![复制到 ExampleApp](/images/react_native_packing_native_module_for_android_n_ios/p6.png)
<img alt="" src="{{ site.url }}/images/react_native_packing_native_module_for_android_n_ios/p6.png" style="border: 1px solid black" >

选择 ExampleApp -- TARGETS -> General -> Embedded Binaries, 加入
AipOcrSdk.framework 和 AipBase.framework
![加入AipOcrSdk.framework 和 AipBase.framework](/images/react_native_packing_native_module_for_android_n_ios/p7.png)

上述工作完成后，iOS部分就能使用了。

## 打包 Android 部分

参照[百度 OCR Android SDK 开发者文档](http://ai.baidu.com/docs#/OCR-Android-SDK/top)，将下载包libs目录中的 ocr-sdk.jar 文件，拷贝到工程 node_module/react-native-baidu-ocr/android/libs 目录中。

将 ocr-sdk.jar 加入工程依赖，即修改 ./react-native-baidu-ocr/android/build.gradle

  ```
  ...
  dependencies {
      compile 'com.facebook.react:react-native:+'
      compile files('libs/ocr-sdk.jar')       //加入这行
  }

  ```
将下载包libs目录下armeabi，arm64-v8a，armeabi-v7a，x86文件夹按需添加到 android studio工程src/main/jniLibs目录中
link 部分

加入以下代码到 ./ExampleApp/android/settings.gradle
```
include ':react-native-baidu-ocr'
project(':react-native-baidu-ocr').projectDir = new File(rootProject.projectDir, '../node_modules/react-native-baidu-ocr/android')
```

加入以下代码到 ./ExampleApp/android/app/build.gradle
```
dependencies {
   ...
    compile project(':react-native-baidu-ocr'') // 加入
   ...
}
```

打开 ./ExampleApp/android/app/src/main/java/[...]/MainActivity.java
  - 在文件开头加入 `import com.reactlibrary.RNBaiduOcrPackage;`
  - 在 `getPackages()` 方法里加入 `new RNBaiduOcrPackage()`

参照 baidu ocr android 官方文档，实现继承了 ReactContextBaseJavaModule 的 Java 类 [RNBaiduOcrModule.java](https://github.com/chasecs/react-native-baidu-ocr/blob/master/android/src/main/java/com/reactlibrary/RNBaiduOcrModule.java)。

## Android 代码指南

由于 baidu ocr 的 ios 和 android 返回数据的格式不一致，为例 RN APP 调用方便，这里只好将 Android 部分的返回数据格式，改成和ios一样。

百度的文档里没有提供 ocr-sdk.jar 里诸如 IDCardResult 等 Class 的文档，无法直接获知各个类可调用的方法。不过，通过 Android Studio 打开从百度官方下载的 OCRDemo 项目，可以看到 jar 包的代码，由此知道有哪些可调用的方法。

![ocr-sdk.jar的代码](/images/react_native_packing_native_module_for_android_n_ios/p8.png)


Android 部分的代码实现，主要就是将原生SDK返回的数据，通过 `com.facebook.react.bridge.WritableMap` 等类的方法，映射到它们对应的JavaScript类型，例如：

```java
/*
*convert GeneralResult into WritableMap
*@param GeneralResult result
*/
private WritableMap resultToWritableMap(GeneralResult result){
  WritableMap response = Arguments.createMap();
  WritableArray wordList = Arguments.createArray();;
  for (WordSimple wordSimple : result.getWordList()) {
      WordSimple word = wordSimple;
      WritableMap wordItem = Arguments.createMap();
      wordItem.putString("words", word.getWords());
      wordList.pushMap(wordItem);
  }
  response.putArray("words_result", wordList);
  response.putInt("words_result_num", result.getWordsResultNumber());
  response.putInt("direction", result.getDirection());
  return response;
}
```
`GeneralResult` 是百度SDK提供的类，最后的 `response` 是可以在 RN APP 中被 JavaScript 理解的 Object。

Android 代码实现后，这个原生模块库的打包即告完成，可以在当前的RN项目里使用。

## 分享到 NPM

不过，要想让这个原生模块可以给你的其它项目或者他人使用，还有一些额外的工作，最简单的就是，写好使用文档，发布到 npm。

上传到 npm 之前，把 node_modules/ 里的 react-native-baidu-ocr 复制粘贴到 ExampleApp 项目以外的地方。最好 `git init`, 然后再上传，具体操作参考 [Getting Started with NPM](https://gist.github.com/coolaj86/1318304)

上传成功后，其它项目就可以通过 npm install 来使用这个库。

## 参考资料

- 打包相关  
  - [Native Modules for React Native Android](https://shift.infinite.red/native-modules-for-react-native-android-ac05dbda800d)  
  - [React Native Bridging Modules for Android from Scratch on Windows](http://www.codepool.biz/react-native-bridging-modules-android.html)  
  - [`React/RCTBridgeModule.h` file not found](https://stackoverflow.com/questions/41663002/react-rctbridgemodule-h-file-not-found/43340802)  
  - ['RCTBridgeModule.h' file not found error](https://github.com/itinance/react-native-fs/pull/238)  
  - [Making libraries for React Native](https://medium.com/gbox-crew-blog/making-libraries-for-react-native-14a8f5006697#.mb7gojizs)  
  - [如何在 React Native 中写一个自定义模块](http://www.jianshu.com/p/73ef53244a7b)  
  - [React Native and Native Modules: The Android SyncAdapter](https://blog.callstack.io/react-native-and-native-modules-the-android-syncadapter-517ddf851bf4)  
  - [How to create a React Native Android Library](http://cmichel.io/how-to-create-react-native-android-library/)  
  - [React Native: Packaging and Sharing Native Modules](https://eastcodes.com/packaging-and-sharing-react-native-modules)  
- iOS framework 相关  
  - [iOS里的动态库和静态库](https://www.zybuluo.com/qidiandasheng/note/603907)  
  - [iOS 动态库(Dynamic框架)的创建以及引用添加(Embed Binary方式嵌入)](http://blog.csdn.net/xy_26207005/article/details/71108558)  
