---
layout: post
title: Bitmap 的使用
tags:
  - Android
  - Bitmap
---
```light
frameworks/base/graphics/java/android/graphics/
    - Bitmap.java
    - BitmapFactory.java

frameworks/base/libs/hwui/jni/
    - Bitmap.cpp
    - BitmapFactory.cpp

frameworks/base/libs/hwui/hwui/
    - Bitmap.h
    - Bitmap.cpp
```

- Android 3.0 到 Android 7.1：像素数据保存在 java heap
- Android 8.0 以及之后： 像素数据保存在 native heap

## BitmapFactory
```java
public static Bitmap decodeFile(String pathName, Options opts);
public static Bitmap decodeResource(Resources res, int id, Options opts);
public static Bitmap decodeByteArray(byte[] data, int offset, int length, Options opts);
public static Bitmap decodeStream(InputStream is, Rect outPadding, Options opts);
...
```

<!-- Title: BitmapFactory decode
BitmaFactory->BitmaFactory: decodeFile()
BitmaFactory->BitmaFactory: decodeStream()
BitmaFactory->BitmaFactory: decodeStream()
BitmaFactory->BitmaFactory: decodeStreamInternal()
BitmaFactory-\->BitmaFactory.cpp: nativeDecodeStream()
BitmaFactory.cpp->BitmaFactory.cpp:nativeDecodeStream()
BitmaFactory.cpp->BitmaFactory.cpp:doDecode()
BitmaFactory.cpp->BitmaFactory.cpp:... -->

<!-- Title: BitmapFactory decode2
participant BitmaFactory\n(native)
participant SkBitmap\n(native)
participant HeapAllocator\n(native)
participant Bitmap\n(native)
participant Bitmap\n(jni)
participant Bitmap\n(java)
BitmaFactory\n(native)->Bitmap\n(jni):doDecode()
SkBitmap\n(native)->HeapAllocator\n(native):tryAllocPixels()
HeapAllocator\n(native)->Bitmap\n(native):allocPixelRef()
Bitmap\n(native)->Bitmap\n(native):allocateHeapBitmap
Bitmap\n(native)-\->Bitmap\n(jni):bitmap
Bitmap\n(jni)->Bitmap\n(java): createBitmap() -->

<!-- Title: createBitmap
Bitmap\n(java)->Bitmap\n(java): createBitmap()
Bitmap\n(java)-\->Bitmap\n(jni): nativeCreate()
Bitmap\n(jni)->Bitmap\n(native): Bitmap_creator()
Bitmap\n(native)->Linux: allocateHeapBitmap()
Linux->Linux:calloc()
Linux-\->Bitmap\n(native):addr
Bitmap\n(native)-\->Bitmap\n(jni):bitmap
Bitmap\n(jni)-\->Bitmap\n(java): createBitmap() -->


