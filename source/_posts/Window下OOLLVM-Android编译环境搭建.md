---
layout: pager
title: Window下OOLLVM Android编译环境搭建
date: 2018-06-17 14:08:46
tags: [Android,NDK,OOLLVM]
description:  Window下OOLLVM Android编译环境搭建
---

Window下OOLLVM Android编译环境搭建
<!--more-->

****什么是OOLLVM？****
===
```
LLVM是由瑞士应用科学大学和瑞士西部艺术大学Yverdon-les-Bains（HEIG-VD）于2010年6月发起的一个项目。这个项目的目的是提供一个LLVM编译套件的开源代码
能够通过代码混淆和防篡改提供更高的软件安全性。由于我们目前主要工作在中间代表（IR）级别，因此我们的工具与所有编程语言（C，C ++，Objective-C，Ada和Fortran）
以及目标平台（x86，x86-64，PowerPC，PowerPC-64 ，ARM，Thumb，SPARC，Alpha，CellSPU，MIPS，MSP430，SystemZ和XCore）目前由LLVM支持。
```

****OOLLVM 官网提供的有用信息****
===
OOLLVM官网 https://github.com/obfuscator-llvm/obfuscator
Wiki介绍 https://github.com/obfuscator-llvm/obfuscator/wiki
```
Build: 编译的方式：如果是mac，或者ubuntu可以直接采用这种方式来编译

git clone https://github.com/Qrilee/llvm-obfuscator

mkdir build

cd build

cmake -DCMAKE_BUILD_TYPE=Release ../Obfuscator-LLVM/

make -j7


Build pass for windows: window下编译要注意的地方：

  MinGW64 for Windows
 
  Cmake 3.9 rc5 for Windows x64
 
Build pass for linux:

  gcc&g++ 7.2.0
 
  cmake 3.8.0
 
Use:使用的方式，我们可以通过在build.gradle 中将这些内容引进来，后面会介绍怎么添加

-mllvm -bcf -mllvm -fla -mllvm -sub -mllvm -sobf

Function:功能介绍

Instructions Substitution -mllvm -sub
Bogus Control Flow -mllvm -bcf
Control Flow Flattening -mllvm -fla
String Obfuscation -mllvm -sobf
```

****NDK开发真的安全吗?****
===
```
在开发android的时候，通常会将一些关键代码放JNI (C/C++)里真的很安全吗?
很多Android开发者都认为 把关键代码放到C/C++里 然后打包静态库 然后破解者就无法破解 
我想说 你太嫩了，下面来演示下OOLLVM的作用 
```

****不使用OOLLVM开发NDK****
===
```java
直接创建一个AndroidStudio 项目，勾选上支持c++支持，然后添加下面的函数

jint JNICALL
Java_com_test_Test(JNIEnv *env, jclass t,jint k) {
 int key = 8;
 return key^k;//假设这是个算法
}

得到so文件 然后打开 IDA pro(反汇编工具) 没有的百度下载
可以看到 我们的jni函数名: Java_com_test_Test 
```

![结果显示](/uploads/OLLVM编译/NDK正常编译.png)

```
点进去
然后按F5 查看伪代码 
```
![结果显示](/uploads/OLLVM编译/查看伪代码.png)

```java
//IDA-pro 伪代码效果
int __fastcall Java_com_test_Test(int a1, int a2, int a3)
{
  return a3 ^ 8;
}

如图 a3则是jint k , int key=8 于是相当于 k^8, 则为a3^8; 
看到了吧 so的代码都能看到了 和我们的jni没编译前的代码基本一致, 你还觉得jni安全么
```


****不使用OOLLVM开发NDK****
===
```java
上面的代码都不用修改，直接在app目录下的build.gradle中添加这些内容，这些内容是后面编译OOLLVM产生的内容,这里先看效果
externalNativeBuild {
    cmake {
        cppFlags " -mllvm -sub -mllvm -fla -mllvm -bcf"
        cFlags " -mllvm -sub -mllvm -fla -mllvm -bcf"
    }
}
	
编译so后 重新使用IDA pro反汇编这个so文件,查看c/c++代码 如下….
//IDA-pro  进行OLLVM混淆-伪代码效果
//尴尬了 一个普通的算法突然混淆成这样 大大增加了破解者的破解难度!!!
//....反正我是看不懂了...
unsigned int __fastcall Java_com_test_Test(int a1, int a2, int a3)
{
  signed int v3; // r4@1
  signed int v4; // r5@1
  unsigned int v5; // r1@1
  signed int v6; // r3@5
  bool v7; // zf@9
  signed int v8; // r5@9
  unsigned int v9; // r2@9
  signed int v10; // r1@9
  signed int v11; // r4@13
  unsigned __int8 v13; // [sp+2h] [bp-16h]@3
  unsigned __int8 v14; // [sp+3h] [bp-15h]@5
  unsigned int v15; // [sp+4h] [bp-14h]@0

  v3 = 0;
  v4 = 0;
  v5 = x * ~-x & (x * ~-x ^ 0xFFFFFFFE);
  if ( !v5 )
    v4 = 1;
  v13 = v4;
  if ( y < 10 )
    v3 = 1;
  v6 = 0;
  v14 = v3;
  if ( v5 )
    v5 = 1;
  if ( y > 9 )
    v6 = 1;
  v7 = ((v6 | v5) ^ 1 | v4 ^ v3) == 0;
  v8 = -1313485305;
  v9 = a3 & 0xFFFFFFF7 | 8 * (((unsigned int)~a3 >> 3) & 1);
  v10 = -1265641972;
  if ( !v7 )
    v10 = 1153232101;
  while ( 1 )
  {
    v11 = v8;
    if ( v8 > 1153232100 )
      break;
    v8 = -1766943702;
    if ( v11 != -1265641972 )
    {
      if ( v11 == -1766943702 )
      {
        v15 = v9;
        v8 = v10;
      }
      else
      {
        if ( v11 != -1313485305 )
          goto LABEL_22;
        v8 = -1265641972;
        if ( ((unsigned __int8)(v13 & v14) | v13 ^ v14) & 1 )
          v8 = -1766943702;
      }
    }
  }
  if ( v8 != 1153232101 )
  {
    while ( 1 )
LABEL_22:
      ;
  }
  return v15;
}

以上就是混淆效果 
你再看看原来没混淆的

//IDA-pro 未进行OLLVM混淆-伪代码效果
int __fastcall Java_com_test_Test(int a1, int a2, int a3)
{
   return a3 ^ 8;
}
```

****OOLLVM Window编译****
===
```
这里先介绍下OOLLVM的使用流程，将源代码下载下来之后，通过替换NDK中的内容，比如我当前的目录为D:\sdk\android-ndk-r14b\toolchains\llvm\prebuilt\windows-x86_64\
那么我们就要将编译出来的OOLLVM的内容，替换这个目录下面的bin目录下的内容,所以基于大部分的开发人员还是屌丝，都是采用Window下开发，所以我们就有必要在Window下编译，
当然mac，ubuntu系统我们也会介绍怎么编译，那俩个相对来说比较简单

1. 准备工作

安装MinGW64 for Windows 和 Cmake
```
MinGW64 for Windows下载对应的url: https://sourceforge.net/projects/mingw-w64/?source=typ_redirect
```
Cmake对应的有在线安装，还是zip包的形式，我采用的是zip包的形式，这样就不用绑定到电脑
```
Cmake对应的下载url: https://cmake.org/files/v3.12/cmake-3.12.0-rc1-win64-x64.zip
```
要注意的是分清是64位还是32位，如果是64位就下载x86对应的

2.设置环境变量，下面是我设置在path中环境变量的路径
F:\Cmake\cmake-3.12.0-rc1-win64-x64\bin\;F:\Mingw64\mingw64\bin;

3.检验MingW64 是否安装成功
直接打开一个cmd命令，然后输入gcc -v 如果出现了这些内容就说明安装成功了
```
![结果显示](/uploads/OLLVM编译/Mingw安装检验.png)
```
4.下载OLLVM
git clone -b llvm-4.0 https://github.com/obfuscator-llvm/obfuscator.git
如果直接编译源码，会在编译的过程中出现问题，所以我就直接把我踩的坑直接贴出来

问题1：修改CryptoUtils.h，修改的内容如下：
#if defined(__i386) || defined(__i386__) || defined(_M_IX86) ||                \
    defined(INTEL_CC) || defined(_WIN64) || defined(_WIN32)

问题二：替换CryptoUtils.cpp
```
可以在这里下载 : https://github.com/rrrfff/Obfuscator-LLVM/blob/master/lib/Transforms/Obfuscation/CryptoUtils.cpp
```
问题三：修改 OrcRemoteTargetClient.h 修改的内容如下：
将Expected<std::vector<char>> readMem(char *Dst, JITTargetAddress Src,uint64_t Size) {
修改为：Expected<std::vector<uint8_t>> readMem(char *Dst, JITTargetAddress Src,uint64_t Size) {

5.编译:
直接打开一个cmd命令，进入到下载的源码目录，这里最好在这个源码目录的外面再新建一层文件夹
创建文件夹 mkdir build
cd build
cmake -G "MinGW Makefiles" -DCMAKE_BUILD_TYPE:String=Release ../obfuscator/
mingw32-make -j16
数字可根据电脑配置进行选择，编译完成后，会在build/bin下看到编译完成的二进制文件，编译时间比较长。
```
编译成功之后的结果为下面这样的
![结果显示](/uploads/OLLVM编译/OOLLVMWindow编译成功.png)

```
2. 备份源 bin 文件：
备份 ndk-bundle/toolchains 的 llvm 的bin目录, 比如：D:\sdk\android-ndk-r14b-windows-x86_64\android-ndk-r14b\toolchains\llvm\prebuilt\windows-x86_64\
我们可以将原本这个目录下的bin目录改为bin_back

3.拷贝编译的 bin 文件：
将我们编译的OOLLVM bin目录整个拷贝到D:\sdk\android-ndk-r14b-windows-x86_64\android-ndk-r14b\toolchains\llvm\prebuilt\windows-x86_64\

4.拷贝 ollvm 依赖的 buildin lib 和 include 头文件，不拷贝会导致编译时多头文件和类型找不到
```

****OOLLVM macOS Ubuntu编译****
===
```
1.  环境准备
建议操作系统：macOS Ubuntu
Obfuscator-llvm 安装：
1. 获取 Obfuscator-llvm 代码并进行编译：
git clone -b llvm-3.6.1 https://github.com/obfuscator-llvm/obfuscator.git
mkdir build
cd build
cmake -DCMAKE_BUILD_TYPE:String=Release ../obfuscator/
make –j8
2. 备份源 bin 文件：
备份 ndk-bundle/toolchains 的 llvm 的目录, 目录位于：
/Users/${user}/Library/Android/sdk/ndk-
bundle/toolchains/llvm/prebuilt/darwin-x86_64/bin
其中${user}代表当前用户名称。
备份命令如下所示：
mv /Users/${user}/Library/Android/sdk/ndk-
bundle/toolchains/llvm/prebuilt/darwin-x86_64/bin
/Users/${user}/Library/Android/sdk/ndk-
bundle/toolchains/llvm/prebuilt/darwin-x86_64/bin_bak
3.拷贝编译的 bin 文件：
cp -rv bin /Users/${user}/Library/Android/sdk/ndk-
bundle/toolchains/llvm/prebuilt/darwin-x86_64/
拷贝 ollvm 依赖的 buildin lib 和 include 头文件，不拷贝会导致编译时多头文
件和类型找不到
cp -rv lib /Users/${user}/Library/Android/sdk/ndk-
bundle/toolchains/llvm/prebuilt/darwin-x86_64/
cp -rv include /Users/${user}/Library/Android/sdk/ndk-
bundle/toolchains/llvm/prebuilt/darwin-x86_64/
```

总结：这里只是记录下，这俩天来编译这个玩意要注意的点已经踩的坑，开发NDK最麻烦的就是编译环境的问题

		
		