---
layout: pager
title: Lame库编译 以及最新版NDK改动
date: 2018-07-21 14:48:22
tags: [Lame,Android,NDK]
description:  Lame库编译 以及最新版NDK改动
---

Lame库编译 以及最新版NDK改动
<!--more-->

****简介****
===
```
LAME 是最好的MP3编码器，编码高品质MP3的最好也是唯一的选择。LAME本身是控制台程序，需要加外壳程序才比较容易使用，也可以在别的软件(比如EAC)中间调用。是一款出色的MP3压缩程序，
它使用了独创的人体听音心理学模型和声学模型，改变了人们对MP3高音发哑、低音发破的音质的印象。 总结来说也就是可以用来压缩原始流，也即是pcm的数据，转为mp3

官网地址  ： http://lame.sourceforge.net/
```

****NDK改动****
===
```java
这俩天在使用Android 目前最新版本的NDK也就是 NDK-R17B 编译Lame库,原本这个库是非常的简单编译的，如果是在NDkR15b之前，但是在NdkR15之后，就有所改动了，下面分别使用ndkR14b 以及NdkR17b

NDK R14b 编译脚本

#!/bin/bash
ANDROID_HOME=/home/yuhui/workSpace/android-ndk-r14b
TOOLCHAIN=$ANDROID_HOME/toolchain
PATH=$TOOLCHAIN/bin:$PATH
HOST=arm-linux-androideabi
PREFIX=$ANDROID_HOME/usr/local
LOCAL_DIR=$ANDROID_HOME/usr/local
DEST=$ANDROID_HOME/usr/local
CC="$HOST-gcc -march=armv7-a"
CXX=$HOST-g++
LDFLAGS="-L$DEST/lib -march=armv7-a"
CPPFLAGS="-I$DEST/include -march=armv7-a"
CXXFLAGS=$CFLAGS


# c-ares library build
  PKG_CONFIG_PATH=$PREFIX/lib/pkgconfig/ LD_LIBRARY_PATH=$PREFIX/lib/ ./configure --host=$HOST --build=`dpkg-architecture -qDEB_BUILD_GNU_TYPE`
--prefix=$PREFIX  --enable-static --disable-shared --disable-frontend 

  make
  make install

echo "build lame complete!"

发现正常的编译成功，可以生成目标文件
```
![结果显示](/uploads/Lame/Lame生成结果.png)
![结果显示](/uploads/Lame/Lame项目结构.png)
```java
Mp3Encoder.cpp内容

#include "Mp3Encoder.h"
int Mp3Encoder::Init(const char *pcmFilePath, const char *mp3FilePath, int sampleRate, int channels, int bitRate)
{
    int ret = 0;
    pcmFile = fopen(pcmFilePath,"rb");
    if(pcmFile == NULL)
    {
        ret = -1;
        return ret;
    }

    mp3File = fopen(mp3FilePath,"wb");
    if(mp3File == NULL)
    {
        ret = -2;
        return ret;
    }
    //初始化lame 得到
    lameClient = lame_init();
    if(lameClient == NULL)
    {
        ret = -3;
        return ret;
    }
    //设置对应的参数
    lame_set_in_samplerate(lameClient,sampleRate);
    lame_set_out_samplerate(lameClient,sampleRate);
    lame_set_num_channels(lameClient,channels);
    lame_set_brate(lameClient,bitRate/1000);
    lame_init_params(lameClient);
    return ret;
}

void Mp3Encoder::Encoder() {
    //每次读取一段bufferSize大小的pcm数据buffer，然后再编码该buffer，但是在编码buffer之前得把该buffer的左右声道分开
    //再送入Lame编码器，最终将编码之后的数据写入到Mp3文件中
    int bufferSize = 1024 * 256;
    //分配内存空间,分配一个大小为bufferSize/2 数量的 short数组
    short * buffer = new short[bufferSize/2];
    short * leftBuffer = new short[bufferSize/4];
    short * rightBuffer = new short[bufferSize/4];

    //用来存储编码之后的内容
    unsigned char * mp3_buffer = new unsigned char[bufferSize];
    size_t readBufferSize = 0;
    //读取pcm的数据，这里是每次读取2个字节，读取数量为bufferSize/2,short所以每次读取2个字节
    while((readBufferSize = fread(buffer,2,bufferSize/2,pcmFile)) > 0)
    {
        //左右声道分开
        for(int i = 0;i<readBufferSize;i++)
        {
            if(i % 2 == 0)
            {
                leftBuffer[i/2] = buffer[i];
            }else{
                rightBuffer[i/2] = buffer[i];
            }
        }

        //编码
        size_t wroteSize = lame_encode_buffer(lameClient,leftBuffer,rightBuffer,readBufferSize/2,mp3_buffer,bufferSize);
        //写文件内容
        fwrite(mp3_buffer,1,wroteSize,mp3File);
    }

    //释放内存
    delete[] buffer;
    delete[] leftBuffer;
    delete[] rightBuffer;
    delete[] mp3_buffer;
}

Mp3Encoder::Mp3Encoder()
{

}

Mp3Encoder::~Mp3Encoder()
{
    if(pcmFile)
    {
        fclose(pcmFile);
        pcmFile = NULL;
    }
    if(mp3File)
    {
        fclose(mp3File);
        mp3File = NULL;
    }
    if(lameClient)
    {
        lame_close(lameClient);
        lameClient = NULL;
    }
}

void Mp3Encoder::Destory() {
    if(pcmFile)
    {
        fclose(pcmFile);
        pcmFile = NULL;
    }
    if(mp3File)
    {
        fclose(mp3File);
        mp3File = NULL;
    }
    if(lameClient)
    {
        lame_close(lameClient);
        lameClient = NULL;
    }
}

Mp3Encoder.h内容

#ifndef LAMEMP3ENCODER_MP3ENCODER_H
#define LAMEMP3ENCODER_MP3ENCODER_H

#include <cstdio>
#include <lame/lame.h>

class Mp3Encoder
{
private:
    FILE* pcmFile;
    FILE* mp3File;
    lame_t lameClient;

public:
    Mp3Encoder();
    ~Mp3Encoder();
    int Init(const char * pcmFilePath,const char * mp3FilePath,int sampleRate,int channels,int bitRate);
    void Destory();
    void Encoder();
};

#endif //LAMEMP3ENCODER_MP3ENCODER_H

native-lib.cpp文件内容

#include <jni.h>
#include "com_example_administrator_lamemp3encoder_Mp3Encoder.h"
#include "Mp3Encoder.h"

Mp3Encoder * encoder;


JNIEXPORT jint JNICALL Java_com_example_administrator_lamemp3encoder_Mp3Encoder_init
        (JNIEnv * env, jobject obj, jstring pcmFilePath, jint audioChannels, jint bitRate, jint sampleRate, jstring mp3FilePath)
{
    const char * pcmPath = env->GetStringUTFChars(pcmFilePath,NULL);
    const char * mp3Path = env->GetStringUTFChars(mp3FilePath,NULL);

    //构建一个对象
    encoder = new Mp3Encoder();
    int ret = encoder->Init(pcmPath,mp3Path,sampleRate,audioChannels,bitRate);
    env->ReleaseStringUTFChars(pcmFilePath,pcmPath);
    env->ReleaseStringUTFChars(mp3FilePath,mp3Path);
    return ret;
}


JNIEXPORT void JNICALL Java_com_example_administrator_lamemp3encoder_Mp3Encoder_encode
        (JNIEnv * env, jobject obj)
{
    encoder->Encoder();
}


JNIEXPORT void JNICALL Java_com_example_administrator_lamemp3encoder_Mp3Encoder_destroy
        (JNIEnv *env, jobject obj)
{
    encoder->Destory();
}

CMakeLists.txt文件内容:

cmake_minimum_required(VERSION 3.4.1)
add_library( open-lame
             STATIC
             IMPORTED)
set_target_properties( open-lame
                       PROPERTIES IMPORTED_LOCATION
                       ${CMAKE_SOURCE_DIR}/src/main/cpp/lib/libmp3lame.a )

#添加头文件的查找目录
include_directories( ${CMAKE_SOURCE_DIR}/src/main/cpp/include )


add_library( # Sets the name of the library.
             native-lib

             # Sets the library as a shared library.
             SHARED

             # Provides a relative path to your source file(s).
             src/main/cpp/Mp3Ecnoder.cpp
             src/main/cpp/native-lib.cpp )


find_library( # Sets the name of the path variable.
              log-lib

              # Specifies the name of the NDK library that
              # you want CMake to locate.
              log )


target_link_libraries( # Specifies the target library.
                       native-lib

                       # Links the target library to the log library
                       # included in the NDK.
                       ${log-lib}

                       open-lame )
					   
					   
Android Studio NDK 版本情况					   
```
![结果显示](/uploads/Lame/NDK最新版本.png)
![结果显示](/uploads/Lame/Android Studio NDK设置.png)
```java				   
通过上图可以知道，当前使用NDK为最新版		
之后执行编译会产生下面的这样的结果：
```
![结果显示](/uploads/Lame/错误日志.jpg)				  				   
```java
下面是这个问题的反馈  https://github.com/android-ndk/ndk/issues/445#issuecomment-313322546

大致就是说 NDK r14之前，我们为每个API版本提供了一组libc标头。在许多情况下，这些标头不正确。许多公开的API不存在，而其他API则没有公开API。
在NDK r14中（作为选择加入功能），我们将它们统一到一组标题中。在r15中，默认使用这些。此单个标头路径用于每个平台级别。使用API​​级别防护 也即是说在R14是可以的,
我们可以改local.properties 中NDK的路径为n14 然后试试，确实可以编译通过，链接也没有问题

下面来说说怎么样解决使用最新版的NDK出现的问题，答案可以从下面的链接得到结果
https://android.googlesource.com/platform/ndk/+/ndk-r15-release/docs/UnifiedHeaders.md

我实验了前面的几种解决方案					   
```
![结果显示](/uploads/Lame/失败方案.png)
```java
发现前面的这几种方案，在最新版的Ndk R17b 已经不生效了，甚至编译不通过，上面的哪篇文章中也介绍到，这种方案可能在Ndk R16之后就会移除，比如在编译编译链的时候使用
$NDK/build/tools/make_standalone_toolchain.py --deprecated-headers ...  会发现R17 根本就不支持 --deprecated-headers

下面是最后一种，也是可行的方案，而且刚刚好适合我们这种情况，使用了开源项目是使用AutoConf构建的 原理就是使用Clang 而不要使用Gcc 了，Gcc会在Ndk R18被抛弃掉
```
![结果显示](/uploads/Lame/可行方案.png)
![结果显示](/uploads/Lame/推荐使用Clang.png)
```java

交叉编译链实现:

#!/bin/bash
ANDROID_HOME=/home/yuhui/workSpace/android-ndk-r17b
$ANDROID_HOME/build/tools/make_standalone_toolchain.py \
   --arch arm --api 16 --stl=libc++ \
   --install-dir $ANDROID_HOME/toolchain

上面这个是最新版的交叉编译设置 这里要注意的一点就是，最好这个--api的设置跟你android应用程序的 minSdkVersion 16 保持一直，对于--stl的选项，本身是有几种的
```
![结果显示](/uploads/Lame/支持的stl设置.png)
```java
但是如果你设置其他的化 他会有提示,比如这里设置str=gnustl，会有这样的提示
```
![结果显示](/uploads/Lame/stl设置过时的信息.png)
```java
经过测试只有设置libc++才不会有这个警告

NdkR17编译脚本实现

#!/bin/bash
ANDROID_HOME=/home/yuhui/workSpace/android-ndk-r17b
TOOLCHAIN=$ANDROID_HOME/toolchain
HOST=arm-linux-androideabi
PREFIX=/home/yuhui/workSpace/FFmpeg/LameBuild
DEST=$ANDROID_HOME/usr/local

export CC="$TOOLCHAIN/bin/arm-linux-androideabi-clang -march=armv7-a"
export CXX="$TOOLCHAIN/bin/arm-linux-androideabi-clang++ -march=armv7-a"
export LDFLAGS="-L$DEST/lib -pie -march=armv7-a"
export CPPFLAGS="-I$DEST/include -march=armv7-a"
export CXXFLAGS=$CPPFLAGS
export CFLAGS=$CPPFLAGS

./configure \
	--host=$HOST \
	--prefix=$PREFIX \
	--enable-static \
	--disable-shared \
	--disable-frontend 

  make
  make install

echo "build lame complete!"

发现也可以编译通过，最终也能生成目标文件，这里要注意的是对于 export CC="$TOOLCHAIN/bin/arm-linux-androideabi-clang -march=armv7-a" 中的export不能去掉，如果去掉的化，使用的就是系统
默认的Gcc了，对于到底是不是使用了我们设置的clang来编译，我们可以在Lame库的源码下面，默认在编译的时候，会生成一个config.log文件，我们可以从里面得到信息
```
![结果显示](/uploads/Lame/使用Clang.png)
```java
如果我把export去掉的化

#!/bin/bash
ANDROID_HOME=/home/yuhui/workSpace/android-ndk-r17b
TOOLCHAIN=$ANDROID_HOME/toolchain
HOST=arm-linux-androideabi
PREFIX=/home/yuhui/workSpace/FFmpeg/LameBuild
DEST=$ANDROID_HOME/usr/local
CC="$TOOLCHAIN/bin/arm-linux-androideabi-clang -march=armv7-a"
CXX="$TOOLCHAIN/bin/arm-linux-androideabi-clang++ -march=armv7-a"
LDFLAGS="-L$DEST/lib -pie -march=armv7-a"
CPPFLAGS="-I$DEST/include -march=armv7-a"
CXXFLAGS=$CPPFLAGS
CFLAGS=$CPPFLAGS

./configure \
	--host=$HOST \
	--prefix=$PREFIX \
	--enable-static \
	--disable-shared \
	--disable-frontend 

  make
  make install

echo "build lame complete!"

虽然可以编译通过，也能生成目标文件，但是这里使用的是gcc，系统Linux默认的环境，这个是错误的,对应的我们可以查看编译的文件config.log内容
```
![结果显示](/uploads/Lame/使用系统gcc生成的文件.png)
```java
最新版的NDK不支持armeabi了，对于build.gradle下面的这个写法，不能写为armeabi了，如果写了会得到这样的错误

指定编译的版本
ndk {
    abiFilters 'armeabi-v7a'
}
```
![结果显示](/uploads/Lame/最新版NDK不支持armeabi.png)
```java
最终我们将编译出来的内容，拷贝出来，只要替换到libmp3lame.a 库就可以了，然后执行编译，生成的结果为：
```
![结果显示](/uploads/Lame/编译成功.jpg)

****测试生成的结果****
===
```java
NDK Native方法声明：

public class Mp3Encoder
{
    public native int init(String pcmFilePath,int audioChannels,int bitRate,int sampleRate,String mp3FilePath);

    public native void encode();

    public native void destroy();
}

public class MainActivity extends AppCompatActivity
{
    private Mp3Encoder mp3Encoder;

    static
    {
        System.loadLibrary("native-lib");
    }

    @Override
    protected void onCreate(Bundle savedInstanceState)
    {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        mp3Encoder = new Mp3Encoder();

        findViewById(R.id.button).setOnClickListener(new View.OnClickListener()
        {
            @Override
            public void onClick(View view)
            {
                String pcmPath = "/mnt/sdcard/voice441000.pcm";
                int audioChannels = 2;
                int bitRate = 128 * 1024;
                int sampleRate = 44100;
                String mp3Path = "/mnt/sdcard/vocal.mp3";
                int ret = mp3Encoder.init(pcmPath,audioChannels,bitRate,sampleRate,mp3Path);
                if(ret != 0)
                {
                    Toast.makeText(MainActivity.this,"初始化失败",Toast.LENGTH_SHORT).show();
                    mp3Encoder.destroy();
                    return;
                }

                mp3Encoder.encode();
                mp3Encoder.destroy();
            }
        });

    }
}

```
![结果显示](/uploads/Lame/检测Lame库是否生成正确.png)




