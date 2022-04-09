---
layout: post
title: Android 图形系统（四）—— VSYNC 在应用端的分发流程
author: JasonWu
tags:
  - Android
  - Graphics
---

Title: I'm title
Choreographer->FrameDisplayEventReceiver: Choreographer()
FrameDisplayEventReceiver->FrameDisplayEventReceiver:DisplayEventReceiver()
FrameDisplayEventReceiver->FrameDisplayEventReceiver:DisplayEventReceiver(VSYNC_SOURCE_APP)
FrameDisplayEventReceiver->NativeDisplayEventReceiver\n(native):nativeInit()
NativeDisplayEventReceiver\n(native)->DisplayEventReceiver\n(native):NativeDisplayEventReceiver()
DisplayEventReceiver\n(native)->DisplayEventReceiver\n(native):DisplayEventReceiver()
DisplayEventReceiver\n(native)-->SurfaceFlinger\n(native):IServiceManager::getService("SurfaceFlinger").\ncreateDisplayEventConnection()
SurfaceFlinger\n(native)->Scheduler\n(native):createDisplayEventConnection()
Scheduler\n(native)->EventThread(app)\n(native):createConnectionInternal()
EventThread(app)\n(native)->EventThreadConnection\n(native):createEventConnection()
EventThreadConnection\n(native)->EventThread(app)\n(native):onFirstRef()
EventThread(app)\n(native)->EventThread(app)\n(native):registerDisplayEventConnection()
NativeDisplayEventReceiver\n(native)->NativeDisplayEventReceiver\n(native):initialize()


FrameDisplayEventReceiver->Choreographer: onVsync()
Choreographer->Choreographer: doFrame()
Choreographer->Choreographer: doCallbacks(CALLBACK_TRAVERSAL)
Choreographer->Choreographer: doCallbacks(CALLBACK_TRAVERSAL)