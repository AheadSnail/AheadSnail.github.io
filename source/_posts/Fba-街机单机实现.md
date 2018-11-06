---
layout: pager
title: Fba 街机单机实现
date: 2018-10-05 15:55:30
tags: [Android,NDK,Fba]
description:  Fba Android版本 街机单机实现
---

### 概述

> Fba Android版本 街机单机实现

<!--more-->

### 简介

这里总结下之前做的街机项目，要不然过段时间之后会忘的一干二净，目前街机主流的好几种，比如 fc,Fba,Mame,小鸡等， 而目前做的比较好的就是悟饭的游戏厅了，里面集成了多种的街机版本,但是Fba仍然是他们的主流，之前有尝试过使用Mame，但是因为项目工程太大，编译出来的项目也是非常大，所以通过分析悟饭的做法，采用了Fba，而且最主要的一个原因是Window下的实现是支持对战的，对于其他的fc,小鸡等官网不支持对战的，所以我们参考他对应window的实现

> 这里是Fba的官网  https://www.fbalpha.com/

### 实现的效果

单机的效果
![结果显示](/uploads/Fba/单机效果.png)

本地对战效果 ，玩家A显示
![结果显示](/uploads/Fba/玩家A效果.gif)

玩家B显示
![结果显示](/uploads/Fba/玩家B效果.gif)

### Fba Window编译

编译Window 其实在官网就有体现 https://www.fbalpha.com/compile/ ，按照他的编译效果最终编译结果为

![结果显示](/uploads/Fba/AFbaWindow编译后的结果.png)

双击编译之后的exe文件，就可以打开这个游戏了，我们可以找到对应的游戏文件，放到roms文件夹下面，扫描roms，选择游戏，就可以玩游戏了
![结果显示](/uploads/Fba/Window单机实现.png)

### Fba Android编译

Fba 单机的实现，其实在Github上面就有了，链接地址为 https://github.com/Cpasjuste/libarcade ，当然这个直接下载下来是不能运行的，缺少了部分的内容，接下来就是修改对应的错误，错误修改完之后，由于这个项目放在是5年前开发的，现在的Fba官网早就更新了多个版本，所以接下来我们要参照他编译的makefile 整合当前最新的代码 而且可以参考https://github.com/libretro/fbalpha 这个是当前最新的Fba  可以编译出对应的lib库的实现，当然也是支持Android对应的编译实现，所以可以结合这俩个项目来整理，所以下面来分析下这个项目的MakeFile

```MakeFile
首先是  Application.mk文件 
APP_ABI			:= armeabi-v7a , x86
APP_PLATFORM		:= android-9
APP_STL			:= stlport_static
NDK_TOOLCHAIN_VERSION	:= 4.9
APP_SHORT_COMMANDS := true

APP_SHORT_COMMANDS := true 当编译的项目比较大的时候，在链接的时候会出现错误，大致的意思就是链接的内容过长，此时可以指定这个选项

接着是 最外层的 MakeFile.mk文件
include $(call all-subdir-makefiles)
LOCAL_SHORT_COMMANDS := true

include $(call all-subdir-makefiles)意思就是说   当前目录下没有需要编译的文件，请向子目录深入  这样可以做到不会将所有的编译脚本写在一起，类似 分模块编译

src 里面 Android.mk 文件的内容
LOCAL_PATH := $(call my-dir)
MY_PATH := $(LOCAL_PATH)

include $(CLEAR_VARS)
LOCAL_PATH := $(MY_PATH)
include $(LOCAL_PATH)/android/AndroidInclude.mk
include $(LOCAL_PATH)/burn/Android.mk
include $(LOCAL_PATH)/burner/Android.mk
include $(LOCAL_PATH)/cpu/Android.mk
include $(LOCAL_PATH)/dep/libs/lib7z/Android.mk
include $(LOCAL_PATH)/dep/libs/libpng/Android.mk

LOCAL_MODULE := arcade
LOCAL_ARM_MODE   := arm

ANDROID_OBJS := android/android.cpp android/android_main.cpp android/android_run.cpp \
		android/android_softfx.cpp android/android_sdlfx.cpp \
		android/android_input.cpp android/android_snd.cpp \
		android/android_stated.cpp

INTF_DIR := $(LOCAL_PATH)/intf
INTF_OBJS := $(wildcard $(INTF_DIR)/*.cpp)

INTF_AUDIO_DIR := $(LOCAL_PATH)/intf/audio
INTF_AUDIO_OBJS := $(wildcard $(INTF_AUDIO_DIR)/*.cpp)

INTF_AUDIO_SDL_DIR := $(LOCAL_PATH)/intf/audio/sdl
INTF_AUDIO_SDL_OBJS := $(wildcard $(INTF_AUDIO_SDL_DIR)/*.cpp)

INTF_INPUT_DIR := $(LOCAL_PATH)/intf/input
INTF_INPUT_OBJS := $(wildcard $(INTF_INPUT_DIR)/*.cpp)

INTF_INPUT_SDL_DIR := $(LOCAL_PATH)/intf/input/sdl
INTF_INPUT_SDL_OBJS := $(wildcard $(INTF_INPUT_SDL_DIR)/*.cpp)

INTF_VIDEO_DIR := $(LOCAL_PATH)/intf/video
INTF_VIDEO_OBJS := $(wildcard $(INTF_VIDEO_DIR)/*.cpp)

INTF_VIDEO_SDL_DIR := $(LOCAL_PATH)/intf/video/sdl
INTF_VIDEO_SDL_OBJS := $(wildcard $(INTF_VIDEO_SDL_DIR)/*.cpp)

LOCAL_SRC_FILES += $(ANDROID_OBJS) \
		$(LIB7Z) $(LIBPNG) $(BURN) $(BURNER) $(CPU) \
		$(subst $(LOCAL_PATH)/,,$(INTF_OBJS)) \
		$(subst $(LOCAL_PATH)/,,$(INTF_AUDIO_OBJS)) \
		$(subst $(LOCAL_PATH)/,,$(INTF_AUDIO_SDL_OBJS)) \
		$(subst $(LOCAL_PATH)/,,$(INTF_INPUT_OBJS)) \
		$(subst $(LOCAL_PATH)/,,$(INTF_INPUT_SDL_OBJS)) \
		$(subst $(LOCAL_PATH)/,,$(INTF_VIDEO_OBJS)) \
		$(subst $(LOCAL_PATH)/,,$(INTF_VIDEO_SDL_OBJS))

#$(warning $(LOCAL_SRC_FILES)) 
LOCAL_SRC_FILES := $(filter-out intf/input/sdl/inp_sdl.cpp, $(LOCAL_SRC_FILES))
LOCAL_SRC_FILES := $(filter-out intf/video/vid_softfx.cpp, $(LOCAL_SRC_FILES))
LOCAL_SRC_FILES := $(filter-out intf/video/sdl/vid_sdlopengl.cpp, $(LOCAL_SRC_FILES))
LOCAL_SRC_FILES := $(filter-out intf/video/sdl/vid_sdlfx.cpp, $(LOCAL_SRC_FILES))
LOCAL_SRC_FILES := $(filter-out intf/audio/sdl/aud_sdl.cpp, $(LOCAL_SRC_FILES))
LOCAL_SRC_FILES := $(filter-out intf/audio/aud_interface.cpp, $(LOCAL_SRC_FILES))
LOCAL_SRC_FILES := $(filter-out burner/sdl/main.cpp, $(LOCAL_SRC_FILES))
LOCAL_SRC_FILES := $(filter-out burner/sdl/media.cpp, $(LOCAL_SRC_FILES))
LOCAL_SRC_FILES := $(filter-out burner/sdl/run.cpp, $(LOCAL_SRC_FILES))
LOCAL_SRC_FILES := $(filter-out burner/sdl/stated.cpp, $(LOCAL_SRC_FILES))
LOCAL_SRC_FILES := $(filter-out burn/drv/pgm/pgm_sprite_create.cpp, $(LOCAL_SRC_FILES))
LOCAL_SRC_FILES := $(filter-out cpu/adsp2100_intf.cpp, $(LOCAL_SRC_FILES))
LOCAL_SRC_FILES := $(filter-out cpu/tms34010_intf.cpp, $(LOCAL_SRC_FILES))
LOCAL_SRC_FILES := $(filter-out cpu/mips3_intf.cpp, $(LOCAL_SRC_FILES))
LOCAL_SRC_FILES := $(filter-out cpu/upd7810/7810ops.c, $(LOCAL_SRC_FILES))
LOCAL_SRC_FILES := $(filter-out cpu/upd7810/7810tbl.c, $(LOCAL_SRC_FILES))

LOCAL_STATIC_LIBRARIES := minizip
LOCAL_SHARED_LIBRARIES := SDL kailleraDemo
LOCAL_LDLIBS := -lGLESv1_CM -llog -lz

include $(BUILD_SHARED_LIBRARY)

首先这里编译的目标文件为arcade ,他需要依赖 一个静态库minizip ，并且需要俩个动态库  SDL kailleraDemo
LOCAL_SHARED_LIBRARIES: 
表示模块在运行时要依赖的共享库（动态库），在链接时就需要，以便在生成文件时嵌入其相应的信息。
LOCAL_LDLIBS: 
编译模块时要使用的附加的链接器选项。这对于使用‘-l’前缀传递指定库的名字是有用的。

使用第三方的动态库，静态库需要采用预编译( BUILD_SHARED_LIBRARY 或 PREBUILT_STATIC_LIBRARY)

比如我们的对战库，虽然也是自己编译的源文件，但是如果只是提供一个so的化，就要类似这样的 方式来编译进来，叫做预编译，针对第三方提供的so，或者lib，如果是源码的化就不是这样

LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)
LOCAL_MODULE := kailleraDemo
LOCAL_SRC_FILES := $(TARGET_ARCH_ABI)/libkailleraDemo.so
LOCAL_EXPORT_C_INCLUDES := $(LOCAL_PATH)
include $(PREBUILT_SHARED_LIBRARY)

LOCAL_STATIC_LIBRARIES LOCAL_SHARED_LIBRARY  来加入自己制订的动态库或者是静态库,比如这里的 minizip  SDL kailleraDemo 等

minizip 库编译脚本
LOCAL_PATH:= $(call my-dir)
include $(CLEAR_VARS)
common_SRC_FILES := \
adler32.c \
compress.c \
crc32.c \
deflate.c \
gzclose.c \
gzlib.c \
gzread.c \
gzwrite.c \
infback.c \
inffast.c \
inflate.c \
inftrees.c \
trees.c \
uncompr.c \
zutil.c

common_CFLAGS := -O3
LOCAL_SRC_FILES := $(common_SRC_FILES)
LOCAL_CFLAGS += $(common_CFLAGS)
LOCAL_MODULE:= minizip
LOCAL_ARM_MODE   := arm
include $(BUILD_STATIC_LIBRARY)

如果使用的系统库(静态库 / 动态库 ) 采用如下即可实现:  LOCAL_LDLIBS := -lm -llog -ljnigraphics -lz

include $(LOCAL_PATH)/android/AndroidInclude.mk  MakeFile的include 跟c语言中的Include是一样的，也即是将整个文件包含进来,注意了在include的时候要注意路径，$(LOCAL_PATH)
表示当前MakeFile的目录，所以当前的include 其他的文件，都是相对这个目录来说的，这里为什么要包含进来，是这样的对于下面要编译的源文件会有体现 
比如 $(LIB7Z) $(LIBPNG) $(BURN) $(BURNER) $(CPU) 都是在对应的不同目录下的MakeFile 这样做的好处也相当于是分模块的意思

LOCAL_SRC_FILES += $(ANDROID_OBJS) \
		$(LIB7Z) $(LIBPNG) $(BURN) $(BURNER) $(CPU) \
		$(subst $(LOCAL_PATH)/,,$(INTF_OBJS)) \
		$(subst $(LOCAL_PATH)/,,$(INTF_AUDIO_OBJS)) \
		$(subst $(LOCAL_PATH)/,,$(INTF_AUDIO_SDL_OBJS)) \
		$(subst $(LOCAL_PATH)/,,$(INTF_INPUT_OBJS)) \
		$(subst $(LOCAL_PATH)/,,$(INTF_INPUT_SDL_OBJS)) \
		$(subst $(LOCAL_PATH)/,,$(INTF_VIDEO_OBJS)) \
		$(subst $(LOCAL_PATH)/,,$(INTF_VIDEO_SDL_OBJS))

		
这里看一个 include $(LOCAL_PATH)/cpu/Android.mk 
CPU_PATH := $(call my-dir)

CPU := 	    $(subst $(LOCAL_PATH)/,, \
			$(wildcard $(CPU_PATH)/*.cpp) \
			$(wildcard $(CPU_PATH)/arm/*.cpp) \
			$(wildcard $(CPU_PATH)/arm7/*.cpp) \
			$(wildcard $(CPU_PATH)/h6280/*.cpp) \
			$(wildcard $(CPU_PATH)/hd6309/*.cpp) \
			$(wildcard $(CPU_PATH)/i8039/*.cpp) \
			$(wildcard $(CPU_PATH)/i8051/*.cpp) \
			$(wildcard $(CPU_PATH)/konami/*.cpp) \
			$(wildcard $(CPU_PATH)/m6502/*.cpp) \
			$(wildcard $(CPU_PATH)/m6800/*.cpp) \
			$(wildcard $(CPU_PATH)/m6805/*.cpp) \
			$(wildcard $(CPU_PATH)/m6809/*.cpp) \
			$(wildcard $(CPU_PATH)/m68k/m68kcpu.c) \
			$(wildcard $(CPU_PATH)/m68k/m68kopac.c) \
			$(wildcard $(CPU_PATH)/m68k/m68kopdm.c) \
			$(wildcard $(CPU_PATH)/m68k/m68kopnz.c) \
			$(wildcard $(CPU_PATH)/m68k/m68kops.c) \
			$(wildcard $(CPU_PATH)/nec/*.cpp) \
			$(wildcard $(CPU_PATH)/pic16c5x/*.cpp) \
			$(wildcard $(CPU_PATH)/s2650/*.cpp) \
			$(wildcard $(CPU_PATH)/sh2/*.cpp) \
			$(wildcard $(CPU_PATH)/tlcs90/*.cpp) \
			$(wildcard $(CPU_PATH)/tms32010/*.cpp) \
			$(wildcard $(CPU_PATH)/upd7725/*.cpp) \
			$(wildcard $(CPU_PATH)/upd7810/*.c*) \
			$(wildcard $(CPU_PATH)/v60/*.cpp) \
			$(wildcard $(CPU_PATH)/z80/*.cpp))
			
这里要注意 $(call my-dir) 跟 $(LOCAL_PATH) 的区别，我们可以通过MakeFile的打印日志来打印具体的内容，比如在当前的MakeFile下添加下面的这俩行，下面是打印的结果
$(warning "Test....." $(call my-dir))
$(warning "Test....." $(LOCAL_PATH))
```
打印的结果显示

![结果显示](/uploads/Fba/call_dir结果.png)

通过结果可以知道，$(call my_dir) 代表的是当前makefile的目录 所以为 jni/src/cpu 而$(LOCAL_PATH) 结果为  jni/src  所以这个代表的是src MakeFile的目录，为什么会这样，因为当前的这个Makefile是 src 的Android.mk文件包含进来的，而且在Android.mk文件中有这样的写法LOCAL_PATH := $(call my-dir),所以当前的Local_PATH 就为 当前这个src目录下的Makefile的路径，当然如果我在当前 cpu/路径下的Android.mk文件写上这句话，

![结果显示](/uploads/Fba/添加之后的结果.jpg)

可以看到当前 $(LOCAL_PATH) 重新改为了 jni/src/cpu  至于后面的错误，是因为你在这边改变了src/Android.mk文件的 LOCAL_PATH的内容，所以导致那边会出现这个问题，也即是多了前面的内容

![结果显示](/uploads/Fba/错误的结果.png)

搞懂了这俩个的区别，继续分析

```MakeFile
CPU := 	    $(subst $(LOCAL_PATH)/,, \
			$(wildcard $(CPU_PATH)/*.cpp) \
			$(wildcard $(CPU_PATH)/arm/*.cpp) \
			....
```		
首先 subst 函数 是MakeFile的字符串替换函数 ，比如 $(subst FROM, TO, TEXT)，即将字符串TEXT中的子串FROM变为TO。
wildcard 函数是用来获取目标文件的 比如一般我们可以使用“$(wildcard *.c)”来获取工作目录下的所有的.c文件列表。
复杂一些用法；可以使用“$(patsubst %.c,%.o,$(wildcard *.c))”，首先使用“wildcard”函数获取工作目录下的.c文件列表；
之后将列表中所有文件名的后缀.c替换为.o。这样我们就可以得到在当前目录可生成的.o文件列表。因此在一个目录下可以使用如下内容的Makefile来将工作目录下的所有的.c文件进行编译并最后连接成为
一个可执行文件

所以对于我们上面的写法就是获取当前 jni/src/cpu目录下对应的所有的.cpp文件，然后将他们jni/src 替换成空字符串，结果就变为/cpu/*.cpp类似的结果, 所以外面就可以这样调用了直接使用这个$(CPU)
就可以代表当前目录下的所有要编译的文件了

![结果显示](/uploads/Fba/截取之后的结果.png)

```MakeFile
LOCAL_SRC_FILES := $(filter-out intf/input/sdl/inp_sdl.cpp, $(LOCAL_SRC_FILES))  filter-out 函数可以做到过滤的作用，就是当前的inp_sdl.cpp文件从$(LOCAL_SRC_FILES)中移除,比如
$(filter-out $(mains),$(objects))  实现了去除变量“objects”中“mains”定义的字串（文件名）功能。它的返回值 为“foo.o bar.o”。

最后我们的头文件指定路径
LOCAL_C_INCLUDES := $(call my-dir) \
			$(INC_PATH)/../SDL/include $(INC_PATH) \
			$(INC_PATH)/burn $(INC_PATH)/burn/devices $(INC_PATH)/burn/drv $(INC_PATH)/burn/snd \
			$(INC_PATH)/burn/drv \
			....
这个头文件指定路径是非常重要的，如果不指定的化，就要改变源码的原有的include 代码，所以当你发现在编译源码的时候发现 某个源码文件里面找不到什么头文件的时候，就要查看是否有指定 LOCAL_C_INCLUDES			
```

### Eclipse中配置JNI

虽然AndroidStudio 中早就支持NDK的配置，包括俩种方式一种是Makfile,一种是CmakeList，虽然这俩种配置，都可以做到 方便的阅读代码，可以调试代码，但是这俩种对应MakeFile的书写，或者CmakeList的写法，支持的不够好，比如MakeFile的语法关键字就没有像Eclipse中这样有标红的显示，让人不知道是否有写对,我将这个Eclipse项目移植到AndroidStuio编译,对应JNI的代码是不用做任何修改的只要在build.gradle中配置一下就好

![结果显示](/uploads/Fba/buildGradle写法.png)
AndroidStudio Makefile显示
![结果显示](/uploads/Fba/AndroidStuioMakeFile的写法.png)

Eclipse Makefile显示
![结果显示](/uploads/Fba/EclipseMakeFile显示.png)

**是不是Eclipse 对这方面支持的更友好，下面介绍Eclipse中配置JNI：**

首先要在eclipse中配置NDK的路径，这里采用的是最新版的ndk和最新版的eclipse，在填写ndk的路径的时候，不能填写到根目录下面，一定要到对应的 build目录下，有ndk-build.cmd的这个目录下，才能识别，以前老的版本这个是在根目录下面的

![结果显示](/uploads/Fba/EclipseNdk路径配置.png)

新建一个普通的android项目，然后在项目的右键点击属性找到Android-Tools的选项，选择support-native

![结果显示](/uploads/Fba/Eclipse新建NDK项目.png)

会弹出一个对话框，叫你输入so的目标名称,点击完成，会默认的帮你添加这些东西

![结果显示](/uploads/Fba/Eclipse默认生成的内容.png)

编写对应的jni代码，会发现识别不了

![结果显示](/uploads/Fba/Eclipse不识别NDK.png)

选择项目的属性右键，选择propertity，在这个选项卡中

![结果显示](/uploads/Fba/Eclipse配置识别NDK.png)
![结果显示](/uploads/Fba/Eclipse配置识别NDK二.png)

添加你JNi对应的版本中的jni.h的路径

![结果显示](/uploads/Fba/NDK选择文件路径.png)

选择完之后，点击ok，再回来看

![结果显示](/uploads/Fba/Eclipse配置NDK结果.png)

就已经支持了，ctr可以进入jni.h的文件里面，说明支持完成,支持Eclipse 配置NDK完成


