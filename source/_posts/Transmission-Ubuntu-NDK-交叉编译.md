---
layout: pager
title: Transmission Ubuntu NDK 交叉编译
date: 2018-06-27 17:40:19
tags: [Android,NDK,Transmission]
description:  Transmission Ubuntu NDK 交叉编译
---

Transmission Ubuntu NDK 交叉编译
<!--more-->

****什么是Transmission？****
===
```
Transmission是一种BitTorrent客户端，特点是一个跨平台的后端和其上的简洁的用户界面。Transmission以MIT许可证和GNU通用公共许可证双许可证授权，因此是一款自由软件
优点是
开源跨平台，由社区志愿者开发
绝无各种广告及浏览器工具栏插件等
完全免费，绝无收费高级版与免费基础版等区别
数据加密、损坏修复
来源交换 (支持Bittorrent、Ares、迅雷、Vuze和μTorrent等等)
硬件资源消耗极低，甚至比某些命令行BT工具都要低
可以选择种子中要下载的文件
支持encryption、web界面、远程控制、磁力链接、DHT、uTP、uPnP、NAT-PMP
支持目录监控、全局或单一速度限制
制作种子、快速继续
黑名单，可以按时升级(资料来自PeerGuardian和PeerBlock)
单一监听端口、带宽计划、整理(过滤)
HTTPS tracker支持以及tracker编辑功能支持
IPv6支持
对应不同平台有着特定的图形用户界面。
```

****为什么要研究Transmission？****
===
```
为什么要研究这个东西，是因为之前搞的Aria2 p2p下载，后面在测试的时候发现了一个很重大的问题，就是内网之间不支持连接，这样内网之间的用户也就无法贡献速度，原来Aria2采用的是
TCP连接，而Transmission 支持utp的协议，也即是 udp的一种，udp是支持在内网连接的，下面是这俩中穿透的原理

至于为什么要用UDP，这是由于UDP的某些特性。UDP通信需要先绑定本地机器的端口，完成后就可以从这个端口收发数据，至于从哪里收，发到哪里，可以在收发数据的时候再决定，
这也就意味着我可以用这一个端口同时和多个对象通信，只要我收发数据的时候指定不同的对象即可。当本地机器用UDP向英特网上的某个服务器发送数据的时候，
这个映射不但能用来和这个服务器进行数据交互，也能用来接收其他主机发来的数据。NAT穿透就是本地主机向公网上的某台服务器发送数据，这时服务器就可以获得NAT对这台主机的映射，
在之前举得例子中就是55.66.77.88:5000这个地址。由于NAT会将55.66.77.88:5000收到的数据转发至本地主机，所以公网上的其他机器可以从服务器获取到55.66.77.88:5000这个网络地址，
然后通过这个网络地址向私网下的机器发出数据。

而至于为什么不用TCP，也是由于TCP的某些特性。TCP通信的步骤与UDP不同，它需要先在两个对象之间建立一个专用通道，再用这个通道收发数据。也就是说外人无法插手。
这样一来，虽然其他机器可以通过服务器获取到NAT的映射对象，也没办法利用它向私网下的机器发出数据。
```

****Transmission ubuntu 编译****
===
官网 https://transmissionbt.com/
transmission编译 https://github.com/transmission/transmission/wiki/Building-Transmission
```
1.首先在ubuntu 执行下面的命令，用来安装一些依赖 
sudo apt-get install build-essential automake autoconf libtool pkg-config intltool libcurl4-openssl-dev libglib2.0-dev libevent-dev libminiupnpc-dev libgtk-3-dev libappindicator3-dev

2.下载安装libevent库,当然你也可以下载最新的libevent库
cd /var/tmp
$ wget https://github.com/downloads/libevent/libevent/libevent-2.0.18-stable.tar.gz
$ tar xzf libevent-2.0.18-stable.tar.gz
$ cd libevent-2.0.18-stable
$ CFLAGS="-Os -march=native" ./configure && make

3. 下载安装Transmission，当然你可以下载Tramsmission最新的版本
cd /var/tmp
$ wget http://download-origin.transmissionbt.com/files/transmission-2.51.tar.bz2
$ tar xjf transmission-2.51.tar.bz2
$ cd transmission-2.51
CFLAGS="-Os -march=native" ./configure && make && checkinstall
```
如果正常没有错误的化，是这样的
![结果显示](/uploads/Transmision 交叉编译/ubuntu编译.png)

生成的结果：
![结果显示](/uploads/Transmision 交叉编译/ubuntu生成的文件.png)

****Transmission ubuntu NDK 交叉编译****
===
```
如果采用交叉编译的化，就要自己生成这些依赖库，这里需要手动的编译的库文件有libcurl ,libevent openssl,所以要提前下载好这些文件，对应的我们编译都是采用最新版本的库
```
libevent 最新库的下载地址  https://github.com/libevent/libevent/releases/download/release-2.1.8-stable/libevent-2.1.8-stable.tar.gz
openssl 最新库的下载地址  https://www.openssl.org/source/openssl-1.0.2o.tar.gz
curl  最新库的下载地址    https://curl.haxx.se/download/curl-7.60.0.tar.gz
transmission 最新的下载地址  https://raw.githubusercontent.com/transmission/transmission-releases/master/transmission-2.94.tar.xz

编译openssl
```java
下面是编译的脚本文件

#!/bin/sh
OPENSSL_SOURCE="openssl-1.0.2m"
FAT="TS-Android"

SCRATCH="scratch"
# must be an absolute path
THIN=`pwd`/"thin"
NDK="/home/yuhui/workSpace/android-ndk-r10e"
SYSROOT_PREFIX="$NDK/platforms/android-16/arch-"

build_openssl() {
    ARCH=$1
    PLATFROM=$2
    if [ $ARCH = "arm64" ]
    then
        ACT_ARCH="aarch64"
    else
        ACT_ARCH=$ARCH
    fi
    if [ -z $PLATFROM ]
    then
        PLATFROM=linux-androideabi
    fi
    SYSROOT=${SYSROOT_PREFIX}${ARCH}
    export CC="$NDK/toolchains/${ACT_ARCH}-${PLATFROM}-4.9/prebuilt/linux-x86_64/bin/${ACT_ARCH}-${PLATFROM}-gcc --sysroot=${SYSROOT}"
    CWD=`pwd`

    echo "building openssl for $ARCH..."
    #mkdir -p "$SCRATCH/$ARCH"
    cd "$CWD/$OPENSSL_SOURCE"
    export CFLGAS="-I$SYSROOT/usr/include -I$CWD/$OPENSSL_SOURCE"
    export LDFLAGS="-L$SYSROOT/usr/lib"
    $CWD/$OPENSSL_SOURCE/Configure android \
        --prefix=$THIN/$ARCH #\ 
    #    --cross-compile-prefix=${ACT_ARCH}-${PLATFROM}
    make install
}

build_openssl "arm"
echo "build openssl success"
exit 1


这里需要注意的是,如果GCC 不是这样指定的化，会出现 error: C compiler cannot create executables的错误 ,sysroot 也即是指定环境的意思
export CC="$NDK/toolchains/${ACT_ARCH}-${PLATFROM}-4.9/prebuilt/linux-x86_64/bin/${ACT_ARCH}-${PLATFROM}-gcc --sysroot=${SYSROOT}"

下面是介绍为什么要这样写的原因 
这个一般是跟ndk相关的错误,某些头文件或者obj文件找不到。可以编写个简单的hello world源文件测试

#include 
int main() { 
printf("Hello world."); 
return 0; 
} 
使用ndk中的编译器进行编译,如

export $ANDROID_NDK=/home/longjing/tools/Android/android-ndk-r15c 
$ANDROID_NDK/toolchains/arm-linux-androideabi-4.9/prebuilt/linux-x86_64/bin/arm-linux-androideabi-gcc test.c 
编译果然报错,找不到头文件

../test.c:16:19: fatal error: stdio.h: No such file or directory 
#include 
^ 
compilation terminated. 
既然是测试文件,那干脆就去掉include语句,再删除printf语句,重新编译看下结果

//#include 
int main() { 
//printf("Hello world."); 
return 0; 
} 
这下就告诉你obj文件找不到了

/home/longjing/tools/Android/android-ndk-r15c/toolchains/arm-linux-androideabi-4.9/prebuilt/linux-x86_64/bin/../lib/gcc/arm-linux-androideabi/4.9.x/../../../../arm-linux-androideabi/bin/ld: error: cannot open crtbegin_dynamic.o: No such file or directory 
/home/longjing/tools/Android/android-ndk-r15c/toolchains/arm-linux-androideabi-4.9/prebuilt/linux-x86_64/bin/../lib/gcc/arm-linux-androideabi/4.9.x/../../../../arm-linux-androideabi/bin/ld: error: cannot open crtend_android.o: No such file or directory 
/home/longjing/tools/Android/android-ndk-r15c/toolchains/arm-linux-androideabi-4.9/prebuilt/linux-x86_64/bin/../lib/gcc/arm-linux-androideabi/4.9.x/../../../../arm-linux-androideabi/bin/ld: error: cannot find -lc 
/home/longjing/tools/Android/android-ndk-r15c/toolchains/arm-linux-androideabi-4.9/prebuilt/linux-x86_64/bin/../lib/gcc/arm-linux-androideabi/4.9.x/../../../../arm-linux-androideabi/bin/ld: error: cannot find -ldl 
collect2: error: ld returned 1 exit status 
使用find命令在ndk目录下搜索下crtend_android.o文件

cd /home/longjing/tools/Android/android-ndk-r15c 
find . -iname crtend_android.o 
```
![结果显示](/uploads/Transmision 交叉编译/c canot create.jpg)
```
可以看到,obj文件都是存在的。那么怎么解决呢?既然找不到,那就给你个路径
$ANDROID_NDK/toolchains/arm-linux-androideabi-4.9/prebuilt/linux-x86_64/bin/arm-linux-androideabi-gcc test.c --sysroot=$ANDROID_NDK/platforms/android-9/arch-arm 
最后重新编译,错误消失了。
```
![结果显示](/uploads/Transmision 交叉编译/test编译成功.png)
```
通过上面的脚本文件可以知道，我们的编译成功之后存放的目录为/thin/arm/ 目录下，如果这个目录就会存放编译产生的文件，其中lib目录如果有下面的俩个红色的就代表成功了，
至于这里这么多是因为没有再次的编译了，这是全部编译成功产生的文件
```
![结果显示](/uploads/Transmision 交叉编译/openssl编译结果.png)
编译libevent
```java
下面是编译的脚本文件

LIBEVENT_SOURCE="libevent-2.1.8-stable"
FAT="TS-Android"
SCRATCH="scratch"
# must be an absolute path
THIN=`pwd`/"thin"
NDK="/home/yuhui/workSpace/android-ndk-r10e"
SYSROOT_PREFIX="$NDK/platforms/android-16/arch-"

build_libevent() {
    ARCH="$1"
    PLATFROM="$2"

    if [ -z $PLATFROM ]
    then
        PLATFROM="linux-androideabi"
    fi

    if [ $ARCH = "arm64" ]
    then
        PLATFROM="linux-android"
        ACT_ARCH="aarch64"
    else
        ACT_ARCH=$ARCH
    fi

    SYSROOT=${SYSROOT_PREFIX}${ARCH}
    CC="$NDK/toolchains/${ACT_ARCH}-${PLATFROM}-4.9/prebuilt/linux-x86_64/bin/${ACT_ARCH}-${PLATFROM}-gcc --sysroot=${SYSROOT}"

    export CC="$CC"

    CWD=`pwd`

    echo "building curl for $ARCH..."
    mkdir -p "$SCRATCH/$ARCH-libevent"
    cd "$SCRATCH/$ARCH-libevent"
	
    #最新的libevent 需要使用到openssl，所以要引进对应的头文件，以及知道连接的库的地址
    CFLAGS="-I${SYSROOT_PREFIX}${ARCH}/usr/include -I${THIN}/$ARCH/include"
    LDFLAGS="-L${SYSROOT_PREFIX}${ARCH}/usr/lib -L${THIN}/$ARCH/lib"
	
    export LDFLAGS="$LDFLAGS"

    export CFLAGS="$CFLAGS"
    $CWD/$LIBEVENT_SOURCE/configure --host=$ACT_ARCH-$PLATFROM \
        --prefix=$THIN/$ARCH
    make install
}

这里要注意的，这里一定要使用ndkr11以下的版本，要不然会出现找不到arc4random_addrandom 之类的错误，这主要是因为ndkr11以上，有对这个函数最修改
下面分别在对应的r10e ，r14b中查询这个函数
```
r10e 查询的结果为：
![结果显示](/uploads/Transmision 交叉编译/arc4andomR10查询结果.png)

r14b中查询的结果为：
![结果显示](/uploads/Transmision 交叉编译/arc4r14b查询结果.png)

可以看到俩者是有区别的，更加详细的信息可以查询
arc4random_addrandom的改动 https://github.com/android-ndk/ndk/issues/48

所以这里要使用ndkr11以下的版本来编译，要不然会编译不过

```
通过上面的脚本文件可以知道，我们的编译成功之后存放的目录为/thin/arm/ 目录下，如果这个目录就会存放编译产生的文件，其中lib目录如果有下面的俩个红色的就代表成功了，
至于这里这么多是因为没有再次的编译了，这是全部编译成功产生的文件
```
![结果显示](/uploads/Transmision 交叉编译/libevent编译结果.png)

编译curl
```java
下面是编译的脚本文件

CURL_SOURCE="curl-7.60.0"
FAT="TS-Android"
SCRATCH="scratch"
# must be an absolute path
THIN=`pwd`/"thin"
NDK="/home/yuhui/workSpace/android-ndk-r10e"
SYSROOT_PREFIX="$NDK/platforms/android-16/arch-"


build_Curl() {
    ARCH="$1"
    PLATFROM="$2"

    if [ -z $PLATFROM ]
    then
        PLATFROM="linux-androideabi"
    fi

    if [ $ARCH = "arm64" ]
    then
        PLATFROM="linux-android"
        ACT_ARCH="aarch64"
    else
        ACT_ARCH=$ARCH
    fi

    SYSROOT=${SYSROOT_PREFIX}${ARCH}
    CC="$NDK/toolchains/${ACT_ARCH}-${PLATFROM}-4.9/prebuilt/linux-x86_64/bin/${ACT_ARCH}-${PLATFROM}-gcc --sysroot=${SYSROOT}"

    export CC="$CC"

    CWD=`pwd`

    echo "building curl for $ARCH..."
    mkdir -p "$SCRATCH/$ARCH-curl"
    cd "$SCRATCH/$ARCH-curl"
    CFLAGS="-I${SYSROOT_PREFIX}${ARCH}/usr/include"
    #-Din_addr_t=uint32_t 的意思是说给  in_addr_t 赋值
    export CFLAGS="$CFLAGS -Din_addr_t=uint32_t"
    $CWD/$CURL_SOURCE/configure --host=$ACT_ARCH-$PLATFROM \
        --with-ssl \
        --prefix=$THIN/$ARCH
    make install
}

#执行这个函数,传递的参数为arm
build_Curl "arm"
echo "build Curl success"
exit 1

通过上面的脚本文件可以知道，我们的编译成功之后存放的目录为/thin/arm/ 目录下，如果这个目录就会存放编译产生的文件，其中lib目录如果有下面的俩个红色的就代表成功了，
至于这里这么多是因为没有再次的编译了，这是全部编译成功产生的文件
```
![结果显示](/uploads/Transmision 交叉编译/curl编译结果.png)

最终这三个库编译出来的内容为
![结果显示](/uploads/Transmision 交叉编译/外部依赖库编译结果.png)


编译 transmission2.9.4
```java
下面是编译的脚本文件

SOURCE="transmission-2.94"
FAT="TS-Android"
SCRATCH="scratch"
# must be an absolute path
THIN=`pwd`/"thin"

NDK="/home/yuhui/workSpace/android-ndk-r10e"

SYSROOT_PREFIX="$NDK/platforms/android-16/arch-"

#配置transmission的选项
CONFIGURE_FLAGS="--enable-largefile --enable-utp --enable-lightweight --build=x86_64-unknown-linux-gnu"

#指定编译的类型
ARCHS="arm"

#给COMPILE 赋值，所以下面的 if [ "$COMPILE" ]就会进去
COMPILE="y"
#跟上面是一样的
#LIPO="y"

DEPLOYMENT_TARGET="6.0"

NEED_RECONFIGURE=yes
if [ "$COMPILE" ]
then
	CWD=`pwd`
	for ARCH in $ARCHS
	do
		ACT_ARCH=$ARCH
		PLATFROM="linux-androideabi"
		if [ $ARCH = "arm64" ]
		then
			ACT_ARCH="aarch64"
			PLATFROM="linux-android"
		fi

		echo "building $ARCH..."
		mkdir -p "$SCRATCH/$ARCH"
		cd "$SCRATCH/$ARCH"

        //指定include的位置
        CFLAGS="-I${SYSROOT_PREFIX}${ARCH}/usr/include -I${THIN}/$ARCH/include"

        export LIBCURL_CFLAGS="-I${THIN}/$ARCH/include"
        export LIBCURL_LIBS="-L${THIN}/$ARCH/lib"
        export LIBEVENT_CFLAGS="-I${THIN}/$ARCH/include"
        export LIBEVENT_LIBS="-L${THIN}/$ARCH/lib"
		
        #我们需要链接上面我们编译出来的外部库
        export LIBS="-lcrypto -levent -lssl -lcurl"
        export PATH=$PATH:"$NDK/toolchains/${ACT_ARCH}-${PLATFROM}-4.9/prebuilt/linux-x86_64/bin/"

        SYSROOT=${SYSROOT_PREFIX}${ARCH}

        #指定gcc ，注意后面的 --sysroot=${SYSROOT} 不能忘记
        CC="$NDK/toolchains/${ACT_ARCH}-${PLATFROM}-4.9/prebuilt/linux-x86_64/bin/${ACT_ARCH}-${PLATFROM}-gcc --sysroot=${SYSROOT}"
        export CC="${CC}"
        CXXFLAGS="$CFLAGS"
		
        #指定连接的库
        LDFLAGS="-L${SYSROOT_PREFIX}${ARCH}/usr/lib -L${THIN}/$ARCH/lib"
		
        #注意的点
        export CFLAGS="$CFLAGS -Din_addr_t=uint32_t  -D__android__  -Din_port_t=uint16_t"
        export CXXFLAGS="$CFLAGS"
        export LDFLAGS="$LDFLAGS"

	#echo "configure message for $THIN/$ARCH"

        if [ $NEED_RECONFIGURE = "yes" ]
        then
		$CWD/$SOURCE/configure \
                $CONFIGURE_FLAGS \
                --host=arm-linux-androideabi \
                --prefix=$THIN/$ARCH || exit 1; #--prefix=$THIN/$ARCH 指定文件生成的目录在 thin/arm/下
        fi
        make clean;
		make ; make install || exit 1
		cd $CWD
	done
fi

if [ "$LIPO" ]
then
	echo "building fat binaries..."
	mkdir -p $FAT/lib
	set - $ARCHS
	CWD=`pwd`
	cd $THIN/$1/lib
	for LIB in *.a
	do
		cd $CWD
		echo lipo -create `find $THIN -name $LIB` -output $FAT/lib/$LIB 1>&2
		lipo -create `find $THIN -name $LIB` -output $FAT/lib/$LIB || exit 1
	done

	cd $CWD
	cp -rf $THIN/$1/include $FAT
fi

echo Done

在上面的脚本中有这样的一句 export CFLAGS="$CFLAGS -Din_addr_t=uint32_t  -D__android__  -Din_port_t=uint16_t"，后面的-Din_addr_t=uint32_t  -D__android__  -Din_port_t=uint16_t
的意思是给in_addr_t ,in_port_t 赋值，定义__android__ 这样的宏，可以看出来，我们要对应的修改代码，原本也是transimssion的官网介绍默认是不支持android的，也即是android的编译要自己做一些
修改，下面是修改的内容

第一处是在transmission源码中的third-party\miniupnp\miniwget.c文件
修改的内容为：
#if defined(__sun) || defined(sun) || defined(__android__)
#define MIN(x,y) (((x)<(y))?(x):(y))
#endif

第二处修改的内容在 transmission源码中的libtransmission目录下的variant.c文件
修改的内容为：
#ifndef __android__
#include <locale.h> /* setlocale() */
#endif

static void
use_numeric_locale (struct locale_context * context,
                    const char            * locale_name)
{
#ifdef HAVE_USELOCALE

  context->new_locale = newlocale (LC_NUMERIC_MASK, locale_name, NULL);
  context->old_locale = uselocale (context->new_locale);

#else

#if defined (HAVE__CONFIGTHREADLOCALE) && defined (_ENABLE_PER_THREAD_LOCALE)
  context->old_thread_config = _configthreadlocale (_ENABLE_PER_THREAD_LOCALE);
#endif

#ifndef __android__
  context->category = LC_NUMERIC;
#endif

  tr_strlcpy (context->old_locale, setlocale (context->category, NULL), sizeof (context->old_locale));
  setlocale (context->category, locale_name);

#endif
}

bool
tr_variantGetReal (const tr_variant * v, double * setme)
{
  bool success = false;

  if (!success && ((success = tr_variantIsReal (v))))
    *setme = v->val.d;

  if (!success && ((success = tr_variantIsInt (v))))
    *setme = v->val.i;

  if (!success && tr_variantIsString (v))
    {
      char * endptr;
#ifndef __android__
      struct locale_context locale_ctx;
#endif
      //struct locale_context locale_ctx;
      double d;

      /* the json spec requires a '.' decimal point regardless of locale */
#ifndef __android__
      use_numeric_locale (&locale_ctx, "C");
#endif
      //use_numeric_locale (&locale_ctx, "C");

      d  = strtod (getStr (v), &endptr);
      //restore_locale (&locale_ctx);
#ifndef __android__
      restore_locale (&locale_ctx);
#endif

      if ((success = (getStr (v) != endptr) && !*endptr))
        *setme = d;
    }
  return success;
}

struct evbuffer *
tr_variantToBuf (const tr_variant * v, tr_variant_fmt fmt)
{
#ifndef __android__
  struct locale_context locale_ctx;
#endif
	
  //struct locale_context locale_ctx;
  struct evbuffer * buf = evbuffer_new();

  //use_numeric_locale (&locale_ctx, "C");
#ifndef __android__
  /* parse with LC_NUMERIC="C" to ensure a "." decimal separator */
  use_numeric_locale (&locale_ctx, "C");
#endif

  evbuffer_expand (buf, 4096); /* alloc a little memory to start off with */

  switch (fmt)
    {
      case TR_VARIANT_FMT_BENC:
        tr_variantToBufBenc (v, buf);
        break;

      case TR_VARIANT_FMT_JSON:
        tr_variantToBufJson (v, buf, false);
        break;

      case TR_VARIANT_FMT_JSON_LEAN:
        tr_variantToBufJson (v, buf, true);
        break;
    }

  //restore_locale (&locale_ctx);
#ifndef __android__
  /* restore the previous locale */
  restore_locale (&locale_ctx);
#endif
  return buf;
}

int
tr_variantFromBuf (tr_variant      * setme,
                   tr_variant_fmt    fmt,
                   const void      * buf,
                   size_t            buflen,
                   const char      * optional_source,
                   const char     ** setme_end)
{
  int err;
  struct locale_context locale_ctx;

  //use_numeric_locale (&locale_ctx, "C");
#ifndef __android__
  /* parse with LC_NUMERIC="C" to ensure a "." decimal separator */
  use_numeric_locale (&locale_ctx, "C");
#endif

  switch (fmt)
    {
      case TR_VARIANT_FMT_JSON:
      case TR_VARIANT_FMT_JSON_LEAN:
        err = tr_jsonParse (optional_source, buf, buflen, setme, setme_end);
        break;

      default /* TR_VARIANT_FMT_BENC */:
        err = tr_variantParseBenc (buf, ((const char*)buf)+buflen, setme, setme_end);
        break;
    }

  //restore_locale (&locale_ctx);
  /* restore the previous locale */
#ifndef __android__
  restore_locale (&locale_ctx);
#endif
  return err;
}

第三处修改的内容在 transmission源码中的libtransmission目录下的platform-quote.c文件
修改的内容为：
#elif defined (__sun)
  #include <sys/fs/ufs_quota.h> /* quotactl */
 #elif defined (__android__)
  #include <linux/quota.h>
 #else
  #include <sys/quota.h> /* quotactl() */
 #endif
 
 
 /* disabel quota on android */
#ifndef __android__
#ifndef _WIN32
static const char *
getdev (const char * path)
{



#endif /* disabel quota on android */

static int64_t
tr_getDiskFreeSpace (const char * path)
{


struct tr_device_info *
tr_device_info_create (const char * path)
{
  struct tr_device_info * info;

  info = tr_new0 (struct tr_device_info, 1);
  info->path = tr_strdup (path);
#ifndef _WIN32
#ifndef __android__
  info->device = tr_strdup (getblkdev (path));
  info->fstype = tr_strdup (getfstype (path));
#endif
#endif

  return info;
}

int64_t
tr_device_info_get_free_space (const struct tr_device_info * info)
{
  int64_t free_space;

  if ((info == NULL) || (info->path == NULL))
    {
      errno = EINVAL;
      free_space = -1;
    }
  else
    {
#ifndef __android__
      free_space = tr_getQuotaFreeSpace (info);

      if (free_space < 0)
#endif
        free_space = tr_getDiskFreeSpace (info->path);
    }

  return free_space;
}

第四处修改的内容在 transmission源码中的libtransmission目录下的util.c文件
修改的内容为：

double
tr_truncd (double x, int precision)
{
  char * pt;
  char buf[128];
  const int max_precision = (int) log10 (1.0 / DBL_EPSILON) - 1;
  tr_snprintf (buf, sizeof (buf), "%.*f", max_precision, x);
#ifndef __android__ // android doesn't support lconv ?
  if ((pt = strstr (buf, localeconv ()->decimal_point)))
    pt[precision ? precision+1 : 0] = '\0';
#endif
  return atof (buf);
}

第五处修改的内容在 transmission 源码中的 libtransmission目录下的util.h文件
修改的内容为：

/** @brief Private libtransmission function to update tr_time ()'s counter */
static inline void tr_timeUpdate (time_t now) { __tr_current_time = now; }

#ifdef WIN32
 #include <windef.h> /* MAX_PATH */
 #define TR_PATH_MAX (MAX_PATH + 1)
#else
 #include <limits.h> /* PATH_MAX */
 #ifdef PATH_MAX
  #define TR_PATH_MAX PATH_MAX
 #else
  #define TR_PATH_MAX 4096
 #endif
#endif

/** @brief Portability wrapper for htonll () that uses the system implementation if available */
uint64_t tr_htonll (uint64_t);

这些内容修改完之后，就可以执行脚本的执行了，如果正常执行的没有错误的化，会生成下面的文件 因为脚本指定了 --prefix=$THIN/$ARCH 所以会在/thin/arm/目录下生成对应的内容
```
![结果显示](/uploads/Transmision 交叉编译/transmission编译结果.png)

****Transmission ubuntu NDK 交叉编译总结****
===
```
至此Transmission交叉编译成功,我们为什么要先在ubuntu下面交叉编译，是为了后面移植到android studio 编译做准备，接下来会介绍怎么在Android Studio 中采用CmakeList直接编译源码
弄成一个可以调试的NDK项目,这样对于我们的开发能做到事半功倍的效果
```
