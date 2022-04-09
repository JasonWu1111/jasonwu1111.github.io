---
layout: post
title: 应用进程的创建流程分析
author: JasonWu
tags:
  - Android
  - Process
  - Application
---

## 前言
每个应用启动必须先创建一个进程，每个进程**有各自独立的资源空间**，用于运行整个 App 所需要的一切资源。

一个新的应用进程的创建，涉及到以下两个核心的系统进程：
- **`Zygote`** 进程：**是 Android 系统的首个 Java 进程**，如其含义“受精卵”那样，是所有应用进程的父进程。为了在 64 位架构 CPU 的设备上兼容支持 32 位的应用，Zygote 进程共有两个：Zygote 和 Zygote64。分别对应用来 fork 出 32 位还是 64 位的应用进程。
  
- **`system_server`** 进程：**系统服务进程**，负责承载 ActivityManagerService、PackageManagerService 等多个关键系统服务的进程，本身也是 Zygote 进程 fork 出来的子进程。

默认情况下，同一应用的所有组件均在相同的进程中运行，**进程名默认为应用的包名**。如果需要运行多进程，则可在清单文件中各类组件元素（`<activity>`、`<service>`、`<receiver>` 和 `<provider>`）中设置 `android:process` 属性，此属性将指定该组件应在哪个进程中运行。


一个应用只有其在清单文件声明的组件，才可以响应来自其他进程的调用。以最常见的在桌面点击一个应用图标启动一个新的应用为例，其进程的启动流程示意如下：

1. 桌面应用处理图标的点击事件，生成启动目标应用主启动 Activity 的 **Intent**；
   
2. 桌面应用通过 Intent 通知**系统服务进程**来启动目标应用主启动 Activity；
   
3. 系统服务进程判断启动的目标组件所属进程未启动时，进一步通信到 **Zygote** 进程来 `fork()` 出此应用进程。

## 应用进程启动
### 基础组件启动触发进程创建
启动一个应用的任一四大基础组件 **Activity**、**Service**、**ContentProvider** 和 **BroadcastReceiver** 都可以通过系统进程来创建该目标应用的进程。一个组件

以 [Activity 的启动为例](/2021/03/16/activity-launch/)，发起调用的进程（如 launcher 进程）会通过 **Binder** 与 `system_server` 进程通信，系统服务 **ActivityTaskManagerService** 在启动目标 Activity 时，会先此 Activity 所属的进程是否在运行中。判断的方法实现如下：

```java
boolean isProcessRunning() {
    WindowProcessController proc = app;
    if (proc == null) {
        // mAtmService 表示 ActivityTaskManagerService
        // processName 为目标 Activity 所属的进程
        proc = mAtmService.mProcessNames.get(processName, info.applicationInfo.uid);
    }
    return proc != null && proc.hasThread();
}
```

若目标 Activity 所属的进程名未在运行中，则会逐步调用，通过 **Process** 类的静态方法来启动一个新的应用进程：
![](/img/posts/post-process-start.svg)
<!-- Title: 应用进程启动（一）
Note right of ActivityTaskManagerService: systerm_server 进程
ActivityTaskManagerService->ActivityManagerService$\nLocalService: startProcessAsync()
ActivityManagerService$\nLocalService->ActivityManagerService: startProcess()
ActivityManagerService->ProcessList: startProcessLocked()
ProcessList->ProcessList: startProcessLocked()
ProcessList->Process: startProcess()
Process->Process: start()
Process->Process: ... -->

```white
frameworks/base/services/core/java/com/android/server/wm/
    - ActivityTaskManagerService.java

frameworks/base/services/core/java/com/android/server/am/
    - ActivityManagerService.java
    - ProcessList.java

frameworks/base/core/java/android/os/
    - Process.java
```

### 与 Zygote 进程建立 Socket 通信

然后，创建 **ZygoteProcess** 对象，负责与 **Zygote** 进程建立 **Socket** 通信：
![](/img/posts/post-process-start2.svg)
<!-- Title: 应用进程启动（二）
Process->ZygoteProcess: start()
ZygoteProcess->ZygoteProcess: start()
ZygoteProcess->ZygoteProcess: startViaZygote()
ZygoteProcess-\->Zygote: connect()
Note right of ZygoteProcess: <- system_server 进程｜Zygote 进程 -> 
ZygoteProcess->ZygoteProcess: zygoteSendArgsAndGetResult()
ZygoteProcess->ZygoteProcess: attemptZygoteSendArgsAndGetResult()
ZygoteProcess-\->Zygote: write()
Zygote->Zygote: ... -->
```white
frameworks/base/core/java/android/os/
    - ZygoteProcess.java

frameworks/base/core/java/com/android/internal/os/
    - Zygote.java

frameworks/base/core/java/android/net/
    - LocalSocket.java
    - LocalSocketAddress.java
```

这里主要做了以下几件事：
- 根据目标要启动进程支持的 abi 来选择与 **Zygote** 还是 **Zygote64**（分别对应 32 和 64 位）建立 **Socket** 通信；
- 将新进程启动需要的 uid、gid、runtime-flags 等一系列参数拼接成一个字符串；
- 通过 Socket 通道向 `Zygote` 进程发送该参数，进入阻塞，等待 Socket 另一端 `Zygote` 进程返回新创建的应用进程 pid。

### Zygote 进程 fork 出应用进程
<待补充...>

<!-- ```white
Title: 应用进程启动（三）
participant RuntimeInit
ZygoteInit->ZygoteServer: main()
ZygoteServer->ZygoteConnection: runSelectLoop()
ZygoteConnection->Zygote: processOneCommand()
Zygote->Zygote: forkAndSpecialize()
Zygote-\->ZygoteConnection: pid
Note over RuntimeInit,Zygote: 进入子进程
ZygoteConnection->ZygoteInit: handleChildProc()
ZygoteInit->RuntimeInit: zygoteInit()
RuntimeInit->RuntimeInit: applicationInit()
RuntimeInit->RuntimeInit: findStaticMain()
RuntimeInit-\->ZygoteInit: runnable
ZygoteInit->ZygoteInit: runnable.run()

Title: 应用进程启动（四）
Zygote->Zygote: forkAndSpecialize()
Zygote-\->JNI: nativeForkAndSpecialize()
JNI->Linux: ForkCommon()
Linux->Linux: fork()
Linux-\->Zygote: pid

Title: 应用进程启动（五）
participant ActivityManagerService
participant ActivityThread$ApplicationThread
participant ActivityThread
participant LoadedApk
participant Instrumentation
ActivityThread->ActivityThread: main()
ActivityThread->ActivityThread: attach()
ActivityThread-\->ActivityManagerService: attachApplication()
ActivityManagerService->ActivityManagerService: attachApplication()
ActivityManagerService->ActivityManagerService: attachApplicationLocked()
ActivityManagerService-\->ActivityThread$ApplicationThread: bindApplication()
ActivityThread$ApplicationThread->ActivityThread: bindApplication() 
ActivityThread->LoadedApk: handleBindApplication()
LoadedApk->Instrumentation: makeApplication()
Instrumentation->Application: newApplication()
Application->Application: attach()
Application->Application: attachBaseContext()
Application->Application: onCreate()
``` -->









