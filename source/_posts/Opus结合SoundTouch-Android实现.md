---
layout: pager
title: Opus结合SoundTouch Android实现
date: 2018-07-08 21:57:07
tags: [Opus,SoundTouch,NDK,Android]
description:  Opus结合SoundTouch Android实现
---

Opus结合SoundTouch Android实现
<!--more-->

****简介****
===
```
前面一篇文章介绍到SoundTouch可以做到变声的效果，大致的效果是可以做到变速，变音调，还可以结合使用，官网也有提供Android版本的实现，也即是上一篇文章中介绍到的
下载下来的源码目录里面有一个Android-lib的目录，甚至连NDK的Android.mk文件都写好了，是一个可以在Eclipse直接运行的

SoundTouch的优缺点:
优点是:
SoundTouch是一个很小的开源库，源码文件很少，是纯c++写法
缺点是:
SoundTouch 提供的变音效果不够多，最起码是比QQ的效果少
SoundTouch只能操作Wav文件，不能操作其他格式的文件
```

****目标****
===
```
查看SoundTouch的demo可以知道，他操作的也是文件的形式，也就是要给出一个源文件，以及处理后要生成的目标文件，由于操作的为WAV的无损的格式，所以输出的文件也是非常的巨大的
对应传输来说，是非常不友好的，QQ的变音处理是这样的，首先录制声音的时候会生成一个pcm格式的文件，然后当你选择变音的时候，会在边操作边生成一个目标文件，目标文件名为
.slk格式，每次选择一种变音就会相应的生成一个目标文件，这样估计是防止多次的压缩处理吧，当你点击发送成功的时候，会删除刚刚变音生成的目标文件，包括原本的pcm数据
只会保留一个当前发送的变音源文件,QQ存放录音路径为:/tencent/MobileQQ/qq号码/ppt/后面是加上时间格式的文件夹/.....slk文件

使用Opus结合SoundTouch能实现一个效果，我们可以在录音的时候，就压缩成一个标准的Opus文件，如果有选择变音，那么我们可以在播放的时候，使用Opus解压缩成一个原始流
然后传递给SoundTouch处理，将处理之后的结果，再丢给AudioTrack来播放，这样就能做到一个效果，我们可只保留一个源文件，而不会生成对应的中间产物

```

****编译****
===
```java
首先要修改SoundTouch的代码，因为demo默认处理的是一个完整的文件，我们要改成流的形式，下面是关键的代码

/**
*
* @param handle  c++中的对象指针
* @param inputData 要处理的源数据
* @param inputLength 要处理的源数据的长度
* @param outputData  处理完之后接受内容的buff
* @param outputMaxLength  最大可以处理的长度
* @return
*/
private native final int processFile(long handle, short [] inputData,int inputLength, short [] outputData,int outputMaxLength);

/**
* 刷新缓冲区的内容
* @param handle
* @param outputData 处理完之后接受内容的buff
* @param outputMaxLength  最大可以处理的长度
* @return
*/
private native final int flushData(long handle,short [] outputData,int outputMaxLength);

JNIEXPORT jint JNICALL Java_example_com_opussoundtouchdemo_SoundTouch_flushData
(JNIEnv *env, jobject thiz, jlong handle, jshortArray outputArray, jint outputMaxLength)
{
    int ret = 0;
    //标识每次传递的数据，处理之后，能得到多少的字节
    int pageCount = 0;
    SoundTouch *ptr = (SoundTouch*)handle;
    int nSamples = 0;

    jshort* output_data = env->GetShortArrayElements(outputArray,0);
    ptr->flush();
    do
    {
         nSamples = ptr->receiveSamples(output_data+pageCount, outputMaxLength);
         if(nSamples > 0)
         {
            //获取处理完之后的内容，写到目标文件里面
            pageCount += nSamples;
         }

    } while (nSamples != 0);

    //释放内存
     env->ReleaseShortArrayElements(outputArray, output_data, 0);

     if (env->ExceptionCheck())
     {
         ret = -1;
         return ret;
     }
     //返回最终处理的结果长度,因为java里面不能传递一个指针，只能通过返回值来做处理
     return pageCount;
}

//处理数据
JNIEXPORT jint JNICALL Java_example_com_opussoundtouchdemo_SoundTouch_processFile
(JNIEnv *env, jobject thiz, jlong handle, jshortArray inputArray, jint inputLength, jshortArray outputArray,jint outputMaxLength)
{
    int ret = 0;
    //标识每次传递的数据，处理之后，能得到多少的字节
    int pageCount = 0;
    SoundTouch *ptr = (SoundTouch*)handle;
    int nSamples = 0;

    //将java中的数组变成c对应的数组
    jshort* input_data = env->GetShortArrayElements(inputArray,0);
    jshort* output_data = env->GetShortArrayElements(outputArray,0);

     LOGD("processFile SoundTouch::processFile: inputLength %d", inputLength);

    try
    {

         nSamples = inputLength / 1;
         ptr->putSamples(input_data, nSamples);
         do
          {
               //outputArray+pageCount 指针的位移，防止覆盖之前的数据
               //receiveSamples 第一个参数接受要写数据的buff，第二个参数代表最多能写多少个
               nSamples = ptr->receiveSamples(input_data, inputLength);
                //获取处理完之后的内容，写到目标文件里面
                if(nSamples > 0)
                {
                    //内容的拷贝,因为short类型为2个字节，所以内容拷贝的话，要拷贝nSamples个short类型的话，就要*2,
                    memcpy(output_data + pageCount, input_data, nSamples * 2);
                    pageCount += nSamples;
                    LOGD("processFile SoundTouch pageCount == %d\n",pageCount);
                }
          } while (nSamples != 0);

    }
    catch (const runtime_error &e)
    {
    	const char *err = e.what();
        // An exception occurred during processing, return the error message
        LOGD("JNI exception in SoundTouch::processFile: %s", err);
         //抛出异常
         _setErrmsg(err);
         ret = -1;
         return ret;
    }
     //释放内存
     env->ReleaseShortArrayElements(inputArray, input_data, 0);
     env->ReleaseShortArrayElements(outputArray, output_data, 0);

     if (env->ExceptionCheck())
     {
       ret = -1;
       return ret;
     }
    //返回最终处理的结果长度,因为java里面不能传递一个指针，只能通过返回值来做处理
    return pageCount;
}

对应这里private native final int processFile(long handle, short [] inputData,int inputLength, short [] outputData,int outputMaxLength); 
为什么处理之后的返回的数据是一个short 类型的,原因是AudioTracker 处理float类型的数据有Api的限制，所以这里返回的结果为short类型


下面是文件的结构图
```
![结果显示](/uploads/SoundTouchopus/文件目录结构图.png)
```java
下面是对应的Android.mk文件编写
LOCAL_PATH := $(call my-dir)

$(warning ${LOCAL_PATH})

include $(CLEAR_VARS)
LOCAL_MODULE    := FLAc
LOCAL_SRC_FILES := ${LOCAL_PATH}/Opus/lib/libFLAC.a
include $(PREBUILT_STATIC_LIBRARY)

include $(CLEAR_VARS)
LOCAL_MODULE    := Ogg
LOCAL_SRC_FILES := ${LOCAL_PATH}/Opus/lib/libogg.a
include $(PREBUILT_STATIC_LIBRARY)


include $(CLEAR_VARS)
LOCAL_MODULE    := Opus
LOCAL_SRC_FILES := ${LOCAL_PATH}/Opus/lib/libopus.a
include $(PREBUILT_STATIC_LIBRARY)


include $(CLEAR_VARS)
LOCAL_CPPFLAGS := -Wall -std=c++11 -DANDROID -D__SOFTFP__ -frtti -DHAVE_PTHREAD -DSOUNDTOUCH_DISABLE_X86_OPTIMIZATIONS -finline-functions -ffast-math -O2 -fexceptions
LOCAL_C_INCLUDES = ${LOCAL_PATH}/SoundTouch/include
LOCAL_MODULE    := soundtouch

LOCAL_SRC_FILES := \
            ${LOCAL_PATH}/SoundTouch/src/AAFilter.cpp \
            ${LOCAL_PATH}/SoundTouch/src/FIFOSampleBuffer.cpp \
            ${LOCAL_PATH}/SoundTouch/src/FIRFilter.cpp \
            ${LOCAL_PATH}/SoundTouch/src/cpu_detect_x86.cpp \
            ${LOCAL_PATH}/SoundTouch/src/sse_optimized.cpp \
            ${LOCAL_PATH}/SoundTouch/src/RateTransposer.cpp \
            ${LOCAL_PATH}/SoundTouch/src/SoundTouch.cpp \
            ${LOCAL_PATH}/SoundTouch/src/InterpolateCubic.cpp \
            ${LOCAL_PATH}/SoundTouch/src/InterpolateLinear.cpp \
            ${LOCAL_PATH}/SoundTouch/src/InterpolateShannon.cpp \
            ${LOCAL_PATH}/SoundTouch/src/TDStretch.cpp \
            ${LOCAL_PATH}/SoundTouch/src/BPMDetect.cpp  \
            ${LOCAL_PATH}/SoundTouch/src/PeakFinder.cpp

include $(BUILD_STATIC_LIBRARY)


#此变量指向的构建脚本用于取消定义下面“开发者定义的变量”一节中列出的几乎全部 LOCAL_XXX 变量。（模块之前使用这个，会清除LOCAL的系统变量） 在描述新模块之前
include $(CLEAR_VARS)

LOCAL_MODULE        := opusSoundTouch

#这是要编译的源代码文件列表。 只要列出要传递给编译器的文件
LOCAL_SRC_FILES     := \
        ${LOCAL_PATH}/Opus/opustools/src/opus_header.c \
        ${LOCAL_PATH}/Opus/opustools/src/picture.c \
        ${LOCAL_PATH}/Opus/opustools/src/resample.c \
        ${LOCAL_PATH}/Opus/opustools/src/audio-in.c \
        ${LOCAL_PATH}/Opus/opustools/src/diag_range.c \
        ${LOCAL_PATH}/Opus/opustools/src/flac.c \
        ${LOCAL_PATH}/Opus/opustools/src/lpc.c \
        ${LOCAL_PATH}/Opus/opustools/win32/unicode_support.c \
        ${LOCAL_PATH}/Opus/opustools/src/wav_io.c \
        ${LOCAL_PATH}/Opus/opustools/src/opusinfo.c \
        ${LOCAL_PATH}/Opus/opustools/src/info_opus.c \
        ${LOCAL_PATH}/mydecoder.c \
        ${LOCAL_PATH}/myencode.c \
        ${LOCAL_PATH}/soundtouch-jni.cpp


LOCAL_C_INCLUDES := \
                    ${LOCAL_PATH}/Opus/include \
                    ${LOCAL_PATH}/Opus/include/opus \
                    ${LOCAL_PATH}/Opus/opustools/include \
                    ${LOCAL_PATH}/Opus/opustools/src \
                    ${LOCAL_PATH}/SoundTouch/include

LOCAL_CFLAGS 	:= -w -std=c11 -Os -fno-strict-aliasing -fprefetch-loop-arrays
LOCAL_CFLAGS 	+= -DHAVE_CONFIG_H -DOPUSTOOLS -DSPX_RESAMPLE_EXPORT= -DRANDOM_PREFIX=opustools -DOUTSIDE_SPEEX -DFLOATING_POINT -fno-math-errno -DANDROID -D__SOFTFP__

LOCAL_CPPFLAGS 	:= -DBSD=1 -ffast-math -Os -funroll-loops -std=c++11
LOCAL_CPPFLAGS  += -DHAVE_CONFIG_H -DOPUSTOOLS -DSPX_RESAMPLE_EXPORT= -DRANDOM_PREFIX=opustools -DOUTSIDE_SPEEX -DFLOATING_POINT -DANDROID -D__SOFTFP__

LOCAL_LDLIBS 	:= -llog

LOCAL_STATIC_LIBRARIES := FLAc Ogg Opus soundtouch

include $(BUILD_SHARED_LIBRARY)
```
上面的MakeFile文件中要注意的一个点
```java
这里要注意的一个是-DANDROID -D__SOFTFP__ 这个涉及到了SoundTouch 决定SAMPLETYPE 类型，在STTypes.h中有这样代码
#if (defined(__SOFTFP__) && defined(ANDROID))
        // For Android compilation: Force use of Integer samples in case that
        // compilation uses soft-floating point emulation - soft-fp is way too slow
        #undef  SOUNDTOUCH_FLOAT_SAMPLES
        #define SOUNDTOUCH_INTEGER_SAMPLES      1
#endif

#ifdef SOUNDTOUCH_INTEGER_SAMPLES
    // 16bit integer sample type
    typedef short SAMPLETYPE;
    // data type for sample accumulation: Use 32bit integer to prevent overflows
    typedef long  LONG_SAMPLETYPE;

    #ifdef SOUNDTOUCH_FLOAT_SAMPLES
    // check that only one sample type is defined
    #error "conflicting sample types defined"
    #endif // SOUNDTOUCH_FLOAT_SAMPLES

    #ifdef SOUNDTOUCH_ALLOW_X86_OPTIMIZATIONS
         / Allow MMX optimizations (not available in X64 mode)
        #if (!_M_X64)
            #define SOUNDTOUCH_ALLOW_MMX   1
        #endif
    #endif
#else

因为前面介绍SoundTouch native接口说到，要使用short 类型的 ，所以这里的SAMPLETYPE 类型也要为short，所以要加上这个宏的定义

Application.mk 文件的编写：
APP_PLATFORM := android-14
APP_ABI := armeabi-v7a
NDK_TOOLCHAIN_VERSION := 4.9
APP_STL := gnustl_static

#APP_OPTIM 将此可选变量定义为 release 或 debug。在构建应用的模块时可使用它来更改优化级别。
#APP_CFLAGS 此变量用于存储构建系统在为任何模块编译任何 C 或 C++ 源代码时传递到编译器的一组 C 编译器标志 也即是全局的
#APP_LDFLAGS 此变量包含构建系统在仅构建 C++ 源文件时传递到编译器的一组 C++ 编译器标志 也即是全局的
#APP_PLATFORM  此变量包含目标 Android 平台的名称。例如，android-3 指定 Android 1.5 系统映像
#APP_STL  默认情况下，NDK 构建系统为 Android 系统提供的最小 C++ 运行时库 (system/lib/libstdc++.so) 提供 C++ 标头
#NDK_TOOLCHAIN_VERSION 将此变量定义为 4.9 或 4.8 以选择 GCC 编译器的版本。 64 位 ABI 默认使用版本 4.9 ，32 位 ABI 默认使用版本 4.8


修改app目录下的build.gradle文件
defaultConfig{
    ...
    externalNativeBuild {
        ndkBuild {
	    //指定构建的架构
	    abiFilters "armeabi-v7a"
    }
}
	
//指定JNI的目录
sourceSets.main.jniLibs.srcDirs = ['/src/main/jni/']

externalNativeBuild {
        ndkBuild {
            //指定Android.mk文件的目录
            path "/src/main/jni/Android.mk"
    }
}

执行编译产生的结果：
```
![结果显示](/uploads/SoundTouchopus/OpusSoundTouch编译之后内容.png)

```java
录音部分跟之前的Opus一样，这里主要讲下播放部分,下面是界面的展示
```
![结果显示](/uploads/SoundTouchopus/界面视图.png)
```java
我们可以在播放的时候，设置好，音调跟音速 然后点击播放按钮

File sdCard = Environment.getExternalStorageDirectory();
File dir = new File(sdCard.getAbsolutePath() + this.audioFolder);
File audioFile = new File(dir, this.audioFile);

//配置，音高，配置节奏，配置 播放的速度
ProcessAudioModel audioModel = new ProcessAudioModel();
audioModel.setSpeed(1.0f);
audioModel.setPitch(Float.parseFloat(editPitch.getText().toString()));
audioModel.setTempo(0.01f * Float.parseFloat(editTempo.getText().toString()));
if (audioFile.exists())
{
    mOpusPlayer.startPlay(audioFile.getAbsolutePath(), audioModel);
}

会开启一个线程来执行解压缩
 public void startPlay(String fileName, ProcessAudioModel processAudioModel)
{
    this.mFileName = fileName;
    this.mProcessAudioModel = processAudioModel;
    try
    {
        mOpusDecoder = new OpusDecoder(this.mFileName);
    }
    catch (Exception e)
    {
        e.printStackTrace();
    }
    mShoudStopPlaying = false;
    mIsPlaying = true;
    //开启线程来播放
    RecordPlayThread rpt = new RecordPlayThread();
    Thread th = new Thread(rpt);
    th.start();
}

OpusDecoder 解码部分的实现：
public void decode(ProcessAudioModel processAudioModel) throws Exception
{
    SoundTouch soundTouch = new SoundTouch();
    //设置音高
    soundTouch.setTempo(processAudioModel.getTempo());
    //设置节奏
    soundTouch.setPitchSemiTones(processAudioModel.getPitch());
    //设置播放的速度
    soundTouch.setPlayRate(processAudioModel.getSpeed());

    File cacheFile = new File(mSrcDecoderpath);
    mFileInputStream = new FileInputStream(cacheFile);
    readBuffer = new byte[bufferSize];
    short[] decoded = new short[65535];
    short[] audioProcessOutput = new short[65535];
    while (true)
    {
        int read = mFileInputStream.read(readBuffer);
        if (read != bufferSize)//如果读取到了文件的结尾的化，返回值不是bufferSize大小
        {
            mIsLastPart = true;
        }

        //首先通过Opus解压缩为一个原始的数据流
        if ((mDecoderSize = nativeDecodeBytes(readBuffer,read, decoded)) > 0)
        {
            Log.d(TAG, "nativeDecodeBytes length "+mDecoderSize);
            //然后将处理之后的原始流，交给SoundTouch来处理，返回处理之后的流
            int processLength = soundTouch.processAudioData(decoded, mDecoderSize,audioProcessOutput,mDecoderSize);
            if(processLength > 0)
            {
                Log.d(TAG, "processAudioOutLength length "+processLength);
                //返回处理之后的流，直接用AudioTracker播放
                mBytesRead += track.write(audioProcessOutput, 0, processLength);
                track.setStereoVolume(1.0f, 1.0f);// 设置当前音量大小
                track.play();
            }
        }

        //读取到了文件的结尾
        if(mIsLastPart)
        {
            //到了文件的结尾，刷新缓冲区的内容
            int processLength = soundTouch.flushAudioData(audioProcessOutput,4096);
            if(processLength > 0)
            {
                Log.d(TAG, "processAudioOutLength length "+processLength);
                mBytesRead += track.write(audioProcessOutput, 0, processLength);
                track.setStereoVolume(1.0f, 1.0f);// 设置当前音量大小
                track.play();
            }
            break;
        }
    }

    track.stop();
    track.release();
    // 释放这个资源
    int ret = nativeReleaseDecoder();
    int soundTouchRet = soundTouch.close();
    if(ret != 0 || soundTouchRet != 0)
    {
        Log.d(TAG, "opusDecoder free error or soundTouchRet free error");
    }
    mFileInputStream.close();
    mFileInputStream = null;
}
```
SoundTouch Native接口提供
```java
public final class SoundTouch
{
    public native final static String getVersionString();
    
    private native final void setTempo(long handle, float tempo);

    private native final void setPitchSemiTones(long handle, float pitch);
    
    private native final void setRate(long handle, float speed);

    /**
     *
     * @param handle  c++中的对象指针
     * @param inputData 要处理的源数据
     * @param inputLength 要处理的源数据的长度
     * @param outputData  处理完之后接受内容的buff
     * @param outputMaxLength  最大可以处理的长度
     * @return
     */
    private native final int processFile(long handle, short [] inputData,int inputLength, short [] outputData,int outputMaxLength);

    /**
     * 刷新缓冲区的内容
     * @param handle
     * @param outputData 处理完之后接受内容的buff
     * @param outputMaxLength  最大可以处理的长度
     * @return
     */
    private native final int flushData(long handle,short [] outputData,int outputMaxLength);

    public native final static String getErrorString();

    private native final static long newInstance();
    
    private native final int deleteInstance(long handle);
    
    long handle = 0;

    /**
     * 构造函数的执行，分配内存空间，初始话资源
     */
    public SoundTouch()
    {
    	handle = newInstance();    	
    }

    /**
     * 释放内存
     * @return
     */
    public int close()
    {
    	int res = deleteInstance(handle);
    	handle = 0;
        return res;
    }

    /**
     * 处理声音
     * @param inputData  源的声音
     * @param inputLength  源声音的长度
     * @param outputData   输出的目标集合
     * @param outputMaxLength   最大的输出的目标的长度
     * @return
     */
    public int processAudioData(short [] inputData,int inputLength, short [] outputData,int outputMaxLength)
    {
        return processFile(handle, inputData, inputLength,outputData,outputMaxLength);
    }


    /**
     * 设置节奏
     * @param tempo
     */
    public void setTempo(float tempo)
    {
    	setTempo(handle, tempo);
    }

    /**
     * 刷新缓冲区
     * @param outputData 输出的目标集合
     * @param outputMaxLength 最大的输出的目标的长度
     */
    public int flushAudioData(short [] outputData,int outputMaxLength)
    {
        return flushData(handle,outputData,outputMaxLength);
    }


    /**
     * 设置音高
     * @param pitch
     */
    public void setPitchSemiTones(float pitch)
    {
    	setPitchSemiTones(handle, pitch);
    }


    /**
     * 设置播放的速度
     * @param speed
     */
    public void setPlayRate(float speed)
    {
        setRate(handle, speed);
    }
}


```

****总结****
===
```
上面就是关键的代码实现了，目前采用的录音的采样率为8000，录制出来的文件大小，在同等时间上跟微信，qq是差不多的
```