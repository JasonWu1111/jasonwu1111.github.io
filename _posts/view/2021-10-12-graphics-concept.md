---
layout: post
title: Android 图形系统（一）—— 概念综述
author: JasonWu
tags:
  - Android
  - Graphics
---

## SurfaceControl 和 Surface
Surface 对象使应用能够渲染要在屏幕上显示的图像。通过 SurfaceHolder 接口，应用可以编辑和控制 Surface。

```java
public final class ViewRootImpl implements ViewParent, ... {
    ...
    public final Surface mSurface = new Surface();
    private final SurfaceControl mSurfaceControl = new SurfaceControl();
    ...
}
```

<!-- ViewRootImpl->ViewRootImpl: performTraversals()
Note right of ViewRootImpl: 首次测量或者窗口大小有变化时
ViewRootImpl->IWindowSession$BinderProxy: relayoutWindow()
IWindowSession$BinderProxy-\->Session: relayout(mSurfaceControl)
Note left of Session: <- 应用进程 | 系统服务进程 ->
Session->WindowManagerService:relayout(outSufaceControl)
WindowManagerService->WindowManagerService: relayoutWindow(outSufaceControl)
WindowManagerService->WindowStateAnimator: createSurfaceControl(outSufaceControl)
WindowStateAnimator->WindowSurfaceController: createSurfaceLocked()
WindowSurfaceController->WindowSurfaceController: <init>()
WindowSurfaceController->SurfaceControl: new
SurfaceControl->SurfaceControl: <init>(...)
SurfaceControl->JNI: nativeCreate() 
JNI->SurfaceComposerClient\n(native): nativeCreate()
SurfaceComposerClient\n(native)->SurfaceComposerClient\n(native): createSurfaceChecked()
SurfaceComposerClient\n(native)->SurfaceFlinger\n(native): createSurface()
SurfaceFlinger\n(native)->SurfaceFlinger\n(native): createLayer()
SurfaceFlinger\n(native)->SurfaceFlinger\n(native): addClientLayer() -->

## Canvas & Skia
<!-- - lockCanvas() 
  locks the buffer for rendering on the CPU and returns a Canvas to use for drawing.
- unlockCanvasAndPost() 
  unlocks the buffer and sends it to the compositor.
- lockHardwareCanvas() 
  locks the buffer for rendering on the GPU and returns a canvas to use for drawing.


frameworks/native/libs/gui/include/gui/BufferQueueProducer.h -->

### lockCanvas
<!-- Surface->Surface: lockCanvas()
Surface->Surface\n(native): nativeLockCanvas(canvas)
Surface\n(native)->Surface\n(native): lock()
Surface\n(native)->BufferQueueProducer\n(native): dequeueBuffer(ANativeWindowBuffer* backBuffer)
BufferQueueProducer\n(native)->BufferQueueCore\n(native): dequeueBuffer(int* outSlot)
BufferQueueCore\n(native)-\->BufferQueueProducer\n(native): mFreeBuffers.front()
Surface\n(native)->Surface\n(native): backBuffer->lockAsync()
Surface\n(native)->Surface\n(native): mLockedBuffer = backBuffer -->

### unlockCanvasAndPost
<!-- Surface->Surface: unlockCanvasAndPost(canvas)
Surface->Surface\n(native): nativeUnlockCanvasAndPost(canvas)
Surface\n(native)->Surface\n(native): unlockAndPost()
Surface\n(native)->Surface\n(native): mLockedBuffer->unlockAsync()
Surface\n(native)->BufferQueueProducer\n(native): queueBuffer(mLockedBuffer)
BufferQueueProducer\n(native)->BufferQueueCore\n(native): queueBuffer()
BufferQueueCore\n(native)->BufferQueueCore\n(native): mQueue.push_back(BufferItem) -->


## SurfaceFlinger
<!-- Title: I'm title
main_surfaceflinger.cpp->SurfaceFlinger\n(native): main()
SurfaceFlinger\n(native)->SurfaceFlinger\n(native): createSurfaceFlinger()
SurfaceFlinger\n(native)->SurfaceFlinger\n(native): SurfaceFlinger()
SurfaceFlinger\n(native)->SurfaceFlinger\n(native): init()
SurfaceFlinger\n(native)->SurfaceFlinger\n(native): run() -->

## Hardware Composer HAL



## Layer

## Display

## OpenGL ES 和 Vulkan






