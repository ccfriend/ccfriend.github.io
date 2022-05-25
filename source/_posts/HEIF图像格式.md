---
title: HEIF图像格式
date: 2022-05-23 21:44:33
categories: image
tags: [image, HEIF]
---

# 什么是HEIF格式
HEIF格式的全名为[高效率图像格式（High Efficiency Image Format ，HEIF）](https://en.wikipedia.org/wiki/High_Efficiency_Image_File_Format), 是一种图像容器格式，它所生成的图像文件相对较小，且图像质量也高于较早的 JPEG 标准，基于高效视频压缩格式（也称为 HEVC 或 H.265）是由动态图像专家组（MPEG）在2013年推出的新格式，参见 https://nokiatech.github.io/heif/.
<!--more-->

**TODO**
为什么这里会是nokia公司的介绍作为HEIF的主要参考？Nokia是当时支持HEIF的主要参与者。

可以看下HEIF和JPG路标图：
![img](../images/heif_jpg.png)


# HEIF支持情况
1. 以下这些描述来自2017年，苹果刚发布支持HEIF

苹果在iOS11系统中引入HEIF格式用于替代原来的JPG格式的图片。使用HEVC的编码技术存储图像数据，进一步减少存储容量和提升图像质量。 据WWDC17数据，使用HEIF会达到JPEG压缩比的2倍。使用iOS11系统的iPhone手机，在相机的设置中格式选择高效，拍摄下来的照片会保存为HEIF格式。

由于目前其他系统(Windows/Android)还不支持该格式图片的显示，所以在将iPhone中的照片导入到Windows PC时，iOS系统会将其转码为JPG格式图片。目前只有macOS High Sierra版本的mac系统支持HEIF格式图片，在将iPhone中的图片导入到该系统的mac上时可以保持原有的格式（一般后缀名为.heic，表示图像的编码格式为HEVC格式）。
iOS设备通过AirDrop发给其他iOS设备时，如果接收方为iOS 10及以下OS版本时，发送方也会将heif图片转码成JPG格式发送。

2. 厂商支持情况

最早被苹果公司的 iPhone 所使用，并且也在 Google 的 Android P 手机系统开始支持。微软也于 Windows 10 Build 17123 预览版开始，新增了对 HEIF 图像格式的系统原生支持。

3. web端支持情况

nokia的展示页面可以看到HEIF demo的图片，是基于Nokia自己的解码库完成。对于浏览器来说，很多新版本的浏览器已经支持。

# HEIF获取
## HEIF文件
**TODO** 可以看到获取的heif文件有的是heif后缀，有的是heic后缀，其实是有很多后缀的。

## 怎么获取HEIF
1. 设备拍照
   
通过上面的描述，其实可以看到，如果你手头有支持HEIF的设备，如iOS 11以后的设备，或者Android P以后的设备（需要厂商支持），就可以用设备拍一张照片就可以得到了。

2. 网络下载

当然，也可以从网络端下载，最简单的就是从Nokia官网。

在gh-pages分支下存储了网页上展示的heif原始资源图片：
https://github.com/nokiatech/heif/tree/gh-pages/content

以及更多的一些heif测试资源，包含了基本码流和配置文件：
https://github.com/nokiatech/heif_conformance

3. 工具转换

最后，就是使用Nokia的库自己转换制作heif图片：
https://github.com/nokiatech/heif <br>
支持C++、JAVA API，可以在windows，Android平台方便移植使用。

## 怎么打开HEIF
同样的，移动端用上述设备就可以了；如果下载到windows和mac(没有，自行查找)怎么打开？win11默认是支持的，如果是老版本的windows，可以从windows store中搜索heif。

# HEIF 特性
从nokia的官网可以看到这些描述：

HEIF files can store:
- Still Images, their thumbnails and related metadata such as EXIF or XMP information
- Image Collections, their thumbnails and related metadata such as EXIF or XMP information
- Image sequences such as cinemagraphs or image bursts and related metadata
- Unprocessed image and processed images in the same file with proper labelling and referencing
- Image derivation information such as rotation, overlay and grid view along with the images, so that different image derivations can make use of the same image data set
- Auxiliary image data such as depth map and alpha channel along with the images
- Audio tracks and cover images along with still images and image sequences
- Well-defined information about content context to signal e.g. stereo pairs, time-synchronized capture, image bursts and image album groups

HEIF files also inherit many properties of ISOBMFF such as edit lists, media alternatives, media data groupings. Moreover, an MP4 file can be “branded” so that it can also contain images and image sequences as well as video and audio (i.e. dual branding)

| type | description |
| ----------| ------------ |
|静态图|包含缩略图和metadata|
|图像集合|包含缩略图和metadata|
|图像序列|电影图像和图片连拍，以及metadata|
|混合图片|包含处理的和未处理的图在一个文件里|
|图片派生信息|如旋转、叠加和网格视图|
|辅助图像数据|如深度图，alpha图|
|音轨数据和封面图|静态图和图像序列中可以包含这些信息|
|图像信息上下文|如立体图像，时间同步，图片连拍，图像组|

# HEIF 解析

## HEIF解析工具
虽然系统有照片等工具可以直接展示HEIF，但是还是有很大局限性的。从Nokia上下载的各种heic图，在windows上的默认照片工具不是全都支持，而且对于开发而言，需要深入解析到每张图具体的格式内容。

金山云基于QT开发了一款在MAC上的查看工具，和MP4Info类似，通过QT编译后，也可以在windows上运行：
https://github.com/ksvc/MediaParser

其实最好的是基于Nokia的库实现解析，支持规格比较完整。

## 格式解析
部分参考：https://www.jianshu.com/p/b016d10a087d <br>
部分参考：https://zhuanlan.zhihu.com/p/35847861

### ISO BMFF
HEIF格式是基于 ISO Base Media File Format格式衍生出来的图像封装格式，所以它的文件格式同样符合ISO Base Media File Format (ISO/IEC 14496-12)中的定义（ ISOBMFF）。

文件中所有的数据都存储在称为Box的数据块结构中，每个文件由若干个Box组成，每个Box有自己的类型和长度。在一个Box中还可以包含子Box，最终由一系列的Box组成完整的文件内容，结构如下图所示，图中每个方块即代表一个Box。常见的MP4文件同样是ISOBMFF结构，所以HEIF文件结构和MP4文件结构基本一致，只是用到的Box类型有区别。

HEIF文件如果是单幅的静态图片的话，使用item的形式保存数据，所有item单独解码；如果保存的为图片序列的话，使用trak的方式保存，这种trak的形式同mp4的audio、video track类似， 每个轨道存储一张图片。

<b>ISOBMFF格式说明

![img](https://upload-images.jianshu.io/upload_images/2926667-337a35837bb3325e.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

### item 格式
heif单张图片是item形式存储，可以使用工具来查看其中的信息。

### ImageCollection 多图

### ImageBurst 连拍

### ImageSequence 动图

### Image Derivations 图片派生信息

### Auxiliary Image Storage 图片辅助数据
   
### 包含用户描述信息的图片

# 转换参考
1. 基于Android平台
基于Android平台接口和能力，实现了一个简单的转换demo，参考：
https://github.com/ccfriend/HeifDemo

2. 基于nokia的库做转换
**TODO** 开发中
