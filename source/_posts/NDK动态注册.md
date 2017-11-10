---
title: NDK动态注册
date: 2017-10-06 00:02:19
tags: [Android,NDK,动态注册]
description: NDK动态注册
---

NDK动态注册
<!--more-->

**静态注册跟动态注册的区别**
===

```
静态注册：
每个class都需要使用javah生成一个头文件，并且生成的名字很长书写不便；初次调用时需要依据名字搜索对应的JNI层函数来建立关联关系，会影响运行效率
用javah 生成头文件方便简单
```

```
动态注册：
使用一种数据结构JNINativeMethod来记录java native函数和JNI函数的对应关系
移植方便（一个java文件中有多个native方法，java文件的包名更换后）
```

**分析MediaPlay动态注册的实现**

```java
查看MediaPlay.cpp文件中动态注册的实现
我们知道在加载完so之后，就会调用这个方法。
jint JNI_OnLoad(JavaVM* vm, void* /* reserved */)
{
    JNIEnv* env = NULL;
    jint result = -1;

    if (vm->GetEnv((void**) &env, JNI_VERSION_1_4) != JNI_OK) {
        ALOGE("ERROR: GetEnv failed\n");
        goto bail;
    }
    assert(env != NULL);
	......

    if (register_android_media_MediaPlayer(env) < 0) {
        ALOGE("ERROR: MediaPlayer native registration failed\n");
        goto bail;
    }
	
	....这边还有注册其他的方法
    if (register_android_media_ExifInterface(env) < 0) {
        ALOGE("ERROR: ExifInterface native registration failed");
        goto bail;
    }

    /* success -- return valid version number */
	要注意这个返回值只能是JNI_VERSION_1_2,JNI_VERSION_1_4,JNI_VERSION_1_6，其他的都不支持
    result = JNI_VERSION_1_4; 

bail:
    return result;
}
register_android_media_MediaPlayer 函数的实现()
# define NELEM(x) ((int) (sizeof(x) / sizeof((x)[0])))
// This function only registers the native methods
static int register_android_media_MediaPlayer(JNIEnv *env)
{
    return AndroidRuntime::registerNativeMethods(env,
                "android/media/MediaPlayer", gMethods, NELEM(gMethods));
}
AndroidRuntime::registerNativeMethods() 函数的实现为：
其实就是调用了(*engv) ->RegisterNatives(engv, clazz, gMethods, NELEM(gMethods)的方法
/*static*/ int AndroidRuntime::registerNativeMethods(JNIEnv* env,
    const char* className, const JNINativeMethod* gMethods, int numMethods)
{
    return jniRegisterNativeMethods(env, className, gMethods, numMethods);
}


下面是要动态注册的方法的结构体数组,这个结构体的为
typedef struct {  
 const char* name; /*Java 中函数的名字*/  
 const char* signature; /*描述了函数的参数和返回值*/  
 void* fnPtr; /*函数指针,指向 C 函数*/   注意这个为jni层函数的实现。。除了方法的名称之后，其他的都给静态注册的原型一样
 } JNINativeMethod;
 
static const JNINativeMethod gMethods[] = {
    {"_prepare",            "()V",                              (void *)android_media_MediaPlayer_prepare},
    {"prepareAsync",        "()V",                              (void *)android_media_MediaPlayer_prepareAsync},
    {"_start",              "()V",                              (void *)android_media_MediaPlayer_start},
    {"_stop",               "()V",                              (void *)android_media_MediaPlayer_stop},
    {"getVideoWidth",       "()I",                              (void *)android_media_MediaPlayer_getVideoWidth},
    {"getVideoHeight",      "()I",                              (void *)android_media_MediaPlayer_getVideoHeight},
    {"setPlaybackParams", "(Landroid/media/PlaybackParams;)V", (void *)android_media_MediaPlayer_setPlaybackParams},
    {"getPlaybackParams", "()Landroid/media/PlaybackParams;", (void *)android_media_MediaPlayer_getPlaybackParams},
    {"setSyncParams",     "(Landroid/media/SyncParams;)V",  (void *)android_media_MediaPlayer_setSyncParams},
    {"getSyncParams",     "()Landroid/media/SyncParams;",   (void *)android_media_MediaPlayer_getSyncParams},
    {"seekTo",              "(I)V",                             (void *)android_media_MediaPlayer_seekTo},
    {"_pause",              "()V",                              (void *)android_media_MediaPlayer_pause},
    {"isPlaying",           "()Z",                              (void *)android_media_MediaPlayer_isPlaying},
};
```

**动态注册的实践**
===

```java
# define NELEM(x) ((int) (sizeof(x) / sizeof((x)[0])))
/*
 * Class:     com_dn_tim_dn_lsn_9_FileUtils
 * Method:    diff
 * Signature: (Ljava/lang/String;Ljava/lang/String;I)V
 */
JNIEXPORT void JNICALL native_diff
        (JNIEnv *env, jclass clazz, jstring path, jstring pattern_Path, jint file_num)
{

    LOGI("JNI begin 动态注册的方法 ");

}

给出要动态注册的方法数组
static const JNINativeMethod gMethods[] = {
        {
                "diff","(Ljava/lang/String;Ljava/lang/String;I)V",(void*)native_diff
        }
};


static int registerNatives(JNIEnv* engv)
{
    LOGI("registerNatives begin");
    jclass  clazz;
	//找到要动态注册的类名
    clazz = (*engv) -> FindClass(engv, "com/dn/tim/dn_lsn_9/FileUtils");

    if (clazz == NULL) {
        LOGI("clazz is null");
        return JNI_FALSE;
    }

	(*engv) ->RegisterNatives(engv, clazz, gMethods, NELEM(gMethods)) < 0)如果小于0表示注册失败
    if ((*engv) ->RegisterNatives(engv, clazz, gMethods, NELEM(gMethods)) < 0) {
        LOGI("RegisterNatives error");
        return JNI_FALSE;
    }
    return JNI_TRUE;
}

JNIEXPORT jint JNI_OnLoad(JavaVM* vm, void* reserved)
{

    LOGI("jni_OnLoad begin");
	
	//这些内容直接照抄过来
    JNIEnv* env = NULL;
    jint result = -1;
    if ((*vm)->GetEnv(vm,(void**) &env, JNI_VERSION_1_4) != JNI_OK) {
        LOGI("ERROR: GetEnv failed\n");
        return -1;
    }
    assert(env != NULL);
	

    registerNatives(env);
	
	//返回值也是，不能随便写
    return JNI_VERSION_1_4;
}
```
