---
layout: post
title: Activity 的启动过程分析
tags:
  - Android
  - Activity
---

**Activity** 为开发者最常接触和使用的系统基础组件，这注定了 Activity 启动的流程整体较为复杂，每个 Android 系统版本都会对 Activity 的启动过程做一定的优化和重构，但核心的流程思路是一致。本文继续以 **Android 11** 的代码为基础，详解 Activity 的启动流程。

## ActivityTaskManager 发起 startActivity 请求
在 Android 10 以前，包含 Activity 在内的四大组件的启动与管理，都是通过系统服务 **ActivityManagerService** 来负责。而从 Android 10 开始，Activity 的这部分抽离了出来，改到了在新的 **ActivityTaskManagerService** 中来处理。

与其他常用的系统服务一样，应用进程内也有相应的 **ActivityTaskManager** 和 ActivityTaskManagerService 对应，两者间通过 `Binder` 完成 IPC 通信。

启动一个新的 Activity 从 `startActivity()` 方法开始，逐步调用到 ActivityTaskManager 发起通信：

![](/img/posts/post-activity-start.png)

<!-- Title: Activity 的启动过程（一）
participant Activity
participant Instrumentation
participant ActivityTaskManager
participant ActivityTaskManagerService

Activity->Activity: startActivity(intent)
Activity->Instrumentation: startActivityForResult()
Instrumentation->ActivityTaskManager: execStartActivity()
ActivityTaskManager-\->ActivityTaskManagerService: startActivity()
ActivityTaskManagerService->ActivityTaskManagerService: startActivity()\n...
ActivityTaskManagerService-\->Instrumentation: result -->

```white
frameworks/base/core/java/android/app/
    - Activity.java
    - Instrumentation.java
    - IActivityTaskManager.aidl
    - ActivityTaskManager.java

frameworks/base/services/core/java/com/android/server/wm/    
    - ActivityTaskManagerService.java
```

其中实现的 `Binder` 通信对应的 aidl 文件为 **IActivityTaskManager.aidl**：
```java
interface IActivityTaskManager {
    int startActivity(in IApplicationThread caller, ..., in Intent intent, ...);
    ...
}
```

**ActivityTaskManager** 内持有一个单例实现为 IActivityTaskManager 接口的 client 端实现，代码示例如下：
```java
@SystemService(Context.ACTIVITY_TASK_SERVICE)
public class ActivityTaskManager {
    ...
    private static final Singleton<IActivityTaskManager> IActivityTaskManagerSingleton =
            new Singleton<IActivityTaskManager>() {
                @Override
                protected IActivityTaskManager create() {
                    final IBinder b = ServiceManager.getService(Context.ACTIVITY_TASK_SERVICE);
                    return IActivityTaskManager.Stub.asInterface(b);
                }
            };
    ...
}
```

而 **ActivityTaskManagerService** 运行在系统服务 `system_server` 进程，作为 IActivityTaskManager 接口的 server 端。

## Activity 任务逻辑处理

其后在 ActivityTaskManagerService 中进一步通过 **ActivityStarter** 来处理目标 Activity 的启动逻辑，流程如下：

![](/img/posts/post-activity-start2.png)

```white
frameworks/base/services/core/java/com/android/server/wm/
    - ActivityStarter.java
    - ActivityRecord.java
    - ActivityStack.java
```

<!-- Title: Activity 的启动过程（二）
ActivityTaskManagerService->ActivityTaskManagerService: startActivity()
ActivityTaskManagerService->ActivityStarter: startActivityAsUser()
ActivityStarter->ActivityStarter: execute()
ActivityStarter->ActivityStarter: executeRequest()
ActivityStarter->ActivityStarter: startActivityUnchecked(ActivityRecord r)
ActivityStarter->ActivityStarter: startActivityInner(r)
ActivityStarter->ActivityStarter: ... -->

在 ActivityStarter 中会创建一个用于**在系统服务中描述当前 Activity 的 `ActivityRecord` 对象**，其包含了如 token、Activity 的状态、intent 等各类信息，是系统服务层面用于记录 Activity 相关的各种操作后的**核心对象**。

在 **ActivityStarter** 的 `startActivityInner()` 方法中，完成了一系列与 Activity 任务相关的关键操作：

- `singleTop` 或 `singleTask` 启动模式下，Intent 标志统一带上 `FLAG_ACTIVITY_NEW_TASK`；<br>&nbsp;
- 查找是否有可复用的 Activity 任务（ActivityStack 对象），若没有则按需创建新的任务；<br>&nbsp;
- 在前台的任务的栈顶 Activity 是否刚好为要启动的 Activity 的实例，若是且启动模式为 `singleTop` or `singleTask` 或者启动 Intent 标志带有 `FLAG_ACTIVITY_SINGLE_TOP`，通知已有栈顶实例回调 `onNewIntent()`，本次启动不再创建新 Activity 实例。<br>&nbsp;

换言之，依据启动模式和 Intent 标志，**并不是每次调用 `startActivity()` 都会启动并创建新的 Activity 实例**。

## 启动新的 Activity

我们先看下会创建新的 Activity 实例往下的调用链路。

![](/img/posts/post-activity-start3.png)

<!-- Title: Activity 的启动过程（三）
ActivityStarter->RootWindowContainer: startActivityInner()
RootWindowContainer->ActivityStack: resumeFocusedStacksTopActivities()
ActivityStack->ActivityStack: resumeTopActivityUncheckedLocked()
ActivityStack->ActivityStackSupervisor: resumeTopActivityInnerLocked()
ActivityStackSupervisor->ActivityStackSupervisor: startSpecificActivity()
ActivityStackSupervisor->ActivityStackSupervisor: realStartActivityLocked()
ActivityStackSupervisor->ActivityStackSupervisor: ... -->

```white
frameworks/base/services/core/java/com/android/server/wm/
    - RootWindowContainer.java
    - ActivityStackSupervisor.java
```

**ActivityStackSupervisor**，原意设计为管理、处理系统中的各个 ActivityStack 对象。从 Android 10 开始，引入了 **RootWindowContainer** 表示整个设备的根窗口容器，逐步承担并替换部分 ActivityStackSupervisor 的职责。

在上面的调用栈，最终通过 ActivityStackSupervisor 的 `realStartActivityLocked()` 真正执行新 Activity 实例的启动。

从 ActivityTaskManagerService 开始，以上执行的流程都发生在系统服务进程中。其后，会通知到应用进程负责创建 Activity 实例对象，回调生命周期等一系列操作。该跨进程通信的过程同样是通过 **Binder** 实现：

![](/img/posts/post-activity-start4.png)

<!-- Title: Activity 的启动过程（四）
ActivityStackSupervisor->ClientLifecycleManager: realStartActivityLocked()
ClientLifecycleManager->ClientTransaction: scheduleTransaction()
ClientTransaction->IApplicationThread$Proxy: schedule()
IApplicationThread$Proxy-\->ActivityThread$ApplicationThread: scheduleTransaction()
Note right of IApplicationThread$Proxy: <- 应用进程 ｜ 系统服务进程 ->
ActivityThread$ApplicationThread->ActivityThread$ApplicationThread: ... -->

```white
frameworks/base/services/core/java/com/android/server/wm/
    - ClientLifecycleManager.java

frameworks/base/core/java/android/app/servertransaction/
    - ClientTransaction.java
    - LaunchActivityItem.java
    - ResumeActivityItem.java

frameworks/base/bin/main/android/app/
    - IApplicationThread.aidl
```

上面，Android Framework 引入了额外的可序列化对象 **ClientTransaction** 类用来包含一系列的消息事件发送到到应用进程来处理。其运作的原理是：
1. 在系统服务进程中，创建多个可序列化的 **ClientTransactionItem** 对象，每个对象代表一个具体要处理的消息事件，并添加到 ClientTransaction 内；
2. ClientTransaction 内持有 **IApplicationThread 的 BinderProxy 对象**，调用其 `scheduleTransaction()` 方法实现 Binder 跨进程通信，复制自身对象数据（包括其内的各个 ClientTransactionItem）到应用进程的 **ActivityThread** 中；
3. ActivityThread 进一步把 ClientTransaction 对象分发至 **TransactionExecutor** 来处理，其会分别执行各个 ClientTransactionItem 对象实现的回调方法来完成事件的处理。

> IApplicationThread 的 BinderProxy 对象由 ActivityTaskManagerService  的 `startActivity()` 方法传入

![](/img/posts/post-activity-start7.png)

<!-- Title: Activity 的启动过程（五）
ActivityThread$ApplicationThread->ActivityThread: scheduleTransaction\n(ClientTransaction transaction)
ActivityThread->ActivityThread: scheduleTransaction()
ActivityThread->ActivityThread$H: sendMessage()
ActivityThread$H->TransactionExecutor: handleMessage()
TransactionExecutor->TransactionExecutor: execute() -->

```white
frameworks/base/core/java/android/app/servertransaction/
    - TransactionExecutor.java
```

在 Activity 启动这一流程中，ClientTransaction 共添加了两个 ClientTransactionItem：**LaunchActivityItem** 和 **ResumeActivityItem**，分别对应 Activity 生命周期中的 `onCreate()` 和 `onResume()` 行为。实际上，Activity 生命周期中的大部分回调都有对应的 item:

|生命周期|ClientTransactionItem 子类
|--|--
|`ON_CREATE`|LaunchActivityItem
|`ON_START`|StartActivityItem
|`ON_RESUME`|ResumeActivityItem
|`ON_PAUSE`|PauseActivityItem
|`ON_STOP`|StopActivityItem
|`ON_DESTROY`|DestroyActivityItem

小结一下，Android 启动的主流程中，在系统服务进程和应用进程之间共**进行两次 Binder IPC 通信**，其 Client / Server 端对应关系如下：

![](/img/posts/post-activity-start5.png)

在第二次的 Binder IPC 通信中，ClientTransaction 对象的流转过程如下：

![](/img/posts/post-activity-start6.png)

## Activity 生命周期的回调

正常一个新 Activity 的启动，其生命周期的回调理应是：

![](/img/posts/post-activity-start11.png)

而在上文的讲述的 **ClientTransaction** 对象中只带有 **LaunchActivityItem** 和 **ResumeActivityItem**，其各自的回调只会执行到 “`onCreate`” 和 “`onResume`” 这两个链路，把中间缺少的 “`onStart`” 是怎么来的呢？

这里，Android 系统引入了 **TransactionExecutorHelper** 类来对将要执行的 Activity 生命周期做梳理：只需要传入此次 Transaction 任务的**开始**和**结束**状态，依据 Activity 生命周期的顺序，自动补上中间的状态。

比如在 Activity 的启动过程中，开始状态为 `ON_CREATE`，结束状态 `ON_RESUME`，则会补充上中间的 `ON_START`。Activity 完整生命周期的顺序在 `ActivityLifecycleItem` 中有定义：

```java
public abstract class ActivityLifecycleItem extends ClientTransactionItem {
    ...
    public static final int UNDEFINED = -1;
    public static final int PRE_ON_CREATE = 0;
    public static final int ON_CREATE = 1;
    public static final int ON_START = 2;
    public static final int ON_RESUME = 3;
    public static final int ON_PAUSE = 4;
    public static final int ON_STOP = 5;
    public static final int ON_DESTROY = 6;
    public static final int ON_RESTART = 7;
    ...
}
```

```white
frameworks/base/core/java/android/app/servertransaction/
    - TransactionExecutorHelper.java
    - ActivityLifecycleItem.java
```

最后，整理一下启动过程中，Activity 各个生命周期的调用链路，流程大致上都是一致的：

- `onCreate()` 方法的回调流程：

![](/img/posts/post-activity-start8.png)

<!-- Title: Activity 的启动过程（六）- onCreate 回调
TransactionExecutor->TransactionExecutor: execute()
TransactionExecutor->LaunchActivityItem: executeCallbacks()
LaunchActivityItem->ActivityThread: execute()
ActivityThread->ActivityThread: handleLaunchActivity()
ActivityThread->Instrumentation: performLaunchActivity()
Instrumentation->Activity: callActivityOnCreate()
Activity->Activity: performCreate()
Activity->Activity: onCreate() -->

- `onStart()` 方法的回调流程：

![](/img/posts/post-activity-start9.png)

<!-- Title: Activity 的启动过程（七）- onStart 回调
participant TransactionExecutor
participant ActivityThread
participant Instrumentation
participant Activity

TransactionExecutor->TransactionExecutor: execute()
TransactionExecutor->TransactionExecutor: executeLifecycleState()
TransactionExecutor->TransactionExecutor: cycleToPath()
TransactionExecutor->ActivityThread: performLifecycleSequence()
ActivityThread->Activity: handleResumeActivity()
Activity->Instrumentation: performStart()
Instrumentation->Activity: callActivityOnStart()
Activity->Activity: onStart() -->

- `onResume()` 方法的回调流程：

![](/img/posts/post-activity-start10.png)

<!-- Title: Activity 的启动过程（八）- onResume 回调
participant TransactionExecutor
participant ResumeActivityItem
participant ActivityThread
participant Instrumentation
participant Activity

TransactionExecutor->TransactionExecutor: execute()
TransactionExecutor->ResumeActivityItem: executeLifecycleState()
ResumeActivityItem->ActivityThread: execute()
ActivityThread->ActivityThread: handleResumeActivity()
ActivityThread->Activity: performResumeActivity()
Activity->Instrumentation: performResume()
Instrumentation->Activity: callActivityOnResume()
Activity->Activity: onResume() -->

<!-- Title: Activity 的启动过程（九）
ActivityStarter->ActivityStarter: startActivityInner()
ActivityStarter->ActivityStarter: deliverToCurrentTopIfNeeded(ActivityStack topStack)
ActivityStarter->ActivityRecord: deliverNewIntent(ActivityRecord top)
ActivityRecord->ClientLifecycleManager: deliverNewIntentLocked(intent)
ClientLifecycleManager->ClientTransaction: scheduleTransaction() -->

至此，一个新 Activity 的完整启动流程梳理完毕。
