---
layout: post
title: Android 图形系统（二）—— SurfaceFlinger 的启动与初始化
tags:
  - Android
  - Graphics
---

## surfaceflinger 进程启动
当系统 *`init`* 进程启动之后，会开始解析 *system/core/rootdir/init.rc* 文件，在 init.rc 文件中，申明了启动 **core** 的一系列服务：
```
...
on boot
...
    class_start core
...
```

其中包含了 *`surfaceflinger`* 服务（进程）。其信息定义在 *frameworks/native/services/surfaceflinger/surfaceflinger.rc*：
```
service surfaceflinger /system/bin/surfaceflinger
    class core animation // 申明此服务属于 core 类
    user system
    group graphics drmrpc readproc
    capabilities SYS_NICE
    onrestart restart zygote // 重启时出发 zygote 进程的重启
    task_profiles HighPerformance
    socket pdx/system/vr/display/client     stream 0666 system graphics u:object_r:pdx_display_client_endpoint_socket:s0
    socket pdx/system/vr/display/manager    stream 0666 system graphics u:object_r:pdx_display_manager_endpoint_socket:s0
    socket pdx/system/vr/display/vsync      stream 0666 system graphics u:object_r:pdx_display_vsync_endpoint_socket:s0
```

*`surfaceflinger`* 进程启动后，便可以通过 `ps` 命令查看到此进程的信息：
```shell
➜  ~ adb shell ps | grep surface
system         820     1  317136  18004 0                   0 S surfaceflinger
```

进程启动时，会先回调 `main()` 函数（定义在 *frameworks/native/services/surfaceflinger/main_surfaceflinger.cpp*）：
```cpp
int main(int, char**) {
    ...
    // 启动 HAL 层图形分配器服务
    startGraphicsAllocatorService();

    // 限制进程 binder 线程池个数上限为 4
    ProcessState::self()->setThreadPoolMaxThreadCount(4);
    // 启动该线程池
    sp<ProcessState> ps(ProcessState::self());
    ps->startThreadPool();

    // 创建 SurfaceFlinger 对象实例
    sp<SurfaceFlinger> flinger = surfaceflinger::createSurfaceFlinger();

    // 设置进程为高优先级以及前台调度策略
    setpriority(PRIO_PROCESS, 0, PRIORITY_URGENT_DISPLAY);
    set_sched_policy(0, SP_FOREGROUND);
    ...
    if (cpusets_enabled()) set_cpuset_policy(0, SP_SYSTEM);

    // 初始化 SurfaceFlinger
    flinger->init();

    // 注册对应服务“SurfaceFlinger”到 ServiceManager 中
    sp<IServiceManager> sm(defaultServiceManager());
    sm->addService(String16(SurfaceFlinger::getServiceName()), flinger, false,
                   IServiceManager::DUMP_FLAG_PRIORITY_CRITICAL | IServiceManager::DUMP_FLAG_PROTO);

    // 启动显示服务
    startDisplayService();

    // 进程调度模式设置为实时进程的FIFO
    if (SurfaceFlinger::setSchedFifo(true) != NO_ERROR) {
        ALOGW("Couldn't set to SCHED_FIFO: %s", strerror(errno));
    }

    // 在当前主线程中开始运行 SurfaceFlinger 内部消息处理机制
    flinger->run();

    return 0;
}
```

整理下，`main()` 函数主要完成了以下一些工作：
![](/img/posts/post-sf-main.png)

接着来深入看下 **SurfaceFlinger** 实例的创建和初始化过程。

## SurfaceFlinger 的启动
承接上面的 `main()` 函数中对 **SurfaceFlinger** 的创建以及初始化操作，梳理出 SurfaceFlinger 整个启动过程主要方法调用时序如下：

![](/img/posts/post-sf-launch.svg)
<!-- Title: SurfaceFlinger 的启动
SurfaceFlinger\n(native)->SurfaceFlinger\n(native): SurfaceFlinger(Factory&)
SurfaceFlinger\n(native)->SurfaceFlinger\n(native): onFirstRef() // 被 sp 强指针首次引用时调用
SurfaceFlinger\n(native)->SurfaceFlinger\n(native): init() // 初始化
SurfaceFlinger\n(native)->SurfaceFlinger\n(native): run() // 运行消息处理机制 -->

```white
frameworks/native/services/surfaceflinger/SurfaceFlinger.h
frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
```

在 SurfaceFlinger 实例中，包含了一系列重要的对象来执行各种复杂的工作，下面将选取其中关键的部分逐一深入分析，最终构建出 SurfaceFlinger 完整的工作机理。

### 工厂类 Factory
SurfaceFlinger 实例中包含了一个工厂类成员变量 **mFactory**，类型为 *surfaceflinger::Factory*，负责统一创建其他核心的对象实例：
```cpp
class SurfaceFlinger : ...
public:
    surfaceflinger::Factory& getFactory() { return mFactory; }
    ...

private:
    surfaceflinger::Factory& mFactory;
    ...
```

**Factory** 中定义了一系列创建各种对象的方法，示例如下：
```cpp
class Factory {
public:
    using SetVSyncEnabled = std::function<void(bool)>;
    virtual std::unique_ptr<DispSync> createDispSync(const char* name, bool hasSyncFramework) = 0;
    virtual std::unique_ptr<EventControlThread> createEventControlThread(SetVSyncEnabled) = 0;
    virtual std::unique_ptr<HWComposer> createHWComposer(const std::string& serviceName) = 0;
    virtual std::unique_ptr<MessageQueue> createMessageQueue() = 0;
    ...

```

在 SurfaceFlinger 的构造函数中，传入了 Factory 的实现类示例 **DefaultFactory**：

```cpp
sp<SurfaceFlinger> createSurfaceFlinger() {
    static DefaultFactory factory;

    return new SurfaceFlinger(factory);
}

// SurfaceFlinger 构造函数
SurfaceFlinger::SurfaceFlinger(Factory& factory, SkipInitializationTag)
      : mFactory(factory), ...) {}

SurfaceFlinger::SurfaceFlinger(Factory& factory) : SurfaceFlinger(factory, SkipInitialization) {
    ...
}
```

```white
frameworks/native/services/surfaceflinger/
    - SurfaceFlingerDefaultFactory.h
    - SurfaceFlingerDefaultFactory.cpp
    - SurfaceFlingerFactory.h
    - SurfaceFlingerFactory.cpp
```

### 消息队列 MessageQueue
消息队列的机制在 Android Framework 中随处可见（详细介绍可见 [Android 线程消息机制（二）—— Native 消息处理](/2021/06/23/thread-message-native)），其设计同样应用在 SurfaceFlinger 中。

对此，定义了一个专职的类 *android:impl:MessageQueue*：
```cpp
class MessageQueue final : public android::MessageQueue {
    // 消息处理类 Handler
    class Handler : public MessageHandler {
        enum { eventMaskInvalidate = 0x1, eventMaskRefresh = 0x2, eventMaskTransaction = 0x4 };
        MessageQueue& mQueue;
        int32_t mEventMask;
        std::atomic<nsecs_t> mExpectedVSyncTime;

    public:
        explicit Handler(MessageQueue& queue) : mQueue(queue), mEventMask(0) {}
        virtual void handleMessage(const Message& message);
        void dispatchRefresh();
        void dispatchInvalidate(nsecs_t expectedVSyncTimestamp);
    };

    friend class Handler;

    sp<SurfaceFlinger> mFlinger;  // 关联至 SurfaceFlinger
    sp<Looper> mLooper; // 消息队列核心：Looper
    sp<EventThreadConnection> mEvents;
    gui::BitTube mEventTube; // 专用的事件管道
    sp<Handler> mHandler;

    static int cb_eventReceiver(int fd, int events, void* data);
    int eventReceiver(int fd, int events);

public:
    ~MessageQueue() override = default;
    void init(const sp<SurfaceFlinger>& flinger) override;
    void setEventConnection(const sp<EventThreadConnection>& connection) override;

    void waitMessage() override;
    void postMessage(sp<MessageHandler>&&) override;

    void invalidate() override;
    void refresh() override;
};
```

以 **Looper** 的运作原理，这里 MessageQueue 提供了两种 **消息发送 -> 消息处理** 的运作机制，分别是：

![](/img/posts/post-sf-mq.svg)
<!-- Title: MessageQueue 消息机制（一）
MessageQueue\n(native)->Looper\n(native): postMessage(handler)
Looper\n(native)->Looper\n(native): sendMessage(handler, msg)
Looper\n(native)->Looper\n(native): ...
Looper\n(native)-\->Handler\n(native):
Handler\n(native)->Handler\n(native): handleMessage(msg) -->

![](/img/posts/post-sf-mq2.svg)
<!-- Title: MessageQueue 消息机制（二）
MessageQueue\n(native)->Looper\n(native): setEventConnection(connection)
Looper\n(native)->Looper\n(native): addFd(mEventTube.getFd())
Looper\n(native)->Looper\n(native): ...
Looper\n(native)-\->MessageQueue\n(native):
MessageQueue\n(native)->MessageQueue\n(native): cb_eventReceiver()
MessageQueue\n(native)->Handler\n(native): eventReceiver()
Handler\n(native)->Handler\n(native): ... -->

```white
frameworks/native/services/surfaceflinger/Scheduler/
    - MessageQueue.h
    - MessageQueue.cpp

frameworks/native/libs/gui/include/private/gui/
    - BitTube.h

system/core/libutils/include/utils/
    - Looper.h
system/core/libutils/
    - Looper.cpp
```

下面来看下 MessageQueue 在 SurfaceFlinger 中运用。

在 SurfaceFlinger 中，有成员变量 **mEventQueue**：
```cpp
class SurfaceFlinger : ...
private:
    std::unique_ptr<MessageQueue> mEventQueue;
    ...
```

同样是 SurfaceFlinger 的构造方法中创建了 MessageQueue 实例：
```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
SurfaceFlinger::SurfaceFlinger(Factory& factory, SkipInitializationTag)
      : mFactory(factory),
        ...
        mEventQueue(mFactory.createMessageQueue()),
        ...) {}

// frameworks/native/services/surfaceflinger/SurfaceFlingerDefaultFactory.cpp
std::unique_ptr<MessageQueue> DefaultFactory::createMessageQueue() {
    return std::make_unique<android::impl::MessageQueue>();
}
```

当 SurfaceFlinger 实例首次被 sp 强指针类型引用时，会回调 `onFirstRef()` 方法，在这里执行了消息队列的初始化：
```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
void SurfaceFlinger::onFirstRef() {
    mEventQueue->init(this);
}

// frameworks/native/services/surfaceflinger/Scheduler/MessageQueue.cpp
void MessageQueue::init(const sp<SurfaceFlinger>& flinger) {
    mFlinger = flinger;
    // 为当前线程创建 Looper
    mLooper = new Looper(true);
    mHandler = new Handler(*this);
}
```

最后，MessageQueue 中的 Looper 还需要让它真正地 “loop” 起来，在前面提到的 SurfaceFlinger 启动过程最后一个方法 `run()` 中，调用了 MessageQueue 的 `waitMessage()` 方法：
```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
void SurfaceFlinger::run() {
    while (true) {
        mEventQueue->waitMessage();
    }
}

// frameworks/native/services/surfaceflinger/Scheduler/MessageQueue.cpp
void MessageQueue::waitMessage() {
    do { // 进入死循环
        IPCThreadState::self()->flushCommands();
        int32_t ret = mLooper->pollOnce(-1);
        ...
    } while (true);
}
```

至此，整个消息机制真正地运行了起来。

<!-- ```
SurfaceFlinger\n(native)->SurfaceFlinger\n(native): onTransact(code = 1006)
SurfaceFlinger\n(native)->MessageQueue\n(native): signalRefresh()
MessageQueue\n(native)->MessageQueue.Handler\n(native): refresh()
MessageQueue.Handler\n(native)->Looper\n(native): dispatchRefresh()
Looper\n(native)->Looper\n(native): sendMessage(MessageQueue::REFRESH)
Looper\n(native)->Looper\n(native): ...
Note left of Looper\n(native): 向 Looper 中发送新消息,\n当消息要处理时，回调至 Handler
MessageQueue.Handler\n(native)->MessageQueue\n(native): handleMessage()
MessageQueue\n(native)-\->SurfaceFlinger\n(native): 
SurfaceFlinger\n(native)->SurfaceFlinger\n(native): onMessageReceived()
SurfaceFlinger\n(native)->SurfaceFlinger\n(native): onMessageRefresh()
``` -->

### CompositionEngine
**CompositionEngine**，顾名思义，合成引擎，其主要是充当 **SurfaceFlinger** 与 **[RenderEngine（渲染引擎）](#renderengine)& [HWComposer（硬件混合渲染器）](#hwcomposer)**之间的桥梁。

```cpp
class CompositionEngine {
public:
    virtual ~CompositionEngine();

    // Create a composition Display
    virtual std::shared_ptr<Display> createDisplay(const DisplayCreationArgs&) = 0;
    virtual std::unique_ptr<compositionengine::LayerFECompositionState>
    createLayerFECompositionState() = 0;

    virtual HWComposer& getHwComposer() const = 0;
    virtual void setHwComposer(std::unique_ptr<HWComposer>) = 0;

    virtual renderengine::RenderEngine& getRenderEngine() const = 0;
    virtual void setRenderEngine(std::unique_ptr<renderengine::RenderEngine>) = 0;
    ...

private:
    std::unique_ptr<HWComposer> mHwComposer; // 硬件混合渲染器
    std::unique_ptr<renderengine::RenderEngine> mRenderEngine; // 渲染引擎
    ...
};
```
```white
frameworks/native/services/surfaceflinger/CompositionEngine/include/compositionengine/
    - CompositionEngine.h

frameworks/native/services/surfaceflinger/CompositionEngine/src/
    - CompositionEngine.cpp
```

在 SurfaceFlinger 中，有成员变量 **mCompositionEngine**
```cpp
class SurfaceFlinger : ...
private:
    std::unique_ptr<compositionengine::CompositionEngine> mCompositionEngine;
    ...
```

同样是 SurfaceFlinger 的构造方法中创建了 CompositionEngine 实例：
```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
SurfaceFlinger::SurfaceFlinger(Factory& factory, SkipInitializationTag)
      : mFactory(factory),
        ...
        mEventQueue(mFactory.createMessageQueue()),
        mCompositionEngine(mFactory.createCompositionEngine()),
        ...) {}

// frameworks/native/services/surfaceflinger/SurfaceFlingerDefaultFactory.cpp
std::unique_ptr<compositionengine::CompositionEngine> DefaultFactory::createCompositionEngine() {
    return compositionengine::impl::createCompositionEngine();
}

std::unique_ptr<compositionengine::CompositionEngine> createCompositionEngine() {
    return std::make_unique<CompositionEngine>();
}
```

在初始化 `init()` 函数中，分别创建了渲染引擎 **RenderEngine** 和硬件混合渲染器 **HWComposer** 
```cpp
void SurfaceFlinger::init() {
    ...
    mCompositionEngine->setRenderEngine(renderengine::RenderEngine::create(
            renderengine::RenderEngineCreationArgs::Builder()
                .setPixelFormat(static_cast<int32_t>(defaultCompositionPixelFormat))
                .setImageCacheSize(maxFrameBufferAcquiredBuffers)
                .setUseColorManagerment(useColorManagement)
                .setEnableProtectedContext(enable_protected_contents(false))
                .setPrecacheToneMapperShaderOnly(false)
                .setSupportsBackgroundBlur(mSupportsBlur)
                .setContextPriority(useContextPriority
                        ? renderengine::RenderEngine::ContextPriority::HIGH
                        : renderengine::RenderEngine::ContextPriority::MEDIUM)
                .build()));
    mCompositionEngine->setTimeStats(mTimeStats);
    mCompositionEngine->setHwComposer(getFactory().createHWComposer("default"));
    ...
}
```
### RenderEngine
承接上面的初始化过程，渲染引擎 **RenderEngine** 的创建是在 `SurfaceFlinger::init()` 方法里完成的，而渲染引擎的真正实现类为 **renderengine::gl::GLESRenderEngine**。

```cpp
std::unique_ptr<impl::RenderEngine> RenderEngine::create(const RenderEngineCreationArgs& args) {
    char prop[PROPERTY_VALUE_MAX];
    property_get(PROPERTY_DEBUG_RENDERENGINE_BACKEND, prop, "gles");
    if (strcmp(prop, "gles") == 0) {
        return renderengine::gl::GLESRenderEngine::create(args);
    }
    return renderengine::gl::GLESRenderEngine::create(args);
}
```

**[EGL](https://www.khronos.org/egl)** 是一套渲染 API（如 [OpenGL ES](https://www.khronos.org/opengles/)）和原生窗口系统之间的**接口**。**GLESRenderEngine** 正是通过调用 EGL 各接口方法来实现创建，下面列举出创建渲染引擎过程中使用到的 EGL 函数：

|函数|含义
|--|--
|eglGetDisplay|返回一个 EGL 显示连接
|eglInitialize|初始化一个 EGL 显示连接
|eglGetConfigs|返回一个显示的所有 EGL 帧缓冲区配置的列表
|eglChooseConfig|返回一个匹配特定参数的 EGL 帧缓冲区配置的列表
|eglGetConfigAttrib|返回一个 EGL 帧缓冲区配置的信息
|eglCreateContext|创建一个新的 EGL 渲染上下文
|eglCreatePbufferSurface|创建一个新的 EGL 像素缓冲 surface
|eglMakeCurrent|附着一个 EGL 渲染上下文到 surfaces 中

更多其他的方法说明可查看 [EGL Reference Pages](https://www.khronos.org/registry/EGL/sdk/docs/man/)。

**GLESRenderEngine** 的 `create()` 方法实现如下：
```cpp
std::unique_ptr<GLESRenderEngine> GLESRenderEngine::create(const RenderEngineCreationArgs& args) {
    // 获取默认的显示连接，并进行初始化
    EGLDisplay display = eglGetDisplay(EGL_DEFAULT_DISPLAY);
    if (!eglInitialize(display, nullptr, nullptr)) {
        LOG_ALWAYS_FATAL("failed to initialize EGL");
    }

    const auto eglVersion = eglQueryStringImplementationANDROID(display, EGL_VERSION);
    const auto eglExtensions = eglQueryStringImplementationANDROID(display, EGL_EXTENSIONS);
    ...

    GLExtensions& extensions = GLExtensions::getInstance();
    extensions.initWithEGLStrings(eglVersion, eglExtensions);

    // 选取合适的 EGL 配置，优先是 OpenGL ES 3.0，若失败再尝试 OpenGL ES 2.0
    EGLConfig config = EGL_NO_CONFIG;
    if (!extensions.hasNoConfigContext()) {
        config = chooseEglConfig(display, args.pixelFormat, /*logConfig*/ true);
    }
    ...
    // 创建 EGL 上下文
    EGLContext ctxt = createEglContext(display, config, protectedContext, useContextPriority,
                                       Protection::UNPROTECTED);
    ...
    // 绑定 EGL 上下文到默认的显示中
    EGLBoolean success = eglMakeCurrent(display, dummy, dummy, ctxt);
    extensions.initWithGLStrings(glGetString(GL_VENDOR), glGetString(GL_RENDERER),
                                 glGetString(GL_VERSION), glGetString(GL_EXTENSIONS));
    ...
    // 创建 GLESRenderEngine 实例
    GlesVersion version = parseGlesVersion(extensions.getVersion());
    std::unique_ptr<GLESRenderEngine> engine;
    switch (version) {
        ...
        // 需要 OpenGL ES 2.0 及以上
        case GLES_VERSION_2_0:
        case GLES_VERSION_3_0:
            engine = std::make_unique<GLESRenderEngine>(args, display, config, ctxt, dummy,
                                                        protectedContext, protectedDummy);
            break;
    }

    return engine;
}
```

### 与 HWComposer 连接
承接上面 **[CompositionEngine](#compositionengine)**

**HWComposer（硬件混合渲染器）**，提供了一个 Callback 类 ComposerCallback：

```cpp
namespace HWC2 {
class ComposerCallback {
 public:
     virtual void onHotplugReceived(int32_t sequenceId, hal::HWDisplayId display,
                                    hal::Connection connection) = 0;
     virtual void onRefreshReceived(int32_t sequenceId, hal::HWDisplayId display) = 0;
     virtual void onVsyncReceived(int32_t sequenceId, hal::HWDisplayId display, int64_t timestamp,
                                  std::optional<hal::VsyncPeriodNanos> vsyncPeriod) = 0;
     virtual void onVsyncPeriodTimingChangedReceived(
             int32_t sequenceId, hal::HWDisplayId display,
             const hal::VsyncPeriodChangeTimeline& updatedTimeline) = 0;
     virtual void onSeamlessPossible(int32_t sequenceId, hal::HWDisplayId display) = 0;

     virtual ~ComposerCallback() = default;
};
}
```

**SurfaceFlinger** 本身集成自
```cpp
class SurfaceFlinger : ...,
                       private HWC2::ComposerCallback,
                       ...
```

```cpp
void SurfaceFlinger::init() {
    ...
    mCompositionEngine->setHwComposer(getFactory().createHWComposer("default"));
    mCompositionEngine->getHwComposer().setConfiguration(this, getBE().mComposerSequenceId);
    ...
```

## SurfaceFlinger 初始化总结

```cpp
void SurfaceFlinger::init() {
    ...
    // Get a RenderEngine for the given display / config (can't fail)
    // TODO(b/77156734): We need to stop casting and use HAL types when possible.
    // Sending maxFrameBufferAcquiredBuffers as the cache size is tightly tuned to single-display.
    mCompositionEngine->setRenderEngine(renderengine::RenderEngine::create(
            renderengine::RenderEngineCreationArgs::Builder()
                .setPixelFormat(static_cast<int32_t>(defaultCompositionPixelFormat))
                .setImageCacheSize(maxFrameBufferAcquiredBuffers)
                .setUseColorManagerment(useColorManagement)
                .setEnableProtectedContext(enable_protected_contents(false))
                .setPrecacheToneMapperShaderOnly(false)
                .setSupportsBackgroundBlur(mSupportsBlur)
                .setContextPriority(useContextPriority
                        ? renderengine::RenderEngine::ContextPriority::HIGH
                        : renderengine::RenderEngine::ContextPriority::MEDIUM)
                .build()));
    mCompositionEngine->setTimeStats(mTimeStats);

    // 创建 HWComposer 对象
    mCompositionEngine->setHwComposer(getFactory().createHWComposer(getBE().mHwcServiceName));
    mCompositionEngine->getHwComposer().setConfiguration(this, getBE().mComposerSequenceId);

    // 首先处理一次显示设备热插拔事件
    processDisplayHotplugEventsLocked();
    ...
    // 初始化绘制状态
    mDrawingState = mCurrentState;

    // 初始化显示
    initializeDisplays();
    ...
    // 启动开机动画
    if (mStartPropertySetThread->Start() != NO_ERROR) {
        ALOGE("Run StartPropertySetThread failed!");
    }
}
```