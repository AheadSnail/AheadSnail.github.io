---
layout: pager
title: 微信终端跨平台组件 Mars环境配置
date: 2020-10-19 10:44:59
tags: [Android,NDK,Mars]
description:  微信终端跨平台组件 Mars环境配置
---

### 概述

> 微信终端跨平台组件 Mars环境配置

<!--more-->

### 简介
首先看看关于Mars的介绍
> Mars 是微信官方的终端基础组件, 是一个业务性无关,平台性无关 使用C++ 编写的基础组件。目前已接入微信 Android、iOS、Mac、Windows、WP、UWP 等客户端。
注意：目前支持Android、iOS、Mac、Window,Wp 平台，其他平台会在后续的版本中很快支持
它主要包括以下几个部分：
Comm：基础库，包括socket、线程、消息队列、协程等基础工具；
Xlog：通用日志模块，充分考虑移动终端的特点，提供高性能、高可用、安全性、容错性的日志功能
SDT：网络诊断模块；
STN：信令传输网络模块，负责终端与服务器的小数据信令通道。包含了微信终端在移动网络上的大量优化经验与成果，经历了微信海量用户的考验。

整体的架构
![结果显示](/uploads/Mars/mars架构.png)
[Mars Github地址](https://github.com/Tencent/mars#mars_cn)

> 微信的Mars组件，作为微信客户端的一部分，多年数以亿计的用户验证，是非常值得期待的，对于我来说，这俩年都在做NDK，也编写过一些跨平台的代码，对于Mars我是非常的有兴趣的，深入的了解,肯定会学到很多有用的内容，先不说网络优化这方面，单纯的基础库 (Comm)就能学到很多的内容，所以接下来就是深入的了解这个库, 这里直接从Master分支拿到最新的代码，但是拿到代码之后使用AndroidStudio直接打开是不能编译通过的，查看官网的编译可以知道，内部要使用 python 来执行对应不同平台 编译脚本，而且很多人使用Mars都是直接拿到编译的so，直接放到应用程序当中直接使用，这样就不能看到Mars的精华所在，而且这种方式配置的话，是不能调试查看代码的，而我们知道AndroidStudio是可以直接配置识别的，而且如果配置得当运行的时候是可以调试的，为了后续代码的更方便的查看我们需要一个这样的环境

### 目标
可以在AndroidStudio中配置一个方便看代码调试代码的环境 


### Linux平台的编译
至于为什么需要在Linux下面编译是因为我们需要在采集编译的参数配置等,也为了更好的理解编译脚本，为后续移植到AndroidStudio做好足够的基础，编译的脚本可以查看官网的介绍
[Mars 编译介绍](https://github.com/Tencent/mars/wiki/Mars-Android-%E6%8E%A5%E5%85%A5%E6%8C%87%E5%8D%97)

首先要配置NDK_ROOT的环境，配置完成之后，执行 python build_android.py 结果
```java
 Enter menu:
 1. Clean && build mars.
 2. Build incrementally mars.
 3. Clean && build xlog.
 4. Exit
在选择1、2选项时，编译出文件为单独的 libc++_shared.so 。
在/mars/libraries/mars_android_sdk/libs/armeabi 目录下。

在选择3选项时，编译出文件为单独的 libc++_shared.so 。
在/mars/libraries/mars_xlog_sdk/libs/armeabi 目录下。

编译出来的文件都是libc++_shared.so，没有生成marsxlog.so文件

当然最简单的就是直接选择123，接着执行编译,在编译的过程中会出现下面的问题
/mapped_file.hpp:295: error: undefined reference to 'mars_boost::iostreams::mapped_file_source::is_open() const'
/mapped_file.hpp:299: error: undefined reference to 'mars_boost::iostreams::mapped_file_source::flags() const'
```

解决方案是
```java
mars/log CMakeLists.txt中修改:
if(MSVC)
add_definitions(/FI"../../comm/projdef.h")

include_directories(../comm/windows)
include_directories(../comm/windows/zlib)
elseif(ANDROID)
file(GLOB SELF_ANDROID_SRC_FILES RELATIVE ${PROJECT_SOURCE_DIR}
../comm/xlogger/xloggerbase.c
jni/*.cc
../mk_template/JNI_OnLoad.cpp)

list(APPEND SELF_SRC_FILES ${SELF_ANDROID_SRC_FILES})

get_filename_component(EXPORT_EXP_FILE jni/export.exp ABSOLUTE)
set (CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -Wl,--version-script=${EXPORT_EXP_FILE}")
//增加:
link_directories(../comm)
link_libraries(comm)
link_directories(../boost)
link_libraries(mars-boost)

endif()
```

如果中间没有出现问题，那么编译的结果会这样
![结果显示](/uploads/Mars/linux编译成功.png)


so所在的位置
![结果显示](/uploads/Mars/编译生成的so.png)

由于内部是采用CmakeList来构建的，我们知道在编译的时候是会生成MakeFile的，所以我们可以在Linux下面使用Eclipse来打开这个工程,我们只要将配置对应的MakeFile就能让这个工程识别
![结果显示](/uploads/Mars/Eclipse下面查看.png)

虽然这样配置是可以的，而且非Android JNI的接口都不会报错的，而且查看代码的跳转也是没有问题的，而且可以直接编译,但是对于JNI的接口是不能识别到的
![结果显示](/uploads/Mars/Eclipse识别成功.png)

由于我们编译的是Android下面的so，所以我们也无法直接在linux下面直接运行这个so,所以我们接下来就是要配置到AndroidStudio中，当然这次Linux的编译可以用来收集编译的信息，比如我们查看整体是怎么样编译的，对应的编译脚本是怎么样的，只有理解清楚，我们才能在AndroidStudio中执行编译，对应不理解的地方，比如参数什么的，我们可以在 CmakeList的 message来输出，能更好的理解编译的脚本

### 脚本执行的过程
```MakeFile
cmake_minimum_required (VERSION 3.6)

#设置变量
set(CMAKE_INSTALL_PREFIX "${CMAKE_BINARY_DIR}" CACHE PATH "Installation directory" FORCE)
message(STATUS "CMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX}")

include_directories(openssl/include)

#添加子目录，每个子目录下面都有对应的CmakeList脚本，可以做到类似模块化的编译
add_subdirectory(comm comm)
add_subdirectory(boost boost)
add_subdirectory(app app)
add_subdirectory(baseevent baseevent)
add_subdirectory(log xlog)
add_subdirectory(sdt sdt)
add_subdirectory(stn stn)
 
# for zstd  首先配置编译的为静态库
option(ZSTD_BUILD_STATIC "BUILD STATIC LIBRARIES" ON)
option(ZSTD_BUILD_SHARED "BUILD SHARED LIBRARIES" OFF)
#配置zstd的路径变量
set(ZSTD_SOURCE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/zstd")
set(LIBRARY_DIR ${ZSTD_SOURCE_DIR}/lib)
include(GNUInstallDirs)
#添加子目录，这里要添加的是zstd 真正要编译的CmakeList的路径目录
add_subdirectory(zstd/build/cmake/lib zstd)

project (mars)

#包含其他的CmakeList文件,类似于C/C++中的include 使用
include(comm/CMakeUtils.txt)

include_directories(.)
include_directories(..)

set(SELF_LIBS_OUT ${CMAKE_SYSTEM_NAME}.out)

#如果是Android环境下的编译
if(ANDROID)

	#找到系统的log，libz库
    find_library(log-lib log)
    find_library(z-lib z)

	#该指令的作用主要是指定要链接的库文件的路径，该指令有时候不一定需要。因为find_package和find_library指令可以得到库文件的绝对路径。
	#不过你自己写的动态库文件放在自己新建的目录下时，可以用该指令指定该目录的路径以便工程能够找到。
    link_directories(app baseevent log sdt stn comm boost zstd)

    # marsxlog 首先编译的 marsxlog.so所以这里要设置编译的so的名字
    set(SELF_LIB_NAME marsxlog)

	#file可以用来搜索符合的cc编译文件, 其实这里的文件都是Android对应的接口文件,其他的内容都是库的内容，与平台无关
    file(GLOB SELF_SRC_FILES libraries/mars_android_sdk/jni/JNI_OnLoad.cc
            libraries/mars_xlog_sdk/jni/import.cc)
	
	#首先将上面搜索出来的文件，编译 该指令的主要作用就是将指定的源文件生成链接文件，然后添加到工程中去
    add_library(${SELF_LIB_NAME} SHARED ${SELF_SRC_FILES})
    install(TARGETS ${SELF_LIB_NAME} LIBRARY DESTINATION ${SELF_LIBS_OUT} ARCHIVE DESTINATION ${SELF_LIBS_OUT})

	#设置连接的参数
    get_filename_component(EXPORT_XLOG_EXP_FILE libraries/mars_android_sdk/jni/export.exp ABSOLUTE)
    set(SELF_XLOG_LINKER_FLAG "-Wl,--gc-sections -Wl,--version-script='${EXPORT_XLOG_EXP_FILE}'") 
    if(ANDROID_ABI STREQUAL "x86_64" OR ANDROID_ABI STREQUAL "x86")
        set(SELF_XLOG_LINKER_FLAG "-Wl,--gc-sections -Wl,--version-script='${EXPORT_XLOG_EXP_FILE}' -Wl,--no-warn-shared-textrel") 
    endif()
	
	#之后将执行连接操作，因为这个库要使用到很多其他模块的内容，所以这里要将其他模块的内容，链接起来
    target_link_libraries(${SELF_LIB_NAME} "${SELF_XLOG_LINKER_FLAG}"
                            xlog
                            mars-boost
                            comm
                            libzstd_static
                            ${log-lib}
                            ${z-lib}
                            )
    #　测试..............                        
    message( WARNING  "test ..................")
 	
	#接下来是 编译 marsstn
    set(SELF_LIB_NAME marsstn)

	#file可以用来搜索符合的cc编译文件
    file(GLOB SELF_SRC_FILES libraries/mars_android_sdk/jni/*.cc)

	# 该指令的主要作用就是将指定的源文件生成链接文件，然后添加到工程中去。该指令常用的语法如下：
    add_library(${SELF_LIB_NAME} SHARED ${SELF_SRC_FILES})

    install(TARGETS ${SELF_LIB_NAME} LIBRARY DESTINATION ${SELF_LIBS_OUT} ARCHIVE DESTINATION ${SELF_LIBS_OUT})
    link_directories(${SELF_LIBS_OUT})

if(NOT MSVC)
    set(CMAKE_FIND_LIBRARY_PREFIXES "lib")
    set(CMAKE_FIND_LIBRARY_SUFFIXES ".so" ".a")
endif()
	#找到openssl路径下面的对应的.a文件，对于这里就是 libssl.a和libcrypto.a 这里要注意的是找的时候要合适的平台来查找
    find_library(CRYPT_LIB crypto PATHS openssl/openssl_lib_android/${ANDROID_ABI} NO_DEFAULT_PATH NO_CMAKE_FIND_ROOT_PATH)
    find_library(SSL_LIB ssl PATHS openssl/openssl_lib_android/${ANDROID_ABI} NO_DEFAULT_PATH NO_CMAKE_FIND_ROOT_PATH)

	#测试
    message( WARNING  "test 1231..................")

	#之后将执行连接操作，因为这个库要使用到很多其他模块的内容，所以这里要将其他模块的内容，链接起来
    target_link_libraries(${SELF_LIB_NAME} "-Wl,--gc-sections"
                        ${log-lib}
                        stn
                        sdt
                        app
                        baseevent
                        comm
                        mars-boost
                        marsxlog
                        ${SSL_LIB}
                        ${CRYPT_LIB})

elseif(APPLE)

endif()
```

接着我们随便看一个子目录下面的模块是怎么样编译的，比如这里选择 add_subdirectory(log xlog)

```MakeFile
cmake_minimum_required (VERSION 3.6)

#设置变量
set(CMAKE_INSTALL_PREFIX "${CMAKE_BINARY_DIR}" CACHE PATH "Installation directory" FORCE)
message(STATUS "CMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX}")

project (xlog)

set(SELF_LIBS_OUT ${CMAKE_SYSTEM_NAME}.out)

#包含其他的CmakeList文件
include(../comm/CMakeUtils.txt)
include(../comm/CMakeExtraFlags.txt)

#指定要引入的头文件
include_directories(.)
include_directories(src)
include_directories(..)
include_directories(../..)
include_directories(../comm)
include_directories(../crypt)
include_directories(../../..)
include_directories(../crypt/micro-ecc-master)

#找到合适的文件，这里要查找的是 src 下面的cc文件 ，之后将查找到内容 添加到 SELF_SRC_FILES 
file(GLOB SELF_TEMP_SRC_FILES RELATIVE ${PROJECT_SOURCE_DIR} src/*.cc src/*.h)
source_group(src FILES ${SELF_TEMP_SRC_FILES})
list(APPEND SELF_SRC_FILES ${SELF_TEMP_SRC_FILES})

##找到合适的文件，这里要查找的是 crypt 下面的cc文件 ，之后将查找到内容 添加到 SELF_SRC_FILES 
file(GLOB SELF_TEMP_SRC_FILES RELATIVE ${PROJECT_SOURCE_DIR} crypt/*.cc crypt/*.h)
source_group(crypt FILES ${SELF_TEMP_SRC_FILES})
list(APPEND SELF_SRC_FILES ${SELF_TEMP_SRC_FILES})

##找到合适的文件，这里要查找的是 crypt/ 下面的cc文件 ，之后将查找到内容 添加到 SELF_SRC_FILES 
file(GLOB SELF_TEMP_SRC_FILES RELATIVE ${PROJECT_SOURCE_DIR} crypt/micro-ecc-master/*.c crypt/micro-ecc-master/*.h)
source_group(crypt\\micro-ecc-master FILES ${SELF_TEMP_SRC_FILES})
list(APPEND SELF_SRC_FILES ${SELF_TEMP_SRC_FILES})


if(MSVC)
    add_definitions(/FI"../../comm/projdef.h")

    include_directories(../comm/windows)
    include_directories(../comm/windows/zlib)
    
elseif(ANDROID)
	#查找log下的 Android的脚本文件
    file(GLOB SELF_ANDROID_SRC_FILES RELATIVE ${PROJECT_SOURCE_DIR}
            ../comm/xlogger/xloggerbase.c
            ../comm/xlogger/xlogger.cc
            jni/*.cc
            ../mk_template/JNI_OnLoad.cpp)
        
	#将查找到的内容，添加到 SELF_SRC_FILES
    list(APPEND SELF_SRC_FILES ${SELF_ANDROID_SRC_FILES})
	
    get_filename_component(EXPORT_EXP_FILE jni/export.exp ABSOLUTE)
    set (CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -Wl,--version-script=${EXPORT_EXP_FILE}")
    
	#指定要依赖的模块
    link_directories(../comm)
    link_libraries(comm)
    
    #1，link_libraries用在add_executable之前，target_link_libraries用在add_executable之后
	#link_libraries用来链接静态库，target_link_libraries用来链接导入库，即按照header file + .lib + .dll方式隐式调用动态库的.lib库
    
	#指定要依赖的模块
    link_directories(../boost)
    link_libraries(mars-boost)	

endif()

#add_library(${PROJECT_NAME} STATIC ${SELF_SRC_FILES})
#install(TARGETS ${PROJECT_NAME} ARCHIVE DESTINATION ${SELF_LIBS_OUT})

BuildWithUnitTest("${PROJECT_NAME}" "${SELF_SRC_FILES}")

可以看到这里并没有类似编译成log.a这样的配置，也就是类似 add_library，其实这样的写法，放到了 include(../comm/CMakeUtils.txt)，前面有把他包含进来，而且最后的一个调用BuildWithUnitTest
其实就是调用到这个CmakeList的函数

message(STATUS "==============config ${PROJECT_NAME}====================")

# just debug;release
set(CMAKE_CONFIGURATION_TYPES "Debug;Release" CACHE STRING "" FORCE)
#添加宏的定义
add_definitions(-DXLOGGER_TAG="mars::${PROJECT_NAME}")
set_property(GLOBAL PROPERTY USE_FOLDERS ON)

if(MSVC)
    # add DEBUG macro .. release has NDEBUG defaultly
    set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} /DDEBUG")

    if(CMAKE_CL_64)
        add_definitions(-D_WIN64 -DWIN64)
    endif()

    add_definitions(-D_WIN32 -DWIN32 -DUNICODE -D_UNICODE -DNOMINMAX -D_LIB)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /MP /Zc:threadSafeInit-")
    
    # generate pdb file
    set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /Zi")
    set(CMAKE_SHARED_LINKER_FLAGS_RELEASE "${CMAKE_EXE_LINKER_FLAGS_RELEASE} /DEBUG /OPT:REF /OPT:ICF")
 
    set(CompilerFlags
        CMAKE_CXX_FLAGS
        CMAKE_CXX_FLAGS_DEBUG
        CMAKE_CXX_FLAGS_RELEASE
        CMAKE_C_FLAGS
        CMAKE_C_FLAGS_DEBUG
        CMAKE_C_FLAGS_RELEASE)
        
    foreach(CompilerFlag ${CompilerFlags})
        string(REPLACE "/MD" "/MT" ${CompilerFlag} "${${CompilerFlag}}")
    endforeach()

else()
    set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -O0 -DDEBUG")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fno-exceptions")
    
endif()
    

if(ANDROID)
    #配置编译的配置
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fpic -std=gnu++14")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fdata-sections")

elseif(APPLE)
    # for gen xcode project file
    set(CMAKE_XCODE_ATTRIBUTE_CLANG_CXX_LANGUAGE_STANDARD "gnu++14")
    set(CMAKE_XCODE_ATTRIBUTE_CLANG_CXX_LIBRARY "libc++")

    # for build
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=gnu++14 -stdlib=libc++")
    set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} -g -gline-tables-only")
    set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -g -gline-tables-only")
    set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} -g -O0")
    set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -g -O0")
    
    if (NOT DEFINED ENABLE_BITCODE)
        set(ENABLE_BITCODE TRUE CACHE BOOL "Wheter or not to enable bitcode")
    endif()

    if (ENABLE_BITCODE)
        set(CMAKE_XCODE_ATTRIBUTE_ENABLE_BITCODE "YES")
    else()
        set(CMAKE_XCODE_ATTRIBUTE_ENABLE_BITCODE "NO")
    endif()

    set(CMAKE_XCODE_ATTRIBUTE_STRIP_STYLE "Debuging")
    set(CMAKE_XCODE_ATTRIBUTE_IPHONEOS_DEPLOYMENT_TARGET "10.0")
    set(CMAKE_XCODE_ATTRIBUTE_OTHER_LDFLAGS "-ObjC")
    set(CMAKE_XCODE_ATTRIBUTE_DEBUG_INFORMATION_FORMAT "dwarf-with-dsym")
    set(CMAKE_XCODE_ATTRIBUTE_CLANG_DEBUG_INFORMATION_LEVEL[variant=Debug] "default")
    set(CMAKE_XCODE_ATTRIBUTE_CLANG_DEBUG_INFORMATION_LEVEL[variant=Release] "line-tables-only")

    if(DEFINED IOS_DEPLOYMENT_TARGET)
        message(STATUS "setting IOS_DEPLOYMENT_TARGET=${IOS_DEPLOYMENT_TARGET}")
        set(CMAKE_XCODE_ATTRIBUTE_IPHONEOS_DEPLOYMENT_TARGET "${IOS_DEPLOYMENT_TARGET}")
    endif()

elseif(UNIX)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 -fPIC -ffunction-sections -fdata-sections -Os")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fPIC -ffunction-sections -fdata-sections -Os")
endif()
 
#刚才  BuildWithUnitTest("${PROJECT_NAME}" "${SELF_SRC_FILES}")  就会执行到这个方法
macro(BuildWithUnitTest projname sourcefiles)
    set(SRCFILES "${sourcefiles}")
    #　测试..............                        
    message( WARNING  "BuildWithUnitTest ......... ${sourcefiles}")
    
    if (UNITTEST)  
        add_library(${projname}.test STATIC ${SRCFILES})
        install(TARGETS ${projname}.test ARCHIVE DESTINATION ${CMAKE_SYSTEM_NAME}.out)
        message( WARNING  "BuildWithUnitTest test add_library......... ${projname}")
    else()
        list(FILTER SRCFILES EXCLUDE REGEX ".*_unittest.cc$")
        list(FILTER SRCFILES EXCLUDE REGEX ".*_mock.cc$")

		message( WARNING  "BuildWithUnitTest add_library......... ${projname}")
		
		#这里构建的是静态文件
        add_library(${projname} STATIC ${SRCFILES})
        install(TARGETS ${projname} ARCHIVE DESTINATION ${CMAKE_SYSTEM_NAME}.out)
    endif()
endmacro()
```

其他的模块也是类似这样的配置,就不一一介绍,接下来就是直接在AndroidStudio下面的配置了

### AndroidStudio下面的配置
这里就不区分模块了，这里直接将每个模块要真正参与编译的内容，查找出来，直接编译，下面的是编译maxlog的配置
```MakeFile
cmake_minimum_required(VERSION 3.6)

set(PATH_TO_MEDIACORE ${CMAKE_SOURCE_DIR}/mars)

include_directories(${CMAKE_SOURCE_DIR}/)
include_directories(${PATH_TO_MEDIACORE}/)
include_directories(${PATH_TO_MEDIACORE}/openssl/include)
include_directories(${PATH_TO_MEDIACORE}/comm)
include_directories(${PATH_TO_MEDIACORE}/comm/xlogger)
include_directories(${PATH_TO_MEDIACORE}/boost)
include_directories(${PATH_TO_MEDIACORE}/libraries/mars_android_sdk/jni)
include_directories(${PATH_TO_MEDIACORE}/sdt)
include_directories(${PATH_TO_MEDIACORE}/stn)
include_directories(${PATH_TO_MEDIACORE}/app)
include_directories(${PATH_TO_MEDIACORE}/zstd/lib/)
include_directories(${PATH_TO_MEDIACORE}/zstd/lib/common/)
include_directories(${PATH_TO_MEDIACORE}/zstd/lib/compress/)
include_directories(${PATH_TO_MEDIACORE}/zstd/lib/decompress/)
include_directories(${PATH_TO_MEDIACORE}/zstd/lib/dictBuilder/)
include_directories(${PATH_TO_MEDIACORE}/zstd/lib/deprecated/)

#App
file(GLOB FILES_APP_SRC "${PATH_TO_MEDIACORE}/app/src/*.cc")
file(GLOB FILES_APP "${PATH_TO_MEDIACORE}/app/*.cc")
file(GLOB FILES_APP_JNI "${PATH_TO_MEDIACORE}/app/jni/*.cc")

#baseevent
file(GLOB FILES_BASE_EVENT "${PATH_TO_MEDIACORE}/baseevent/*.cc")
file(GLOB FILES_BASE_EVENT_SRC "${PATH_TO_MEDIACORE}/baseevent/src/*.cc")
file(GLOB FILES_BASE_EVENT_JNI "${PATH_TO_MEDIACORE}/baseevent/jni/*.cc")
#boost
add_definitions(-DBOOST_NO_EXCEPTIONS)
set(SELF_BOOST_SRC_FILES
        ${PATH_TO_MEDIACORE}/boost/libs/atomic/src/lockpool.cpp
        ${PATH_TO_MEDIACORE}/boost/libs/date_time/src/gregorian/date_generators.cpp
        ${PATH_TO_MEDIACORE}/boost/libs/date_time/src/gregorian/gregorian_types.cpp
        ${PATH_TO_MEDIACORE}/boost/libs/date_time/src/gregorian/greg_month.cpp
        ${PATH_TO_MEDIACORE}/boost/libs/date_time/src/gregorian/greg_weekday.cpp
        ${PATH_TO_MEDIACORE}/boost/libs/date_time/src/posix_time/posix_time_types.cpp
        ${PATH_TO_MEDIACORE}/boost/libs/exception/src/clone_current_exception_non_intrusive.cpp
        ${PATH_TO_MEDIACORE}/boost/libs/filesystem/src/codecvt_error_category.cpp
        ${PATH_TO_MEDIACORE}/boost/libs/filesystem/src/operations.cpp
        ${PATH_TO_MEDIACORE}/boost/libs/filesystem/src/path.cpp
        ${PATH_TO_MEDIACORE}/boost/libs/filesystem/src/path_traits.cpp
        ${PATH_TO_MEDIACORE}/boost/libs/filesystem/src/portability.cpp
        ${PATH_TO_MEDIACORE}/boost/libs/filesystem/src/unique_path.cpp
        ${PATH_TO_MEDIACORE}/boost/libs/filesystem/src/utf8_codecvt_facet.cpp
        ${PATH_TO_MEDIACORE}/boost/libs/filesystem/src/windows_file_codecvt.cpp
        ${PATH_TO_MEDIACORE}/boost/libs/iostreams/src/file_descriptor.cpp
        ${PATH_TO_MEDIACORE}/boost/libs/iostreams/src/mapped_file.cpp
        ${PATH_TO_MEDIACORE}/boost/libs/smart_ptr/src/sp_collector.cpp
        ${PATH_TO_MEDIACORE}/boost/libs/smart_ptr/src/sp_debug_hooks.cpp
        ${PATH_TO_MEDIACORE}/boost/libs/system/src/error_code.cpp
        ${PATH_TO_MEDIACORE}/boost/libs/thread/src/future.cpp)

file(GLOB SELF_BOOST_ANDROID_SRC_FILE
        ${PATH_TO_MEDIACORE}/boost/libs/coroutine/src/*.cpp
        ${PATH_TO_MEDIACORE}/boost/libs/coroutine/src/detail/*.cpp
        ${PATH_TO_MEDIACORE}/boost/libs/coroutine/src/posix/*.cpp
        ${PATH_TO_MEDIACORE}/boost/libs/context/src/*.cpp
        ${PATH_TO_MEDIACORE}/boost/libs/context/src/posix/*.cpp)
enable_language(ASM)
file(GLOB SELF_BOOST_ASM_FILES
        ${PATH_TO_MEDIACORE}/boost/libs/context/src/asm/jump_arm_aapcs_elf_gas.S
        ${PATH_TO_MEDIACORE}/boost/libs/context/src/asm/make_arm_aapcs_elf_gas.S)
#comm
#file(GLOB FILES_BASE_EVENT_JNI "${PATH_TO_MEDIACORE}/baseevent/jni/*.cc")
file(GLOB FILES_COMMOM_LAYER
        ${PATH_TO_MEDIACORE}/comm/*.cc
        ${PATH_TO_MEDIACORE}/comm/*.c )
file(GLOB FILES_COMMOM_ASSERT_LAYER ${PATH_TO_MEDIACORE}/comm/assert/*.c)
file(GLOB FILES_COMMOM_CRYPT_LAYER
        ${PATH_TO_MEDIACORE}/comm/crypt/*.cc
        ${PATH_TO_MEDIACORE}/comm/crypt/*.c)
file(GLOB FILES_COMMOM_NETWORK_LAYER
        ${PATH_TO_MEDIACORE}/comm/network/*.cc
        ${PATH_TO_MEDIACORE}/comm/network/*.c
        )
file(GLOB FILES_COMMOM_SOCKET_LAYER ${PATH_TO_MEDIACORE}/comm/socket/*.cc)
file(GLOB FILES_COMMOM_XLOGGER_LAYER
        ${PATH_TO_MEDIACORE}/comm/xlogger/*.c
        ${PATH_TO_MEDIACORE}/comm/xlogger/*.cc)
file(GLOB FILES_COMMOM_COREPATTERN_LAYER ${PATH_TO_MEDIACORE}/comm/corepattern/*.cc)
file(GLOB FILES_COMMOM_DNS_LAYER ${PATH_TO_MEDIACORE}/comm/dns/*.cc)
file(GLOB FILES_COMMOM_MESSAGEQUEUE_LAYER ${PATH_TO_MEDIACORE}/comm/messagequeue/*.cc)
file(GLOB FILES_COMMOM_SOCKET ${PATH_TO_MEDIACORE}/comm/unix/socket/*.cc)
add_definitions(-DUSING_XLOG_WEAK_FUNC)
file(GLOB FILES_COMMOM_JNI
        ${PATH_TO_MEDIACORE}/comm/android/*.cc
        ${PATH_TO_MEDIACORE}/comm/android/*.c
        ${PATH_TO_MEDIACORE}/comm/jni/*.cc
        ${PATH_TO_MEDIACORE}/comm/jni/*.c
        ${PATH_TO_MEDIACORE}/comm/jni/util/*.cc)
#log
file(GLOB SELF_LOG_FILES  ${PATH_TO_MEDIACORE}/log/src/*.cc)
file(GLOB SELF_LOG_CRYPT_SRC_FILES ${PATH_TO_MEDIACORE}/log/crypt/*.cc)
file(GLOB SELF_LOG_CRYPT_MICO_FILES ${PATH_TO_MEDIACORE}/log/crypt/micro-ecc-master/*.c)
file(GLOB SELF_LOG_JNI_INTERFACE_FILES ${PATH_TO_MEDIACORE}/log/jni/*.cc )
#sdt
file(GLOB SELF_SDT_FILES ${PATH_TO_MEDIACORE}/sdt/*.cc)
file(GLOB SELF_SDT_SRC_FILES ${PATH_TO_MEDIACORE}/sdt/src/*.cc)
file(GLOB SELF_SDT_ACTIVECHECK_SRC_FILES ${PATH_TO_MEDIACORE}/sdt/src/activecheck/*.cc)
file(GLOB SELF_SDT_CHECK_IMPL_SRC_FILES ${PATH_TO_MEDIACORE}/sdt/src/checkimpl/*.cc)
file(GLOB SELF_SDT_CHECK_TOOLS_FILES ${PATH_TO_MEDIACORE}/sdt/src/tools/*.cc)
file(GLOB SELF_SDT_ANDROID_SRC_FILES ${PATH_TO_MEDIACORE}/sdt/jni/*.cc)
#stn
file(GLOB SELF_STN_SRC_FILES ${PATH_TO_MEDIACORE}/stn/src/*.cc)
file(GLOB SELF_STN_FILES  ${PATH_TO_MEDIACORE}/stn/*.cc )
file(GLOB SELF_STN_ANDROID_SRC_FILES ${PATH_TO_MEDIACORE}/stn/jni/*.cc)

file(GLOB SELF_LOG_JNI_FILES ${CMAKE_SOURCE_DIR}/mars/libraries/mars_xlog_sdk/jni/import.cc)
file(GLOB SELF_MARS_ANDROID_LIBRARY_FILES
        ${CMAKE_SOURCE_DIR}/mars/libraries/mars_android_sdk/jni/JNI_OnLoad.cc
        ${CMAKE_SOURCE_DIR}/mars/libraries/mars_android_sdk/jni/longlink_packer.cc
        ${CMAKE_SOURCE_DIR}/mars/libraries/mars_android_sdk/jni/shortlink_packer.cc )

#zsd
file(GLOB ZSTD_CommonSources ${PATH_TO_MEDIACORE}/zstd/lib/common/*.c)
file(GLOB ZSTD_CompressSources ${PATH_TO_MEDIACORE}/zstd/lib/compress/*.c)
file(GLOB ZSTD_DecompressSources ${PATH_TO_MEDIACORE}/zstd/lib/decompress/*.c)
file(GLOB ZSTD_DictBuilderSources ${PATH_TO_MEDIACORE}/zstd/lib/dictBuilder/*.c)
file(GLOB ZSTD_DeprecatedSources ${PATH_TO_MEDIACORE}/zstd/lib/deprecated/*.c)

add_library(marlog SHARED
        ${FILES_APP_SRC}
        ${FILES_APP}
        ${FILES_APP_JNI}
        ${FILES_BASE_EVENT}
        ${FILES_BASE_EVENT_SRC}
        ${FILES_BASE_EVENT_JNI}
        ${SELF_BOOST_SRC_FILES}
        ${SELF_BOOST_ANDROID_SRC_FILE}
        ${SELF_BOOST_ASM_FILES}
        ${FILES_COMMOM_LAYER}
        ${FILES_COMMOM_ASSERT_LAYER}
        ${FILES_COMMOM_CRYPT_LAYER}
        ${FILES_COMMOM_NETWORK_LAYER}
        ${FILES_COMMOM_SOCKET_LAYER}
        ${FILES_COMMOM_XLOGGER_LAYER}
        ${FILES_COMMOM_COREPATTERN_LAYER}
        ${FILES_COMMOM_DNS_LAYER}
        ${FILES_COMMOM_MESSAGEQUEUE_LAYER}
        ${FILES_COMMOM_SOCKET}
        ${FILES_COMMOM_JNI}
        ${SELF_SDT_FILES}
        ${SELF_SDT_SRC_FILES}
        ${SELF_SDT_ACTIVECHECK_SRC_FILES}
        ${SELF_SDT_CHECK_IMPL_SRC_FILES}
        ${SELF_SDT_CHECK_TOOLS_FILES}
        ${SELF_SDT_ANDROID_SRC_FILES}
        ${SELF_STN_SRC_FILES}
        ${SELF_STN_FILES}
        ${SELF_STN_ANDROID_SRC_FILES}
        ${SELF_LOG_JNI_FILES}
        #log
        ${SELF_LOG_FILES}
        ${SELF_LOG_CRYPT_SRC_FILES}
        ${SELF_LOG_CRYPT_MICO_FILES}
        ${SELF_LOG_JNI_INTERFACE_FILES}
        ${SELF_MARS_ANDROID_LIBRARY_FILES}
        #zstd
        ${ZSTD_CommonSources}
        ${ZSTD_CompressSources}
        ${ZSTD_DecompressSources}
        ${ZSTD_DictBuilderSources}
        ${ZSTD_DeprecatedSources}
        )
# 添加两个预编译库
add_library(openssl-crypto
        STATIC
        IMPORTED)
set_target_properties(openssl-crypto
        PROPERTIES IMPORTED_LOCATION
        ${CMAKE_SOURCE_DIR}/mars/openssl/openssl_lib_android/${ANDROID_ABI}/libcrypto.a )
add_library(openssl-ssl
        STATIC
        IMPORTED)
set_target_properties(openssl-ssl
        PROPERTIES IMPORTED_LOCATION
        ${CMAKE_SOURCE_DIR}/mars/openssl/openssl_lib_android/${ANDROID_ABI}/libssl.a )

find_library(log-lib log)
find_library(z-lib z)

#下面是连接的操作
get_filename_component(EXPORT_XLOG_EXP_FILE mars/libraries/mars_android_sdk/jni/export.exp ABSOLUTE)
set(SELF_XLOG_LINKER_FLAG "-Wl,--gc-sections -Wl,--version-script='${EXPORT_XLOG_EXP_FILE}'")
target_link_libraries(marlog "${SELF_XLOG_LINKER_FLAG}"
        ${log-lib}
        ${z-lib}
        openssl-crypto
        openssl-ssl
        )	
```

编译成功，可以看到编译的文件达到185个
![结果显示](/uploads/Mars/AndroidStudio编译成功.png)

AndroidStudio下面每个函数都方便查看和跳转，没有出现不能识别的错误
![结果显示](/uploads/Mars/AndroidStudio识别成功.png)


### 参考链接
1. [Tencent/mars](https://github.com/Tencent/mars#android_cn)
2. [系统-编译失败](https://github.com/Tencent/mars/issues/842)
3. [Mars Android 接入指南](https://github.com/Tencent/mars/wiki/Mars-Android-%E6%8E%A5%E5%85%A5%E6%8C%87%E5%8D%97)
4. [cmake学习笔记](https://blog.csdn.net/bigdog_1027/article/details/79113342)

















