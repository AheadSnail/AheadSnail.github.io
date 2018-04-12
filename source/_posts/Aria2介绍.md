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

在上面可以发现我们编译出来的内容达到了70多M，而官网提供的一个zip包里面，解压缩也就4M，然来发现我们少执行了一个脚本android-release (个人认为这是官网的一个缺失，应该告知)
```
android-relese 里面会执行一些优化，其中最重要的就是调用了linux下面的strip工具
下面是这个工具的具体介绍：
strip工具
strip命令减少XCOFF对象文件的大小。strip命令从XCOFF对象文件中有选择地除去行号
信息、重定位信息、调试段、typchk 段、注释段、文件头以及所有或部分符号表。一旦您
使用该命令，则很难调试文件的符号；因此，通常应该只在已经调试和测试过的生成模块上
使用 strip  命令。使用 strip  命令减少对象文件所需的存储量开销。

    对于每个对象模块，strip 命令除去给出的选项所指定的信息。对于每个归档文件，
strip 命令从归档中除去全局符号表。可以使用 ar -s 命令将除去的符号表恢复到归档文件
或库文件中。

    没有选项的 strip 命令除去行号信息、重定位信息、符号表、调试段、typchk 段和注
释段。
    -e      在对象文件的可选头中设置F_LOADONLY 标志。如果对象文件放置在归档中，则该
        标志告知绑定程序（ld  命令），在与此归档链接时应忽略该对象文件中的符号。
    -E     复位（关闭）对象文件的可选头中的 F_LOADONLY 位。（请参阅 -e 标志。）
    -H     除去对象文件头、任何可选的头以及所有段的头部分。

    注： 不除去符号表信息。

    -l     （小写 L）从对象文件中除去行号信息。
    -r     除了外部符号和静态符号条目，将全部符号表信息除去。不除去重定位信息。同时
        除去调试段和 typchk 段。这个选项产生一个对象文件，该对象文件仍可以用作输
        入到链接编辑器（ld 命令）中。
    -t     除去大多数符号表信息，但并不除去函数符号或行号信息。
    -V     打印 strip 命令的版本号。
    -x     除去符号表信息，但并不除去静态或外部符号信息。 -x 标志同时除去重定位信息，
        因此将不可能链接到该文件。
    -X mode     指定应检查 strip 的对象文件的类型。 mode 必须是下列之一：
        32
            只处理 32 位对象文件 
        64
            只处理 64 位对象文件 
        32_64
            既处理 32 位对象文件，又处理 64 位对象文件
```

在通过android-relase命令之后，就会生成一个对应版本的zip文件，
![结果显示](/uploads/Aria2编译/android-release.png)

打开这个压缩文件，查看文件的大小，下面显示就跟官网提供的是差不多的。所以这就可以证明了我们的编译android是没有任务的问题的，这个是我们继续往下研究的一个前提(重点)
![结果显示](/uploads/Aria2编译/压缩之后aria2c.png)

验证这个编译的产物是否可以运行（Linux 下面运行这个可执行文件）
```
将这个可执行的文件，通过adb shell命令传输动 android 手机上面，注意我们上面编译的是一个arm版本的，如果放到64位的上面是不能运行的，运行的时候会出现
not found/not executable:32-bit ELF file

对于有root的权限的手机，可以随便放到sdk的下面，如果没有权限的用户，可以放到 /data/local/tmp的目录下面，这是一个linux系统的临时目录，是不用权限的

验证android的 编译的是否是可以行的。在命令下面，进入到这个/data/local/tmp 目录下面，前提是你已经把aria2跟要下载的种子文件一起放到里面
然后执行命令./aria2c 1.toerrate 就会显示结果
```

结果显示
![结果显示](/uploads/Aria2编译/aria2Android运行.png)

