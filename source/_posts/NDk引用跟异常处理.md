---
title: NDk引用跟异常处理
date: 2017-10-04 10:50:00
tags: [Android,NDK,Exception]
description: NDk引用跟异常处理
---

NDk引用跟异常处理
<!--more-->

java代码
```java
public native void exception();
test.exception();
System.out.println("javaException");
```

ndk代码
```C
/*
* Class:     com_example_jni_demo2_Jni_Test
* Method:    exception
* Signature: ()V
*/
JNIEXPORT void JNICALL Java_com_example_jni_demo2_Jni_1Test_exception
(JNIEnv * env, jobject obj)
{
	//得到obj对应的class
	jclass cls = (*env)->GetObjectClass(env, obj);
	//得到这个class的成员域,如果没有找到的话，就会出现异常,
	jfieldID fid = (*env)->GetFieldID(env, cls, "key", "Ljava/lang/String;");

	//c语言中的异常出错不是立刻就有的。。。这个用来判断发生异常的时候Ndk是否会立即的终止
	printf("exception\n");
}
```
![结果显示](/uploads/ndk异常.png)

从运行结果可以发现ndk中产生了异常并不会立刻的终止应用程序。。代码还是会往下面执行，而java是不可以的
而且对于java层的人来说，他并不不知道ndk的代码哪里会有异常产生，哪里需要try-catch，所以解决方案就是
```java
/*
* Class:     com_example_jni_demo2_Jni_Test
* Method:    exception
* Signature: ()V
*/
JNIEXPORT void JNICALL Java_com_example_jni_demo2_Jni_1Test_exception
(JNIEnv * env, jobject obj)
{
	//得到obj对应的class
	jclass cls = (*env)->GetObjectClass(env, obj);
	//得到这个class的成员域,如果没有找到的话，就会出现异常
	jfieldID fid = (*env)->GetFieldID(env, cls, "key", "Ljava/lang/String;");

	//检查是否发生了异常的处理
	jthrowable ex = (*env)->ExceptionOccurred(env);
	//判断是否有异常
	if (ex != NULL)
	{
		//清空JNI产生的异常,这样就不会将java的异常产生在java层，影响java
		(*env)->ExceptionClear(env);
		//IllegalArgumentException
		jclass newExceptionJclz = (*env)->FindClass(env, "java/lang/IllegalArgumentException");
		if (newExceptionJclz == NULL)
		{
			printf("exception\n");
			return ;
		}
		我们主动的抛出一个异常出来
		(*env)->ThrowNew(env, newExceptionJclz, "Throw exception from JNI");
	}

	//c语言中的异常出错不是立刻就有的。。。
	printf("exception\n");
}
```
![结果显示](/uploads/ndk异常处理.png)
这样就可以抛出我们自己的异常，这样java层try-catch

**异常处理**
===
```
本地代码通常有两种方式来处理一个异常：
1、一旦发生异常，立即返回，让调用者处理这个异常。
2、通过 ExceptionClear 清除异常，然后执行自己的异常处理代码。
当一个异常发生后，必须先检查、处理、清除异常后再做其它 JNI 函数调用，否
则的话，结果未知。当前线程中有异常的时候，你可以调用的 JNI 函数非常少，
11.8.2 节列出了这些 JNI 函数的详细列表。通常来说，当有一个未处理的异常
时，你只可以调用两种 JNI 函数：异常处理函数和清除 VM 资源的函数
```

**NDK引用**
 - 1、JNI 支持三种引用：局部引用、全局引用、弱全局引用（下文简称“弱引用”）。
 - 2、局部引用和全局引用有不同的生命周期。当本地方法返回时，局部引用会被自动释放。而全局引用和弱引用必须手动释放。
 - 3、局部引用或者全局引用会阻止 GC 回收它们所引用的对象，而弱引用则不会。
 - 4、不是所有的引用可以被用在所有的场合。例如，一个本地方法创建一个局部引用并返回后，再对这个局部引用进行访问是非法的
 
 局部引用
 
 ```java
 /*
局部引用
定义的方式多样:FindClass,NewObject,GetObjectClass,NewString.....
释放局部引用的方式：方法调用完JVM会自动的释放,我们自己手动的释放DeleteLocalRef();
不能在多线程里面使用
* Class:     com_example_jni_demo2_Jni_Test
* Method:    localRelf
* Signature: ()V
*/
JNIEXPORT void JNICALL Java_com_example_jni_demo2_Jni_1Test_localRelf
(JNIEnv * env, jobject obj)
{
	int i = 0;
	for (i = 0; i < 5; i++)
	{
		jclass cls = (*env)->FindClass(env, "java/util/Date");
		//得到对应的函数method,这边得到他的构造函数，所有的构造函数都是<init>,最后一个参数为函数的签名
		jmethodID method = (*env)->GetMethodID(env, cls, "<init>", "()V");
		//创建了一个Date类型的引用,这边创建的局部的引用
		jobject obj = (*env)->NewObject(env, cls, method);
		//使用这个引用

		//释放局部引用的方法
		(*env)->DeleteLocalRef(env, cls);
		(*env)->DeleteLocalRef(env, method);
		(*env)->DeleteLocalRef(env, obj);

	}
}
```

全局引用
 ```java
jstring global_cstr;
/*
全局引用
可以跨线程使用，跨方法使用
创建全局引用的唯一的方法 NewGlobalRef
* Class:     com_example_jni_demo2_Jni_Test
* Method:    CreateGlobalRelf
* Signature: ()V
*/
JNIEXPORT void JNICALL Java_com_example_jni_demo2_Jni_1Test_CreateGlobalRelf
(JNIEnv * env, jobject jobj)
{
	jobject obj = (*env)->NewStringUTF(env, "sdfdfdf");
	//创建全局引用的唯一的方法
	global_cstr = (*env)->NewGlobalRef(env, obj);
}

/*
全局引用可以跨方法使用
* Class:     com_example_jni_demo2_Jni_Test
* Method:    getGlobalRef
* Signature: ()Ljava/lang/String;
*/
JNIEXPORT jstring JNICALL Java_com_example_jni_demo2_Jni_1Test_getGlobalRef
(JNIEnv * env, jobject obj)
{
	return global_cstr;
}

/*
全局引用的释放的唯一方法DeleteGlobalRef,全局引用要释放。。。要不然会内存溢出
* Class:     com_example_jni_demo2_Jni_Test
* Method:    releaseGlobalRef
* Signature: ()V
*/
JNIEXPORT void JNICALL Java_com_example_jni_demo2_Jni_1Test_releaseGlobalRef
(JNIEnv * env, jobject obj)
{
	(*env)->DeleteGlobalRef(env, global_cstr);
}
```
弱全局引用

```java
//weak全局引用,他也可以跨线程，跨方法使用
//他不会阻止GC的操作
static jclass g_weak_cls;
JNIEXPORT void JNICALL Java_com_example_jni_demo2_Jni_1Test_CreateWeakRelf
(JNIEnv * env, jobject jobj)
{
	jclass cls_string = (*env)->FindClass(env, "java/lang/String");
	//创建weak 引用
	g_weak_cls = (*env)->NewWeakGlobalRef(env, cls_string);
}

//释放weak 全局引用
JNIEXPORT void JNICALL Java_com_example_jni_demo2_Jni_1Test_releaseWeakRef
(JNIEnv * env, jobject obj)
{
	(*env)->DeleteWeakGlobalRef(env, g_weak_cls);
}
```

缓存策略和weak 引用联合使用带来的问题
```java
/*
缓存策略和weak 引用联合使用带来的问题
* Class:     com_example_jni_demo2_Jni_Test
* Method:    weakAndglobalCached
* Signature: ()V
*/
JNIEXPORT void JNICALL Java_com_example_jni_demo2_Jni_1Test_weakAndglobalCached
(JNIEnv * env, jobject obj)
{
	//定义一个静态的局部变量
	static jclass cls_string;
	if (cls_string == NULL)
	{
		printf("Java_JniMain_AccessCache out\n");
		//给局部静态变量赋值一个局部变量,这个静态的局部变量就成了一个野指针
		cls_string = (*env)->GetObjectClass(env, obj);
	}
	//这里也有可能发生异常
	if (cls_string == NULL)
	{
		printf(" cls_string null \n");
		return;
	}
	//使用局部静态变量
	jmethodID methodId = (*env)->GetMethodID(env, cls_string, "getRef", "(I)I");
	//检查是否有异常发生
	jthrowable ex = (*env)->ExceptionOccurred(env);
	if (ex != NULL)
	{
		jclass newExc;
		//让java代码继续执行
		//(*env)->ExceptionDescribe(env);//输出关于这个信号的异常描述
		(*env)->ExceptionClear(env);//清除异常
		printf("c Exception happend\n");
	}
	//调用方法
	//jint in = (*env)->CallIntMethod(env, obj, methodId, 1);
	printf("result ");
}

上面代码中，我们省略了和我们的讨论无关的代码。因为 GetObjectClass 返回一个对
jclass 对象的局部引用，上面的代码中缓存 cls_string 做法是错
误的。假设一个本地方法 C.f 调用了 Java_com_example_jni_demo2_Jni_1Test_weakAndglobalCached
JNIEXPORT jstring JNICALL
Java_C_f(JNIEnv *env, jobject this)
{
...
return Java_com_example_jni_demo2_Jni_1Test_weakAndglobalCached(env,this);
}
C.f 方法返回后，VM 释放了在这个方法执行期间创建的所有局部引用，也包含对
jclass 类的引用 cls_string。当再次调用 cls_string 时，会试图访问一个无
效的局部引用，从而导致非法的内存访问甚至系统崩溃。
释放一个局部引用有两种方式，一个是本地方法执行完毕后 VM 自动释放，另外
一个是程序员通过 DeleteLocalRef 手动释放。
既然 VM 会自动释放局部引用，为什么还需要手动释放呢？因为局部引用会阻止
它所引用的对象被 GC 回收。
局部引用只在创建它们的线程中有效，跨线程使用是被禁止的。不要在一个线程
中创建局部引用并存储到全局引用中，然后到另外一个线程去使用。
5.1.2 全局引用
全局引用可以跨方法、跨线程使用，直到它被手动释放才会失效。同局部引用一
```


