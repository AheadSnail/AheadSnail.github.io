---
layout: pager
title: Transmission Android CmakeList编译过程
date: 2018-06-30 09:24:52
tags: [Android,NDK,Transmission,CmakeList]
description:  Transmission Android CmakeList编译过程
---

Transmission Android CmakeList编译过程
<!--more-->


****简介****
===
```
上一篇文章中介绍了怎么样在ubuntu下面的采用Ndk,交叉编译链，编译transmission，这篇文章将介绍怎么在Android Studio下面采用CmakeList的方式来编译transmission库，至于为什么要在
android Studio中编译是为了我们开发的方便，在Android Studio中采用最新的CmakeList编译的化，是支持断点调试代码，方便我们阅读，修改代码
```

****移植Transmission遇到的问题****
===
```
既然要移植到Android Studio下面，那么就涉及到要编译哪些文件，还有编译需要的链接的参数，定义的宏等，这些都是我们需要采集的
因为Transmission是采用autoconf来维护，自动的生成Makefile的，所以我们可以参考Ubuntu交叉编译之后产生Makefile文件，从里面采集我们需要的这些信息


还记得前面一篇文章中介绍到在编译libevent库的时候采用的NDKr10e ,那是因为Ndk11以上有对这些函数做调整，但是在Android Studio中如果想采用CmakeList来编译的化，最低的Ndk版本不能低于r12
要不然会提示下面的这些错误
```

```java
CMake Error at D:/sdk/sdk/cmake/3.6.4111459/android.toolchain.cmake:345 (message):
  Missing file:
  D:/sdk/android-ndk-r10e-windows-x86_64/android-ndk-r10e/source.properties.
  Please use NDK r12+.
Call Stack (most recent call first):
  D:/sdk/sdk/cmake/3.6.4111459/share/cmake-3.6/Modules/CMakeDetermineSystem.cmake:98 (include)
  CMakeLists.txt
CMake Error: CMAKE_C_COMPILER not set, after EnableLanguage
CMake Error: CMAKE_CXX_COMPILER not set, after EnableLanguage

所以我们必须要将libevent库采用Ndr12以上来编译，那么这就意味着，我们不能逃避在编译libevent库试出现的 找不到arc4random_addrandom实现的问题,
更多的详细信息可以参考
https://github.com/android-ndk/ndk/issues/48

通过上面介绍可知，ndk中是有头文件的，但是确找不到对应的实现，下面介绍怎么解决这个问题,这个函数触发是在evutil_rand.c文件中，有这样的代码
#ifdef EVENT__HAVE_ARC4RANDOM
.....
#else /* !EVENT__HAVE_ARC4RANDOM { */
....
#include "./arc4random.c"
....
#endif /* } !EVENT__HAVE_ARC4RANDOM */

其中arc4random.c文件中就有定义这个函数
#ifndef ARC4RANDOM_NOADDRANDOM
ARC4RANDOM_EXPORT void
arc4random_addrandom(const unsigned char *dat, int datlen)
{
	int j;
	ARC4_LOCK_();
	if (!rs_initialized)
		arc4_stir();
	for (j = 0; j < datlen; j += 256) {
		/* arc4_addrandom() ignores all but the first 256 bytes of
		 * its input.  We want to make sure to look at ALL the
		 * data in 'dat', just in case the user is doing something
		 * crazy like passing us all the files in /var/log. */
		arc4_addrandom(dat + j, datlen - j);
	}
	ARC4_UNLOCK_();
}
#endif

evutil_rand.c文件中，有这样的代码，这就是触发调用的入口
void
evutil_secure_rng_add_bytes(const char *buf, size_t n)
{
	arc4random_addrandom((unsigned char*)buf,
	    n>(size_t)INT_MAX ? INT_MAX : (int)n);
}

通过上面可以知道，如果EVENT__HAVE_ARC4RANDOM = 0,也即是没有定义这个宏的时候，就会将arc4random.c 包含进来，这样 arc4random_addrandom 就会有定义了，下面查看一下这个宏是怎么产生的
由于transmission是采用autoconf来维护的，在他生成makefile文件之前，要先通过configure文件的检查，这个文件主要是检查当前的系统的环境，下面是configure.ac文件的内容

dnl Checks for library functions.
AC_CHECK_FUNCS([ \
  accept4 \
  arc4random \
  arc4random_buf \
  eventfd \
  ...
])
AC_CHECK_FUNCS：检查C标准库中是否存在函数。 如果找到，则定义预处理器宏HAVE_ [function]。它生成一个测试程序，声明一个具有相同名称的函数，然后编译并链接它。
```
关于autocof语法可以参考 https://aheadsnail.github.io/2018/04/11/AutoConfig-%E5%B8%B8%E8%A7%81%E7%9A%84%E5%AE%8F%E8%AF%A6%E8%A7%A3/
```java
所以上面会去检查系统是否中是否有arc4random 函数，如果有就会定义宏，这些宏生成在event-config.h文件中
/* Define to 1 if you have the `arc4random' function. */
#define EVENT__HAVE_ARC4RANDOM 1
/* Define to 1 if you have the `arc4random_buf' function. */
#define EVENT__HAVE_ARC4RANDOM_BUF 1

通过生成的event-config.h文件可以知道，系统是可以检测到这个函数的，所以会将EVENT__HAVE_ARC4RANDOM 置为1，这样在使用的时候，就会以为有这个函数存在去使用系统的函数，但本质是这个函数
在NDK中是只有声明，并没有定义，所以会在编译libevent库的时候出现找不到实现的问题，系统的函数这个检测我们是没有办法干预的，但是我们可以在使用这个宏的时候，手动的让这个宏改为0，这样
就会以为系统没有这个函数，libevent自己定义这个函数，因为会走 #include "./arc4random.c"

下面是修改的内容：
#undef EVENT__HAVE_ARC4RANDOM //手动的取消这个宏
#ifdef EVENT__HAVE_ARC4RANDOM
....
修改完之后，重新进行编译，发现又出现了
```
![结果显示](/uploads/Transmision 交叉编译/libeventndk14编译.png)
```java
从错误的信息可以看出来，这个函数的定义跟系统的这个对应的这个函数重定义了，系统的函数肯定是不能修改的，那么我们就有必要修改libevent库的函数，就改一个名字而已，下面是修改的内容

arc4random.c中总的要修改下面的内容
#ifndef ARC4RANDOM_NOADDRANDOM
ARC4RANDOM_EXPORT void
my_arc4random_addrandom(const unsigned char *dat, int datlen)
{
	int j;
	ARC4_LOCK_();
	if (!rs_initialized)
		arc4_stir();
	for (j = 0; j < datlen; j += 256) {
		/* arc4_addrandom() ignores all but the first 256 bytes of
		 * its input.  We want to make sure to look at ALL the
		 * data in 'dat', just in case the user is doing something
		 * crazy like passing us all the files in /var/log. */
		arc4_addrandom(dat + j, datlen - j);
	}
	ARC4_UNLOCK_();
}
#endif

evutil_rand.c中总的要修改的内容：
#undef EVENT__HAVE_ARC4RANDOM //手动的取消这个宏
#ifdef EVENT__HAVE_ARC4RANDOM
....

void
evutil_secure_rng_add_bytes(const char *buf, size_t n)
{
	my_arc4random_addrandom((unsigned char*)buf,
	    n>(size_t)INT_MAX ? INT_MAX : (int)n);
}

修改完之后，重新编译，通过了，至此这个库已经修改完成

编译transmission,遇到下面的这个问题
```
![结果显示](/uploads/Transmision 交叉编译/ndkr14endpowent.png)

经查阅这个函数也是NDK的一个坑，详细信息可以参考
endpowent https://github.com/android-ndk/ndk/issues/77

```
这个函数在系统的头文件pwd.h文件中，跟他一起配套使用的方式是getpwuid，endpwent函数一般用来关闭用getpwent打开的密码文件。

从上面的文章中可知这个函数是没有实现的，那我们可以在Android的源码中查找这个函数，因为这个是系统的库，那么这个函数肯定存在android的源码中
```
查找的结果
![结果显示](/uploads/Transmision 交叉编译/endpewent查找结果.png)
```java
其中有这样的查询结果 /bionic/libc/bionic/ndk_cruft.cpp:void endpwent() { }，我们可以进入对应的目录找到这个文件，下面是关键的内容

// This was never implemented in bionic, only needed for ABI compatibility with the NDK.
// In the M time frame, over 1000 apps have a reference to this!
void endpwent() { }

// This was removed from BSD.
void arc4random_stir(void) {
  // The current implementation stirs itself as needed.
}

// This was removed from BSD.
void arc4random_addrandom(unsigned char*, int) {
  // The current implementation adds randomness as needed.
}

可以看到这些函数都是连接找不到实现的，都在同一个文件，而跟endpwent 配套使用的 getpwuid 函数的实现是在 ./bionic/libc/bionic/stubs.cpp 目录
passwd* getpwuid(uid_t uid) { // NOLINT: implementing bad function.
  passwd_state_t* state = g_passwd_tls_buffer.get();
  if (state == NULL) {
    return NULL;
  }

  passwd* pw = android_id_to_passwd(state, uid);
  if (pw != NULL) {
    return pw;
  }
  // Handle OEM range.
  pw = oem_id_to_passwd(uid, state);
  if (pw != NULL) {
    return pw;
  }
  return app_id_to_passwd(uid, state);
}

可以看到这个函数是有实现的，而且这个函数跟前面的endpwent并不在同一个文件，而且在查找google的测试代码中，也并没有调用endpwent 这个函数,下面是测试的代码
static void check_getpwuid_r(const char* username, uid_t uid, uid_type_t uid_type) {
  passwd pwd_storage;
  char buf[512];
  int result;

  errno = 0;
  passwd* pwd = NULL;
  result = getpwuid_r(uid, &pwd_storage, buf, sizeof(buf), &pwd);
  ASSERT_EQ(0, result);
  ASSERT_EQ(0, errno);
  SCOPED_TRACE("getpwuid_r");
  check_passwd(pwd, username, uid, uid_type);
}

通过上面的内容，我们可以直接，简单的将这个函数屏蔽掉，这样就可以编译通过了，最终生成下面的内容
```
![结果显示](/uploads/Transmision 交叉编译/transmission编译结果.png)

****Android CmakeList的编写****
===
```java
cmake_minimum_required(VERSION 3.4.1)

#设置变量 自定义变量使用SET(OBJ_NAME xxxx)，使用时${OBJ_NAME}
set(My_ARIC_SRC ${CMAKE_SOURCE_DIR}/src/main/cpp/src)
#message( WARNING  "${My_ARIC_SRC}")

#定义宏
add_definitions( -Din_addr_t=uint32_t
 -D__android__
 -Din_port_t=uint16_t
 -D__TRANSMISSION__
 -DPACKAGE_DATA_DIR="${CMAKE_SOURCE_DIR}"
 -DPACKAGE_NAME=\"transmission\"
 -DPACKAGE_TARNAME=\"transmission\"
 -DPACKAGE_VERSION=\"2.94\"
 -DPACKAGE_STRING=\"transmission\ 2.94\"
 -DPACKAGE_BUGREPORT=\"https://github.com/transmission/transmission\"
 -DPACKAGE_URL=\"\"
 -DPACKAGE=\"transmission\"
 -DVERSION=\"2.94\"
 -DSTDC_HEADERS=1
 -DHAVE_SYS_TYPES_H=1
 -DHAVE_SYS_STAT_H=1
 -DHAVE_STDLIB_H=1
 -DHAVE_STRING_H=1
 -DHAVE_MEMORY_H=1
 -DHAVE_STRINGS_H=1
 -DHAVE_INTTYPES_H=1
 -DHAVE_STDINT_H=1
 -DHAVE_UNISTD_H=1
 -DHAVE_DLFCN_H=1
 -DLT_OBJDIR=\".libs/\"
 -DSTDC_HEADERS=1
 -DTIME_WITH_SYS_TIME=1
 -DHAVE_STDBOOL_H=1
 -DHAVE_PREAD=1
 -DHAVE_PWRITE=1
 -DHAVE_STRLCPY=1
 -DHAVE_DAEMON=1
 -DHAVE_DIRNAME=1
 -DHAVE_BASENAME=1
 -DHAVE_STRCASECMP=1
 -DHAVE_LOCALTIME_R=1
 -DHAVE_MEMMEM=1
 -DHAVE_STRSEP=1
 -DHAVE_SYSLOG=1
 -DHAVE_VALLOC=1
 -DHAVE_POSIX_MEMALIGN=1
 -DHAVE_MKDTEMP=1
 -DHAVE_PTHREAD=1
 -DHAVE_GETMNTENT=1
 -DHAVE_DECL_POSIX_FADVISE=0
 -DWITH_UTP=1
 -DGETTEXT_PACKAGE=\"transmission-gtk\"
 -DHAVE_LOCALE_H=1
 -DHAVE_LC_MESSAGES=1
 -DTR_LIGHTWEIGHT=1
 -DENABLE_STRNATPMPERR
 )

#添加头文件的查找目录
include_directories(
${CMAKE_SOURCE_DIR}/src/main/cpp/src/include
${CMAKE_SOURCE_DIR}/src/main/cpp/src/include/libtransmission
${CMAKE_SOURCE_DIR}/src/main/cpp/src/third-party/libb64
${CMAKE_SOURCE_DIR}/src/main/cpp/src/third-party/libutp
${CMAKE_SOURCE_DIR}/src/main/cpp/src/third-party
${CMAKE_SOURCE_DIR}/src/main/cpp/src/third-party/libnatpmp
)

#用来输出参数 ${CMAKE_SOURCE_DIR} 代表的就为CmakeList的路径，这里即为app目录
#message( WARNING  "${CMAKE_SOURCE_DIR}")

#add_library():添加库，分为两种，一种是需要编译为库的代码，一种是已经编译好的库文件。 一般.so文件，还有STATIC，一般.a文件。IMPORTED表示引用的不是生成的。
add_library( open-event
             STATIC
             IMPORTED)

# set_target_properties：对于已经编译好的so文件需要引入，所以需要设置 这里cares是名字，然后是PROPERTIES IMPORTED_LOCATION加上库的路径。
set_target_properties( open-event
                       PROPERTIES IMPORTED_LOCATION
                       ${CMAKE_SOURCE_DIR}/src/main/cpp/lib/libevent.a )

add_library( open-event_core
             STATIC
             IMPORTED)

set_target_properties( open-event_core
                       PROPERTIES IMPORTED_LOCATION
                       ${CMAKE_SOURCE_DIR}/src/main/cpp/lib/libevent_core.a )

add_library( open-event_extra
             STATIC
             IMPORTED)

set_target_properties( open-event_extra
                       PROPERTIES IMPORTED_LOCATION
                       ${CMAKE_SOURCE_DIR}/src/main/cpp/lib/libevent_extra.a )

add_library( open-event_openssl
             STATIC
             IMPORTED)

set_target_properties( open-event_openssl
                       PROPERTIES IMPORTED_LOCATION
                       ${CMAKE_SOURCE_DIR}/src/main/cpp/lib/libevent_openssl.a )

add_library( open-event_potheads
             STATIC
             IMPORTED)

set_target_properties( open-event_potheads
                       PROPERTIES IMPORTED_LOCATION
                       ${CMAKE_SOURCE_DIR}/src/main/cpp/lib/libevent_pthreads.a )

# 添加两个预编译库
add_library(openssl-crypto
    STATIC
    IMPORTED)

set_target_properties(openssl-crypto
                      PROPERTIES IMPORTED_LOCATION
                      ${CMAKE_SOURCE_DIR}/src/main/cpp/lib/libcrypto.a )

add_library(openssl-ssl
  STATIC
  IMPORTED)

set_target_properties(openssl-ssl
                    PROPERTIES IMPORTED_LOCATION
                    ${CMAKE_SOURCE_DIR}/src/main/cpp/lib/libssl.a )

add_library( open-curl
             STATIC
             IMPORTED)

set_target_properties( open-curl
                       PROPERTIES IMPORTED_LOCATION
                       ${CMAKE_SOURCE_DIR}/src/main/cpp/lib/libcurl.a )

add_library( open-z
             STATIC
             IMPORTED)

set_target_properties( open-z
                       PROPERTIES IMPORTED_LOCATION
                       ${CMAKE_SOURCE_DIR}/src/main/cpp/lib/libz.a )

#dht的源码
set(dht_src ${My_ARIC_SRC}/third-party/dht/dht.c )

#libb64的源码
set(bb64_src ${My_ARIC_SRC}/third-party/libb64/cdecode.c  ${My_ARIC_SRC}/third-party/libb64/cencode.c )

#libnatpmp的源码
set(natpmp_src ${My_ARIC_SRC}/third-party/libnatpmp/getgateway.c
        ${My_ARIC_SRC}/third-party/libnatpmp/natpmp.c
        ${My_ARIC_SRC}/third-party/libnatpmp/wingettimeofday.c )

#libutp的源码
set(utp_src ${My_ARIC_SRC}/third-party/libutp/utp.cpp  ${My_ARIC_SRC}/third-party/libutp/utp_utils.cpp )

#miniupnp的源码
set(miniupnp_src ${My_ARIC_SRC}/third-party/miniupnp/connecthostport.c
       ${My_ARIC_SRC}/third-party/miniupnp/igd_desc_parse.c
       ${My_ARIC_SRC}/third-party/miniupnp/minisoap.c
       ${My_ARIC_SRC}/third-party/miniupnp/minissdpc.c
       ${My_ARIC_SRC}/third-party/miniupnp/miniupnpc.c
       ${My_ARIC_SRC}/third-party/miniupnp/miniwget.c
       ${My_ARIC_SRC}/third-party/miniupnp/minixml.c
       ${My_ARIC_SRC}/third-party/miniupnp/portlistingparse.c
       ${My_ARIC_SRC}/third-party/miniupnp/receivedata.c
       ${My_ARIC_SRC}/third-party/miniupnp/upnpcommands.c
       ${My_ARIC_SRC}/third-party/miniupnp/upnpreplyparse.c
       )

#libtransmission的源码
set(libtransmission_src
        ${My_ARIC_SRC}/libtransmission/announcer.c
        ${My_ARIC_SRC}/libtransmission/announcer-http.c
        ${My_ARIC_SRC}/libtransmission/announcer-udp.c
        ${My_ARIC_SRC}/libtransmission/bandwidth.c
        ${My_ARIC_SRC}/libtransmission/bitfield.c
        ${My_ARIC_SRC}/libtransmission/blocklist.c
        ${My_ARIC_SRC}/libtransmission/cache.c
        ${My_ARIC_SRC}/libtransmission/clients.c
        ${My_ARIC_SRC}/libtransmission/completion.c
        ${My_ARIC_SRC}/libtransmission/ConvertUTF.c
        ${My_ARIC_SRC}/libtransmission/crypto.c
        ${My_ARIC_SRC}/libtransmission/crypto-utils.c
        ${My_ARIC_SRC}/libtransmission/crypto-utils-fallback.c
        ${My_ARIC_SRC}/libtransmission/error.c
        ${My_ARIC_SRC}/libtransmission/fdlimit.c
        ${My_ARIC_SRC}/libtransmission/file.c
        ${My_ARIC_SRC}/libtransmission/handshake.c
        ${My_ARIC_SRC}/libtransmission/history.c
        ${My_ARIC_SRC}/libtransmission/inout.c
        ${My_ARIC_SRC}/libtransmission/list.c
        ${My_ARIC_SRC}/libtransmission/log.c
        ${My_ARIC_SRC}/libtransmission/magnet.c
        ${My_ARIC_SRC}/libtransmission/makemeta.c
        ${My_ARIC_SRC}/libtransmission/metainfo.c
        ${My_ARIC_SRC}/libtransmission/natpmp.c
        ${My_ARIC_SRC}/libtransmission/net.c
        ${My_ARIC_SRC}/libtransmission/peer-io.c
        ${My_ARIC_SRC}/libtransmission/peer-mgr.c
        ${My_ARIC_SRC}/libtransmission/peer-msgs.c
        ${My_ARIC_SRC}/libtransmission/platform.c
        ${My_ARIC_SRC}/libtransmission/platform-quota.c
        ${My_ARIC_SRC}/libtransmission/port-forwarding.c
        ${My_ARIC_SRC}/libtransmission/ptrarray.c
        ${My_ARIC_SRC}/libtransmission/quark.c
        ${My_ARIC_SRC}/libtransmission/resume.c
        ${My_ARIC_SRC}/libtransmission/rpcimpl.c
        ${My_ARIC_SRC}/libtransmission/rpc-server.c
        ${My_ARIC_SRC}/libtransmission/session.c
        ${My_ARIC_SRC}/libtransmission/stats.c
        ${My_ARIC_SRC}/libtransmission/torrent.c
        ${My_ARIC_SRC}/libtransmission/torrent-ctor.c
        ${My_ARIC_SRC}/libtransmission/torrent-magnet.c
        ${My_ARIC_SRC}/libtransmission/tr-dht.c
        ${My_ARIC_SRC}/libtransmission/tr-lpd.c
        ${My_ARIC_SRC}/libtransmission/tr-udp.c
        ${My_ARIC_SRC}/libtransmission/tr-utp.c
        ${My_ARIC_SRC}/libtransmission/tr-getopt.c
        ${My_ARIC_SRC}/libtransmission/trevent.c
        ${My_ARIC_SRC}/libtransmission/upnp.c
        ${My_ARIC_SRC}/libtransmission/utils.c
        ${My_ARIC_SRC}/libtransmission/variant.c
        ${My_ARIC_SRC}/libtransmission/variant-benc.c
        ${My_ARIC_SRC}/libtransmission/variant-json.c
        ${My_ARIC_SRC}/libtransmission/verify.c
        ${My_ARIC_SRC}/libtransmission/watchdir.c
        ${My_ARIC_SRC}/libtransmission/watchdir-generic.c
        ${My_ARIC_SRC}/libtransmission/web.c
        ${My_ARIC_SRC}/libtransmission/webseed.c
        ${My_ARIC_SRC}/libtransmission/wildmat.c
        ${My_ARIC_SRC}/libtransmission/watchdir-inotify.c
        ${My_ARIC_SRC}/libtransmission/file-posix.c
        ${My_ARIC_SRC}/libtransmission/crypto-utils-openssl.c
        )


#daemon的源码
set(daemon_src ${My_ARIC_SRC}/daemon/daemon.c
 ${My_ARIC_SRC}/daemon/daemon-posix.c
 ${My_ARIC_SRC}/daemon/remote.c )

#cli的源码
set(cli_src ${My_ARIC_SRC}/cli/cli.c )

find_library( # Sets the name of the path variable.
              log-lib
              # Specifies the name of the NDK library that
              # you want CMake to locate.
              log )

add_library( # Sets the name of the library.
             transmission

             # Sets the library as a shared library.
             SHARED

            ${dht_src}
            ${bb64_src}
            ${natpmp_src}
            ${utp_src}
            ${miniupnp_src}
            ${libtransmission_src}
            ${daemon_src}
            ${cli_src}

            #自己的源码文件
            src/main/cpp/jni_interface.cpp
            )


#链接库文件，第一个代表要生成的target，后面的item为要连接的item,对于如果是通过 IMPORTED_NO_SONAME的方式来导入的
#这里注意不推荐使用全路径的方式来导入，一般要通过set_target_properties来指定
target_link_libraries( transmission

                       # Links the target library to the log library
                       # included in the NDK.

                       ${log-lib}
                       open-z
                       open-event
                       open-event_core
                       open-event_extra
                       open-event_openssl
                       open-event_potheads
                       open-curl
                       #最后一个openssl 库文件要放在最后来连接。。。要不然会出现找不到函数的定义问题。。。注意
                       openssl-ssl openssl-crypto
                       )
					   
目录工程为：					  
```
![结果显示](/uploads/Transmision 交叉编译/transmisionsCmamekList移植.jpg)

编译的结果为：
![结果显示](/uploads/Transmision 交叉编译/CmakeList编译结果.png)


****验证是否可以真的下载****
===
```java
官网的transmission中有一个cli模块，这个是一个命令行模式的，通过了解他的代码，需要传递的参数，大致下了个简单的demo，验证是否可以真的下载

@Override
public void onClick(View view)
{
    switch (view.getId())
    {
        case R.id.but_start:
            //是否有权限
            if (PermissionManager.getInstance().isHasNecessaryPermissions(this))
            {
                String[] commands = new String[3];
                //第一个参数
                //设置下载文件保存的目录
                commands[0] = "-w=" + FileUtils.getTransmissionDataPath();
                commands[1] = "-g=" + FileUtils.getTransmissionConfigPath();
                //设置种子文件的路径
                commands[2] = FileUtils.getTransmissionTorrentPath();
                Transmission.initTransmission(commands);
            }
            else
            {
                Toast.makeText(MainActivity.this,"没有权限",Toast.LENGTH_SHORT).show();
            }
            break;
    }
}
上面是传递的参数，至于这些要传递什么参数，只能去看cli.c的代码,上面是我了解之后，需要传递的参数

/**
 * 初始化Transmission
*/
public static native int initTransmission(String[] pArgs);

/**
 * 初始化Transmission，开启线程
 * @param env
 * @param jstr
 */
JNIEXPORT jint JNICALL Java_com_example_com_transmissionandroidproject_Transmission_initTransmission
        (JNIEnv * env, jclass jstr,jobjectArray pArgs)
{
    //pArgs 传递的是命令参数
    argc = env->GetArrayLength(pArgs);
    LOGD("pArgs length %d", argc);

    //将java对应的每一个参数解析到char * argv数组中
    for (int i = 0; i < argc; i++) {
        jstring js = (jstring) env->GetObjectArrayElement(pArgs, i);
        argv[i] = (char *) env->GetStringUTFChars(js, 0);
        LOGD("pArgs argv %s", argv[i]);
    }

    int rev = 0;
    //2创建子线程,用于下载，检测等,创建成功之后，就会执行run的回调
    if (pthread_create(&pthread_tid, NULL, run, NULL)) {
        //创建失败
        LOGD("cannot create the thread\n");
        pthread_detach(pthread_tid);
        rev = 1;
        return rev;
    }
    return rev;
}

extern "C" {
extern int cli_tr_main(int argc, char *argv[]);
}

/**
 * 线程执行的方法回调
 * @param pVoid
 * @return
 */
void *run(void *pVoid) {
    cli_tr_main(argc, argv);
    return NULL;
}

//命令行的入口函数 这个是cli.c文件的main 函数，这里直接改了个函数名
int cli_tr_main (int argc, char * argv[])
{
  tr_session  * h;
  tr_ctor     * ctor;
  tr_torrent  * tor = NULL;
  tr_variant       settings;
  const char  * configDir;
  uint8_t     * fileContents;
  size_t        fileLength;
  const char  * str;
  
  .....
}
```

下面是下载的结果,可以看出来，我们的移植是没有出现问题的
![结果显示](/uploads/Transmision 交叉编译/transmission结果验证.png)




