---
title: 采用VisualStudio跟Eclipse开发NDK
date: 2017-10-04 11:01:43
tags: [Android,NDK,VisualStudio,Eclipse]
description: 采用VisualStudio跟Eclipse开发NDK
---

采用VisualStudio跟Eclipse开发NDK
<!--more-->

**JNI开发的流程**
===
![结果显示](/uploads/ndk开发流程.png)

1.编写native 方法
```
//声明本地的方法
public native static String getStringFromC();
//因为不是静态的方法，所以要通过对象的方式来访问这个成员函数
public native String getStringFromC2();
```

2.javah 命令，生成对应的.h头文件之后生产对应的头文件，这里我们采用javah来生成
```
在jdk1.6只能在build，classes目录下面运行javah +对应的包名加上类名来生产
在jdk1.7中，可以在src的下面 javah +对应的包名加上类名来生产 ，包名类名的获取，
可以在对应类的属性中点击右键copy Qualaifiled name 获取，不是直接在Jni_Test里面右键的是下面类的右键，这个要注意了
```
![结果显示](/uploads/ndkJavah生成头文件.png)

回车，就可以对应的头文件
![结果显示](/uploads/ndkJavah.png)

3.复制.h头文件到visual studio 对应的工程里面，再通过添加现有项的方式将这个头文件添加进工程里面
4.从jdk里面查找对应的jni.h跟 jni_mid.h头文件(JNI一开始是跟随java的，而不是android)
![结果显示](/uploads/ndkVisualStudio.png)

5.实现.h头文件中的声明函数
![结果显示](/uploads/ndk函数实现.png)

6.配置项目的属性，生成一个dll动态库，（生成的过程中，移除没有必要的文件）,dll库不需要提供main函数的入口
![结果显示](/uploads/visual生成dll.png)

7.再java中加载动态库 (system.loadLibrary())方式来添加进来

```java
static {
		System.loadLibrary("NDKDemo2");
}
```

8.触发native 函数,这里要注意生成的库要跟调用的库要保持一致，比如生成的是32位的，eclipse为64位的这样是不行的
![结果显示](/uploads/ndk加载so.png)

简单的区分动态库和静态库的区别：
```
静态库文件的后缀名为.a结尾的，静态库的特点是要在编译的时候，就要添加进来
而动态库是动态添加的也即是只有你要使用的时候才会添加进来，在window下动态库为.dll后缀结尾，linux下为.so结尾
```

静态方法跟非静态方法的区别：
```
每个native函数，都至少有俩个参数（JNIEnv，jclass/jObject）
1.静态的方法 ,因为是静态的，所以这个jclz就代表静态的类
JNIEXPORT jstring JNICALL Java_JniMain_getStringFromC(JNIEnv * env, jclass jclz)
2.非静态方法  ,因为不是静态的，所以这里传递的jobz为调用这个方法的对象
JNIEXPORT jstring JNICALL Java_JniMain_getStringFromC2(JNIEnv * env, jobject jobz)
3.java调用的地方区别
System.out.println(getStringFromC());
System.out.println(new JniMain().getStringFromC2());
```

理解下生成的.h头文件的代码解析：
```C
#ifndef _Included_JniMain   这个是条件编译，好处是可以防止重复的引人同一个文件出现错误
#define _Included_JniMain   
#ifdef __cplusplus          这个的意思是说，如果是采用了c++来编译的化
extern "C" {
#endif                      这个是跟#ifdef __cplusplus一对的
/*
 * Class:     JniMain
 * Method:    getStringFromC
 * Signature: ()Ljava/lang/String;
 */
JNIEXPORT jstring JNICALL Java_JniMain_getStringFromC
  (JNIEnv *, jclass);

/*
 * Class:     JniMain
 * Method:    getStringFromC2
 * Signature: ()Ljava/lang/String;
 */
JNIEXPORT jstring JNICALL Java_JniMain_getStringFromC2
  (JNIEnv *, jobject);

#ifdef __cplusplus                 这个的意思是说，如果是采用了c++来编译的化
} 
#endif                             这个是跟#ifdef __cplusplus一对的
#endif                             这个是跟#ifndef _Included_JniMain一对的
也即是说如果采用的是c++来编译的话：
extern "C" {
/*
 * Class:     JniMain
 * Method:    getStringFromC
 * Signature: ()Ljava/lang/String;
 */
JNIEXPORT jstring JNICALL Java_JniMain_getStringFromC(JNIEnv *, jclass);

/*
 * Class:     JniMain
 * Method:    getStringFromC2
 * Signature: ()Ljava/lang/String;
 */
JNIEXPORT jstring JNICALL Java_JniMain_getStringFromC2(JNIEnv *, jobject);
} 
如果采用的是c来编译的话，即为:
/*
 * Class:     JniMain
 * Method:    getStringFromC
 * Signature: ()Ljava/lang/String;
 */
JNIEXPORT jstring JNICALL Java_JniMain_getStringFromC (JNIEnv *, jclass);

/*
 * Class:     JniMain
 * Method:    getStringFromC2
 * Signature: ()Ljava/lang/String;
 */
JNIEXPORT jstring JNICALL Java_JniMain_getStringFromC2 (JNIEnv *, jobject);

jni.h头文件里面为什么c编译的JNIEnv是一个二级指针，而c++编译的JNIEnv 是一级指针：
#ifdef __cplusplus
typedef JNIEnv_ JNIEnv;
#else
typedef const struct JNINativeInterface_ *JNIEnv;
#endif
```

```
首先 下面的条件编译如果是采用C++编译的话，JNIEnv 是JNIEnv_数据类型的别名,而JNIEnv_结构体里面存储了
一个这样的成员变量，这个就是使用c编译使用的结构体，而且可以看出来其实c++结构体里面只是将c编译的结构体进一步的封装，
最后还是调用了c结构体里面的内容比如：functions->GetVersion(this);所以c++为了跟c的区分，要不然c++跟c的使用都差不多，
还有一点就是 jstring (JNICALL *NewStringUTF)(JNIEnv *env, const char *utf); c结构体里面的函数指针接收的参数是要传递JNIEnv *参数的，所以c里面选择了使用二级指针，c++使用一级指针
const struct JNINativeInterface_ *functions;
struct JNIEnv_ {
    const struct JNINativeInterface_ *functions;
#ifdef __cplusplus

    jint GetVersion() {
        return functions->GetVersion(this);
    }


JNIEXPORT jstring JNICALL
Java_Prompt_getLine(JNIEnv *env, jobject this, jstring prompt);
其中，JNIEXPORT 和 JNICALL 这两个宏（被定义在 jni.h）确保这个函数在本地库外可见， 并且 C 编译器会进行正确的调用转换。C 函数的名字构成有些讲究，在 11.3 中会有一个详 细的解释。
```

JNI里面跟java里面数据类型的对应关系:基本数据类型的对应：
![结果显示](/uploads/NDk基本数据类型对应关系.png)
引用数据类型的对应
![结果显示](/uploads/NDK引用数据类型.png)

```
本地方法声明中的参数类型在本地语言中都有对应的类型。JNI 定义了一个 C/C++类型的集 合，集合中每一个类型对应于 JAVA 中的每一个类型。
JAVA 中有两种类型：基本数据类型（int,float,char 等）和引用类型（类，对象，数组等）。 JNI 对基本类型和引用类型的处理是不同的。基本类型的映射是一对一的。
例如 JAVA 中的 int 类型直接对应 C/C++中的 jint（定义在 jni.h 中的一个有符号 32 位整数）。12.1.1 包含了 JNI 中所有基本类型的定义。
JNI 把 JAVA 中的对象当作一个 C 指针传递到本地方法中，这个指针指向 JVM 中的内部数 据结构，而内部数据结构在内存中的存储方式是不可见的。
本地代码必须通过在 JNIEnv 中 选择适当的 JNI 函数来操作 JVM 中的对象。例如，对于 java.lang.String 对应的 JNI 类型是 jstring，但本地代码只能通过 GetStringUTFChars 
这样的 JNI 函数来访问字符串的内容。所有的 JNI 引用都是 jobject 类型，对了使用方便和类型安全，JNI 定义了一个引用类型集合，
 集合当中的所有类型都是 jobject 的子类型。这些子类型和 JAVA 中常用的引用类型相对应。 例如，jstring 表示字符串，jobjectArray 表示对象数组。
 ```