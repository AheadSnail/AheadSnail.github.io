---
layout: pager
title: AndroidStudio CmakeList 编译 Aria2
date: 2018-06-17 10:28:07
tags: [AndroidStudio,CmakeList,Aria2]
description:  AndroidStudio CmakeList 编译 Aria2为一个动态库
---

### 概述

> AndroidStudio CmakeList 编译 Aria2为一个动态库

<!--more-->


### 简介
> 上一篇文章中，介绍了AutoConf，AutoMake，LibTool的基本语法，已经简单的使用，如何生成还有是怎么样来维护MakeFile的，之前都有分析过，大致我们要维护的文件就有configure.ac文件，还有Makefile.am文件，其中的configure.ac文件主要是用来检测环境的，使我们可以做到条件编译，比如检测当前的系统，检测当前环境是否含有某个库，检测函数等，根据检测的结果可以生成不同的变量传递到我们的MakeFile.am或者直接传递到Makefile文件中，还有一个文件就是Makefile.am文件，这个用来定义我们当前要编译的文件，还有编译的参数，连接的参数等，还可以接受到configure.ac文件检测的结果来做到条件的编译，有选择的编译需要的文件

我们为什么需要去了解这个文件
> 是因为当前的Aria2虽然在ubuntu利用android的交叉编译链可以生成可执行文件，但是我们需要移植到AndroidStudio上面去的化，就需要我们自己来编写CmakeList文件 而且我们需要生成的文件是一个so文件，不是一个可执行的文件，至于为什么要采用CmakeList来写主要是因为Google推荐使用，还有AndroidStudio在CmakeList下面如果配置的好，是可以调试的， 方便我们查看代码，本身目前就缺少一个编译器查看这些源码还有断点调试，所以采用CmakeList来写

因为我们需要手动的写CmakeList所以我们就有必要去了解文件，去了解他是怎么编译的。然后提取我们需要的信息，比如需要编译什么文件 编译的时候传递什么样的参数，需要连接什么库等，

### 确定需要编译的文件
```MakeFile
首先我们在Ubuntu，Aria源码目录下面执行 ./android-config文件，用来检测当前的环境，并且生成对应的变量，以便条件编译，其中configure.ac中有这样的输出文件
AC_CONFIG_FILES([Makefile
                src/Makefile
                src/libaria2.pc
                src/includes/Makefile
                test/Makefile
                po/Makefile.in
                lib/Makefile
                doc/Makefile
                doc/manual-src/Makefile
                doc/manual-src/en/Makefile
                doc/manual-src/en/conf.py
                doc/manual-src/ru/Makefile
                doc/manual-src/ru/conf.py
                doc/manual-src/pt/Makefile
                doc/manual-src/pt/conf.py
                deps/Makefile])
意思就是说用于检测的环境等生成的变量都要在这些文件里面生成一份，可以看到这里有多个MakeFile文件
		
然后执行 ./android-make ，就可以完成android 版本的Aria2的可执行文件，下面是这个文件的内容
if [ -z "$ANDROID_HOME" ]; then
    echo 'No $ANDROID_HOME specified.'
    exit 1
fi
TOOLCHAIN=$ANDROID_HOME/toolchain
PATH=$TOOLCHAIN/bin:$PATH

make "$@"

就是检测下环境变量，然后执行make "$@"，$@就是上面所说的多个MakeFIle文件，这里就会依次的执行编译，而且每一个MakeFile文件的环境变量，编译的参数都是一样的，但是要编译的规则，还有要
编译的文件，是要根据每一个文件夹下面的MakeFile.am文件指定的，这里最主要的就是src/Makefile文件，我们的源码文件都是存储在这个文件
```
[MakeFile.am基本知识](http://www.cnblogs.com/ranson7zop/p/8268196.html)

确定MakeFIle文件中哪些文件是需要编译的
```MakeFile
下面的这些宏都是通过configure.ac检测之后生成的宏
如果当前是window下编译的化
if MINGW_BUILD			   MINGW_BUILD 为false
SRCS += WinConsoleFile.cc WinConsoleFile.h
endif # MINGW_BUILD

如果允许WebSocket的化，默认是支持的
if ENABLE_WEBSOCKET
SRCS += \
	WebSocketInteractionCommand.cc WebSocketInteractionCommand.h\
	WebSocketResponseCommand.cc WebSocketResponseCommand.h\
	WebSocketSession.cc WebSocketSession.h\
	WebSocketSessionMan.cc WebSocketSessionMan.h
endif # ENABLE_WEBSOCKET

如果不允许WebSocket的化
if !ENABLE_WEBSOCKET
SRCS += NullWebSocketSessionMan.h
endif # !ENABLE_WEBSOCKET

如果支持XMLLIB
if HAVE_SOME_XMLLIB
SRCS += \
	ParserStateMachine.h\
	XmlAttr.cc XmlAttr.h\
	XmlParser.cc XmlParser.h
endif # HAVE_SOME_XMLLIB
....
```

如果你不知道，或者不确定那些宏的值，也是可以知道 MakeFIle.am文件，需要什么样的文件，我们知道最终肯定都是由MakeFile来编译的，所以我们可以直接看MakeFile文件，他肯定会将要编译的文件收集起来，不需要编译的文件就注释掉，而且这样比较保险

MakeFIle文件其中指定要编译的文件是用这个变量来收集的
```MakeFIle
SRCS = a2algo.h a2functional.h a2io.h a2iterator.h a2netcompat.h \
	 ......
	 libssl_compat.h $(am__append_1) $(am__append_2) \
	$(am__append_3) $(am__append_4) $(am__append_5) \
	$(am__append_6) $(am__append_7) $(am__append_8) \
	$(am__append_9) $(am__append_10) $(am__append_11) \
	$(am__append_12) $(am__append_13) $(am__append_14) \
	$(am__append_15) $(am__append_16) $(am__append_17) \
	$(am__append_18) $(am__append_19) $(am__append_20) \
	$(am__append_21) $(am__append_22) $(am__append_23) \
	$(am__append_24) $(am__append_25) $(am__append_26) \
	$(am__append_27) $(am__append_28) $(am__append_29) \
	$(am__append_30) $(am__append_31) $(am__append_32) \
	$(am__append_33) $(am__append_34) $(am__append_35) \
	$(am__append_36) $(am__append_37) $(am__append_38) \
	$(am__append_39) $(am__append_40) $(am__append_41) \
	$(am__append_42) $(am__append_43) $(am__append_44) \
	$(am__append_45) $(am__append_46)

其中的$(am__append_1)代表是取这个变量的值，如果这个变量前面有定义就取他的值，如果不存在就定义一个变量
例如：下面就是我们条件编译的内容，前面带有#的代表是注释的文件，这就是我们在MakeFile.am文件中对应的条件编译的内容，如果不需要就是用#注释的，所以我们能没有错误的指定哪些文件是需要编译的
#am__append_1 = WinConsoleFile.cc WinConsoleFile.h    
am__append_2 = \
	WebSocketInteractionCommand.cc WebSocketInteractionCommand.h\
	WebSocketResponseCommand.cc WebSocketResponseCommand.h\
	WebSocketSession.cc WebSocketSession.h\
	WebSocketSessionMan.cc WebSocketSessionMan.h
#am__append_3 = NullWebSocketSessionMan.h
am__append_4 = \
	ParserStateMachine.h\
	XmlAttr.cc XmlAttr.h\
	XmlParser.cc XmlParser.h
...
```

从MakeFIle文件中确定连接的参数，以及对应的编译参数
```MakeFIle
MakeFile文件中的 AM_CPPFLAGS这个变量就代表编译需要传递的编译参数，这个我们是需要的，这里解释下期中的变量
-I$(top_srcdir)/lib ， -I 代表需要导入哪个文件，也即是定位文件，可以用来定义头文件的地方，这样系统就会去对应的目录中去查找对应的文件
-DLOCALEDIR=\"${datarootdir}/locale\" -DHAVE_CONFIG_H  -D代表定义一个宏

AM_CPPFLAGS = \
	-I$(top_srcdir)/lib -I$(top_srcdir)/intl\
	-I$(srcdir)/includes -I$(builddir)/includes\
	-DLOCALEDIR=\"${datarootdir}/locale\" -DHAVE_CONFIG_H \
	 \
	-I/opt/NDK/android-ndk-r14b/usr/local/include \
	 \
	 \
	-I/opt/NDK/android-ndk-r14b/usr/local/include \
	 \
	 \
	-I/opt/NDK/android-ndk-r14b/usr/local/include \
	 \
	 \
	 \
	-I/opt/NDK/android-ndk-r14b/usr/local/include \
	-DCARES_STATICLIB -I/opt/NDK/android-ndk-r14b/usr/local/include \
	-I$(top_builddir)/deps/wslay/lib/includes -I$(top_srcdir)/deps/wslay/lib/includes \
	 \

MakeFile文件中的这个变量代表编译的时候需要连接的库	 
EXTLDADD =  \
	 \
	-L/opt/NDK/android-ndk-r14b/usr/local/lib -lz \
	 \
	 \
	-L/opt/NDK/android-ndk-r14b/usr/local/lib -lexpat \
	 \
	 \
	 \
	-L/opt/NDK/android-ndk-r14b/usr/local/lib -lssl -lcrypto \
	 \
	 \
	 \
	-L/opt/NDK/android-ndk-r14b/usr/local/lib -lssh2 \
	-L/opt/NDK/android-ndk-r14b/usr/local/lib -lcares \
	$(top_builddir)/deps/wslay/lib/libwslay.la \
	 \
	 \	 
```

也可以在Aria2根目录中，执行./android-config命令，改命令会去检测环境，同时最后面会将检测的结果，还有编译的一些参数，打印出来
![结果显示](/uploads/Aria2编译/android-config配置.png)

### AndroidStudio CmakeList编写

这里我们需要连接的库是从Ubuntu，跨平台吧编译Android的时候生成导出来的,下面是我CmakeList文件的编写

```Cmake
cmake_minimum_required(VERSION 3.4.1)

#设置变量 自定义变量使用SET(OBJ_NAME xxxx)，使用时${OBJ_NAME}
set(My_ARIC_SRC ${CMAKE_SOURCE_DIR}/src/main/cpp/src)
#message( WARNING  "${My_ARIC_SRC}")

#定义宏 对应的即为  -DHAVE_CONFIG_H 
add_definitions(-DHAVE_CONFIG_H)

#添加头文件的查找目录
include_directories(${CMAKE_SOURCE_DIR}/src/main/cpp/src/includes)


#用来输出参数 ${CMAKE_SOURCE_DIR} 代表的就为CmakeList的路径，这里即为app目录
#message( WARNING  "${CMAKE_SOURCE_DIR}")

#add_library():添加库，分为两种，一种是需要编译为库的代码，一种是已经编译好的库文件。 一般.so文件，还有STATIC，一般.a文件。IMPORTED表示引用的不是生成的。
add_library( open-cares
             STATIC
             IMPORTED)

# set_target_properties：对于已经编译好的so文件需要引入，所以需要设置 这里cares是名字，然后是PROPERTIES IMPORTED_LOCATION加上库的路径。
set_target_properties( open-cares
                       PROPERTIES IMPORTED_LOCATION
                       ${CMAKE_SOURCE_DIR}/src/main/cpp/lib/libcares.a )

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


add_library( open-expat
             STATIC
             IMPORTED)

set_target_properties( open-expat
                       PROPERTIES IMPORTED_LOCATION
                       ${CMAKE_SOURCE_DIR}/src/main/cpp/lib/libexpat.a )

add_library( open-ssh2
             STATIC
             IMPORTED)

set_target_properties( open-ssh2
                       PROPERTIES IMPORTED_LOCATION
                       ${CMAKE_SOURCE_DIR}/src/main/cpp/lib/libssh2.a )

add_library( open-z
             STATIC
             IMPORTED)

set_target_properties( open-z
                       PROPERTIES IMPORTED_LOCATION
                       ${CMAKE_SOURCE_DIR}/src/main/cpp/lib/libz.a )

set(wslay_Src
     ${My_ARIC_SRC}/wslay/wslay_frame.c
     ${My_ARIC_SRC}/wslay/wslay_event.c
     ${My_ARIC_SRC}/wslay/wslay_queue.c
     ${My_ARIC_SRC}/wslay/wslay_net.c )

.......

find_library( # Sets the name of the path variable.
              log-lib
              # Specifies the name of the NDK library that
              # you want CMake to locate.
              log )

add_library( # Sets the name of the library.
             Aria

             # Sets the library as a shared library.
             SHARED

            ${AriaSrc}
            ${am__append_1_4}
            ${am__append_6_22}
            ${am__append_23_29}
            ${am__append_30}
            ${am__append_31}
            ${wslay_Src}

            #自己的源码文件
            src/main/cpp/native-lib.cpp
            ${My_ARIC_SRC}/main.cc
            )


#链接库文件，第一个代表要生成的target，后面的item为要连接的item,对于如果是通过 IMPORTED_NO_SONAME的方式来导入的
#这里注意不推荐使用全路径的方式来导入，一般要通过set_target_properties来指定
target_link_libraries( Aria

                       # Links the target library to the log library
                       # included in the NDK.

                       ${log-lib}
                       open-z
                       open-expat
                       open-ssh2
                       open-cares
                       #最后一个openssl 库文件要放在最后来连接。。。要不然会出现找不到函数的定义问题。。。注意，要不然会坑死你
                       openssl-ssl openssl-crypto
                       )
```
[CMake语法](https://cmake.org/cmake/help/latest/index.html)


AndroidStudio 文件目录显示
![结果显示](/uploads/Aria2编译/AndroidStudioCmakeList文件编写.png)

AndroidStuido编译的结果
![结果显示](/uploads/Aria2编译/Aria2Android编译结果.png)

![结果显示](/uploads/Aria2编译/Aria2生成的so.png)

### 其中遇到的问题记录

1.posix_memalign() undeclared identifier issue.
这个出现的原因就是：posix_memalign这个函数，在Api16以下是不存在的
解决的方案就是：这个需要我们提高Android Api的最低等级，也即是minSdkVersion 16

2. relocation overflow in R_ARM_THM_CALL
这个出现的原因就是：该错误通常意味着被链接的二进制文件中的一个太大而且跳转/重定位地址根本不适合该指令。
解决的方案就是 我们可以在Android.mk文件中添加 LOCAL_ARM_MODE := arm 如果是在CmakeList里面可以在app的build.gradle中配置 arguments '-DANDROID_ARM_MODE=arm'

3. FAILED: cmd.exe /C "cd . && D:\sdk\sdk\ndk-bundle\toolchains\llvm\prebuilt\windows-x86_64\bin\clang++.exe....
  global.c:48: error: undefined reference to 'ENGINE_load_builtin_engines'
  global.c:48: error: undefined reference to 'ENGINE_register_all_complete'
大致就是说在链接的时候找不到ENGINE_load_builtin_engines 跟  ENGINE_register_all_complete对应的函数实现，但是本质这俩个函数是存在的
这里涉及到了一个链接的先后顺序的问题，这个函数对应的库为openssl，这个库在openssh2中是有用到的
解决方案就是：将openssl库链接放到最后

总结：什么事情都不会那么简单，主要一步一步来，如果遇到问题，就要去解决问题，这个库从我接触到编译出来我想要的大致用了1个月的时间,看来我的耐心确实很好..哈哈