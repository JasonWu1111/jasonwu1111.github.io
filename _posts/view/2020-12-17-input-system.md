---
layout: post
title: Input 系统触摸事件分发原理
author: JasonWu
tags:
  - Android
  - View
  - Window
  - Input
---

当用户在 Android 系统设备上触摸屏幕或者按键操作时，会先触发硬件驱动，硬件驱动受到事件后，会将相应事件写入到设备节点（`/dev/input/` 目录下），这便产生了最原生态的内核事件。比如我的测试设备有以下一些设备节点：
```shell
OnePlus5T:/ $ ls /dev/input/                                                                                           
event0 event1 event2 event3 event4 event5 event6 mice mouse0 
```

有 event0 代表电源键输入，event3 代表音量键输入，event4 代表屏幕输入等。

可通过 `getevent` 命令实时监听某个设备节点的事件写入情况，如一个轻触屏幕的点击会产生以下事件：
```shell
OnePlus5T:/ $ getevent -lt /dev/input/event4                                                                           
[  277099.294712] EV_ABS       ABS_MT_TRACKING_ID   00002e0a            
[  277099.294712] EV_KEY       BTN_TOOL_FINGER      DOWN                
[  277099.294712] EV_ABS       ABS_MT_POSITION_X    00000361            
[  277099.294712] EV_ABS       ABS_MT_POSITION_Y    0000056a            
[  277099.294712] EV_ABS       ABS_MT_TOUCH_MAJOR   00000005            
[  277099.294712] EV_ABS       ABS_MT_TOUCH_MINOR   00000005            
[  277099.294712] EV_SYN       SYN_REPORT           00000000            
[  277099.335669] EV_ABS       ABS_MT_TRACKING_ID   ffffffff            
[  277099.335669] EV_KEY       BTN_TOOL_FINGER      UP                  
[  277099.335669] EV_SYN       SYN_REPORT           00000000 
```

而后这一些输入事件会被输入系统服务来读取以及分发，直至最后被处理消费。

## InputManagerService 的启动
Android 的输入系统服务类为 **InputManagerService**，其在启动时会进一步在 native 层启动两个核心的功能模版 **InputReader** 和 **InputDispatcher**。InputReader 用于输入事件的读取，而 InputDispatcher 用于输入事件的分发。

整体关系如下：
![](/img/posts/post-view-input2.png)

<!-- Title: InputManagerService 的启动
participant InputManagerService
participant InputManager\n(native)
participant InputDispatcher\n(native)
participant InputReader\n(native)
InputManagerService->InputManagerService: start()
InputManagerService->InputManager\n(native): nativeStart()
InputManager\n(native)->InputReader\n(native): start()
InputDispatcher\n(native)->InputDispatcher\n(native): start()
InputReader\n(native)->InputReader\n(native): start() -->

`InputManagerService` 服务启动整体流程的时序图如下：
![](/img/posts/post-view-input1.png)
```white
frameworks/base/services/core/java/com/android/server/input/
    - InputManagerService.java
    
frameworks/base/services/core/jni/
    - com_android_server_input_InputManagerService.cpp

frameworks/native/services/inputflinger/
    - InputManager.cpp

frameworks/native/services/inputflinger/dispatcher/
    - InputDispatcher.cpp

frameworks/native/services/inputflinger/reader/
    - InputReader.cpp
```

## InputReader 读取设备输入事件
`InputReader` 内会启动一个同名的 Looper 线程专用于循环来读取设备的输入事件：
```cpp
status_t InputReader::start() {
    if (mThread) {
        return ALREADY_EXISTS;
    }
    mThread = std::make_unique<InputThread>(
            "InputReader", [this]() { loopOnce(); }, [this]() { mEventHub->wake(); });
    return OK;
}
```

`loopOnce()` 为每次循环执行的方法，其内会通过 **EventHub** 类来扫描设备的输入事件（`/dev/input` 目录下），整理调用流程如下：
![](/img/posts/post-view-input3.png)

<!-- Title: 通过 EventHub 扫描设备的输入事件 
participant InputReader\n(native)
participant EventHub\n(native)

Note right of InputReader\n(native): 循环执行
InputReader\n(native)->EventHub\n(native): loopOnce()
EventHub\n(native)->EventHub\n(native): getEvents()
EventHub\n(native)->EventHub\n(native): scanDevicesLocked()
EventHub\n(native)->EventHub\n(native): scanDirLocked("/dev/input")
InputReader\n(native)->InputReader\n(native): processEventsLocked\n(RawEvent* rawEvents)
InputReader\n(native)->InputReader\n(native): processEventsForDeviceLocked\n(rawEvent) -->

```white
frameworks/native/services/inputflinger/reader/
    - EventHub.h
    - EventHub.cpp
```

其后，**InputReader** 会将同一设备的原始事件 `RawEvent` 打包一起，然后根据事件类型，找到合适的 **InputMapper** 来进一步处理。

**InputMapper** 根据设备类型分成几类，下面列举一些常见的 InputMapper:

|InputMapper|说明
|--|--
|KeyboardInputMapper|键盘类设备
|CursorInputMapper|鼠标类设备
|MultiTouchInputMapper|多点触摸屏类设备
|SingleTouchInputMapper|单点触摸屏类设备

对于触摸事件而言，由 **TouchInputMappper**(Multi or Single) 作处理，并传递到 **InputDispatcher** 中，整体流程如下：
![](/img/posts/post-view-input4.png)

<!-- Title: InputMapper 作事件进一步处理、传递 
participant InputReader\n(native)
participant TouchInputMapper\n(native)
participant QueuedInputListener\n(native)
participant InputDispatcher\n(native)

InputReader\n(native)->TouchInputMapper\n(native): processEventsForDeviceLocked\n(rawEvent)
TouchInputMapper\n(native)->TouchInputMapper\n(native): process(rawEvent)\n...
TouchInputMapper\n(native)->QueuedInputListener\n(native): dispatchMotion()
QueuedInputListener\n(native)->QueuedInputListener\n(native): notifyMotion\n(NotifyMotionArgs* args)
QueuedInputListener\n(native)->QueuedInputListener\n(native): flush()
QueuedInputListener\n(native)->InputDispatcher\n(native): args->notify(args)
InputDispatcher\n(native)-InputDispatcher\n(native): notifyMotion(args)
InputDispatcher\n(native)-InputDispatcher\n(native): enqueueInboundEventLocked\n(EventEntry* entry) -->

```white
frameworks/native/services/inputflinger/reader/mapper/
    - TouchInputMapper.cpp

frameworks/native/services/inputflinger/include/
    - InputListener.h

frameworks/native/services/inputflinger/
    - InputListener.cpp

frameworks/native/services/inputflinger/dispatcher/
    - Entry.h
```

## InputDispatcher 分发事件
**InputDispatcher** 顾名思义，用于分发从 **InputReader** 中读取出来的事件，其在启动同样会开启一个 Looper 线程循环执行事件的分发：
```cpp
status_t InputDispatcher::start() {
    if (mThread) {
        return ALREADY_EXISTS;
    }
    mThread = std::make_unique<InputThread>(
            "InputDispatcher", [this]() { dispatchOnce(); }, [this]() { mLooper->wake(); });
    return OK;
}
```

每次触摸事件分发的处理中，会通过 `findTouchedWindowTargetsLocked()` 方法找到可接受事件的目标窗口，做进一步的派发，整体流程如下：

![](/img/posts/post-view-input5.png)

<!-- Title: InputDispatcher 分发事件至目标窗口
participant InputDispatcher\n(native)
InputDispatcher\n(native)->InputDispatcher\n(native): dispatchOnce()
InputDispatcher\n(native)->InputDispatcher\n(native): dispatchOnceInnerLocked()
InputDispatcher\n(native)->InputDispatcher\n(native): dispatchMotionLocked(MotoinEntry* entry)
Note right of InputDispatcher\n(native): 查找可接受目标事件的窗口\nfindTouchedWindowTargetsLocked(entry)
InputDispatcher\n(native)->InputDispatcher\n(native): dispatchEventLocked(entry, vector<InputTarget> inputTargets) -->

```white
frameworks/native/include/input/
    - InputWindow.h
    - InputWindow.cpp
```

### 查找响应事件的目标窗口
那 **InputDispatcher** 是如果查找到应该处理事件的目标窗口呢？首先看下设备中的各个窗口（包括系统和应用端的）是如何和 InputDispatcher 关联起来的。

对 Android 系统而言，各类型的窗口统一通过系统服务 **WindowManagerService** 来管理增删改等操作的。以窗口新增为例，每当有新的窗口添加时，以 `addWindow()` 为出发点，会逐步通知到 **InputDispatcher** 更新窗口的信息，整体流程如下：

![](/img/posts/post-view-input6.png)

<!-- Title: 更新窗口信息至 InputDispatcher
participant WindowManagerService
participant InputMonitor
participant ...
participant SurfaceFlinger\n(native)
participant InputManager\n(native)
participant InputDispatcher\n(native)
WindowManagerService->InputMonitor: addWindow()
InputMonitor->...: updateInputWindows()
...->SurfaceFlinger\n(native): ...
SurfaceFlinger\n(native)->InputManager\n(native): updateInputWindowInfo()
InputManager\n(native)->InputDispatcher\n(native): setInputWindows\n(vector<InputWindowInfo> infos)
InputDispatcher\n(native)->InputDispatcher\n(native): setInputWindows\n(vector<InputWindowHandle> handles) -->

```white
frameworks/base/services/core/java/com/android/server/wm/
    - WindowManagerService.java
    - InputMonitor.java

frameworks/native/services/surfaceflinger/
    - SurfaceFlinger.cpp
    
frameworks/native/include/input/
    - IInputFlinger.h
    - InputWindow.h
```

其中，每个 **InputWindowInfo** 实例包含了一个窗口的 displayId、token、布局参数等关键信息：
```cpp
struct InputWindowInfo {
    ...
    sp<IBinder> token; // 窗口 token
    int32_t id = -1;
    std::string name;
    int32_t layoutParamsFlags = 0;
    int32_t layoutParamsType = 0;
    ...
    Region touchableRegion; // 可触摸区域
    bool visible = false; // 窗口是否可见
    bool canReceiveKeys = false;
    bool hasFocus = false;
    bool hasWallpaper = false;
    bool paused = false;
    int32_t ownerPid = -1;
    int32_t ownerUid = -1;
    int32_t inputFeatures = 0;
    int32_t displayId = ADISPLAY_ID_NONE; // displayId
    int32_t portalToDisplayId = ADISPLAY_ID_NONE;
    InputApplicationInfo applicationInfo;
    ...
}
```

然后，回到 **InputDispatcher** 内，其在 `findTouchedWindowTargetsLocked()` 方法中会通过一系列的判断条件，从全部的窗口信息中找到应该接受该触摸事件的目标窗口，并分发事件到该窗口上。判断的条件主要有以下几点：
- displayId 一致；
- 窗口当前时可见的；
- 目标事件的触摸点坐标在窗口可触控区域内；

在 **InputDispatcher** 找到分发事件的目标窗口信息后，要怎么实现通信把具体的事件数据发送到该窗口上呢？

由于 InputDispatcher 线程运行在系统服务所在的 **system_server 进程**，每个应用的窗口所操控的 RootView 运行在各自的**应用进程**，这里就需要进行跨进程通信。这里，系统采用了 **socket** 的通信方式。

## InputChannel 通信渠道的建立
### 双端 InputChannel 的创建
首先，在每个窗口视图创建时，会在应用进程的 **ViewRootImpl** `setView()` 方法中创建一个 `InputChannel` 对象，该对象已实现可序列化，然后与 WindowManagerService 的 Binder 跨进程通信中传递：
```java
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView,
            int userId) {
    ...
    // 创建应用进程的 InputChannel 对象
    InputChannel inputChannel = null;
    if ((mWindowAttributes.inputFeatures
            & WindowManager.LayoutParams.INPUT_FEATURE_NO_INPUT_CHANNEL) == 0) {
        inputChannel = new InputChannel();
    }
    ...
    try {
        // Binder 跨进程通信添加窗口，inputChannel 实质内容在 system_server 进程填充
        res = mWindowSession.addToDisplayAsUser(..., inputChannel, ...);
    } ...
}
```

然后，`system_server` 进程中，WindowManagerService 会进一步调用并创建返回两个 `InputChannel` 对象，分别对应 **client** 端和 **server** 端，并把 **client** 的 native 对象数据转移到上一步 ViewRootImpl 创建的 InputChannel 对象中，整理流程如下：

![](/img/posts/post-view-input7.png)

<!-- Title: InputChannel 的创建
participant ViewRootImpl
participant WindowManagerService
participant WindowState
participant InputChannel
participant InputChannel\n(native)

ViewRootImpl-\->WindowManagerService: setView\n(InputChannel inputChannel)
Note right of ViewRootImpl: <- 应用进程 ｜ 系统服务进程 ->
WindowManagerService->WindowState: addWindow\n(outInputChannel)
WindowState->InputChannel: openInputChannel\n(outInputChannel)
InputChannel->InputChannel\n(native): openInputChannelPair()
InputChannel\n(native)->InputChannel\n(native): openInputChannelPair()
InputChannel\n(native)-\->WindowState: {serverChannel, clientChannel}
Note right of WindowState: clientChannel.transferTo(outInputChannel) -->

```white
frameworks/base/core/java/android/view/
    - ViewRootImpl.java
    - InputChannel.java

frameworks/base/services/core/java/com/android/server/wm/
    - WindowState.java

frameworks/base/core/jni/
    - android_view_InputChannel.cpp

frameworks/native/libs/input/
    - InputTransport.cpp
```

流程的最后，在 **InputChannel(native)** 的 `openInputChannelPair()` 方法中，真正实现了 socket 对以及 InputChannel 对象的创建。代码实例如下：

```cpp
status_t InputChannel::openInputChannelPair(const std::string& name,
        sp<InputChannel>& outServerChannel, sp<InputChannel>& outClientChannel) {
    int sockets[2];
    // 创建 socket 对
    if (socketpair(AF_UNIX, SOCK_SEQPACKET, 0, sockets)) {
        ...
        return result;
    }

    int bufferSize = SOCKET_BUFFER_SIZE; // 32KB
    setsockopt(sockets[0], SOL_SOCKET, SO_SNDBUF, &bufferSize, sizeof(bufferSize));
    setsockopt(sockets[0], SOL_SOCKET, SO_RCVBUF, &bufferSize, sizeof(bufferSize));
    setsockopt(sockets[1], SOL_SOCKET, SO_SNDBUF, &bufferSize, sizeof(bufferSize));
    setsockopt(sockets[1], SOL_SOCKET, SO_RCVBUF, &bufferSize, sizeof(bufferSize));

    sp<IBinder> token = new BBinder();

    std::string serverChannelName = name + " (server)";
    android::base::unique_fd serverFd(sockets[0]);
    // 创建 server 端 InputChannel 对象，名称后缀为 (server)
    outServerChannel = InputChannel::create(serverChannelName, std::move(serverFd), token);

    std::string clientChannelName = name + " (client)";
    android::base::unique_fd clientFd(sockets[1]);
    // 创建 client 端 InputChannel 对象，名称后缀为 (client)
    outClientChannel = InputChannel::create(clientChannelName, std::move(clientFd), token);
    return OK;
}
```

在 **client** 和 **server** 的通信渠道 `InputChannel` 对象创建后，分别有以下绑定关系：
  - **InputChannel(client)** ：与窗口视图所在的应用进程的主线程绑定；
  - **InputChannel(server)** ：与 system_server 进程的 InputDispatcher 线程绑定；

### server 端 InputChannel 的注册
我们先看 server 端 `InputChannel` 对象的注册，调用流程如下：

![](/img/posts/post-view-input8.png)

<!-- Title: server 端 InputChannel 的注册
participant WindowState
participant InputManagerService
participant InputManager\n(native)
participant InputDispatcher\n(native)

WindowState-\->InputManagerService: {serverChannel}
InputManagerService->InputManager\n(native): registerInputChannel\n(serverChannel)
InputManager\n(native)->InputDispatcher\n(native): registerInputChannel\n(channel)
InputDispatcher\n(native)->InputDispatcher\n(native): registerInputChannel(channel)
InputDispatcher\n(native)->InputDispatcher\n(native): Looper::addFd(channel->getFd()) -->

具体看 **InputDispatcher** 的 `registerInputChannel()` 方法实现：
```cpp
status_t InputDispatcher::registerInputChannel(const sp<InputChannel>& inputChannel) {
    { // 获取锁
        std::scoped_lock _l(mLock);
        ...
        sp<Connection> connection = new Connection(inputChannel, false, mIdGenerator);

        int fd = inputChannel->getFd();
        mConnectionsByFd[fd] = connection;
        mInputChannelsByToken[inputChannel->getConnectionToken()] = inputChannel;
        // 将该 InputChannel 的 socket fd 添加到 InputDispatcher 的 Looper 中去
        mLooper->addFd(fd, 0, ALOOPER_EVENT_INPUT, handleReceiveCallback, this);
    } // 释放锁

    mLooper->wake(); // connections 改变，唤醒 Looper
    return OK;
}
```

这里把 **socket server 端的 fd 通过 epoll 机制注册监听**，回调方法为 `handleReceiveCallback()`。

### client 端 InputChannel 的注册
同理，client 端 `InputChannel` 的注册机制是类似的，在 ViewRootImpl 创建 InputChannel 对象并跨进程通过 WindowManagerService 填充 native 对象数据后，进一步创建 **InputEventReceiver** 对象用于输入事件的接受并把 InputChannel 的 fd 注册到当前应用主线程的 Looper 中去。

整体调用如下：

![](/img/posts/post-view-input9.png)

<!-- Title: client 端 Input 事件接受的初始化
participant ViewRootImpl
participant WindowInputEventReceiver
participant NativeInputEventReceiver\n(native)

ViewRootImpl->WindowInputEventReceiver: setView()
WindowInputEventReceiver->WindowInputEventReceiver: <init>(inputChannel, looper)
WindowInputEventReceiver->NativeInputEventReceiver\n(native): nativeInit(inputChannel, messageQueue)
NativeInputEventReceiver\n(native)->NativeInputEventReceiver\n(native): <init>
NativeInputEventReceiver\n(native)->NativeInputEventReceiver\n(native): initialize()
NativeInputEventReceiver\n(native)->NativeInputEventReceiver\n(native): setFdEvents()
NativeInputEventReceiver\n(native)->NativeInputEventReceiver\n(native): Looper::addFd(channel->getFd()) -->

```light
frameworks/base/core/java/android/view/
    - InputEventReceiver.java

frameworks/base/core/jni/
    - android_view_InputEventReceiver.cpp
```

至此，双端通信的模型边建立起来：

![](/img/posts/post-view-input10.png)

## 触摸事件分发至应用端窗口
回到 **InputDispatcher** 中的事件分发逻辑，在 `diaptchEventLocked()` 方法执行中，逐步调用，最终通过注册了 socket 实现的 **InputChannel(server)** 发送信息，通知应用进程窗口视图接受触摸事件：

![](/img/posts/post-view-input11.png)

<!-- Title: InputDispatcher 分发事件
participant InputDispatcher\n(native)
participant InputPublisher\n(native)
participant InputChannel\n(native)

InputDispatcher\n(native)->InputDispatcher\n(native): dispatchEventLocked\n(EventEntry* eventEntry)
InputDispatcher\n(native)->InputDispatcher\n(native): prepareDispatchCycleLocked()
InputDispatcher\n(native)->InputDispatcher\n(native): enqueueDispatchEntriesLocked()
InputDispatcher\n(native)->InputPublisher\n(native): startDispatchCycleLocked()
InputPublisher\n(native)->InputChannel\n(native):publishMotionEvent()
InputChannel\n(native)->InputChannel\n(native): sendMessage\n(InputMessage* msg)
InputChannel\n(native)->InputChannel\n(native): socket::send() -->

在应用进程主线程处，**NativeInputEventReceiver** 会接受到信息的到来，并回调 `handleEvent()` 方法开始接手事件的处理，并逐步分发至窗口根视图 RootView 上：

![](/img/posts/post-view-input13.png)

<!-- Title: RootView 对触摸事件的接收
participant NativeInputEventReceiver\n(native)
participant WindowInputEventReceiver
participant ViewRootImpl
participant RootView

NativeInputEventReceiver\n(native)->NativeInputEventReceiver\n(native): handleEvent()
NativeInputEventReceiver\n(native)->WindowInputEventReceiver: consumeEvents()
WindowInputEventReceiver->WindowInputEventReceiver: onInputEvent\n(InputEvent event)
WindowInputEventReceiver->ViewRootImpl: enqueueInputEvent(event)
ViewRootImpl->ViewRootImpl: doProcessInputEvents()
ViewRootImpl->ViewRootImpl: deliverInputEvent()
ViewRootImpl->ViewRootImpl: ...
ViewRootImpl->RootView: processPointerEvent()
RootView->RootView: dispatchPointerEvent()
RootView->RootView: dispatchTouchEvent\n(MotionEvent event) -->

在触摸事件分发完成后，会通过 **InputChannel(client)** 的 socket 发送 `finished` 信号信息回系统服务进程的 InputDispatcher 线程中通知当前输入事件已消费完毕，并开始下一个事件的分发处理：

![](/img/posts/post-view-input12.png)

<!-- Title: 事件消费回调
participant ViewRootImpl
participant WindowInputEventReceiver
participant NativeInputEventReceiver\n(native)
participant InputConsumer\n(native)
participant InputDispatcher\n(native)

ViewRootImpl->ViewRootImpl: deliverInputEvent()
Note right of ViewRootImpl: 事件分发完成
ViewRootImpl->WindowInputEventReceiver: finishInputEvent()
WindowInputEventReceiver->NativeInputEventReceiver\n(native): finishInputEvent()
NativeInputEventReceiver\n(native)->InputConsumer\n(native): finishInputEvent()
InputConsumer\n(native)->InputConsumer\n(native): sendFinishedSignal()
InputConsumer\n(native)->InputConsumer\n(native): sendUnchainedFinishedSignal()
InputConsumer\n(native)-\->InputDispatcher\n(native):mChannel->sendMessage()
Note right of InputConsumer\n(native): <- 应用进程 ｜ 系统服务进程 ->
InputDispatcher\n(native)->InputDispatcher\n(native): handleReceiveCallback()
InputDispatcher\n(native)->InputDispatcher\n(native): finishDispatchCycleLocked()\n...
Note right of InputDispatcher\n(native): 启动下一个事件处理循环
InputDispatcher\n(native)->InputDispatcher\n(native): startDispatchCycleLocked() -->

可见，一个触摸事件分发的完整过程共有两次 socket 通信：
- 系统服务 InputDispatcher 线程 -> 应用主线程：发送新触摸事件信息
- 应用主线程 -> 系统服务 InputDispatcher 线程：发送触摸事件已被消费完成信号

至此，一个完整触摸事件的分发流程便完成。

## 总结
按时间线顺序，整个流程可归纳为：
1. 设备启动时，系统进程启动服务 **InputManagerService**，并创建 **InputReader** 和 **InputDispatcher** 两个 Looper 线程；
2. 在有新窗口创建时，更新窗口信息至 **InputDispatcher**；
3. 同时创建应用进程和系统服务进程间的双端通信渠道 **InputChannel** 并通过 **socket** 实现通信；
4. 屏幕硬件设备被触碰时，产生原始事件写入到 `/dev/input/` 目录下的设备节点中；
5. **InputReader** 线程扫描设备节点并读取设备输入事件，发送至 **InputDispatcher** 中；
6. **InputDispatcher** 内找到需要处理事件的窗口信息，通过 **socket** 发送至目标应用的窗口；
7. 目标应用窗口接受并分发、消费目标事件，再通过 **socket** 通知 **InputDispatcher** 事件已消费完成。

