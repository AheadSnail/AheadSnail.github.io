---
title: NDK实现热更新
date: 2017-10-05 16:42:20
tags: [Android,NDK,热更新]
description: NDK实现热更新
---

NDK实现热更新
<!--more-->

**热更新跟普通更新的比较**
===
普通更新
![结果显示](/uploads/普通更新.png)

增量更新
![结果显示](/uploads/增量更新.png)

区别
![结果显示](/uploads/区别.png)

增量更新需要使用到第三方的库，下面为官网的地址
```
bspatch 官网
http://www.daemonology.net/bsdiff/
因为bspatch里面需要依赖bizp2,下面为bzip的官网的地址
bzip2
http://www.bzip.org/downloads.html
```
![结果显示](/uploads/增量更新官网.png)

使用
```
1. 差分
	依赖bzip2
	（版本较多，动态生成差分包）so/dll
		1. windows平台下：
			a. 使用Eclipse创建服务器工程
			b. 创建一个win32 工程
			c. 展示linux下面编译so库，写一个demo APK 做文件差分
2. 合并

	客户端合并

第一步：
	1. 生成 win 环境差分工具 差分库（动态生成差分包）
```		

**现在window下使用diff，用来生成一个差分包**
===
将源码拷贝到新建的vs工程下面，源码跟头文件分开存放,因为只要生成差分包的功能，这里删掉(bspatch.cpp文件)通过添加现有项将代码包含进来
![结果显示](/uploads/patchvs.png)

会出现以下的错误跟解决方案

严重性	代码	说明	项目	文件	行	禁止显示状态
错误	C4996	'strcat': This function or variable may be unsafe. Consider using strcat_s instead. To disable deprecation, use _CRT_SECURE_NO_WARNINGS. See online help for details.	DnTimDiff	f:\dn-lesson-vip\ndk\dn_lsn11\dntimdiff\dntimdiff\src\bzlib.c	1416

严重性	代码	说明	项目	文件	行	禁止显示状态
错误	C4996	'setmode': The POSIX name for this item is deprecated. Instead, use the ISO C and C++ conformant name: _setmode. See online help for details.	DnTimDiff	f:\dn-lesson-vip\ndk\dn_lsn11\dntimdiff\dntimdiff\src\bzlib.c	1422	
	

vs 找头文件 设置

	右键工程 --->  属性 ---> c++ -----> 附含包目录
	
vs 解决 _CRT_SECURE_NO_WARNINGS

	右键工程 --->  属性 ---> c++ -----> 命令行 添加 -D _CRT_SECURE_NO_WARNINGS -D表示的是定义一个宏的意思，在运行的时候，就会将这个宏添加进去
	
vs 关闭sdl 安全检查
	右键工程 --->  属性 ---> c++------>常规 ---->SDL检查 否
	
window使用这个的过程，其中bsdiff.cpp文件中有这样的主入口函数int main(int argc,char *argv[])
if(argc!=4) errx(1,"usage: %s oldfile newfile patchfile\n",argv[0]);，所以知道我们一定要传递四个参数，第一个参数不要管，第二个参数为老的apk的路径，第三个为新的apk的路径，最后一个为生成的差分包的路径

所以我们在eclipse里面新建的一个java工程，其中的native要传递对应的参数过去
public class DispatchUtils {
	public native void MyDispatch(String oldPath,String newPath,String patchPath);
}
使用javah生成对应的头文件,在1.7以上可以在src里面直接的使用javah 包名加上类名，成功的生成了头文件之后，
将头文件拷贝到vs工程的include 里面，还有jni.h jni_md.h，然后通过添加现有项添加进来

这里要注意的是这里拷贝的jni.h为jdk里面的，ndk里面也有jni.h这个对应的为linux的,下面为对应的关系
jni 是java 语法的概念 jni.h jdk
ndk是android 的概念  jni.h ndk
```
	
![结果显示](/uploads/vspathchJni.png)

```java
编写函数的实现
/*
* Class:     com_yuhui_dispatch_DispatchUtils
* Method:    MyDispatch
* Signature: (Ljava/lang/String;Ljava/lang/String;Ljava/lang/String;)V
*/
JNIEXPORT void JNICALL Java_com_yuhui_dispatch_DispatchUtils_MyDispatch
(JNIEnv * env, jobject obj,  jstring oldPath, jstring newPath, jstring patchPath)
{
	//1 定义要传递的参数
	int argc = 4;
	char *argv[4];
	//2，将java中的字符串，变成c中的字符串，并填充进参数里面
	char *c_olaPath = (char*)env->GetStringUTFChars(oldPath, NULL);
	char *c_newPath = (char*)env->GetStringUTFChars(newPath, NULL);
	char *c_patchPath = (char*)env->GetStringUTFChars(patchPath, NULL);

	argv[0] = "dispatchDemo";
	argv[1] = c_olaPath;
	argv[2] = c_newPath;
	argv[3] = c_patchPath;
	
	//调用main函数
	main(argc, argv);

	//释放引用
	env->ReleaseStringUTFChars(oldPath, c_olaPath);
	env->ReleaseStringUTFChars(newPath, c_newPath);
	env->ReleaseStringUTFChars(patchPath, c_patchPath);
}
生成解决方案,生成成功之后，会在vs的工程下面，debug目录下面有一个dll文件，将这个dll文件拷贝到eclispe里面
然后将测试的apk放到对应的生成dll的vs路径下面

![结果显示](/uploads/diff放置apk路径.png)

java代码配置我们的路径，然后调用
// 路径不能包含中文
public static final String OLD_APK_PATH = "F:/c/visual studio Worksapce1/yuhuiDiffPathch/x64/Debug/appOld.apk";
public static final String NEW_APK_PATH = "F:/c/visual studio Worksapce1/yuhuiDiffPathch/x64/Debug/appNew.apk";
public static final String PATCH_PATH = "F:/c/visual studio Worksapce1/yuhuiDiffPathch/x64/Debug/apk.patch";
```

运行结果为,可以看到差分的才900k
![结果显示](/uploads/热更新diff运行结果为.png)

**Android实现合并**
===
```java
首先写上native方法。。。
public class BsPatch {
    public native static int patch(String oldfile, String newFile, String patchFile);

    static {
        System.loadLibrary("myBaspatch");
    }
}
然后利用javah生成对应的头文件。。可以在app/src/main/java 目录下面利用javah +包名加上类名可以直接生成

cmakelist.txt文件的内容
#添加怎个目录的思路 指定一个变量 添加的时候使用变量值(my_c_path),这样就可以添加整个目录下的文件
file(GLOB my_c_path src/main/cpp/bzip2/*.c)

add_library( # Sets the name of the library.
             myBaspatch

             # Sets the library as a shared library.
             SHARED

             # Provides a relative path to your source file(s).
             ${my_c_path}
             src/main/cpp/bspatch.c )


find_library( # Sets the name of the path variable.
              log-lib

              # Specifies the name of the NDK library that
              # you want CMake to locate.
              log )


target_link_libraries( # Specifies the target library.
                       myBaspatch

                       # Links the target library to the log library
                       # included in the NDK.
                       ${log-lib} )
```
将源代码拷贝过来，可以去掉diff.c文件，因为这里用不到这个差分的功能，只要合并的功能，然后就对应的.c文件，将里面的main函数全部改成其他的名，要不然编译的时候，会出现
函数重名
![结果显示](/uploads/androidStuidoBaspatch.png)



编写native方法
```java
JNIEXPORT jint JNICALL Java_com_example_administrator_mybspatchdemo_BsPatch_patch
		(JNIEnv *env, jclass jazz, jstring oldPath_jstr, jstring newPath_jstr, jstring patchPatch_jst)
{
    int ret= -1;
    LOGD(" jni patch begin");

    char *oldPath = (*env) -> GetStringUTFChars(env, oldPath_jstr, JNI_FALSE);
    char *newPath = (*env) -> GetStringUTFChars(env, newPath_jstr, JNI_FALSE);
    char *patchPath = (*env) -> GetStringUTFChars(env, patchPatch_jst, JNI_FALSE);

    int argc = 4;
    char *argv[4];

    argv[0] = "yuBsPatch";
    argv[1] = oldPath;
    argv[2] = newPath;
    argv[3] = patchPath;

    //如果成功ret等于0
    ret = bspatch_main(argc,argv);
    (*env) -> ReleaseStringUTFChars(env, oldPath_jstr, oldPath);
    (*env) -> ReleaseStringUTFChars(env, newPath_jstr, newPath);
    (*env) -> ReleaseStringUTFChars(env, patchPatch_jst, patchPath);
    return ret;

}
```
执行编译，成功生成so
![结果显示](/uploads/baspatchJni生成成功.png)

