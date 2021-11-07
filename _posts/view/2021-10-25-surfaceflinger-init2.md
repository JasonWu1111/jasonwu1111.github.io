---
layout: post
title: Android 图形系统（三）—— SurfaceFlinger 的启动与 DispSync 机制详解
tags:
  - Android
  - Graphics
---

## Scheduler 初始化

![](/img/posts/post-sf-scheduler.svg)

```
Title: Scheduler 初始化
SurfaceFlinger\n(native)->SurfaceFlinger\n(native): init()
SurfaceFlinger\n(native)->SurfaceFlinger\n(native): processDisplayHotplugEventsLocked()
SurfaceFlinger\n(native)->SurfaceFlinger\n(native): initScheduler()
```

```cpp
void SurfaceFlinger::initScheduler(DisplayId primaryDisplayId) {
    ...
    // start the EventThread
    mScheduler =
            getFactory().createScheduler([this](bool enabled) { setPrimaryVsyncEnabled(enabled); },
                                         *mRefreshRateConfigs, *this);
    mAppConnectionHandle =
            mScheduler->createConnection("app", mPhaseConfiguration->getCurrentOffsets().late.app,
                                         impl::EventThread::InterceptVSyncsCallback());
    mSfConnectionHandle =
            mScheduler->createConnection("sf", mPhaseConfiguration->getCurrentOffsets().late.sf,
                                         [this](nsecs_t timestamp) {
                                             mInterceptor->saveVSyncEvent(timestamp);
                                         });

    mEventQueue->setEventConnection(mScheduler->getEventConnection(mSfConnectionHandle));
    mVSyncModulator.emplace(*mScheduler, mAppConnectionHandle, mSfConnectionHandle,
                            mPhaseConfiguration->getCurrentOffsets());

    mRegionSamplingThread =
            new RegionSamplingThread(*this, *mScheduler,
                                     RegionSamplingThread::EnvironmentTimingTunables());
    // Dispatch a config change request for the primary display on scheduler
    // initialization, so that the EventThreads always contain a reference to a
    // prior configuration.
    //
    // This is a bit hacky, but this avoids a back-pointer into the main SF
    // classes from EventThread, and there should be no run-time binder cost
    // anyway since there are no connected apps at this point.
    const nsecs_t vsyncPeriod =
            mRefreshRateConfigs->getRefreshRateFromConfigId(currentConfig).getVsyncPeriod();
    mScheduler->onPrimaryDisplayConfigChanged(mAppConnectionHandle, primaryDisplayId.value,
                                              currentConfig, vsyncPeriod);
}
```

### DispSync


### EventControlThread

```cpp
EventControlThread::EventControlThread(EventControlThread::SetVSyncEnabledFunction function)
      : mSetVSyncEnabled(std::move(function)) {
    pthread_setname_np(mThread.native_handle(), "EventControlThread");

    pid_t tid = pthread_gettid_np(mThread.native_handle());
    setpriority(PRIO_PROCESS, tid, ANDROID_PRIORITY_URGENT_DISPLAY);
    set_sched_policy(tid, SP_FOREGROUND);
}
```

```white
frameworks/native/services/surfaceflinger/Scheduler/
    - EventControlThread.h
    - EventControlThread.cpp
```