---
layout: pager
title: Aria2介绍 与 Linux下编译 Window,Linux,Android 产物
date: 2018-04-10 10:28:07
tags: [Aria2,Linux,Android,Window]
description: Aria2介绍 与 Linux下编译Window,Linux,Android 产物
---

Aria2介绍 与 Linux 下编译 Window ，Linux，Android 产物
<!--more-->

```简介
今年三月底的时候，收到了老大的一个任务说去研究一下p2p的下载，对于我们这种屌丝来说，当然是老大说什么就是什么了，俗话说拿人钱财替人消灾

首先我们要先去了解这个东西，什么是p2p下载，在搞这个之前，确实不了解，这个p2p技术早就已经存在了，所以关于他的内容也是非常的多
然后去github上面查找对应的开源项目，最终定位到了Aria2 这个开源项目，对于为什么要选择他，因为他是一个命令行的工具，而且好像还支持android平台,下面是官网的介绍

aria2是一款用于下载文件的工具。支持的协议是HTTP（S），FTP，SFTP，BitTorrent和Metalink。aria2可以从多种来源/协议下载文件，并尝试利用您的最大下载带宽。
它支持从HTTP（S）/ FTP / SFTP和BitTorrent同时下载文件，而从HTTP（S）/ FTP / SFTP下载的数据则上传到BitTorrent群。使用Metalink块校验和，aria2在下载文件时自动验证数据块。
```

Linux平台编译
```java
编译的系统为Ubuntu16.04 号称是最稳定的版本

1.
为了从源码包构建aria2，您需要以下开发包（包名可能会因您使用的发行版而异）：
  ● libgnutls-dev（需要HTTPS，BitTorrent，Checksum支持）
  ● nettle-dev（需要BitTorrent，支持Checksum）
  ● libgmp-dev（BitTorrent必需）
  ● libssh2-1-dev（SFTP支持必需）
  ● libc-ares-dev（异步DNS支持需要）
  ● libxml2-dev（Metalink支持必需）
  ● zlib1g-dev（gzip需要，HTTP解压解码支持）
  ● libsqlite3-dev（需要Firefox3 / Chromium cookie支持）
  ● pkg-config（需要检测已安装的库）
你可以使用libgcrypt-dev而不是nettle-dev和libgmp-dev：
您可以使用libssl-dev而不是libgnutls-dev，nettle-dev，libgmp-dev，libgpg-error-dev和libgcrypt-dev：
您可以使用libexpat1-dev而不是libxml2-dev：
如果您从git存储库下载源代码，则必须安装以下软件包以获取autoconf宏：
  ● libxml2-dev
  ● libcppunit-dev
  ● autoconf
  ● automake
  ● autotools-dev
  ● autopoint
  ● libtool

2.
上面需要的包安装完之后，进入到下载下来的源码目录里面这里使用的是最新版Github上面的Maste分支的内容

然后运行以下命令生成构建程序所需的配置脚本和其他文件：
$ autoreconf -i

您还需要Sphinx来构建手册页。
如果您正在为Mac OS X构建aria2，请查看make-release-os.mk GNU Make makefile。
构建aria2的最快方法是首先运行configure脚本：
$ ./configure

要建立静态链接的aria2，请使用ARIA2_STATIC=yes 命令行选项：
$ ./configure ARIA2_STATIC = yes

配置完成后，运行make以编译该程序：
$ make
```

编译完成就会在源码目录下面的src里面有一个可以执行的文件，名为aria2c
![结果显示](/uploads/Aria2编译/linuxAria.png)

测试可以下载
![结果显示](/uploads/Aria2编译/linux运行.png)


Window产物
```java 
编译的系统为Ubuntu16.04 号称是最稳定的版本

编译
  ● Aria2 需要用到 mingw-w64 的交叉编译工具链来编译出来 Windows 下的二进制，所以要首先安装一个 Linux 系统，推荐用 Ubuntu
  ● 新建一个 setup.sh 脚本，来执行环境的安装和相关依赖库的编译
  ● #! /bin/bash

# 改成 x86_64-w64-mingw32 来编译64位版本 这个要对应的为你在Linux上面安装的Mingw64对应的 是64还是32位系统
#安装mingW64可以通过 sudo apt-get install mingw-w64-x86-64-dev 来完成安装
export HOST=x86_64-w64-mingw32

# It would be better to use nearest ubuntu archive mirror for faster
# downloads.
# RUN sed -ie 's/archive\.ubuntu/jp.archive.ubuntu/g' /etc/apt/sources.list

# 安装编译环境 后面的其实就是将官网上面介绍的那样，要安装的软件包，这里只不过是讲这些要安装的软件包通过脚本文件的形势来编译，这样的好处是不用
#一个一个的执行，可以通过脚本文件，一下子执行完成

apt-get update && \
apt-get install -y make binutils autoconf automake autotools-dev libtool pkg-config git curl dpkg-dev gcc-mingw-w64 autopoint libcppunit-dev libxml2-dev libgcrypt11-dev lzip

# 下载依赖库
if [ ! -f "gmp-6.1.2.tar.lz" ]; then 
	curl -L -O https://gmplib.org/download/gmp/gmp-6.1.2.tar.lz 
fi

if [ ! -f "expat-2.2.0.tar.bz2" ]; then 
	curl -L -O http://downloads.sourceforge.net/project/expat/expat/2.2.0/expat-2.2.0.tar.bz2
fi

if [ ! -f "sqlite-autoconf-3160200.tar.gz" ]; then 
	curl -L -O https://www.sqlite.org/2017/sqlite-autoconf-3160200.tar.gz
fi

if [ ! -f "zlib-1.2.11.tar.gz" ]; then 
	curl -L -O http://zlib.net/zlib-1.2.11.tar.gz
fi

if [ ! -f "c-ares-1.12.0.tar.gz" ]; then 
	curl -L -O https://c-ares.haxx.se/download/c-ares-1.12.0.tar.gz
fi

if [ ! -f "libssh2-1.8.0.tar.gz" ]; then 
	curl -L -O http://libssh2.org/download/libssh2-1.8.0.tar.gz
fi

# 动态编译 gmp
tar xf gmp-6.1.2.tar.lz && \
cd gmp-6.1.2 && \
./configure --enable-shared --disable-static --prefix=/usr/local/$HOST --host=$HOST --disable-cxx --enable-fat CFLAGS="-mtune=generic -O2 -g0" && \
make install

# 动态编译 expat
cd ..
tar xf expat-2.2.0.tar.bz2 && \
cd expat-2.2.0 && \
./configure --enable-shared --disable-static --prefix=/usr/local/$HOST --host=$HOST --build=`dpkg-architecture -qDEB_BUILD_GNU_TYPE` && \
make install

# 动态编译 sqlite3
cd ..
tar xf sqlite-autoconf-3160200.tar.gz && cd sqlite-autoconf-3160200 && \
./configure --enable-shared --disable-static --prefix=/usr/local/$HOST --host=$HOST --build=`dpkg-architecture -qDEB_BUILD_GNU_TYPE` && \
make install

# 动态编译 zlib
cd ..
tar xf zlib-1.2.11.tar.gz && \
cd zlib-1.2.11
export BINARY_PATH=/usr/local/$HOST/bin
export INCLUDE_PATH=/usr/local/$HOST/include
export LIBRARY_PATH=/usr/local/$HOST/lib
make install -f win32/Makefile.gcc PREFIX=$HOST- SHARED_MODE=1

# 动态编译 c-ares
cd ..
tar xf c-ares-1.12.0.tar.gz && \
cd c-ares-1.12.0 && \
./configure --enable-shared --disable-static --without-random --prefix=/usr/local/$HOST --host=$HOST --build=`dpkg-architecture -qDEB_BUILD_GNU_TYPE` LIBS="-lws2_32" && \
make install

# 动态编译 libssh2
cd ..
tar xf libssh2-1.8.0.tar.gz && \
cd libssh2-1.8.0 && \
./configure --enable-shared --disable-static --prefix=/usr/local/$HOST --host=$HOST --build=`dpkg-architecture -qDEB_BUILD_GNU_TYPE` --without-openssl --with-wincng LIBS="-lws2_32" && \
make install

下载 Aria2 代码并解压，修改 aria2\mingw-config 文件，增加 --enable-libaria2 编译选项来生成 libaria2 库，并修改ARIA2_STATIC=no 来关闭静态编译，如果想在 windows xp 上运行，则还要将 --with-libssh2 改成 --without-libssh2，否则会出现 bcrypt.dll 缺失报错，最后的修改如下:

test -z "$HOST" && HOST=i686-w64-mingw32
test -z "$PREFIX" && PREFIX=/usr/local/$HOST
./configure \
    --host=$HOST \
    --prefix=$PREFIX \
    --without-included-gettext \
    --disable-nls \
    --with-libcares \
    --without-gnutls \
    --without-openssl \
    --with-sqlite3 \
    --without-libxml2 \
    --with-libexpat \
    --with-libz \
    --with-libgmp \
    --without-libssh2 \
    --without-libgcrypt \
    --without-libnettle \
    --with-cppunit-prefix=$PREFIX \
    --enable-libaria2 \
    ARIA2_STATIC=no \
    CPPFLAGS="-I$PREFIX/include" \
    LDFLAGS="-L$PREFIX/lib" \
    PKG_CONFIG_PATH="$PREFIX/lib/pkgconfig"



3.新建一个 build.sh 来编译 Aria2 库

#! /bin/bash
# 改成 x86_64-w64-mingw32 来编译64位版本
export HOST=x86_64-w64-mingw32
cd aria2 && autoreconf -i && ./mingw-config && make install && $HOST-strip /usr/local/$HOST/bin/libaria2-0.dll

编译完成后，将两个 gcc 的运行库复制到 /usr/local/x86_64-w64-mingw32/bin 中，
路径为 /usr/lib/gcc/x86_64-w64-mingw32/5.3-win32/libgcc_s_seh-1.dll 和 /usr/lib/gcc/x86_64-w64-mingw32/5.3-win32/libstdc++-6.dll
```

把 /usr/local/x86_64-w64-mingw32 文件夹复制到 Windows 中，这个文件夹中的文件就是 Windows 所需要的二进制,这个就是最后到Window中bin目录下的内容
![结果显示](/uploads/Aria2编译/window编译.png)

Android编译
```java
编译的系统为Ubuntu16.04 号称是最稳定的版本


下面是build-libraries.sh的脚本，用来保证编译android的时候，需要的库等
#!/bin/bash

if [[ $ANDROID_HOME == "" ]];then
echo "ANDROID_HOME not defined"
exit 0
fi

# 安装编译环境
apt-get update && \
apt-get install -y make binutils autoconf automake autotools-dev libtool pkg-config git curl dpkg-dev autopoint libcppunit-dev libxml2-dev libgcrypt11-dev

##LIBRARIES_DOWNLOAD_URL##
EXPAT=https://github.com/libexpat/libexpat/releases/download/R_2_2_5/expat-2.2.5.tar.bz2
OPENSSL=https://www.openssl.org/source/openssl-1.0.2o.tar.gz
C_ARES=https://c-ares.haxx.se/download/c-ares-1.12.0.tar.gz
SSH2=http://libssh2.org/download/libssh2-1.8.0.tar.gz
ZLIB=http://zlib.net/zlib-1.2.11.tar.gz


DOWNLOADER="wget -c"
TOOLCHAIN=$ANDROID_HOME/toolchain
PATH=$TOOLCHAIN/bin:$PATH
HOST=arm-linux-androideabi
PREFIX=$ANDROID_HOME/usr/local
LOCAL_DIR=$ANDROID_HOME/usr/local
TOOL_BIN_DIR=$ANDROID_HOME/toolchain/bin
PATH=${TOOL_BIN_DIR}:$PATH
CFLAGS="-march=armv7-a -mtune=cortex-a9"
DEST=$ANDROID_HOME/usr/local
CC=$HOST-gcc
CXX=$HOST-g++
LDFLAGS="-L$DEST/lib"
CPPFLAGS="-I$DEST/include"
CXXFLAGS=$CFLAGS

 cd /tmp
 # zlib library build
  $DOWNLOADER $ZLIB
  tar zxvf zlib-1.2.11.tar.gz
  cd zlib-1.2.11/
  PKG_CONFIG_PATH=$PREFIX/lib/pkgconfig/ LD_LIBRARY_PATH=$PREFIX/lib/ CC=$HOST-gcc STRIP=$HOST-strip RANLIB=$HOST-ranlib CXX=$HOST-g++ AR=$HOST-ar LD=$HOST-ld ./configure --prefix=$PREFIX --static
  make
  make install

 # expat library build
  cd /tmp
  $DOWNLOADER $EXPAT
  tar xf expat-2.2.5.tar.bz2
  cd expat-2.2.5/
  PKG_CONFIG_PATH=$PREFIX/lib/pkgconfig/ LD_LIBRARY_PATH=$PREFIX/lib/ CC=$HOST-gcc CXX=$HOST-g++ ./configure --host=$HOST --build=`dpkg-architecture -qDEB_BUILD_GNU_TYPE` --prefix=$PREFIX --enable-static=yes --enable-shared=no
  make
  make install

 # c-ares library build
  cd /tmp
  $DOWNLOADER $C_ARES
  tar xf c-ares-1.12.0.tar.gz
  cd c-ares-1.12.0/
  PKG_CONFIG_PATH=$PREFIX/lib/pkgconfig/ LD_LIBRARY_PATH=$PREFIX/lib/ CC=$HOST-gcc CXX=$HOST-g++ ./configure --host=$HOST --build=`dpkg-architecture -qDEB_BUILD_GNU_TYPE` --prefix=$PREFIX --enable-static --disable-shared
  make
  make install

 # openssl library build
  cd /tmp
  $DOWNLOADER $OPENSSL
  tar xf openssl-1.0.2o.tar.gz
  cd openssl-1.0.2o/
  PKG_CONFIG_PATH=$PREFIX/lib/pkgconfig/ LD_LIBRARY_PATH=$PREFIX/lib/ CC=$HOST-gcc CXX=$HOST-g++ ./Configure linux-armv4 $CFLAGS --prefix=$PREFIX shared zlib zlib-dynamic -D_GNU_SOURCE -D_BSD_SOURCE --with-zlib-lib=$LOCAL_DIR/lib --with-zlib-include=$LOCAL_DIR/include
  make 
  make install

 # libssh2 library build
  cd /tmp
  $DOWNLOADER $SSH2
  tar xf libssh2-1.8.0.tar.gz
  cd libssh2-1.8.0/
  rm -rf $PREFIX/lib/pkgconfig/libssh2.pc
  PKG_CONFIG_PATH=$PREFIX/lib/pkgconfig/ LD_LIBRARY_PATH=$PREFIX/lib/ CC=$HOST-gcc CXX=$HOST-g++ AR=$HOST-ar RANLIB=$HOST-ranlib ./configure --host=$HOST --without-libgcrypt --with-openssl --without-wincng --prefix=$PREFIX --enable-static --disable-shared
  make
  make install
  
  cd /tmp
  rm -rf c-ares*
  rm -rf sqlite-autoconf*
  rm -rf zlib-*
  rm -rf expat-*
  rm -rf openssl-*
  rm -rf libssh2-*
 echo "all complete!"


下面是build-ndk-toolchain.sh的脚本，用来保证ndk的环境，主要官网介绍最好用ndk-41b以上的版本
#!/bin/bash

mkdir -p /opt/NDK
cd /opt/NDK
apt install wget
wget https://dl.google.com/android/repository/android-ndk-r14b-linux-x86_64.zip
apt install unzip
unzip android-ndk-r14b-linux-x86_64.zip
rm android-ndk-r14b-linux-x86_64.zip

echo "export NDK=/opt/NDK/android-ndk-r14b
export ANDROID_HOME=/opt/NDK/android-ndk-r14b
export PATH=$ANDROID_HOME:$PATH" >> /etc/profile
source /etc/profile


if [[ $ANDROID_HOME == "" ]];then
 echo "ANDROID_HOME not defined"
 exit 0
 fi
 
$ANDROID_HOME/build/tools/make_standalone_toolchain.py \
   --arch arm --api 16 --stl=gnustl \
   --install-dir $ANDROID_HOME/toolchain


cd $ANDROID_HOME

mkdir -p usr/local/lib/pkgconfig

在root的权限下面，执行 ./build-ndk-toolchain.sh 没有错误之后 执行 ./build-libraries.sh 如果有发生了错误，要修正这些错误，比如资源包查找不到的化，自行修改。找到一个可以下载的地址，替换上去
既可

然后进入aria2的源码文件里面 执行./android-config 用来确定当前的配置，这个android-config文件跟源码提供的一样，没有做任何的修改。。然后执行./android-make 如果没有错误的化，就会编译完成
```

最后的编译产物为
![结果显示](/uploads/Aria2编译/android编译.png)
