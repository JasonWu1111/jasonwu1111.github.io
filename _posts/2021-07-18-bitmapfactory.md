---
layout: post
title: BitmapFactory 的使用和原理分析
tags:
  - Android
  - Bitmap
---
# BitmapFactory
**BitmapFactory** 为系统 Framework 提供的用于从不同的来源中解析并新建出 Bitmap 对象（如文件、流、字节数组等）的工厂类。

BitmapFactory 中封装了一系列的静态方法用来从不同类似的资源中解析出 Bitmap 对象，示例如下：
```java
public static Bitmap decodeFile(String pathName, Options opts);
public static Bitmap decodeResource(Resources res, int id, Options opts);
public static Bitmap decodeByteArray(byte[] data, int offset, int length, Options opts);
public static Bitmap decodeStream(InputStream is, Rect outPadding, Options opts);
...
```
> frameworks/base/graphics/java/android/graphics/BitmapFactory.java

其中，**Options** 为 BitmapFactory 的静态内部类，表示可以自定义设置 Bitmap 解析生成相关的选项，各属性以及对应的效果详细说明如下：

### Options 属性

|Options 属性|	说明
|--|--
|Bitmap inBitmap|如果设置，将尝试加载内容时重用此 Bitmap。如果解码操作不能使用这个 Bitmap，decode 方法将会抛出异常。
|boolean inMutable	|设置返回的 Bitmap 是否为可变的。可变的 Bitmap 意味着可以在生成后动态地应用上其他的效果。<br><br>不可以和 `inPreferredConfig  = Bitmap.Config.HARDWARE` 一起设置，因为硬件位图总是不可变的。
|boolean inJustDecodeBounds|  若设置为 true，将不获取位图，不分配内存，但会返回图片的高宽等信息。
|int inSampleSize	|设置图片相对原图缩放的倍数。取值为 2 的次方。如 inSampleSize == 4 表示返回的图片宽和高像素只有原图的 1/4。<br><br>任何小于 1 的值都认为是 1，任何其他值将向下舍入到最接近的 2 次方。
|Bitmap.Config inPreferredConfig	|如果为非空的，解码器将尝试解码到这个配置。如果为空，或者无法满足请求，解码器将尝试根据系统的屏幕深度和原始图像的特征来选择最匹配的配置。默认值为 `Bitmap.Config#ARGB_8888`。
|ColorSpace inPreferredColorSpace|如果为非空的，解码器将尝试解码到这个颜色空间。如果为空，或者无法满足请求，解码器将选择嵌入在图像中的色彩空间或最适合请求的图像配置的色彩空间。
|inPremultiplied|如果为 true（默认值），则生成的位图会将其颜色通道预先乘以 alpha 通道。对于要由 View 系统或 Canvas 直接绘制的图像，不应将此设置为 false。<br><br>将此标志设置为 false 时将 inScaled 设置为 true 可能会导致颜色不正确。
|int inDensity	|用于位图的像素密度。如果设置了 inScaled（默认情况下）并且此密度与 inTargetDensity 不匹配，则位图将在返回之前缩放到目标密度。
|int inTargetDensity	|此位图将绘制到的目标的像素密度。与 inDensity 和 inScaled 结合使用，以确定在返回位图之前是否以及如何缩放位图。
|int inScreenDensity|正在使用的实际屏幕的像素密度。通过设置此项，可以避免将当前处于屏幕密度的位图向上/向下缩放到兼容密度。相反，如果 inDensity 与 inScreenDensity 相同，则位图将保持原样。
|boolean inScaled	|若设置为 tru（默认值），如果 inDensity 和 inTargetDensity 不为 0，则位图将在加载时缩放以匹配 inTargetDensity，而不是在每次将其绘制到 Canvas 时依赖图形系统对其进行缩放。<br><br>如果你需要位图的非缩放版本，则应将其关闭。.9 图会忽略这个标志并且总是被缩放。
|byte[] inTempStorage	|用于解码的临时存储。
|...|...

其中 `inPreferredConfig` 可设置的 **Bitmap.Config** 各像素配置说明如下：
### Bitmap.Config

|Bitmap.Config|说明|每像素占内存|备注
|--|--|--|--
|`ALPHA_8`|每个像素存储在单个半透明 (alpha) 通道。可用于有效存储透明度信息，不存储颜色信息。|1 KB
|`RGB_565`|仅对 RGB 通道进行编码，以红色 5 位精度，绿色 6 位精度，蓝色 5 位精度存储。当使用不需要高色彩保真度的不透明位图时，此配置可能很有用。|2KB
|`ARGB_4444`|三个 RGB 颜色通道和半透明通道存放在 4 位精度。可用于存储半透明信息但也需要节省内存。|2 KB|*从 API 29 开始已被废弃*
|`ARGB_8888`|三个 RGB 颜色通道和半透明通道存放在 8 位精度。此配置可提供最佳质量，应尽可能使用。|4 KB
|`RGBA_F16`|每个通道都存储为半精度浮点值。这种配置特别适用于广色域和 HDR 内容。|8 KB|*从 API 26 开始支持*
|`HARDWARE`|一种特殊的配置，像素数据仅存储在图形内存中。此配置中的位图始终不可变，对于位图的唯一操作应该是将其绘制在屏幕上。|-|*从 API 26 开始支持*

> frameworks/base/graphics/java/android/graphics/Bitmap.java

<!-- Title: BitmapFactory decode
BitmapFactory->BitmapFactory: decodeFile()
BitmapFactory->BitmapFactory: decodeStream()
BitmapFactory->BitmapFactory: decodeStream()
BitmapFactory->BitmapFactory: decodeStreamInternal()
BitmapFactory-\->BitmapFactory.cpp: nativeDecodeStream()
BitmapFactory.cpp->BitmapFactory.cpp:nativeDecodeStream()
BitmapFactory.cpp->BitmapFactory.cpp:doDecode()
BitmapFactory.cpp->BitmapFactory.cpp:... -->

<!-- Title: BitmapFactory decode2
participant BitmapFactory\n(native)
participant SkBitmap\n(native)
participant HeapAllocator\n(native)
participant Bitmap\n(native)
participant Bitmap\n(jni)
participant Bitmap\n(java)
BitmapFactory\n(native)->Bitmap\n(jni):doDecode()
SkBitmap\n(native)->HeapAllocator\n(native):tryAllocPixels()
HeapAllocator\n(native)->Bitmap\n(native):allocPixelRef()
Bitmap\n(native)->Bitmap\n(native):allocateHeapBitmap
Bitmap\n(native)-\->Bitmap\n(jni):bitmap
Bitmap\n(jni)->Bitmap\n(java): createBitmap() -->

# decode 过程分析
根据调用不同的 `decode...()` 方法，会通过 **JNI** ，最终调用到 Native 层的 `doDecode()` 方法来完成解析工作。以 `decodeFile()` 方法为例，其调用栈如下：

![](/img/posts/post-bitmapfactory-decode.png)
<!-- Title: BitmapFactory decode
BitmapFactory->BitmapFactory: decodeFile()
BitmapFactory->BitmapFactory: decodeStream()
BitmapFactory->BitmapFactory: decodeStreamInternal()
BitmapFactory-\->BitmapFactory\n(native): nativeDecodeStream()
BitmapFactory\n(native)->BitmapFactory\n(native):nativeDecodeStream()
BitmapFactory\n(native)->BitmapFactory\n(native):doDecode()
BitmapFactory\n(native)->BitmapFactory\n(native):... -->

```white
frameworks/base/graphics/java/android/graphics/
    - BitmapFactory.java

frameworks/base/libs/hwui/jni/
    - BitmapFactory.cpp

external/skia/include/core/
    - SkStream.h
```

根据图片资源的不同，BitmapFactory 提供了以下几个 Native 方法：
```java
private static native Bitmap nativeDecodeStream(InputStream is, ...);
private static native Bitmap nativeDecodeFileDescriptor(FileDescriptor fd, ...);
private static native Bitmap nativeDecodeAsset(long nativeAsset, ...);
private static native Bitmap nativeDecodeByteArray(byte[] data, ...);
```

不同的资源类型，在 Native 层，最终会转成 [**Skia**](https://skia.org/docs/) 图形库的 **SkStreamRewindable** 对象，传入到 `doDecode()` 方法中，各类型资源转换后的数据流分别为：

|资源类型|转换后
|--|--
|InputStream|SkFrontBufferedStream
|FileDescriptor|SkFILEStream
|Asset|AssetStreamAdaptor
|byte[]|SkMemoryStream


`doDecode()` 方法完成了一系列图片解析的过程，代码较长，整理其核心步骤如下：
1. 从传入的 Options 对象获取各属性设置；
2. 创建图形解码器（codec）；
3. 通过解码器获取图片的宽高等信息，若 `inJustDecodeBounds` 设置为 true，结束返回空；
4. 创建并选择合适的内存分配器；
5. 使用内存分配器为像素数据分配内存；
6. 使用解码器来进行解码获取像素数据；
7. 处理位图 scale 值缩放；
8. 创建 Bitmap 的 Java 对象返回。

整个解析过程，核心是**解码器的解析以及内存分配器对像素数据存储所需内存的分配**。下面将重点分析这两部分：

### 解码器 Codec
解码器的基类是 [**Skia**](https://skia.org/docs/) 图形库的 **SkCodec**，它创建的方式如下：

```cpp
NinePatchPeeker peeker;
SkCodec::Result result;
std::unique_ptr<SkCodec> c = SkCodec::MakeFromStream(std::move(stream), &result,
                                                        &peeker);
```

在 `SkCodec::MakeFromStream()` 方法中，**会先对传入的 stream 读取其头部的 32 个字符**，并根据这部分头部信息来判断图片的格式类型，从而创建具体特定格式解码的解码器并返回。不同格式的图片对应的解码器为：

|图片格式|判断依据|解码器
|--|--|--
|`PNG`|前 8 个字符为 {137, 80, 78, 71, 13, 10, 26, 10}|SkPngCodec
|`JPEG`|前 3 个字符为 {0xFF, 0xD8, 0xFF}|SkJpegCodec
|`WEBP`|前 4 个字符为 "RIFF"，从第 9 位开始的 6 个字符为 "WEBPVP"|SkWebpCodec
|`GIF`|前 6 个字符为 "GIF87a" 或者 "GIF89a"|SkGifCodec
|`ICO`|前 4 个字符为 {'\x00', '\x00', '\x01', '\x00'} 或 {'\x00', '\x00', '\x02', '\x00'}|SkIcoCodec
|`BMP`|前 2 个字符为 {'B', 'M'}|SkBmpCodec
|...|...|...

而在具体解码器中，实际的解码工作会交由第三方库，如 **SkPngCodec** 调用的是 [**libpng**](https://android.googlesource.com/platform/external/libpng/) 库，**SkJpegCodec** 调用的是 [**libjpeg-turbo**](https://android.googlesource.com/platform/external/libjpeg-turbo/) 库等。

图像解码的操作分为以下两步：
- 读取头部数据，解析出宽高尺寸等信息（对应上面 doDecode 过程的第 3 步）；
![](/img/posts/post-skcodec-makefromstream.png)

<!-- Title: SkCodec::MakeFromStream()
BitmapFactory\n(native)->SkCodec: doDecode()
SkCodec->SkCodec: MakeFromStream()
SkCodec->SkCodec: stream->peek(buffer, 32)
SkCodec-\->SkPngCodec: buffer
SkPngCodec->libpng: IsPng()
libpng->libpng: png_sig_cmp()
SkCodec-\->SkPngCodec: stream
SkPngCodec->SkPngCodec: MakeFromStream()
SkPngCodec->libpng: read_header()
libpng->libpng: ... -->

- 解析出图形完整的像素数据（对应上面 doDecode 过程的第 6 步）。
![](/img/posts/post-skcodec-getpixel.png)

<!-- Title: SkSampledCodec::getAndroidPixels()
BitmapFactory\n(native)->SkSampledCodec: doDecode()
SkSampledCodec->SkSampledCodec: getAndroidPixels()
SkSampledCodec->SkCodec: onGetAndroidPixels()
SkCodec->SkPngCodec: getPixels()
SkPngCodec->libpng: onGetPixels()
libpng->libpng: ... -->

```light
frameworks/base/libs/hwui/jni/
    - itmapFactory.cpp

external/skia/src/codec/
    - SkCodec.cpp
    - SkAndroidCodec.cpp
    - SkSampledCodec.cpp

external/libpng/
    - png.c
    - pngpread.c
```

### 内存分配器 Allocator
在解码器解析出完整像素数据前，需要先分配足够空间用来存储数据的内存（对应上面 doDecode 过程的第 5 步）。内存分配器的基类是 **SkBitmap::Allocator**。

实际选择使用的内存分配器如下：
![](/img/posts/post-allocator.png)

内存分配器进一步通过 `allocPixelRef()` 方法来触发内存的分配，不同的分配器的内存分配方式如下：

|分配器|内存分配方式
|--|--
|HeapAllocator（默认）|调用 `calloc()` 函数分配
|RecyclingPixelAllocator|复用传入的 Bitmap 像素数据
|SkBitmap::HeapAllocator|在 skia 库通过 `calloc()` 或者 `malloc()` 函数分配
|ScaleCheckingAllocator|同 SkBitmap::HeapAllocator

以 **HeapAllocator** 为例，它的内存分配的调用栈是：

![](/img/posts/post-allocator-allocate.png)

<!-- Title: HeapAllocator::tryAllocPixels()
BitmapFactory\n(native)->SkBitmap\n(native): doDecode()
SkBitmap\n(native)->HeapAllocator\n(native): tryAllocPixels(allocator)
HeapAllocator\n(native)->Bitmap\n(native): allocPixelRef()
Bitmap\n(native)->Linux: allocateHeapBitmap()
Linux->Linux: calloc() -->

```white
frameworks/base/libs/hwui/jni/
    - GraphicsJNI.h
    - Graphics.cpp

frameworks/base/libs/hwui/hwui/
    - Bitmap.cpp

external/skia/include/core/
    - SkBitmap.h
    - SkBitmap.cpp
```



