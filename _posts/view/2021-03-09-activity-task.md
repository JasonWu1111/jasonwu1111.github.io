---
layout: post
title: Activity 任务、亲和关系与启动模式
tags:
  - Android
  - Activity
---

# Activity 与任务
**任务（Task）** 是一系列 Activity 的集合，本质上是一个管理多个 Activity 的堆栈。

当前 Activity 启动另一个 Activity 时，新的 Activity 将被推送到堆栈顶部；按返回按钮时，当前 Activity 又会从堆栈顶部退出（销毁），堆栈按照“**后进先出**”的对象结构运作。

![](/img/posts/post-activity-task.png)

当堆栈中所有的 Activity 都被移除后，该任务将不复存在。

在 Activity 中可通过以下方法查看当前任务信息：
```java
public int getTaskId() // 获取当前 Activity 的任务对应的 id
public boolean isTaskRoot() // 返回当前 Activity 是否为所在任务的栈根
```

# 任务亲和性
“**亲和性**”表示 Activity 倾向于属于哪个任务。默认情况下，同一应用中的所有 Activity 彼此具有亲和性。因此，**在默认情况下，同一应用中的所有 Activity 都倾向于位于同一任务**。

可以使用 `<activity>` 元素的 `taskAffinity` 属性修改任何给定 Activity 的亲和性（未设置时默认值为应用包名）。

<!-- 亲和性可在两种情况下发挥作用：

当 Activity 的 allowTaskReparenting 属性设为 "true" 时。
在这种情况下，一旦和 Activity 有亲和性的任务进入前台运行，Activity 就可从其启动的任务转移到该任务。

举例来说，假设一款旅行应用中定义了一个报告特定城市天气状况的 Activity。该 Activity 与同一应用中的其他 Activity 具有相同的亲和性（默认应用亲和性），并通过此属性支持重新归属。当您的某个 Activity 启动该天气预报 Activity 时，该天气预报 Activity 最初会和您的 Activity 同属于一个任务。不过，当旅行应用的任务进入前台运行时，该天气预报 Activity 就会被重新分配给该任务并显示在其中。 -->

# 启动模式
Activity 的**启动模式**决定了一个新启动的 Activity 如何推入到目标任务栈中。可以通过两种方式定义不同的启动模式：
- 使用清单文件
- 使用 Intent 标记

### 清单文件配置启动模式
启动模式在可清单文件中通过 `<activity>` 元素的 `launchMode` 属性指定配置，共以下四种：

`standard`
- 默认。系统在启动该 Activity 的任务中创建 Activity 的新实例。
- 每个实例可以属于不同的任务，一个任务可以拥有多个实例。

`singleTop`
- 如果当前任务的栈顶已经是该 Activity 的实例，会通过调用该实例的 onNewIntent() 方法向其传送 Intent，而不是创建 Activity 的新实例；
- 未有实例或实例不在栈顶时，创建新的 Activity 实例入栈。

`singleTask`
- 若目标任务栈已存在（任务亲和性 `taskAffinity` 指定），切换该任务到前台；
  - 如果 Activity 实例已存在，把该实例以上的其他 Activity 全部弹出栈（销毁），并调用 onNewIntent() 方法；
  - 如果 Activity 实例不存在，创建实例入栈顶；
- 若目标任务栈不存在，则系统会创建新任务，并实例化为新任务的根 Activity。

`singleInstance`
- 创建新任务并实例化为新任务的根 Activity；
- 系统不会将任何其他 Activity 启动到包含该实例的任务中，该 Activity 始终是其任务唯一的成员；
- 由该 Activity 启动的任何 Activity 都会在其他的任务中打开。

### 使用 Intent 标记
通过 **Intent 标记（FLAG）**可以灵活设置每次启动 Activity 时与任务的关联关系：

`FLAG_ACTIVITY_NEW_TASK`
- 若目标任务栈（任务亲和性 `taskAffinity` 指定）未存在，则系统会创建新任务，并实例化为新任务的根 Activity；
- 若目标任务栈已存在且栈根为 Activity 的实例，则会把目标任务栈切到前台（不再创建实例，即便栈顶为其他 Activity）；
- 如果栈根不为目标 Activity 的实例，则新建 Activity 的实例入栈（即便栈顶已是该 Activity 的实例）。

`FLAG_ACTIVITY_SINGLE_TOP`
- 与清单文件配置配置的 `singleTop` 启动模式表现相同。

`FLAG_ACTIVITY_CLEAR_TOP`
- 如果要启动的 Activity 已经在当前任务中运行，则把该实例以上的其他 Activity 全部弹出栈（销毁）。<br>
- 当 launchMode 设置为 `standard` 且没有同时设置 `FLAG_ACTIVITY_SINGLE_TOP`，则该实例会被销毁然后重新创建。
- 当 launchMode 设置为其他的，或者同时设置了 `FLAG_ACTIVITY_SINGLE_TOP`，当前实例会回调 onNewIntent() 方法。

`FLAG_ACTIVITY_MULTIPLE_TASK`
- 不与 `FLAG_ACTIVITY_NEW_DOCUMENT` 或 `FLAG_ACTIVITY_NEW_TASK` 任意一个合用，这个标记将会失效;
- 与 `FLAG_ACTIVITY_NEW_TASK` 一起使用时，会始终创建新的任务并实例化目标 Activity。

# 其他管理任务的属性
清单文件中 `<activity>` 元素还有以下一些属性可用于管理 Activity 与任务的关系：

`allowTaskReparenting`
- 当下一次将启动 Activity 的任务转至前台时，Activity 是否能从该任务转移至与其有相似性的任务。默认值为“false”。

- 正常情况下，Activity 启动时会与启动它的任务关联，并在其整个生命周期中一直留在该任务处。当不再显示现有任务时，可以使用该属性强制 Activity 将其父项更改为与其有相似性的任务。该属性通常用于将应用的 Activity 转移至与该应用关联的主任务。

`alwaysRetainTaskState`
- 如果在任务的根 Activity 中将该属性设为 "true"，则不会发生上述默认行为。即使经过很长一段时间后，任务仍会在其堆栈中保留所有 Activity。

`clearTaskOnLaunch`
- 如果在任务的根 Activity 中将该属性设为 "true"，那么只要用户离开任务再返回，堆栈就会被清除到只剩根 Activity。也就是说，它与 alwaysRetainTaskState 正好相反。用户始终会返回到任务的初始状态，即便只是短暂离开任务也是如此。

`finishOnTaskLaunch`
- 该属性与 clearTaskOnLaunch 类似，但它只会作用于单个 Activity 而非整个任务。它还可导致任何 Activity 消失，包括根 Activity。如果将该属性设为 "true"，则 Activity 仅在当前会话中归属于任务。如果用户离开任务再返回，则该任务将不再存在。

