---
title: XCode编译WebKit JavaScriptCore问题解决
---

个人感觉使用XCode编译调试JavaScriptCore是一种正确的方法，之前使用GCC太不方便了，耗费了很多很多时间。



（1）直接编译All Source，然后切换schema到jsc运行 。

（2）不要使用parrellize build 。

（3）run exit with xx、file not found等问题可以通过重试来解决 。

（4）放置老版本SDK进入目录 ▸ ⁨应用程序⁩ ▸ ⁨Xcode.app⁩ ▸ ⁨Contents⁩ ▸ ⁨Developer⁩ ▸ ⁨Platforms⁩ ▸ ⁨MacOSX.platform⁩ ▸ ⁨Developer⁩ ▸ SDKs⁨，重启XCode。

 修改的参数为：

- Base SDK

- macOS Deployment Target

  修改的目标有PAL、WebCore、WebKit、WebKitLegacy：

  - ![image-20190119151730540](XCode编译WebKit JavaScriptCore问题解决.assets/image-20190119151730540.png)
  - ![image-20190119151757424](XCode编译WebKit JavaScriptCore问题解决.assets/image-20190119151757424.png)
  - ![image-20190119151812242](XCode编译WebKit JavaScriptCore问题解决.assets/image-20190119151812242.png)
  - ![image-20190119151832372](XCode编译WebKit JavaScriptCore问题解决.assets/image-20190119151832372.png)

（5）对较老的版本使用legacy build system：例如2019年在MacOS 10.14上使用XCode 10编译2018年1月30日版本时，就需要切换到legacy build system。

（6）Treat warnings as errors设置为否。



另：用新的编译器、操作系统编译老板本时，更多的问题是WebCore包含的多媒体部分和其他内容，它们引用了很多macOS SDK，就是因为它们JSC才编译不过。JSC本身不需要太多的依赖，奈何需要有一个WebCore的动态库，所以构建All Soource是一种简单粗暴的可行方法。 

老版本SDK的下载地址： 

<https://github.com/phracker/MacOSX-SDKs>

