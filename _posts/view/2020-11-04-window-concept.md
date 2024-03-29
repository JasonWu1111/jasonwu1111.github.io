---
layout: post
title: Android 系统 Window 概念理解
author: JasonWu
tags:
  - Android
  - Window
---

# Android 系统 Window 概念理解
Window 是整个 Android View 系统中最基本、核心的概念。本文旨在概括性地介绍 Window 中的一些重要概念，以方便后续对 Window 相关操作的深入学习。

## 如果理解 Window
在 Android 系统中，任何可视化的界面，都是以 **Window** 的概念而存在。包括我们日常开发中最常接触到的 Activity、弹窗 Dialog、系统界面固定的状态栏、虚拟按键等，总之，所有的视图（可见或不可见），都是一个个独立的 “窗口”，受到系统 Framework 服务来统一管理。

![](/img/posts/post-windows.png)


对于普通开发者而说，处理视图直接相关的是 **View** 对象；反而对于 Android 系统来说，**Window 才是系统管理其内视图的单位**。如果我们想新展示一个界面，比如一个 Activity，其本质上是通知系统服务来添加一个新的 Window。

Window 是一个概念，没有一个实质完整的‘类’来描述。对系统来说直接操作的对象是一个 **View**，也就是一个 ViewTree 里最外层的那个 ViewGroup。

然而 **Window 本身并不直接等于 View**, View 对象只表示一个具体展示的视图，除此以外，一个 Window 还包含了窗口类型、token、窗口布局等参数，这些共同形成一个完整的 Window 概念。

![](/img/posts/post-window_element.png)


## 如果查看 Window
手机通过 adb 连接后，可以通过以下命令查看当前手机中打开的全部窗口：
```shell
adb shell dumpsys window | grep -E 'Window #\d{1,} Window'
```

比如，我的一加手机当前“只打开”了微信应用，查看全部的窗口信息如下：
```
➜  ~ adb shell dumpsys window | grep -E 'Window #\d{1,} Window'
  Window #0 Window{578b2e8 u0 InputMethod}:
  Window #1 Window{5164e47 u0 ScreenDecorOverlayBottom}:
  Window #2 Window{917eae u0 ScreenDecorOverlay}:
  Window #3 Window{526fe2e u0 NavigationBar0}:
  Window #4 Window{dd335a7 u0 StatusBar}:
  Window #5 Window{4f8a3b4 u0 AssistPreviewPanel}:
  Window #6 Window{2564720 u0 DockedStackDivider}:
  Window #7 Window{2650353 u0 com.tencent.mm/com.tencent.mm.ui.LauncherUI}:
  Window #8 Window{86826e4 u0 net.oneplus.launcher/net.oneplus.launcher.Launcher}:
  Window #9 Window{7dc0b56 u0 com.oneplus.wallpaper.LiveWallpaper}:
```

另外，还可以通过以下命令查看当前聚焦的窗口：
```
➜  ~ adb shell dumpsys window | grep -E 'mCurrentFocus|mFocusedApp'
  mCurrentFocus=Window{dd335a7 u0 StatusBar}
  mFocusedApp=AppWindowToken{76df2b7 token=Token{e565b6 ActivityRecord{fdcd951 u0 com.tencent.mm/.ui.LauncherUI t3879}}}
```

## Window 的类型
通过 adb 查看手机系统中当前全部的 Window，发现有些窗口是系统常驻的，比如状态栏、导航栏，有些则是属于一个应用的，每个应用可以打开不定数量的窗口。

其中，每个窗口都有一个 int 值来表示**窗口类型**，整体上分为以下三类：

| 类型 | 说明 | 范围
|--|--|--
| APPLICATION_WINDOW | 应用窗口，如 Activity | 1~99
| SUB_WINDOW | 子窗口，不能单独存在，只能附属在父 Window 中，如 Dialog 等 | 1000~1999
| SYSTEM_WINDOW | 系统窗口，需要权限声明，如 Toast 和系统状态栏等 | 2000~2999

> 窗口类型的取值范围定义在 `frameworks/base/core/java/android/view/WindowManager.java` 文件中

## Window 的 token
既然全部的 Window 都受到系统统一的管理，那么 Window 间的**安全性**就是一个很重要的问题了。试想一下，如果其他的应用可以随意在你应用的窗口上绘制更改内容，甚至移除你的应用的窗口，那么系统的安全性将会是极不可靠的。

在 Android 系统中，每个应用窗口窗口对会有其对应的一个 **token**，也就是所谓的“窗口令牌”。一个窗口的 token 就像是它的合格且唯一的凭证一样，只有带合格 token 的窗口，才能够添加进系统中；只有具有相同 token，你才能够对目标窗口进行操作。
> 其中，系统类型的窗口（SYSTEM_WINDOW）比较特殊，它通常是由系统所创建，是不需要 token 的。而对于应用窗口来说，token 则是必须的。

为了 token 不能够被轻松地“仿造”，token 对象被设计为 **Binder 对象**。为什么是 Binder？其实不止是 Window 中，整个 Android 系统大部分的跨进程通信都是依赖了 Binder 的**唯一性**，把它作为 token 来保证安全性。Binder 的唯一性是指，**每个 Binder 实例在系统所有进程中都保持唯一的身份**，这是由 Binder 内核驱动来做保证的。这里就不展开细讲了。

因此，当你向系统请求添加或删除一个 Window 时，如果携带的 token 不匹配，将会抛出 ``BadTokenException``。
