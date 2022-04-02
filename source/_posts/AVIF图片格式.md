---
title: AVIF图片格式
date: 2022-03-30 22:56:21
categories: image
tags: [image, AVIF]
---
# AVIF图片格式

参考：

https://zhuanlan.zhihu.com/p/444624167

https://zhuanlan.zhihu.com/p/355701873


## 什么是AVIF

AVIF（ AV1 Image File Format）是一种由AOM（ Alliance for Open Media）开发的基于AV1编解码器的网络图像格式。这是一种开源免版税的图像格式。AVIF支持全分辨率的10位和12位色彩以及HDR。

<!-- more -->

## AVIF 来源
视频编解码器中的帧内编码工具与图像压缩工具之间并没有显著的差异。在构建下一代视频编解码器时，它必须能够以更高的效率对关键帧进行编码。有了一个性能优于现有标准的图像编码器，在其基础上建立一个图像编码格式就很简单了。

AVIF就是源自AOM创始成员开发的AV1视频编解码标准 。为了研发AVIF，AOM扩展了MPEG推出的图像格式：ISO/IEC 23000–12标准（HEIF）。HEIF已用于存储HEVC/AVC/JPEG编码的图像。使用HEIF的原因是它支持所有现有的图像编码格式，并允许使用有损或无损模式进行压缩，支持各种子采样，多个位深度等等。

HEIF格式还允许存储一系列动画帧，并指定一个alpha通道。由于HEIF格式借鉴了下一代视频压缩技术的经验，因此可以保留诸如色域和HDR信息的元数据。


## AVIF优势
Netflix的博客从 2020年2月就开始大力宣传AVIF标准的优点。该博客[1]详细介绍了AVIF，并提供了一些优于JPEG的技术信息。作为AOM的创始成员 ，Netflix一直是AV1编解码标准的坚定支持者。因此，毫不奇怪，他们会对使用AV1来改善其UI体验感兴趣。

相比于JPEG，AVIF无论是在压缩率上还是在生态上都有着不可小觑的实力。

1. AV1编解码标准和AVIF格式免版税。
2. 所有Chromium浏览器（Chrome 85或更高版本）都支持AVIF。
3. Microsoft Windows 10从19H1开始支持AVIF。
4. AVIF得到了Google、Amazon、Netflix、Microsoft、Intel、Apple等公司的支持。
5. AVIF提供比JPEG更高的压缩效率。
6. AVIF图像格式支持透明度，HDR，宽色域和其他现代功能。

在与其他图像格式的PK中，AVIF表现十分突出。

AVIF（微帧Aurora）vs JPG

测试数据集：Kodak测试集20张 768×512 图片
![img](https://pic1.zhimg.com/80/v2-1adde73a764e356fa387a9ad5c678bbc_1440w.jpg)

![img](https://pic2.zhimg.com/80/v2-35e716ebd1bb906433a8c521071bebcd_1440w.jpg)

## 各种格式对比
AV1是视频流媒体服务和平台游戏规则的改变者。在Google和整个技术生态系统的大力推动下，AV1和AVIF处于较为有利的位置。最新一代的图像编解码器，尤其是AVIF和JPEG XL，相较于JPEG有了显著的改进。

对于支持图像大规模编码的服务和平台，需要做到兼容几种不同的图像编码方法，降低与目标设备/平台的关联性。在选择编解码标准时，需要考量该标准的关键属性是否是最适合您的那一个。

![img](https://pic2.zhimg.com/80/v2-65e7bfe6c16a6f94e8a06cfd3f2c6379_1440w.jpg)

