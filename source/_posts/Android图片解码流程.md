---
title: Android图片解码流程
date: 2022-04-06 10:37:16
categories: image
tags: [image]
---

# Android图片解码流程
Android支持的图片有很多种，压缩图，xml图等等。

对于常用的图片来说，主要是这些网络或资源里存在的图片格式：

- jpeg
- png
- webp
- bmp
- gif
- heif
- ico
- dng

这里主要针对上述图片格式做流程分析，这些压缩图都是有自己的规范的，在解码流程里，这些压缩图会通过规范解压缩得到位图数据，
<!-- more -->

一般针对音视频来说，会使用编码、解码这样的描述，但是对于图片来说，压缩、解压缩也更合适一些，可以从函数的定义（Bitmap.compress），文档的描述中看出来。不过对于文章描述来说，遵从大家的口语一致性，解码对应解压缩，编码对应压缩，从语境上理解即可，不必过于纠正词语的规范性。

先贴上一张大图，通览一下整个结构。

![skia](../images/skia.png)

## 位图Bitmap

Bitmap即位图的翻译，也是Android中位图类的定义。位图一般是指点阵像素图，位图的数据常用RGB颜色域来表达，这个和视频流常用YUV不同。YUV是视频为了降低码率，适应黑白电视的历史而诞生的，是一种传输过程中的数据流。而RGB是需要在显示端，由显示设备对点阵像素进行渲染绘制的。与RGB对应的还有一种三原色，CMYK，这种是减色系，主要用于印刷等行业，其原理是不发光，通过光的反射进行颜色的显示，和RGB正好是反色的。

Bitmap的定义：

https://developer.android.google.cn/reference/android/graphics/Bitmap?hl=en

Bitmap本身可以被创建，被序列化传递，同时也可以送给Canvas，控件等被绘制，是连接图片和图形之间的桥梁。

## Bitmap解码接口

android上用于获取Bitmap的接口有两类，

- BitmapFactory


- ImageDecoder

两类接口都可以方便的从各种输入流中返回Bitmap。既然都可以，为什么还要搞两套接口呢？
BitmapFactory是API1就存在的接口，简单易用，所有的参数都放在Option中，同时包含有输入输出参数，数量多且不易区分。

ImageDecoder是新引入的接口，这套接口的设计模式偏重于Builder方式，每个不同的参数都需要显式的指定参数.

## skia介绍

skia是一个开源的2D图形渲染框架，因为简单易用，移植性好，所以广泛被使用，被google运用到了多个项目工程里，包括Android，Chromium，Flutter等。

skia除了2D图形之外，还包括了一个小的图片解码框架，这部分就被Android 用来做位图解码之用。但是，google是不甘于做拿来主义的，一方面是要做一些优化，另一方面也要针对Android做一些功能扩展。

skia提供的解码接口使用起来很简单，这样使用，通过文件创建SkStream，然后通过SkStream创建SkCodec，最后分配buffer，用于存储解码的数据。

```c++
std::unique_ptr<SkStream> stream(GetResourceAsStream("images/plane.png"));
if (!stream) {
    return;
}

std::unique_ptr<SkCodec> codec(SkCodec::MakeFromStream(std::move(stream)));
if (!codec) {
    ERRORF(r, "failed to create a codec for %s", path);
    return;
}

SkBitmap bm;
bm.allocPixels(codec->getInfo());
SkCodec::Result result = codec->getPixels(codec->getInfo(), bm.getPixels(), bm.rowBytes());
```

skia的源码可以在这里查看：

https://android-opengrok.bangnimang.net/android-12.0.0_r3/xref/external/skia/src/codec/

## 解码流程

BitmapFactory从JNI层开始，引用skia的接口进行处理。原则上，对于Android来说，其实只要合理调用这些接口就行了。

但是实际上，Android不是简单的直接调用的，而在此之外封装了接口，SkAndroidCodec。

可以看到BitmapFactory的JNI代码，逻辑很长。大概分几个步骤：

- 预处理应用传入的参数
- 根据stream创建codec
- 根据输入参数决定codec输出的各种配置，包括 size，colorspace，scale等，size处理比较复杂，涉及到解码后二次缩放等处理
- 是否可以复用传入的bitmap
- 初始化内存分配器
- 初始化解码的参数
- 进行实际解码
- ninepatch的处理，可选
- 解码后的尺寸缩放处理
- 返回java bitmap

```c++
static jobject doDecode(JNIEnv* env, std::unique_ptr<SkStreamRewindable> stream,
                        jobject padding, jobject options, jlong inBitmapHandle,
                        jlong colorSpaceHandle) {
    // Set default values for the options parameters.
    int sampleSize = 1;
    bool onlyDecodeSize = false;
    SkColorType prefColorType = kN32_SkColorType;
    bool isHardware = false;
    bool isMutable = false;
    float scale = 1.0f;
    bool requireUnpremultiplied = false;
    jobject javaBitmap = NULL;
    sk_sp<SkColorSpace> prefColorSpace = GraphicsJNI::getNativeColorSpace(colorSpaceHandle);

    // Update with options supplied by the client.
    if (options != NULL) {
         // 。。。
    }

    // Create the codec.
    NinePatchPeeker peeker; // png 用到
    std::unique_ptr<SkAndroidCodec> codec;
    {
        SkCodec::Result result;
        std::unique_ptr<SkCodec> c = SkCodec::MakeFromStream(std::move(stream), &result,
                                                             &peeker);
        if (!c) {
            SkString msg;
            msg.printf("Failed to create image decoder with message '%s'",
                       SkCodec::ResultToString(result));
            return nullObjectReturn(msg.c_str());
        }

        codec = SkAndroidCodec::MakeFromCodec(std::move(c));
        if (!codec) {
            return nullObjectReturn("SkAndroidCodec::MakeFromCodec returned null");
        }
    }

    // Determine the output size.
    SkISize size = codec->getSampledDimensions(sampleSize);

    int scaledWidth = size.width();
    int scaledHeight = size.height();
    bool willScale = false;

    // Apply a fine scaling step if necessary.
    if (needsFineScale(codec->getInfo().dimensions(), size, sampleSize)) {
        willScale = true;
        scaledWidth = codec->getInfo().width() / sampleSize;
        scaledHeight = codec->getInfo().height() / sampleSize;
    }

    // Set the decode colorType
    SkColorType decodeColorType = codec->computeOutputColorType(prefColorType);
    if (decodeColorType == kRGBA_F16_SkColorType && isHardware &&
            !uirenderer::HardwareBitmapUploader::hasFP16Support()) {
        decodeColorType = kN32_SkColorType;
    }

    sk_sp<SkColorSpace> decodeColorSpace = codec->computeOutputColorSpace(
            decodeColorType, prefColorSpace);

    // Set the options and return if the client only wants the size.
    if (options != NULL) {
          // 。。。
    }

    // Scale is necessary due to density differences.
    if (scale != 1.0f) {
        willScale = true;
        scaledWidth = static_cast<int>(scaledWidth * scale + 0.5f);
        scaledHeight = static_cast<int>(scaledHeight * scale + 0.5f);
    }

    // reuse bitmap
    android::Bitmap* reuseBitmap = nullptr;
    unsigned int existingBufferSize = 0;
    if (javaBitmap != nullptr) {
        reuseBitmap = &bitmap::toBitmap(inBitmapHandle);
        if (reuseBitmap->isImmutable()) {
            ALOGW("Unable to reuse an immutable bitmap as an image decoder target.");
            javaBitmap = nullptr;
            reuseBitmap = nullptr;
        } else {
            existingBufferSize = reuseBitmap->getAllocationByteCount();
        }
    }

    // 内存分配器
    HeapAllocator defaultAllocator;
    RecyclingPixelAllocator recyclingAllocator(reuseBitmap, existingBufferSize);
    ScaleCheckingAllocator scaleCheckingAllocator(scale, existingBufferSize);
    SkBitmap::HeapAllocator heapAllocator;
    SkBitmap::Allocator* decodeAllocator;
    if (javaBitmap != nullptr && willScale) {
        // This will allocate pixels using a HeapAllocator, since there will be an extra
        // scaling step that copies these pixels into Java memory.  This allocator
        // also checks that the recycled javaBitmap is large enough.
        decodeAllocator = &scaleCheckingAllocator;
    } else if (javaBitmap != nullptr) {
        decodeAllocator = &recyclingAllocator;
    } else if (willScale || isHardware) {
        // This will allocate pixels using a HeapAllocator,
        // for scale case: there will be an extra scaling step.
        // for hardware case: there will be extra swizzling & upload to gralloc step.
        decodeAllocator = &heapAllocator;
    } else {
        decodeAllocator = &defaultAllocator;
    }

    // 计算解码器配置的参数
    SkAlphaType alphaType = codec->computeOutputAlphaType(requireUnpremultiplied);

    const SkImageInfo decodeInfo = SkImageInfo::Make(size.width(), size.height(),
            decodeColorType, alphaType, decodeColorSpace);

    SkImageInfo bitmapInfo = decodeInfo;
    if (decodeColorType == kGray_8_SkColorType) {
        // The legacy implementation of BitmapFactory used kAlpha8 for
        // grayscale images (before kGray8 existed).  While the codec
        // recognizes kGray8, we need to decode into a kAlpha8 bitmap
        // in order to avoid a behavior change.
        bitmapInfo =
                bitmapInfo.makeColorType(kAlpha_8_SkColorType).makeAlphaType(kPremul_SkAlphaType);
    }
    SkBitmap decodingBitmap;
    if (!decodingBitmap.setInfo(bitmapInfo) ||
            !decodingBitmap.tryAllocPixels(decodeAllocator)) {
        // SkAndroidCodec should recommend a valid SkImageInfo, so setInfo()
        // should only only fail if the calculated value for rowBytes is too
        // large.
        // tryAllocPixels() can fail due to OOM on the Java heap, OOM on the
        // native heap, or the recycled javaBitmap being too small to reuse.
        return nullptr;
    }

    // Use SkAndroidCodec to perform the decode.
    SkAndroidCodec::AndroidOptions codecOptions;
    codecOptions.fZeroInitialized = decodeAllocator == &defaultAllocator ?
            SkCodec::kYes_ZeroInitialized : SkCodec::kNo_ZeroInitialized;
    codecOptions.fSampleSize = sampleSize;
    SkCodec::Result result = codec->getAndroidPixels(decodeInfo, decodingBitmap.getPixels(),
            decodingBitmap.rowBytes(), &codecOptions);
    switch (result) {
        case SkCodec::kSuccess:
        case SkCodec::kIncompleteInput:
            break;
        default:
            return nullObjectReturn("codec->getAndroidPixels() failed.");
    }

    // This is weird so let me explain: we could use the scale parameter
    // directly, but for historical reasons this is how the corresponding
    // Dalvik code has always behaved. We simply recreate the behavior here.
    // The result is slightly different from simply using scale because of
    // the 0.5f rounding bias applied when computing the target image size
    const float scaleX = scaledWidth / float(decodingBitmap.width());
    const float scaleY = scaledHeight / float(decodingBitmap.height());

    // ninepatch 处理
    // ...

    // 后处理，缩放
    SkBitmap outputBitmap;
    if (willScale) {
        // Set the allocator for the outputBitmap.
        SkBitmap::Allocator* outputAllocator;
        if (javaBitmap != nullptr) {
            outputAllocator = &recyclingAllocator;
        } else {
            outputAllocator = &defaultAllocator;
        }

        SkColorType scaledColorType = decodingBitmap.colorType();
        // FIXME: If the alphaType is kUnpremul and the image has alpha, the
        // colors may not be correct, since Skia does not yet support drawing
        // to/from unpremultiplied bitmaps.
        outputBitmap.setInfo(
                bitmapInfo.makeWH(scaledWidth, scaledHeight).makeColorType(scaledColorType));
        if (!outputBitmap.tryAllocPixels(outputAllocator)) {
            // This should only fail on OOM.  The recyclingAllocator should have
            // enough memory since we check this before decoding using the
            // scaleCheckingAllocator.
            return nullObjectReturn("allocation failed for scaled bitmap");
        }

        SkPaint paint;
        // kSrc_Mode instructs us to overwrite the uninitialized pixels in
        // outputBitmap.  Otherwise we would blend by default, which is not
        // what we want.
        paint.setBlendMode(SkBlendMode::kSrc);

        SkCanvas canvas(outputBitmap, SkCanvas::ColorBehavior::kLegacy);
        canvas.scale(scaleX, scaleY);
        decodingBitmap.setImmutable(); // so .asImage() doesn't make a copy
        canvas.drawImage(decodingBitmap.asImage(), 0.0f, 0.0f,
                         SkSamplingOptions(SkFilterMode::kLinear), &paint);
    } else {
        outputBitmap.swap(decodingBitmap);
    }

    if (padding) {
        peeker.getPadding(env, padding);
    }

    // If we get here, the outputBitmap should have an installed pixelref.
    if (outputBitmap.pixelRef() == NULL) {
        return nullObjectReturn("Got null SkPixelRef");
    }

    if (!isMutable && javaBitmap == NULL) {
        // promise we will never change our pixels (great for sharing and pictures)
        outputBitmap.setImmutable();
    }

    bool isPremultiplied = !requireUnpremultiplied;
    if (javaBitmap != nullptr) {
        bitmap::reinitBitmap(env, javaBitmap, outputBitmap.info(), isPremultiplied);
        outputBitmap.notifyPixelsChanged();
        // If a java bitmap was passed in for reuse, pass it back
        return javaBitmap;
    }

    int bitmapCreateFlags = 0x0;
    if (isMutable) bitmapCreateFlags |= android::bitmap::kBitmapCreateFlag_Mutable;
    if (isPremultiplied) bitmapCreateFlags |= android::bitmap::kBitmapCreateFlag_Premultiplied;

    if (isHardware) {
        sk_sp<Bitmap> hardwareBitmap = Bitmap::allocateHardwareBitmap(outputBitmap);
        if (!hardwareBitmap.get()) {
            return nullObjectReturn("Failed to allocate a hardware bitmap");
        }
        return bitmap::createBitmap(env, hardwareBitmap.release(), bitmapCreateFlags,
                ninePatchChunk, ninePatchInsets, -1);
    }

    // now create the java bitmap
    return bitmap::createBitmap(env, defaultAllocator.getStorageObjAndReset(),
            bitmapCreateFlags, ninePatchChunk, ninePatchInsets, -1);
}
```



### SkAdroidCodec

SkAndroidCodec是google基于SkCodec包装的一个类，定义在：

[external/skia/include/codec/SkAndroidCodec.h](https://android-opengrok.bangnimang.net/android-12.0.0_r3/xref/external/skia/include/codec/SkAndroidCodec.h)

用于对接JNI层的BitmapFactory，针对原始的SkCodec主要扩展了几个功能：

- sampleSize的处理
- dimension的控制

```c++
    /**
     *  Compute the appropriate sample size to get to |size|.
     *
     *  @param size As an input parameter, the desired output size of
     *      the decode. As an output parameter, the smallest sampled size
     *      larger than the input.
     *  @return the sample size to set AndroidOptions::fSampleSize to decode
     *      to the output |size|.
     */
    int computeSampleSize(SkISize* size) const;

    /**
     *  Returns the dimensions of the scaled output image, for an input
     *  sampleSize.
     *
     *  When the sample size divides evenly into the original dimensions, the
     *  scaled output dimensions will simply be equal to the original
     *  dimensions divided by the sample size.
     *
     *  When the sample size does not divide even into the original
     *  dimensions, the codec may round up or down, depending on what is most
     *  efficient to decode.
     *
     *  Finally, the codec will always recommend a non-zero output, so the output
     *  dimension will always be one if the sampleSize is greater than the
     *  original dimension.
     */
    SkISize getSampledDimensions(int sampleSize) const;

    /**
     *  Return (via desiredSubset) a subset which can decoded from this codec,
     *  or false if the input subset is invalid.
     *
     *  @param desiredSubset in/out parameter
     *                       As input, a desired subset of the original bounds
     *                       (as specified by getInfo).
     *                       As output, if true is returned, desiredSubset may
     *                       have been modified to a subset which is
     *                       supported. Although a particular change may have
     *                       been made to desiredSubset to create something
     *                       supported, it is possible other changes could
     *                       result in a valid subset.  If false is returned,
     *                       desiredSubset's value is undefined.
     *  @return true         If the input desiredSubset is valid.
     *                       desiredSubset may be modified to a subset
     *                       supported by the codec.
     *          false        If desiredSubset is invalid (NULL or not fully
     *                       contained within the image).
     */
    bool getSupportedSubset(SkIRect* desiredSubset) const;
    // TODO: Rename SkCodec::getValidSubset() to getSupportedSubset()

    /**
     *  Returns the dimensions of the scaled, partial output image, for an
     *  input sampleSize and subset.
     *
     *  @param sampleSize Factor to scale down by.
     *  @param subset     Must be a valid subset of the original image
     *                    dimensions and a subset supported by SkAndroidCodec.
     *                    getSubset() can be used to obtain a subset supported
     *                    by SkAndroidCodec.
     *  @return           Size of the scaled partial image.  Or zero size
     *                    if either of the inputs is invalid.
     */
    SkISize getSampledSubsetDimensions(int sampleSize, const SkIRect& subset) const;
```

SkAndroidCodec有两个继承类，一个是SkAndroidCodecAdapter，一个是SkSampledCodec。

SkAndroidCodecAdapter负责包装透传到SkCodec类中处理，

SkSampledCodec用于处理SkCodec不支持的，android增加的功能。

这两个类都定义了如下可被继承的方法，用于各个实现的解码器来做处理。

各个实现的功能类可以继承实现：

- onGetSampledDimensions：根据sampleSize返回合适的大小
- onGetSupportedSubset：返回支持的区域
- onGetAndroidPixels：解码返回

其实，对于SkAndroidCodecAdapter来说，这三个方法的实现仍然交给SkCodec来处理的。

```c++
protected:

    SkISize onGetSampledDimensions(int sampleSize) const override;

    bool onGetSupportedSubset(SkIRect* desiredSubset) const override { return true; }

    SkCodec::Result onGetAndroidPixels(const SkImageInfo& info, void* pixels, size_t rowBytes,
            const AndroidOptions& options) override;
```

SkSampledCodec中实现了两个方法：

```c++
private:
    /**
     *  Find the best way to account for native scaling.
     *
     *  Return a size that fCodec can scale to, and adjust sampleSize to finish scaling.
     *
     *  @param sampleSize As an input, the requested sample size.
     *                    As an output, sampling needed after letting fCodec
     *                    scale to the returned dimensions.
     *  @param nativeSampleSize Optional output parameter. Will be set to the
     *                          effective sample size done by fCodec.
     *  @return SkISize The size that fCodec should scale to.
     */
    SkISize accountForNativeScaling(int* sampleSize, int* nativeSampleSize = nullptr) const;

    /**
     *  This fulfills the same contract as onGetAndroidPixels().
     *
     *  We call this function from onGetAndroidPixels() if we have determined
     *  that fCodec does not support the requested scale, and we need to
     *  provide the scale by sampling.
     */
    SkCodec::Result sampledDecode(const SkImageInfo& info, void* pixels, size_t rowBytes,
            const AndroidOptions& options);
```

android在创建的时候，已经定义好了哪些格式走哪些类：

```c++
    switch ((SkEncodedImageFormat)codec->getEncodedFormat()) {
        case SkEncodedImageFormat::kPNG:
        case SkEncodedImageFormat::kICO:
        case SkEncodedImageFormat::kJPEG:
#ifndef SK_HAS_WUFFS_LIBRARY
        case SkEncodedImageFormat::kGIF:
#endif
        case SkEncodedImageFormat::kBMP:
        case SkEncodedImageFormat::kWBMP:
        case SkEncodedImageFormat::kHEIF:
        case SkEncodedImageFormat::kAVIF:
            return std::make_unique<SkSampledCodec>(codec.release());
#ifdef SK_HAS_WUFFS_LIBRARY
        case SkEncodedImageFormat::kGIF:
#endif
#ifdef SK_CODEC_DECODES_WEBP
        case SkEncodedImageFormat::kWEBP:
#endif
#ifdef SK_CODEC_DECODES_RAW
        case SkEncodedImageFormat::kDNG:
#endif
#if defined(SK_CODEC_DECODES_WEBP) || defined(SK_CODEC_DECODES_RAW) || defined(SK_HAS_WUFFS_LIBRARY)
            return std::make_unique<SkAndroidCodecAdapter>(codec.release());
#endif
    }
```



### SkCodec处理流程

SkCodec是skia中定义的用于编解码图片的接口，定义在：

[external/skia/include/codec/SkCodec.h](https://android-opengrok.bangnimang.net/android-12.0.0_r3/xref/external/skia/include/codec/SkCodec.h)

SkStream是对输入数据源的抽象，定义在：

[external/skia/include/core/SkStream.h](https://android-opengrok.bangnimang.net/android-12.0.0_r3/xref/external/skia/include/core/SkStream.h)

#### 接口定义

SkCodec是基类，真正完成解码的是各个实现子类。基类定义了这些接口：

几个关键接口：

- 创建对象：MakeFromStream，MakeFromData
- 读取信息：getInfo，getICCProfile，getEncodedFormat
- 解码：getPixels

```c++
public:
    static std::unique_ptr<SkCodec> MakeFromStream(
            std::unique_ptr<SkStream>, Result* = nullptr,
            SkPngChunkReader* = nullptr,
            SelectionPolicy selectionPolicy = SelectionPolicy::kPreferStillImage);
    /**
     *  Return a reasonable SkImageInfo to decode into.
     *
     *  If the image has an ICC profile that does not map to an SkColorSpace,
     *  the returned SkImageInfo will use SRGB.
     */
    SkImageInfo getInfo() const { return fEncodedInfo.makeImageInfo(); }

    SkISize dimensions() const { return {fEncodedInfo.width(), fEncodedInfo.height()}; }

    /**
     * Return the ICC profile of the encoded data.
     */
    const skcms_ICCProfile* getICCProfile() const

    /**
     *  Return a size that approximately supports the desired scale factor.
     *  The codec may not be able to scale efficiently to the exact scale
     *  factor requested, so return a size that approximates that scale.
     *  The returned value is the codec's suggestion for the closest valid
     *  scale that it can natively support
     */
    SkISize getScaledDimensions(float desiredScale) const
     
     /**
     *  Return (via desiredSubset) a subset which can decoded from this codec,
     *  or false if this codec cannot decode subsets or anything similar to
     *  desiredSubset.
     *
     *  @param desiredSubset In/out parameter. As input, a desired subset of
     *      the original bounds (as specified by getInfo). If true is returned,
     *      desiredSubset may have been modified to a subset which is
     *      supported. Although a particular change may have been made to
     *      desiredSubset to create something supported, it is possible other
     *      changes could result in a valid subset.
     *      If false is returned, desiredSubset's value is undefined.
     *  @return true if this codec supports decoding desiredSubset (as
     *      returned, potentially modified)
     */
    bool getValidSubset(SkIRect* desiredSubset) const
      
     /**
     *  Format of the encoded data.
     */
    SkEncodedImageFormat getEncodedFormat() const { return this->onGetEncodedFormat(); }

    /**
     *  Decode into the given pixels, a block of memory of size at
     *  least (info.fHeight - 1) * rowBytes + (info.fWidth *
     *  bytesPerPixel)
     *
     *  Repeated calls to this function should give the same results,
     *  allowing the PixelRef to be immutable.
     *
     *  @param info A description of the format (config, size)
     *         expected by the caller.  This can simply be identical
     *         to the info returned by getInfo().
     *
     *         This contract also allows the caller to specify
     *         different output-configs, which the implementation can
     *         decide to support or not.
     *
     *         A size that does not match getInfo() implies a request
     *         to scale. If the generator cannot perform this scale,
     *         it will return kInvalidScale.
     *
     *         If the info contains a non-null SkColorSpace, the codec
     *         will perform the appropriate color space transformation.
     *
     *         If the caller passes in the SkColorSpace that maps to the
     *         ICC profile reported by getICCProfile(), the color space
     *         transformation is a no-op.
     *
     *         If the caller passes a null SkColorSpace, no color space
     *         transformation will be done.
     *
     *  If a scanline decode is in progress, scanline mode will end, 
     *  requiring the client to call
     *  startScanlineDecode() in order to return to decoding scanlines.
     *
     *  @return Result kSuccess, or another value explaining the type of failure.
     */
    Result getPixels(const SkImageInfo& info, void* pixels, size_t rowBytes, const Options*);

    /**
     *  Simplified version of getPixels() that uses the default Options.
     */
    Result getPixels(const SkImageInfo& info, void* pixels, size_t rowBytes) {
        return this->getPixels(info, pixels, rowBytes, nullptr);
    }

    Result getPixels(const SkPixmap& pm, const Options* opts = nullptr) {
        return this->getPixels(pm.info(), pm.writable_addr(), pm.rowBytes(), opts);
    }
```

根据流程图可以看到，实际有8个完成具体工作的子类，也可以理解为插件实现。

#### 插件注册

skia里使用了插件的方式，注册了各种格式类型的图片的插件，用于不同格式的解码。

这个插件的定义简单有效，是值得学习的，定义了两个C风格的函数指针作为接口，各个插件需要实现这两个方法，就可以成功注册为一个插件了。

```c++
struct DecoderProc {
    bool (*IsFormat)(const void*, size_t);
    std::unique_ptr<SkCodec> (*MakeFromStream)(std::unique_ptr<SkStream>, SkCodec::Result*);
};
```

以下是注册的函数，skia支持使用者注册自己实现的插件进行解码：

```c++
    // Register a decoder at runtime by passing two function pointers:
    //    - peek() to return true if the span of bytes appears to be your encoded format;
    //    - make() to attempt to create an SkCodec from the given stream.
    // Not thread safe.
    static void Register(
            bool                     (*peek)(const void*, size_t),
            std::unique_ptr<SkCodec> (*make)(std::unique_ptr<SkStream>, SkCodec::Result*));
```

skia框架里默认实现了常见的各种图片格式的注册，Android环境下一般都是用这些默认的：

```c++
static std::vector<DecoderProc>* decoders() {
    static auto* decoders = new std::vector<DecoderProc> {
    #ifdef SK_CODEC_DECODES_JPEG
        { SkJpegCodec::IsJpeg, SkJpegCodec::MakeFromStream },
    #endif
    #ifdef SK_CODEC_DECODES_WEBP
        { SkWebpCodec::IsWebp, SkWebpCodec::MakeFromStream },
    #endif
    #ifdef SK_HAS_WUFFS_LIBRARY
        { SkWuffsCodec_IsFormat, SkWuffsCodec_MakeFromStream },
    #elif defined(SK_USE_LIBGIFCODEC)
        { SkGifCodec::IsGif, SkGifCodec::MakeFromStream },
    #endif
    #ifdef SK_CODEC_DECODES_PNG
        { SkIcoCodec::IsIco, SkIcoCodec::MakeFromStream },
    #endif
        { SkBmpCodec::IsBmp, SkBmpCodec::MakeFromStream },
        { SkWbmpCodec::IsWbmp, SkWbmpCodec::MakeFromStream },
    };
    return decoders;
}
```

以jpeg为例，可以具体看下，每个插件怎么识别类型的：

jpeg文件开头是固定的0xFF, 0xD8, 0xFF：

```c++
bool SkJpegCodec::IsJpeg(const void* buffer, size_t bytesRead) {
    constexpr uint8_t jpegSig[] = { 0xFF, 0xD8, 0xFF };
    return bytesRead >= 3 && !memcmp(buffer, jpegSig, sizeof(jpegSig));
}
```

#### 插件调用

实际使用的时候，就是遍历所有的decoders，根据peek的结果检查是否是对应的格式：

```c++
    // PNG is special, since we want to be able to supply an SkPngChunkReader.
    // But this code follows the same pattern as the loop.
#ifdef SK_CODEC_DECODES_PNG
    if (SkPngCodec::IsPng(buffer, bytesRead)) {
        return SkPngCodec::MakeFromStream(std::move(stream), outResult, chunkReader);
    } else
#endif
    {
        for (DecoderProc proc : *decoders()) {
            if (proc.IsFormat(buffer, bytesRead)) {
                return proc.MakeFromStream(std::move(stream), outResult);
            }
        }

#ifdef SK_HAS_HEIF_LIBRARY
        SkEncodedImageFormat format;
        if (SkHeifCodec::IsSupported(buffer, bytesRead, &format)) {
            return SkHeifCodec::MakeFromStream(std::move(stream), selectionPolicy,
                    format, outResult);
        }
#endif

#ifdef SK_CODEC_DECODES_RAW
        // Try to treat the input as RAW if all the other checks failed.
        return SkRawCodec::MakeFromStream(std::move(stream), outResult);
#endif
    }
```

但是skia这里其实做了各种适配，PNG，RAW，HEIF等都是打乱了原来的架构设计，说明实际世界变化太快，代码设计跟不上了，原来都是静态图，对于包含视频帧的图（HEIF，AVIF）是没有预料到的，导致原来的识别方式无法满足了，还要返回一个format用于区分是HEIF还是AVIF。

#### 插件解码

以jpeg为例可以看下，具体怎么实现的，关于libjpeg的详细使用可以单独再写。

解码使用libjpeg库的方法，jpeg_read_scanlines 按照扫描线读数据进行解码。

```c++
/*
 * Performs the jpeg decode
 */
SkCodec::Result SkJpegCodec::onGetPixels(const SkImageInfo& dstInfo,
                                         void* dst, size_t dstRowBytes,
                                         const Options& options,
                                         int* rowsDecoded) {
    if (options.fSubset) {
        // Subsets are not supported.
        return kUnimplemented;
    }

    // Get a pointer to the decompress info since we will use it quite frequently
    jpeg_decompress_struct* dinfo = fDecoderMgr->dinfo();

    int rows = this->readRows(dstInfo, dst, dstRowBytes, dstInfo.height(), options);
    if (rows < dstInfo.height()) {
        *rowsDecoded = rows;
        return fDecoderMgr->returnFailure("Incomplete image data", kIncompleteInput);
    }

    return kSuccess;
}
int SkJpegCodec::readRows(const SkImageInfo& dstInfo, void* dst, size_t rowBytes, int count,
                          const Options& opts) {
    for (int y = 0; y < count; y++) {
        uint32_t lines = jpeg_read_scanlines(fDecoderMgr->dinfo(), &decodeDst, 1);
        if (0 == lines) {
            return y;
        }

        if (fSwizzler) {
            fSwizzler->swizzle(swizzleDst, decodeDst);
        }

        if (this->colorXform()) {
            this->applyColorXform(dst, swizzleDst, dstWidth);
            dst = SkTAddOffset<void>(dst, rowBytes);
        }

        decodeDst = SkTAddOffset<JSAMPLE>(decodeDst, decodeDstRowBytes);
        swizzleDst = SkTAddOffset<uint32_t>(swizzleDst, swizzleDstRowBytes);
    }

    return count;
}
```

### 解码返回

对于位图来说，解码得到的就是位图pixels数据buffer，和配套的Info信息，包含了大小、format等。对于数据来说，不需要拷贝返回（也不可能，数据太大了，浪费资源），做地址指针的记录就行；而配套信息，需要包装成java层的对象予以返回。

java层的Bitmap提供了一个私有构造方法，通过JNI构造java object返回回去。native层创建成功后，通过 defaultAllocator.getStorageObjAndReset() 返回一个Bitmap对象指针，并传递给BitmapWrapper类，java中持有这个对象的引用，且后续访问数据的时候，是通过这个类来操作的。

```c++
jobject createBitmap(JNIEnv* env, Bitmap* bitmap,
        int bitmapCreateFlags, jbyteArray ninePatchChunk, jobject ninePatchInsets,
        int density) {
    bool isMutable = bitmapCreateFlags & kBitmapCreateFlag_Mutable;
    bool isPremultiplied = bitmapCreateFlags & kBitmapCreateFlag_Premultiplied;
    // The caller needs to have already set the alpha type properly, so the
    // native SkBitmap stays in sync with the Java Bitmap.
    assert_premultiplied(bitmap->info(), isPremultiplied);
    bool fromMalloc = bitmap->pixelStorageType() == PixelStorageType::Heap;
    BitmapWrapper* bitmapWrapper = new BitmapWrapper(bitmap);
    if (!isMutable) {
        bitmapWrapper->bitmap().setImmutable();
    }
    jobject obj = env->NewObject(gBitmap_class, gBitmap_constructorMethodID,
            reinterpret_cast<jlong>(bitmapWrapper), bitmap->width(), bitmap->height(), density,
            isPremultiplied, ninePatchChunk, ninePatchInsets, fromMalloc);

    if (env->ExceptionCheck() != 0) {
        ALOGE("*** Uncaught exception returned from Java call!\n");
        env->ExceptionDescribe();
    }
    return obj;
}
```

看下BitmapWrapper类：

```c++
class BitmapWrapper {
public:
    explicit BitmapWrapper(Bitmap* bitmap)
        : mBitmap(bitmap) { }

    void freePixels() {
        mInfo = mBitmap->info();
        mHasHardwareMipMap = mBitmap->hasHardwareMipMap();
        mAllocationSize = mBitmap->getAllocationByteCount();
        mRowBytes = mBitmap->rowBytes();
        mGenerationId = mBitmap->getGenerationID();
        mIsHardware = mBitmap->isHardware();
        mBitmap.reset();
    }

    bool valid() {
        return mBitmap != nullptr;
    }

    Bitmap& bitmap() {
        assertValid();
        return *mBitmap;
    }

    void assertValid() {
        LOG_ALWAYS_FATAL_IF(!valid(), "Error, cannot access an invalid/free'd bitmap here!");
    }

    void getSkBitmap(SkBitmap* outBitmap) {
        assertValid();
        mBitmap->getSkBitmap(outBitmap);
    }

    bool hasHardwareMipMap() {
        if (mBitmap) {
            return mBitmap->hasHardwareMipMap();
        }
        return mHasHardwareMipMap;
    }

    void setHasHardwareMipMap(bool hasMipMap) {
        assertValid();
        mBitmap->setHasHardwareMipMap(hasMipMap);
    }

    void setAlphaType(SkAlphaType alphaType) {
        assertValid();
        mBitmap->setAlphaType(alphaType);
    }

    void setColorSpace(sk_sp<SkColorSpace> colorSpace) {
        assertValid();
        mBitmap->setColorSpace(colorSpace);
    }

    const SkImageInfo& info() {
        if (mBitmap) {
            return mBitmap->info();
        }
        return mInfo;
    }

    size_t getAllocationByteCount() const {
        if (mBitmap) {
            return mBitmap->getAllocationByteCount();
        }
        return mAllocationSize;
    }

    size_t rowBytes() const {
        if (mBitmap) {
            return mBitmap->rowBytes();
        }
        return mRowBytes;
    }

    uint32_t getGenerationID() const {
        if (mBitmap) {
            return mBitmap->getGenerationID();
        }
        return mGenerationId;
    }

    bool isHardware() {
        if (mBitmap) {
            return mBitmap->isHardware();
        }
        return mIsHardware;
    }

    ~BitmapWrapper() { }肉欸

private:
    sk_sp<Bitmap> mBitmap;
    SkImageInfo mInfo;
    bool mHasHardwareMipMap;
    size_t mAllocationSize;
    size_t mRowBytes;
    uint32_t mGenerationId;
    bool mIsHardware;
};
```

里面所有的操作都依赖Bitmap类，这里的Bitmap是个什么对象？再这里定义的，可以看到，实现里包装了skia的功能类skBitmap。

/[frameworks](https://android-opengrok.bangnimang.net/android-12.0.0_r3/xref/frameworks/)/[base](https://android-opengrok.bangnimang.net/android-12.0.0_r3/xref/frameworks/base/)/[libs](https://android-opengrok.bangnimang.net/android-12.0.0_r3/xref/frameworks/base/libs/)/[hwui](https://android-opengrok.bangnimang.net/android-12.0.0_r3/xref/frameworks/base/libs/hwui/)/[hwui](https://android-opengrok.bangnimang.net/android-12.0.0_r3/xref/frameworks/base/libs/hwui/hwui/)/[Bitmap.h](https://android-opengrok.bangnimang.net/android-12.0.0_r3/xref/frameworks/base/libs/hwui/hwui/Bitmap.h)

```c++
class Bitmap : public SkPixelRef {
public:
    /* The allocate factories not only construct the Bitmap object but also allocate the
     * backing store whose type is determined by the specific method that is called.
     *
     * The factories that accept SkBitmap* as a param will modify those params by
     * installing the returned bitmap as their SkPixelRef.
     *
     * The factories that accept const SkBitmap& as a param will copy the contents of the
     * provided bitmap into the newly allocated buffer.
     */
    static sk_sp<Bitmap> allocateAshmemBitmap(SkBitmap* bitmap);
    static sk_sp<Bitmap> allocateHardwareBitmap(const SkBitmap& bitmap);
    static sk_sp<Bitmap> allocateHeapBitmap(SkBitmap* bitmap);
    static sk_sp<Bitmap> allocateHeapBitmap(const SkImageInfo& info);
    static sk_sp<Bitmap> allocateHeapBitmap(size_t size, const SkImageInfo& i, size_t rowBytes);

    /* The createFrom factories construct a new Bitmap object by wrapping the already allocated
     * memory that is provided as an input param.
     */
#ifdef __ANDROID__ // Layoutlib does not support hardware acceleration
    static sk_sp<Bitmap> createFrom(AHardwareBuffer* hardwareBuffer,
                                    sk_sp<SkColorSpace> colorSpace,
                                    BitmapPalette palette = BitmapPalette::Unknown);

    static sk_sp<Bitmap> createFrom(AHardwareBuffer* hardwareBuffer,
                                    SkColorType colorType,
                                    sk_sp<SkColorSpace> colorSpace,
                                    SkAlphaType alphaType,
                                    BitmapPalette palette);
#endif
    static sk_sp<Bitmap> createFrom(const SkImageInfo& info, size_t rowBytes, int fd, void* addr,
                                    size_t size, bool readOnly);
    static sk_sp<Bitmap> createFrom(const SkImageInfo&, SkPixelRef&);

    int rowBytesAsPixels() const { return rowBytes() >> mInfo.shiftPerPixel(); }

    void reconfigure(const SkImageInfo& info, size_t rowBytes);
    void reconfigure(const SkImageInfo& info);
    void setColorSpace(sk_sp<SkColorSpace> colorSpace);
    void setAlphaType(SkAlphaType alphaType);

    void getSkBitmap(SkBitmap* outBitmap);

    SkBitmap getSkBitmap() {
        SkBitmap ret;
        getSkBitmap(&ret);
        return ret;
    }

    int getAshmemFd() const;
    size_t getAllocationByteCount() const;

    void setHasHardwareMipMap(bool hasMipMap);
    bool hasHardwareMipMap() const;

    bool isOpaque() const { return mInfo.isOpaque(); }
    SkColorType colorType() const { return mInfo.colorType(); }
    const SkImageInfo& info() const { return mInfo; }

    void getBounds(SkRect* bounds) const;

    bool isHardware() const { return mPixelStorageType == PixelStorageType::Hardware; }

    PixelStorageType pixelStorageType() const { return mPixelStorageType; }

#ifdef __ANDROID__ // Layoutlib does not support hardware acceleration
     AHardwareBuffer* hardwareBuffer();
#endif

    /**
     * Creates or returns a cached SkImage and is safe to be invoked from either
     * the UI or RenderThread.
     *
     */
    sk_sp<SkImage> makeImage();

    static BitmapPalette computePalette(const SkImageInfo& info, const void* addr, size_t rowBytes);

    static BitmapPalette computePalette(const SkBitmap& bitmap) {
        return computePalette(bitmap.info(), bitmap.getPixels(), bitmap.rowBytes());
    }

    BitmapPalette palette() {
        if (!isHardware() && mPaletteGenerationId != getGenerationID()) {
            mPalette = computePalette(info(), pixels(), rowBytes());
            mPaletteGenerationId = getGenerationID();
        }
        return mPalette;
    }

  // returns true if rowBytes * height can be represented by a positive int32_t value
  // and places that value in size.
  static bool computeAllocationSize(size_t rowBytes, int height, size_t* size);

  // These must match the int values of CompressFormat in Bitmap.java, as well as
  // AndroidBitmapCompressFormat.
  enum class JavaCompressFormat {
    Jpeg = 0,
    Png = 1,
    Webp = 2,
    WebpLossy = 3,
    WebpLossless = 4,
  };

  bool compress(JavaCompressFormat format, int32_t quality, SkWStream* stream);

  static bool compress(const SkBitmap& bitmap, JavaCompressFormat format,
                       int32_t quality, SkWStream* stream);
private:
    static sk_sp<Bitmap> allocateAshmemBitmap(size_t size, const SkImageInfo& i, size_t rowBytes);

    Bitmap(void* address, size_t allocSize, const SkImageInfo& info, size_t rowBytes);
    Bitmap(SkPixelRef& pixelRef, const SkImageInfo& info);
    Bitmap(void* address, int fd, size_t mappedSize, const SkImageInfo& info, size_t rowBytes);
#ifdef __ANDROID__ // Layoutlib does not support hardware acceleration
    Bitmap(AHardwareBuffer* buffer, const SkImageInfo& info, size_t rowBytes,
           BitmapPalette palette);

    // Common code for the two public facing createFrom(AHardwareBuffer*, ...)
    // methods.
    // bufferDesc is only used to compute rowBytes.
    static sk_sp<Bitmap> createFrom(AHardwareBuffer* hardwareBuffer, const SkImageInfo& info,
                                    const AHardwareBuffer_Desc& bufferDesc, BitmapPalette palette);
#endif

    virtual ~Bitmap();

    SkImageInfo mInfo;

    const PixelStorageType mPixelStorageType;

    BitmapPalette mPalette = BitmapPalette::Unknown;
    uint32_t mPaletteGenerationId = -1;

    bool mHasHardwareMipMap = false;

    union {
        struct {
            SkPixelRef* pixelRef;
        } wrapped;
        struct {
            void* address;
            int fd;
            size_t size;
        } ashmem;
        struct {
            void* address;
            size_t size;
        } heap;
#ifdef __ANDROID__ // Layoutlib does not support hardware acceleration
        struct {
            AHardwareBuffer* buffer;
        } hardware;
#endif
    } mPixelStorage;

    sk_sp<SkImage> mImage;  // Cache is used only for HW Bitmaps with Skia pipeline.
};
```

补充：

Android为了同时支持native的bitmap接口，包装了一个不太容易理解的ABitmap和bitmap对象，用于对外和对内的数据类型转换。

/[frameworks](https://android-opengrok.bangnimang.net/android-12.0.0_r3/xref/frameworks/)/[base](https://android-opengrok.bangnimang.net/android-12.0.0_r3/xref/frameworks/base/)/[libs](https://android-opengrok.bangnimang.net/android-12.0.0_r3/xref/frameworks/base/libs/)/[hwui](https://android-opengrok.bangnimang.net/android-12.0.0_r3/xref/frameworks/base/libs/hwui/)/[apex](https://android-opengrok.bangnimang.net/android-12.0.0_r3/xref/frameworks/base/libs/hwui/apex/)/[include](https://android-opengrok.bangnimang.net/android-12.0.0_r3/xref/frameworks/base/libs/hwui/apex/include/)/[android](https://android-opengrok.bangnimang.net/android-12.0.0_r3/xref/frameworks/base/libs/hwui/apex/include/android/)/[graphics](https://android-opengrok.bangnimang.net/android-12.0.0_r3/xref/frameworks/base/libs/hwui/apex/include/android/graphics/)/[bitmap.h](https://android-opengrok.bangnimang.net/android-12.0.0_r3/xref/frameworks/base/libs/hwui/apex/include/android/graphics/bitmap.h)

```C++
 /**
   * Opaque handle for a native graphics bitmap.
   */
  typedef struct ABitmap ABitmap;
```

其实是没有ABitmap的完整类型定义的，就是一个空的struct。而Bitmap类也没有有效实现，提供了几个接口，用于访问native层的数据和信息。

但是在真正使用的时候，可以看到，是和native的Bitmap对象可以互转的，地址直接赋值，其实就是一个类型。

```C++
    class TypeCast {
    public:
        static inline Bitmap& toBitmapRef(const ABitmap* bitmap) {
            return const_cast<Bitmap&>(reinterpret_cast<const Bitmap&>(*bitmap));
        }

        static inline Bitmap* toBitmap(ABitmap* bitmap) {
            return reinterpret_cast<Bitmap*>(bitmap);
        }

        static inline ABitmap* toABitmap(Bitmap* bitmap) {
            return reinterpret_cast<ABitmap*>(bitmap);
        }
```
