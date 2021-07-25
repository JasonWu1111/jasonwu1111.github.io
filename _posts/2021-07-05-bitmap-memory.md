---
layout: post
title: Bitmap 的内存管理及原理
tags:
  - Android
  - Bitmap
---
## Bitmap 内存的计算
在一般的 Android 应用运行中，图像资源往往占用着很大一部分的内存。在 Android 系统，图像数据会以 **Bitmap** 的形式呈现。一个 Bitmap 中的图形像素数据是其占用内存的主要部分。

通常如果要加载一张图片文件的像素数据到一个 Bitmap 中，每像素所占的大小由 Bitmap 的色彩格式决定：

|格式|每像素占用
|--|--
|RGB_565|2 bytes
|ARGB_4444|2 bytes
|ARGB_8888|4 bytes
|...|...

因此，在不考虑缩放的情况下，一张 1080*720 的图片在默认的 `ARGB_8888` 格式下就会占用内存：
```
1080 * 720 * 4KB = 3MB
```

在一个 Bitmap 对象创建后，可以通过以下实例方法来查看一个 Bitmap 中的像素数据实际分配了多少个字节：

```java
Bitmap.getAllocationByteCount()
```

## Bitmap 内存模型概述
对于 Bitmap 的内存管理，不同的系统版本之间存在以下差异：
- 从 Android 3.0 到 Android 7.1：**像素数据存放在 Dalvik 的 Java 堆中**；
- Android 8.0 及以上：**像素数据存放在 Native 堆中**。

可见，Android 8.0 是 Bitmap 像素数据内存分配的一个分水岭。像素数据存放在 Java 堆和 Native 堆的差别在于：

一个应用运行时，其虚拟机可使用的 Java 内存会受到设备出厂时就规定好的限制，可以通过以下命令来查看：

```
// 默认
➜  ~ adb shell getprop dalvik.vm.heapgrowthlimit
256m

// 设置 android:largeHeap="true" 之后
➜  ~ adb shell getprop dalvik.vm.heapsize
512m
```

因此，当 Bitmap 的像素数据大量得堆积在 Java 堆时，很容易就达到该虚拟机上限进而引发 OOM 奔溃。而相对的，当 Bitmap 的像素数据存放在 Native 堆时，将不会受到此限制，而是可以使用完整个设备的物理内存。显然，像素数据分配在 Native 堆上时，会对应用本身的内存管理更加友好。

## 7.1 及以下 Bitmap 内存管理
我们先来看下 Android 系统 7.1 及以下的 Bitmap 内存管理，此类情况 Bitmap 的像素数据将存放在 Java 堆。

### 像素数据内存分配
在 Bitmap Java 类中，声明了一个 byte 数组用来存储像素数据：
```java
public final class Bitmap implements Parcelable {
    ...
    private byte[] mBuffer;
}
```

以 Bitmap 的 `createBitmap()` 方法为例，每个 Java 层的 Bitmap 对象创建时都会新建一个 Native 的 Bitmap 对象，并逐步在 Native 层通过 JNI 调用  **VMRuntime** 的 `newNonMovableArray()` 方法来在 Java 堆上创建一个地址不变的字节数组，并调用 Bitmap（Java）的构造方法传入，赋值到 `mBuffer`：

![](/img/posts/post-bitmap-create.png)

<!-- Title: Bitmap createBitmap()
Bitmap\n(java)->Bitmap\n(java): createBitmap()
Bitmap\n(java)->Bitmap\n(java): createBitmap(...)
Bitmap\n(java)-\->Bitmap\n(jni):nativeCreate()
Bitmap\n(jni)->GraphicsJNI\n(jni):Bitmap_creator()
GraphicsJNI\n(jni)->VMRuntime\n(java):allocateJavaPixelRef()
VMRuntime\n(java)->VMRuntime\n(java):newNonMovableArray()
VMRuntime\n(java)-\->GraphicsJNI\n(jni):arrayObj
GraphicsJNI\n(jni)-\->GraphicsJNI\n(jni):nativeBitmap
GraphicsJNI\n(jni)-\->Bitmap\n(java):createBitmap() -->

```light
frameworks/base/graphics/java/android/graphics/
    - Bitmap.java

frameworks/base/core/jni/android/graphics/
    - Bitmap.cpp
    - Graphics.cpp

libcore/libart/src/main/java/dalvik/system/
    - VMRuntime.java
```

最后，会通过 **JNI** 来调用 Bitmap(java) 的构造方法，来创建 Java 层的 Bitmap 实例：
```java
public final class Bitmap implements Parcelable {
    private byte[] mBuffer;
    private final long mNativePtr;
    ...

    // 由 JNI 调用
    Bitmap(long nativeBitmap, byte[] buffer, ...) {
        ...
        mBuffer = buffer;
        ...
        mNativePtr = nativeBitmap;
    }
```

由于像素内存分配在 Java 堆上，其释放的时机自然是 **Bitmap 对象没有被引用被虚拟机 GC 回收时**。

### Native 对象的释放
由于每个 Java 的 Bitmap 对象创建时都会在关联一个 Native 的 Bitmap 对象，因此**在 Java 的 Bitmap 对象被虚拟机 GC 回收时，需要通知到 Native 层来主动释放 Native 的 Bitmap 对象**。

在一个 Java 对象准备被 GC 回收时，会回调其 `finalize()` 方法。鉴于此，Android Framework 引入了一个 **BitmapFinalizer** 类，专用来在 Java Bitmap 对象被 GC 回收时重写其 `finalize()` 来释放 Native 内存：
```java
public final class Bitmap implements Parcelable {

    private final long mNativePtr;
    private final BitmapFinalizer mFinalizer;
    ...

    Bitmap(long nativeBitmap, ...) {
        ...
        mNativePtr = nativeBitmap;
        mFinalizer = new BitmapFinalizer(nativeBitmap);
        ...
    }
    ...

    private static class BitmapFinalizer {
        private long mNativeBitmap;
        ...

        BitmapFinalizer(long nativeBitmap) {
            mNativeBitmap = nativeBitmap;
        }

        @Override
        public void finalize() {
            try {
                super.finalize();
            } catch (Throwable t) {
                // Ignore
            } finally {
                ...
                nativeDestructor(mNativeBitmap);
                mNativeBitmap = 0;
            }
        }
    }
}
```

从 `nativeDestructor()` 方法开始，其逐步往下的调用栈如下：

![](/img/posts/post-bitmap-destructor.png)

<!-- Title: Bitmap nativeDestructor()
Bitmap\n(java)-\->Bitmap\n(jni): nativeDestructor
Bitmap\n(jni)->Bitmap\n(native): Bitmap_destructor
Bitmap\n(native)->Bitmap\n(native): detachFromJava()
Bitmap\n(native)->Bitmap\n(native):~Bitmap()
Bitmap\n(native)->Bitmap\n(native):doFreePixels() -->

**Bitmap 占用的内存已实现了自动管理，一般情况下不再需要开发者额外的处理**。不过如果你想更快的释放内存，可以在确定当前 Bitmap Java 对象无需再使用后，调用其 `recycle()` 方法：
```java
public void recycle() {
    if (!mRecycled && mFinalizer.mNativeBitmap != 0) {
        if (nativeRecycle(mFinalizer.mNativeBitmap)) {
            mBuffer = null; // 置空 mBuffer，像素数据等 GC 触发时才真正被回收
            mNinePatchChunk = null;
        }
        mRecycled = true;
    }
}
```

`nativeRecycle()` 方法往下调用链路如下：

![](/img/posts/post-bitmap-recycle.png)

<!-- Title: Bitmap nativeRecycle()
Bitmap\n(java)->Bitmap\n(java):recycle()
Bitmap\n(java)-\->Bitmap\n(jni):nativeRecycle()
Bitmap\n(jni)->Bitmap\n(native):Bitmap_recycle()
Bitmap\n(native)->Bitmap\n(native):freePixels()
Bitmap\n(native)->Bitmap\n(native):doFreePixels() -->

## 8.0 及以上 Bitmap 内存管理
接着来看 Android 系统 8.0 及以上的 Bitmap 内存管理，此类情况 Bitmap 的像素数据分配在 Native 堆上。

### 像素数据内存分配
由于 **Bitmap** 的像素数据分配在 Native 堆上，同样以 Bitmap 的 `createBitmap()` 方法为例，其会逐步调用到 Native 层，最终通过 `calloc()` 函数来分配 Native 内存，其调用链路如下：

![](/img/posts/post-bitmap-create2.png)

<!-- Title: Bitmap createBitmap()
Bitmap\n(java)->Bitmap\n(java): createBitmap()
Bitmap\n(java)-\->Bitmap\n(jni): nativeCreate()
Bitmap\n(jni)->Bitmap\n(native): Bitmap_creator()
Bitmap\n(native)->Linux: allocateHeapBitmap()
Linux->Linux:calloc()
Linux-\->Bitmap\n(native):addr
Bitmap\n(native)->Bitmap\n(native):Bitmap()
Bitmap\n(native)-\->Bitmap\n(jni):bitmap
Bitmap\n(jni)-\->Bitmap\n(java): createBitmap() -->

```light
frameworks/base/graphics/java/android/graphics/
    - Bitmap.java

frameworks/base/libs/hwui/jni/
    - Bitmap.cpp

frameworks/base/libs/hwui/hwui/
    - Bitmap.h
    - Bitmap.cpp
```

存放像素数据的字节数组在 Native 堆上分配后，会在创建 Native 的 Bitmap 对象时，传入该数组的大小和地址，记录在当前 Bitmap 对象中：
```cpp
Bitmap::Bitmap(void* address, size_t size, const SkImageInfo& info, size_t rowBytes)
        : SkPixelRef(info.width(), info.height(), address, rowBytes)
        , mInfo(validateAlpha(info))
        , mPixelStorageType(PixelStorageType::Heap) {
    mPixelStorage.heap.address = address;
    mPixelStorage.heap.size = size;
}
```

### Native 对象的释放
在 Java 层的 Bitmap 对象需要被 GC 回收时，同样需要通知 Native 层来主动释放内存，特别是分配在 Native 堆的像素数据。从 Android 8.0 开始，对于 Bitmap Java 对象回收的监听不再依赖于 `finalize()` 方法，而是改为使用 **Cleaner** 类。

对比于 finalization，使用 Cleaner 类有以下几点好处：
- Cleaner 使用的是虚引用（PhantomReference），是最弱的引用类型，避免了在对象在回调 `finalize()` 时又重新恢复引用等麻烦情况。
- 当 Cleaner 虚引用的对象变得不可达时，会在 reference-handler 线程来直接运行 Cleaner 而不是在 finalizer 线程.
- Cleaner 封装了清理的代码和逻辑，简单易用且轻量级。

首先，在 Java 的 Bitmap 创建时，通过 **NativeAllocationRegistry** 类来注册其回收方法 `nativeGetNativeFinalizer()`：
```java
Bitmap(long nativeBitmap, ..., boolean fromMalloc) {
    ...
    NativeAllocationRegistry registry;
    if (fromMalloc) {
        registry = NativeAllocationRegistry.createMalloced(
                Bitmap.class.getClassLoader(), nativeGetNativeFinalizer(), allocationByteCount);
    } else {
        registry = NativeAllocationRegistry.createNonmalloced(
                Bitmap.class.getClassLoader(), nativeGetNativeFinalizer(), allocationByteCount);
    }
    registry.registerNativeAllocation(this, nativeBitmap);
    ...
}
```
> libcore/luni/src/main/java/libcore/util/NativeAllocationRegistry.java

其中 `nativeGetNativeFinalizer()` 返回了销毁 Native 层 Bitmap 对象方法的地址：
```cpp
static void Bitmap_destruct(BitmapWrapper* bitmap) {
    delete bitmap;
}

static jlong Bitmap_getNativeFinalizer(JNIEnv*, jobject) {
    return static_cast<jlong>(reinterpret_cast<uintptr_t>(&Bitmap_destruct));
}
```
> frameworks/base/libs/hwui/jni/Bitmap.cpp

接着来看 **NativeAllocationRegistry** 的 `registerNativeAllocation()` 方法实现，其会创建 **Cleaner** 对象并开始其运作：

```java
public Runnable registerNativeAllocation(Object referent, long nativePtr) {
    ...
    CleanerThunk thunk;
    CleanerRunner result;
    try {
        thunk = new CleanerThunk();
        Cleaner cleaner = Cleaner.create(referent, thunk);
        result = new CleanerRunner(cleaner);
        registerNativeAllocation(this.size);
    } catch (VirtualMachineError vme /* probably OutOfMemoryError */) {
        applyFreeFunction(freeFunction, nativePtr);
        throw vme;
    }
    thunk.setNativePtr(nativePtr);
    Reference.reachabilityFence(referent);
    return result;
}
```
> libcore/ojluni/src/main/java/sun/misc/Cleaner.java

在 **Cleaner** 对象封装的虚引用的 Bitmap 对象不可达时，会调用其 `clean()` 方法，来回调前面注册的回收方法，调用链路如下：
![](/img/posts/post-bitmap-destructor2.png)

<!-- Title: Bitmap 内存自动回收
Cleaner->CleanerThunk: clean()
CleanerThunk->NativeAllocationRegistry: run()
NativeAllocationRegistry-\->JNI: applyFreeFunction()
JNI->JNI: NativeAllocationRegistry_\napplyFreeFunction()
JNI->Bitmap\n(native): Bitmap_destruct()
Bitmap\n(native)->Linux:~Bitmap()
Linux->Linux:free() -->

```light
libcore/luni/src/main/native/
    - libcore_util_NativeAllocationRegistry.cpp

frameworks/base/libs/hwui/hwui/
    - Bitmap.cpp
```

最终，在 Native 的 Bitmap 对象的析构方法中，通过 `free()` 方法释放分配的像素数据内存：
```cpp
Bitmap::~Bitmap() {
    switch (mPixelStorageType) {
        ...
        case PixelStorageType::Heap:
            free(mPixelStorage.heap.address);
#ifdef __ANDROID__
            mallopt(M_PURGE, 0);
#endif
            break;
        ...
    }
}
```

在确认 Java 的 Bitmap 对象不需要再使用时，为了更快的释放像素数据内存，同样可以调用 Bitmap 的 `recycle()` 方法来直接释放 Native 堆的像素数据内存，其调用栈如下：
![](/img/posts/post-bitmap-recycle2.png)
<!-- Title: Bitmap recycle()
Bitmap\n(java)->Bitmap\n(java): recycle()
Bitmap\n(java)-\->Bitmap\n(jni): nativeRecycle()
Bitmap\n(jni)->BitmapWrapper\n(native): Bitmap_recycle()
BitmapWrapper\n(native)->Bitmap\n(native):freePixels()
Bitmap\n(native)->Bitmap\n(native):reset()
Bitmap\n(native)->Bitmap\n(native):~Bitmap() -->

## 总结

|系统版本|Android 8.0 以下|Android 8.0 及以上
|--|--|--
|Bitmap 像素数据分配|分配在 Java 堆，要注意避免虚拟机的 OOM|分配在 Native 堆，能使用更多的内存
|Native 对象回收|通过 Java 对象回收的 finalizer 机制触发|通过封装虚引用的 Cleaner 机制触发
|recyle() 方法调用|释放 Java 堆上像素数据字节数组引用，等待 GC 触发时才会真正释放内存|直接释放 Native 堆上的像素数据内存


