---
layout: pager
title: Opus 交叉编译实现
date: 2018-07-04 20:32:03
tags: [Opus,Ubuntu,NDK]
description:  Opus Ubuntu实现交叉编译
---
### 简介
> 要将一个开源库移植到Android 上面，那么有一个重要的步骤就是要尝试的采用NDK交叉编译来编译，从而确定这个库是否能够移植到Android上面，如果能够交叉编译成功，后面的步骤
就是移植到Android Studio 中来编译，实现一个可以调试的NDK环境，所以这篇文章就是介绍在Ubuntu下面，采用Ndk 交叉编译最新版的Opus，

由于最新版的NDK 已经不能指定编译armeabi了，所以我们直接编译armv7-a的

### 环境准备
> 1.采用的是ubuntu16.04来编译
2.NDK采用的是android-ndk-r14b
3.下载Opus需要使用的库,具体可以参照官网的最新稳定介绍 https://www.opus-codec.org/downloads/
opus-tools 0.1.10       https://archive.mozilla.org/pub/opus/opus-tools-0.1.10.tar.gz
opusfile 0.9   		https://archive.mozilla.org/pub/opus/opusfile-0.9.tar.gz
libopus 1.2.1		https://archive.mozilla.org/pub/opus/opus-1.2.1.tar.gz
libogg-1.3.3		https://ftp.osuosl.org/pub/xiph/releases/ogg/libogg-1.3.3.tar.gz
flac-1.3.2		https://ftp.osuosl.org/pub/xiph/releases/flac/flac-1.3.2.tar.xz
openssl-1.0.2m		https://www.openssl.org/source/openssl-1.0.2m.tar.gz

需要的文件结构图
![](/uploads/Opus交叉编译/Opus目录结构.jpg)

### 目标
> 目标是我们要编译出opus-tools里面的可执行程序，然后放到手机里面跑，看是否能够正常工作,在此之前，我们可以先看下官网提供的opus-tools可执行的程序
这个是官网提供的最新版的 Window下的可执行程序下载地址  https://archive.mozilla.org/pub/opus/win64/opus-tools-0.1.10-win64.zip
下面了解他们是干嘛用的，提供了什么功能

压缩包文件内容
![](/uploads/Opus交叉编译/官网提供opustools工具.png)

opusdec.exe是用来将一个opus文件还原成一个wav文件
![](/uploads/Opus交叉编译/opusdec使用.jpg)

opusenc.exe 是用来将一个wav文件，压缩成一个opus格式的文件 
![](/uploads/Opus交叉编译/opusenc工具使用.png)

opusinfo.exe 可以用来显示一个opus文件的信息
![](/uploads/Opus交叉编译/opusinfo工具使用.png)

### 交叉编译实现

#### 构建交叉编译链

使用交叉编译链比较方便,下面是构建的脚本

```bash
#!/bin/bash

export ANDROID_HOME=/home/yuhui/WorkSpace/android-ndk-r14b
 
$ANDROID_HOME/build/tools/make_standalone_toolchain.py \
   --arch arm --api 16 \
   --install-dir $ANDROID_HOME/toolchain

cd $ANDROID_HOME

mkdir -p usr/local/lib/pkgconfig
```

上述脚本执行之后的结果
![](/uploads/Opus交叉编译/交叉编译链执行结果.png)

#### 编译  libogg-1.3.3

下面是脚本文件,这里构建一个静态库

```bash
#!/bin/bash
ANDROID_HOME=/home/yuhui/WorkSpace/android-ndk-r14b

TOOLCHAIN=$ANDROID_HOME/toolchain
PATH=$TOOLCHAIN/bin:$PATH
HOST=arm-linux-androideabi
PREFIX=$ANDROID_HOME/usr/local
LOCAL_DIR=$ANDROID_HOME/usr/local
TOOL_BIN_DIR=$ANDROID_HOME/toolchain/bin
PATH=${TOOL_BIN_DIR}:$PATH
DEST=$ANDROID_HOME/usr/local
#-march=CPU  按照特定的CPU进行优化
CC="$HOST-gcc -march=armv7-a"
CXX=$HOST-g++
#LDFLAGS="-L$DEST/lib 指定要连接库的时候，寻找的地方
LDFLAGS="-L$DEST/lib -march=armv7-a"
#CPPFLAGS="-I$DEST/include 指定头文件寻找的地方
CPPFLAGS="-I$DEST/include -march=armv7-a"
CXXFLAGS=$CFLAGS

# c-ares library build
  PKG_CONFIG_PATH=$PREFIX/lib/pkgconfig/ LD_LIBRARY_PATH=$PREFIX/lib/ ./configure --host=$HOST --build=`dpkg-architecture -qDEB_BUILD_GNU_TYPE` 
--prefix=$PREFIX  --enable-static --disable-shared
  make
  make install

echo "build ogg complete!"
```
编译的结果为红色部分指定的
![](/uploads/Opus交叉编译/ogg编译结果.jpg)

#### 编译  flac-1.3.2,

下面是脚本文件,这里构建一个静态库

```bash
#!/bin/bash

ANDROID_HOME=/home/yuhui/WorkSpace/android-ndk-r14b
TOOLCHAIN=$ANDROID_HOME/toolchain
PATH=$TOOLCHAIN/bin:$PATH
HOST=arm-linux-androideabi
PREFIX=$ANDROID_HOME/usr/local
LOCAL_DIR=$ANDROID_HOME/usr/local
TOOL_BIN_DIR=$ANDROID_HOME/toolchain/bin
PATH=${TOOL_BIN_DIR}:$PATH
DEST=$ANDROID_HOME/usr/local
CC="$HOST-gcc -march=armv7-a"
CXX=$HOST-g++
LDFLAGS="-L$DEST/lib -march=armv7-a"
CPPFLAGS="-I$DEST/include -march=armv7-a"
CXXFLAGS=$CFLAGS

# c-ares library build
  PKG_CONFIG_PATH=$PREFIX/lib/pkgconfig/ LD_LIBRARY_PATH=$PREFIX/lib/ ./configure --host=$HOST --build=`dpkg-architecture -qDEB_BUILD_GNU_TYPE` 
--prefix=$PREFIX  --enable-static --disable-shared
  make
  make install

echo "build flac complete!"
```
这里要知道的是编译flac库需要使用到ogg库，所以ogg库放在第一个编译，而且我们的 LDFLAGS="-L$DEST/lib  CPPFLAGS="-I$DEST/include 指定了链接查找的地方，所以可以他可以自动的查找到
编译的结果为红色部分指定的
![](/uploads/Opus交叉编译/flac编译结果.jpg)

#### 编译  libopus 1.2.1

下面是脚本文件,这里构建一个静态库
```bash
#!/bin/bash
ANDROID_HOME=/home/yuhui/WorkSpace/android-ndk-r14b

TOOLCHAIN=$ANDROID_HOME/toolchain
PATH=$TOOLCHAIN/bin:$PATH
HOST=arm-linux-androideabi
PREFIX=$ANDROID_HOME/usr/local
LOCAL_DIR=$ANDROID_HOME/usr/local
TOOL_BIN_DIR=$ANDROID_HOME/toolchain/bin
PATH=${TOOL_BIN_DIR}:$PATH
DEST=$ANDROID_HOME/usr/local
CC="$HOST-gcc -march=armv7-a"
CXX=$HOST-g++
LDFLAGS="-L$DEST/lib -march=armv7-a"
CPPFLAGS="-I$DEST/include -march=armv7-a"
CXXFLAGS=$CFLAGS

  PKG_CONFIG_PATH=$PREFIX/lib/pkgconfig/ LD_LIBRARY_PATH=$PREFIX/lib/ ./configure --host=$HOST --build=`dpkg-architecture -qDEB_BUILD_GNU_TYPE` 
--prefix=$PREFIX
  make
  make install

```
编译的结果为红色部分指定的
![](/uploads/Opus交叉编译/opus编译结果.png)

#### 编译 openssl-1.0.2m

下面是脚本文件

```bash
 #!/bin/bash
ANDROID_HOME=/home/yuhui/WorkSpace/android-ndk-r14b

TOOLCHAIN=$ANDROID_HOME/toolchain
PATH=$TOOLCHAIN/bin:$PATH
HOST=arm-linux-androideabi
PREFIX=$ANDROID_HOME/usr/local
LOCAL_DIR=$ANDROID_HOME/usr/local
TOOL_BIN_DIR=$ANDROID_HOME/toolchain/bin
PATH=${TOOL_BIN_DIR}:$PATH
DEST=$ANDROID_HOME/usr/local
CC="$HOST-gcc -march=armv7-a"
CXX=$HOST-g++
LDFLAGS="-L$DEST/lib -march=armv7-a"
CPPFLAGS="-I$DEST/include -march=armv7-a"
CXXFLAGS=$CFLAGS

  PKG_CONFIG_PATH=$PREFIX/lib/pkgconfig/ LD_LIBRARY_PATH=$PREFIX/lib/ export CROSS_COMPILE=$TOOLCHAIN/bin/arm-linux-androideabi-
./Configure --prefix=$PREFIX android
  make
  make install
```
编译的结果为红色部分指定的
![](/uploads/Opus交叉编译/openssl编译结果.jpg)

#### 编译 opusfile 0.9 

下面是脚本文件

```bash
#!/bin/bash
ANDROID_HOME=/home/yuhui/WorkSpace/android-ndk-r14b
TOOLCHAIN=$ANDROID_HOME/toolchain
PATH=$TOOLCHAIN/bin:$PATH
HOST=arm-linux-androideabi
PREFIX=$ANDROID_HOME/usr/local
LOCAL_DIR=$ANDROID_HOME/usr/local
TOOL_BIN_DIR=$ANDROID_HOME/toolchain/bin
PATH=${TOOL_BIN_DIR}:$PATH
DEST=$ANDROID_HOME/usr/local
CC="$HOST-gcc -march=armv7-a"
CXX=$HOST-g++
LDFLAGS="-L$DEST/lib -march=armv7-a"
CPPFLAGS="-I$DEST/include -march=armv7-a"
CXXFLAGS=$CFLAGS

  PKG_CONFIG_PATH=$PREFIX/lib/pkgconfig/ LD_LIBRARY_PATH=$PREFIX/lib/ ./configure --host=$HOST --build=`dpkg-architecture -qDEB_BUILD_GNU_TYPE` --prefix=$PREFIX
  make
  make install
```
编译的结果为红色部分指定的
![](/uploads/Opus交叉编译/opusfile编译结果.png)

#### 编译 opus-tools 0.1.10 

下面是脚本文件

```bash

#!/bin/bash
ANDROID_HOME=/home/yuhui/WorkSpace/android-ndk-r14b
TOOLCHAIN=$ANDROID_HOME/toolchain
PATH=$TOOLCHAIN/bin:$PATH
HOST=arm-linux-androideabi
PREFIX=$ANDROID_HOME/usr/local
LOCAL_DIR=$ANDROID_HOME/usr/local
TOOL_BIN_DIR=$ANDROID_HOME/toolchain/bin
PATH=${TOOL_BIN_DIR}:$PATH
DEST=$ANDROID_HOME/usr/local
CC="$HOST-gcc -march=armv7-a"
CXX=$HOST-g++
LDFLAGS="-L$DEST/lib -march=armv7-a"
CPPFLAGS="-I$DEST/include -march=armv7-a"
CXXFLAGS=$CFLAGS

PKG_CONFIG_PATH=$PREFIX/lib/pkgconfig/ LD_LIBRARY_PATH=$PREFIX/lib/ ./configure --host=$HOST --build=`dpkg-architecture -qDEB_BUILD_GNU_TYPE` --prefix=$PREFIX
  make
  make install
```
要注意的是这个opus-tools 需要依赖opus库，ogg库，flac库 编译的结果为红色部分指定的
![](/uploads/Opus交叉编译/opustools编译结果.png)

### 验证编译的结果
> 我们可以将编译之后产生的文件拷贝到手机里面，手机里面有一个目录可以不用root进入的，就是/data/local/tmp目录，这个目录是所有的手机都是可以访问的，所以选择放在这个目录
然后进入使用adb shell 进入命令行模式

push到手机的目录以及内容，这里是将整个local的内容push到手机上
![](/uploads/Opus交叉编译/localpush到手机.png)

opusinfo使用结果
![](/uploads/Opus交叉编译/opusinfo手机调用.png)

opusdec使用结果
![](/uploads/Opus交叉编译/opusdec手机实验.jpg)

opusdec使用结果
![](/uploads/Opus交叉编译/opusenc手机实验.jpg)

将压缩之后，还有解压缩生成的文件，可以直接使用音乐播放器播放，qq音乐也是支持直接播放opus文件的，还有google浏览器也是支持的，我们可以根据这个来判断是否生成有问题
![](/uploads/Opus交叉编译/opus文件google浏览器运行.png)

### 总结
> 至此，Opus交叉编译已经完成，接下来一篇文章会介绍怎么集成到Android Studio中，而且实现边录音边压缩，边解压缩边播放
