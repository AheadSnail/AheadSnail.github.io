---
layout: pager
title: 'NdkR17 编译FFmpeg4.0 添加AAC,X264支持'
date: 2018-08-03 14:09:16
tags: [NDK,FFmpeg,AAC,X264]
description:  NdkR17 编译FFmpeg4.0 添加AAC,X264支持
---

NdkR17 编译FFmpeg4.0 添加AAC,X264支持
<!--more-->

****简介****
===
```
FFmpeg是一套可以用来记录、转换数字音频、视频，并能将其转化为流的开源计算机程序。采用LGPL或GPL许可证。它提供了录制、转换以及流化音视频的完整解决方案。
它包含了非常先进的音频/视频编解码库libavcodec，为了保证高可移植性和编解码质量，libavcodec里很多code都是从头开发的。

本文采用最新版的NDK 版本NDkR17 编译最新版的FFmpeg，以此来记录编译的过程
```

****环境准备****
===
```
1.采用的是ubuntu16.04来编译
2.NDK采用的是android-ndk-r17b
3.下载最新版的ffmpeg-4.0.2下载地址为 https://ffmpeg.org/releases/ffmpeg-4.0.2.tar.bz2
4.下载最新版的x264 下载地址为 ftp://ftp.videolan.org/pub/x264/snapshots/last_x264.tar.bz2
5.下载最新版的 fdk-aac-0.1.6 下载地址为 https://jaist.dl.sourceforge.net/project/opencore-amr/fdk-aac/fdk-aac-0.1.6.tar.gz
```

****交叉编译****
===
```java
前面一篇文章介绍到，NDK的改动，推荐使用clang的方式来编译，而且如果是使用了AutoConf维护的开源库，最好是使用单独的交叉编译链

#!/bin/bash
ANDROID_HOME=/home/yuhui/workSpace/android-ndk-r17b
$ANDROID_HOME/build/tools/make_standalone_toolchain.py \
   --arch arm --api 16 --stl=libc++ \
   --install-dir $ANDROID_HOME/toolchain


由于要在FFmpeg中集成进x264，AAC，所以要提起将这俩个库编译，下面是对应的脚本文件

编译AAC

#!/bin/bash
ANDROID_HOME=/home/yuhui/workSpace/android-ndk-r17b
TOOLCHAIN=$ANDROID_HOME/toolchain
HOST=arm-linux-androideabi
PREFIX=/home/yuhui/workSpace/FFmpeg/AAcBuild

export CC=$TOOLCHAIN/bin/arm-linux-androideabi-clang
export CXX=$TOOLCHAIN/bin/arm-linux-androideabi-clang++
export LDFLAGS=" -pie"


./configure \
--host=armv7a \
--prefix=$PREFIX \
--enable-static \
--disable-shared

  make
  make install

echo "build AAc complete!"   
```
![结果显示](/uploads/NDkR17编译FFmpeg4.0/AAC生成文件.png)
```java
这里要检查是否是采用clang来生成的，对应的可以在对应的源文件目录下面的config.log文件中有体现，比如
```
![结果显示](/uploads/NDkR17编译FFmpeg4.0/x264采用Clang编译.jpg)
```java

编译X264

#!/bin/bash
ANDROID_HOME=/home/yuhui/workSpace/android-ndk-r17b
TOOLCHAIN=$ANDROID_HOME/toolchain
HOST=arm-linux-androideabi
PREFIX=/home/yuhui/workSpace/FFmpeg/X264Build

export CC=$TOOLCHAIN/bin/arm-linux-androideabi-clang
export CXX=$TOOLCHAIN/bin/arm-linux-androideabi-clang++
export LDFLAGS=" -pie"

./configure \
--host=arm-linux \
--prefix=$PREFIX \
--enable-static \
--enable-pic \
--enable-strip \
--disable-cli \
--disable-asm \
--extra-cflags=" -mfloat-abi=softfp -mfpu=neon" \
--cross-prefix=$TOOLCHAIN/bin/arm-linux-androideabi-

  make
  make install

echo "build x264 complete!"
```   
![结果显示](/uploads/NDkR17编译FFmpeg4.0/x264生成文件.png)   
```java   
这里要检查是否是采用clang来生成的，对应的可以在对应的源文件目录下面的config.log文件中有体现，比如
```
![结果显示](/uploads/NDkR17编译FFmpeg4.0/AAC采用Clang编译.png)
```java
编译FFmpeg

首先在FFmpeg源码目录新建一个目录为存储AAc,X264库，这里创建为external-libs,里面存放俩个文件夹分别为
AAC,X264 然后分别对应的在这俩个文件夹里面放进刚刚编译生成的文件,分别包括include文件夹，已经lib文件夹都要拷贝过来
```
![结果显示](/uploads/NDkR17编译FFmpeg4.0/external-libs目录.jpg)
![结果显示](/uploads/NDkR17编译FFmpeg4.0/AAc拷贝目录.jpg)

```java

FFmpeg 编译脚本

#!/bin/bash
ANDROID_HOME=/home/yuhui/workSpace/android-ndk-r17b
TOOLCHAIN=$ANDROID_HOME/toolchain
SYSROOT=$TOOLCHAIN/sysroot
PREFIX=/home/yuhui/workSpace/FFmpeg/FFmpegAndroidBuild
EXTERNAL_PATH=/home/yuhui/workSpace/FFmpeg/ffmpeg-4.0/external-libs

./configure \
--enable-static \
--disable-shared \
--disable-stripping \
--disable-ffmpeg \
--disable-ffplay \
--disable-ffprobe \
--disable-avdevice \
--disable-devices \
--disable-indevs \
--disable-outdevs \
--disable-debug \
--disable-asm \
--disable-yasm \
--disable-doc \
--enable-small \
--enable-dct \
--enable-dwt \
--enable-lsp \
--enable-mdct \
--enable-rdft \
--enable-fft \
--enable-version3 \
--enable-nonfree \
--disable-filters \
--disable-postproc \
--disable-bsfs \
--enable-bsf=aac_adtstoasc \
--enable-bsf=h264_mp4toannexb \
--disable-encoders \
--enable-encoder=pcm_s16le \
--enable-encoder=aac \
--enable-encoder=libvo_amrwbenc \
--disable-decoders \
--enable-decoder=aac \
--enable-decoder=mp3 \
--enable-decoder=pcm_s16le \
--disable-parsers \
--enable-parser=aac \
--disable-muxers \
--enable-muxer=flv \
--enable-muxer=wav \
--enable-muxer=adts \
--disable-muxers \
--enable-demuxer=flv \
--enable-demuxer=wav \
--enable-demuxer=aac \
--disable-protocols \
--enable-protocol=rtmp \
--enable-protocol=file \
--enable-libfdk-aac \
--enable-libx264 \
--enable-cross-compile \
--target-os=linux \
--enable-gpl \
--arch=arm \
--toolchain=clang-usan \
--sysroot=$SYSROOT \
--cross-prefix=$TOOLCHAIN/bin/arm-linux-androideabi- \
--extra-cflags=" -I$EXTERNAL_PATH/FDK_AAC/include -I$EXTERNAL_PATH/X264/include " \
--extra-ldflags=" -pie -Wl,-lc -lm -ldl -L$EXTERNAL_PATH/FDK_AAC/lib -L$EXTERNAL_PATH/X264/lib " \
--prefix=$PREFIX

  make j8
  make install

echo "build FFmpeg complete!"


注意:其中 --enable-libx264为打开x264 库编译,--enable-encoder=libx264 设在h264 编码库为x264 编码,--enable-encoder=aac 为打开libfdk-aac 库,
--enable-decoder=aac 和--enable-libfdk-aac设置aac 的编码和解码库为libfdk-aac 编码库,这样就会替换内部的默认aac编码库。

下面是指定这俩个外部库要链接的位置，头文件的位置
--extra-cflags=" -I$EXTERNAL_PATH/FDK_AAC/include -I$EXTERNAL_PATH/X264/include " \
--extra-ldflags=" -pie -Wl,-rpath-link=$SYSROOT/usr/lib -L$SYSROOT/usr/lib -lc -lm -ldl -L$EXTERNAL_PATH/FDK_AAC/lib -L$EXTERNAL_PATH/X264/lib " \

--sysroot=$SYSROOT \  这个要指定为我们生成的交叉编译链下面的sysroot目录，而不要采用默认的sysroot
--toolchain=clang-usan \  这个可以指定我们CC 采用的是 arm-linux-androideabi-clang

有一点要注意的是，生成的AAC,X264 这俩个库的脚本文件的配置，尽量要跟FFmpeg里面的配置一样，要不然连接的时候会出现问题
```
![结果显示](/uploads/NDkR17编译FFmpeg4.0/FFmpeg编译错误.png)
```java
解决办法为 
需要将libavcodec/aaccoder.c里面的B0定义改一下，我是修改为b0，之后make ，编译成功
```
![结果显示](/uploads/NDkR17编译FFmpeg4.0/FFmpeg编译成功.png)
![结果显示](/uploads/NDkR17编译FFmpeg4.0/FFmpeg成功编译文件.png)
```java
如果在执行./configure 的时候出现了错误，可以在ffbuild 目录下的 config.log里面查看详细的错误信息
```
![结果显示](/uploads/NDkR17编译FFmpeg4.0/ffmpegConfig.jpg)

