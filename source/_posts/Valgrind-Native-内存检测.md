---
layout: pager
title: Valgrind Native 内存检测
date: 2019-07-20 14:21:19
tags: [Android,NDK,Valgrind]
description:  Valgrind Native 内存检测
---

### 概述

> Valgrind Native 内存检测

<!--more-->

### 简介
> 最近在研究Android Native 内存检测，虽然网上有很多关于这方面的内容，比如：
使用ddms 检测，这个是很早之前的检测方式，目前sdk下面连ddms都删除了，可以参考这个链接 https://blog.csdn.net/yellowcath/article/details/78085419
使用LeakTracer 检测,亲测并不能检测出来，可以参考这个链接 https://www.wolfcstech.com/2017/01/12/%E4%BD%BF%E7%94%A8LeakTracer%E6%A3%80%E6%B5%8Bandroid%20NDK%20C_C++%E4%BB%A3%E7%A0%81%E4%B8%AD%E7%9A%84memory%20leak/
使用memwatcher 检测，貌似这个检测并不适合用来检测C++ 可以参考这个链接 http://memwatch.sourceforge.net/
最后就是我们的主角了 Valgrind ，官网链接为 http://www.valgrind.org/  虽然可以检测，但是坑还是有很多的

### Valgrind 编译使用
首先下载的内容里面有一个README的文件，这个文件包含了怎么样在各种平台上的编译,对应的也有平台对应的README文件，比如README.android等，我们首先看看在linux下面的表现

首先看README文件中关于Valgrind的编译
![结果显示](/uploads/Valgrind内存检测/Linuex编译介绍.png)

也即是首先要执行./autogen.sh ，然后执行./configure 文件，最终编译的结果
![结果显示](/uploads/Valgrind内存检测/linux编译结果.png)

接下来我们测试下，这个valgrind 在linux上面的表现,首先我们写了一个简单的程序
```C
#include <stdlib.h>
#include <stdio.h>
void f(void)
{
    int*x = (int *)malloc(10 * sizeof(int));
    // x[10]= 0;
	printf("Hello Wrold");
    //problem 1: heap block overrun
}  //problem 2: memory leak -- x not freed

int main(void)
{
    f();
    return 0;
}
```
之后使用gcc编译为一个可以执行的文件，接着使用valgrind来检测，关于valgrind的使用参数可以参考这篇文章
https://www.cnblogs.com/kuangsyx/p/8043526.html

如果在执行的过程中出现了这样的错误
![结果显示](/uploads/Valgrind内存检测/valgrind设置LIb.png)
我们可以执行 export VALGRIND_LIB=../lib/valgrind/,也即是设置你编译好的lib目录下的valgrind目录，设置完之后，继续执行
![结果显示](/uploads/Valgrind内存检测/Valgrind Linux检测的结果.jpg)
结合着文章，我们可以发现检测的结果是非常准确的


关于Valgrind 在 Android 方面的移植可以参考这篇文章  https://www.cnblogs.com/kuliuheng/p/10600737.html 同时仔细的读README.android,README.android_emulator 文件
![结果显示](/uploads/Valgrind内存检测/valgrind Android支持版本.png)

经过测试，对于x86的模拟器是没有一个可行的，Arm的模拟器支持的有4.0.3，4.1其他的版本的也是不行的这是一个很大的坑，也就是并不是所有的手机都能检测，那我们怎么知道是否能检测呢
那也很简单，我们可以将编译好的valgrind，移植到手机上，具体的操作，可以参考前面的链接，执行使用它来检测ls这个应用程序
![结果显示](/uploads/Valgrind内存检测/valgrind检测是否可行.png)

可以发现使用valgrind 检测ls的时候，既然没有检测到分配了多少内存，说明这个手机是不支持的,我们使用arm4.1的模拟器，按照上面的操作，执行检测ls，下面是检测的结果
![结果显示](/uploads/Valgrind内存检测/使用Valgrind能检测的结果.png)
可以发现这个是可以检测出来的

另一个问题要注意的是我们不可以直接的拷贝上面的脚本文件，然后放到手机上，这里要注意一个linux换行符跟window 换行符不同的问题，如果直接移植过来，会出现 #!/system/bin/sh 找不到的错误






