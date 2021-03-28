---
layout: pager
title: so入门加密
date: 2020-09-29 09:09:32
tags: [Android,ELF,SO]
description:  在native层为.so文件进行加固
---
### 简介
Android 系统安全愈发重要，像传统pc安全的可执行文件加固一样，应用加固是Android系统安全中非常重要的一环。目前Android 应用加固可以分为dex加固和Native加固，Native 加固的保护对象为 Native 层的 SO 文件，使用加壳、反调试、混淆、VM 等手段增加SO文件的反编译难度。目前最主流的 SO 文件保护方案还是加壳技术,本篇主要用来记录下native层加密

### 目标
通过修改之后，使用readelf和ida等工具打开，会报各种错误

### ELF文件的理解

elf文件是一种目标文件格式，用于定义不同类型目标文件以什么样的格式，都放了些什么东西。主要   用于linux平台。windows下是PE/COFF格式,可执行文件、可重定位文件(.o)、共享目标文件(.so)、核心转储文件都是以elf文件格式存储的,ELF文件组成部分：文件头、段表(section)、程序头，具体的可以参考下面有关ELF文件的详解

[ELF文件格式解析](https://blog.csdn.net/feglass/article/details/51469511)
[ELF文件格式解析(完)](https://blog.csdn.net/zhangmiaoping23/article/details/82285664)

elf文件视图

![结果显示](/uploads/so加密/elf文件视图.png)
![结果显示](/uploads/so加密/elf装载视图.png)


链接视图是以节（section）为单位，执行视图是以段（segment）为单位。链接视图就是在链接时用到的视图，而执行视图则是在执行时用到的视图。上图左侧的视角是从链接来看的，右侧的视角是执行来看的。总个文件可以分为四个部分：

>ELF header： 描述整个文件的组织。
Program Header Table: 描述文件中的各种segments，用来告诉系统如何创建进程映像的。
sections 或者 segments：segments是从运行的角度来描述elf文件，sections是从链接的角度来描述elf文件，也就是说，在链接阶段，我们可以忽略program header table来处理此文件，在运行阶段可以忽略section header table来处理此程序（所以很多加固手段删除了section header table）。从图中我们也可以看出，segments与sections是包含的关系，一个segment包含若干个section。
Section Header Table: 包含了文件各个segction的属性信息，我们都将结合例子来解释。

因为 so文件是以segment为单位映射到内存的，所以加密的原理我们就可以利用装载视图，对于Session的内容作特定的修改,比如对于so来说,e_entry 入口地址是无意义的，所以我们可以进行修改,当然修改字段也是有讲究的，并不是每个字段都是可以的,还有对于session表中的.init和.fini的特点，这里要注意下
>.init:可执行指令，构成进程的初始化代码，发生在main函数调用之前。
.fini:进程终止指令，发生在main函数调用之后。

以上这么分析感觉有点像c++的构造函数和析构函数，的确构造和析构是由此实现的。

>并且结合GGC的可扩展机制：__attribute__((section(".mytext")));可以把相应的函数和要保护的代码放在自己所定义的节里面,这就引入了我们今天的主题，可以把我们关键的so文件中的核心函数放在自己所定义的节里面，然后进行加密保护，在合适的时机构造解密函数，当然解密函数可以用这个_attribute__((constructor))进行定义;类似于C++构造函数发生在main函数之前。


### Android So的加载过程

具体的加载过程，可以参考下面的链接

[08-SO加载解析过程](https://shuwoom.com/?p=351)
[07-ELF文件格式分析](https://shuwoom.com/?p=286)
[腾讯Bugly干货分享 Android Linker 与 SO 加壳技术](https://zhuanlan.zhihu.com/p/22652847)
[Android ELF系列:ELF文件格式简析到linker的链接so文件原理分析](https://bbs.pediy.com/thread-249589.htm)

大致就是加载解析so，就是解析 ELF 文件，将解析的信息保存到 soinfo中，对于dlopen返回的其实就是这个类型的对象，下面重点看下当解析出soinfo的时候，还做了什么处理

CallConstructors
> 在编译 SO 时，可以通过链接选项-init或是给函数添加属性__attribute__((constructor))来指定 SO 的初始化函数，这些初始化函数在 SO 装载链接后便会被调用，再之后才会将 SO 的 soinfo 指针返回给 dl_open 的调用者。SO 层面的保护手段，有两个介入点, 一个是 jni_onload, 另一个就是初始化函数，比如反调试、脱壳等，逆向分析时经常需要动态调试分析这些初始化函数。完成 SO 的装载链接后，返回到 do_dlopen 函数, do_open 获得 find_library 返回的刚刚加载的 SO 的 soinfo，在将 soinfo 返回给其他模块使用之前，最后还需要调用 soinfo 的成员函数 CallConstructors。

```java
 soinfo* do_dlopen(const char* name, int flags, const Android_dlextinfo* extinfo) {
   ...
   soinfo* si = find_library(name, flags, extinfo);
   if (si != NULL) {
       si->CallConstructors();
   }
   return si;
 }
CallConstructors 函数会调用 SO 的首先调用所有依赖的 SO 的 soinfo 的 CallConstructors 函数，接着调用自己的 soinfo 成员变量 init 和 看 init_array 指定的函数，这两个变量在在解析 dynamic section 时赋值。

 void soinfo::CallConstructors() {
   //如果已经调用过，则直接返回
   if (constructors_called) {
     return;
   }
   // 调用依赖 SO 的 Constructors 函数
   get_children().for_each([] (soinfo* si) {
     si->CallConstructors();
   });
   // 调用 init_func
   CallFunction("DT_INIT", init_func);
   // 调用 init_array 中的函数
   CallArray("DT_INIT_ARRAY", init_array, init_array_count, false);
 }
```
有了以上分析基础后，在需要动态跟踪初始化函数时，我们就知道可以将断点设在 do_dlopen 或是 CallConstructors。

### So简单加密demo实现
首先看加密的实现
```java
#include <jni.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>
#include <elf.h>
#include <sys/mman.h>
#include <stdio.h>
#include <fcntl.h>
#include <elf.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include "PrintLog.h"

JNIEXPORT jint JNICALL
Java_com_example_androidelf_MainActivity_encoder(JNIEnv *env, jobject thiz) {
    char target_section[] = ".textdemo";
    //存储 Session的 String 索引表
    char *shstr = NULL;
    char *content = NULL;
    Elf64_Ehdr ehdr;
    Elf64_Shdr shdr;
    int i;
    unsigned int base, length;
    unsigned short nblock;
    unsigned short nsize;
    unsigned char block_size = 16;

    int fd;

    fd = open("/sdcard/libnative-lib.so", O_RDWR);
    if (fd < 0) {
        LOGD("open libnative-lib.so failed\n");
        goto _error;
    }

    //读取头部  ELF Header的结构体：
    if (read(fd, &ehdr, sizeof(Elf64_Ehdr)) != sizeof(Elf64_Ehdr)) {
        LOGD("Read ELF header error");
        goto _error;
    }

    //定位到shstrndx 对应的 Elf32_Shdr，代表session的段
    lseek(fd, ehdr.e_shoff + sizeof(Elf64_Shdr) * ehdr.e_shstrndx, SEEK_SET);

    //读取shstrndx 中的字符串，存放在str空间中
    if (read(fd, &shdr, sizeof(Elf64_Shdr)) != sizeof(Elf64_Shdr)) {
        LOGD("Read ELF section string table error");
        goto _error;
    }

    //分配内存空间，从shoff位置开始读取section header，存放在shdr,这个做为后续用来存储当前 Session的字符串的索引
    if ((shstr = (char *) malloc(shdr.sh_size)) == NULL) {
        LOGD("Malloc space for section string table failed");
        goto _error;
    }

    //定位到 sh_offset代表此session在文件中的偏移
    lseek(fd, shdr.sh_offset, SEEK_SET);

    //读取字符串的内容
    if (read(fd, shstr, shdr.sh_size) != shdr.sh_size) {
        LOGD("Read string table failed");
        goto _error;
    }

    //定位到 Section Header Table 偏移量
    lseek(fd, ehdr.e_shoff, SEEK_SET);
    // ehdr.e_shnum 代表当前 Section Header Tabele中 含有多少个 Session,下面遍历这些Session
    for (i = 0; i < ehdr.e_shnum; i++) {
        //依次执行读取操作，读取到  shdr 变量中
        if (read(fd, &shdr, sizeof(Elf64_Shdr)) != sizeof(Elf64_Shdr)) {
            LOGD("Find section .text procedure failed");
            goto _error;
        }
        LOGD("session name %s", shstr + shdr.sh_name);
        //依次读取 Section的名字，注意 sh_name 代表Session的名字，他实际上是.shstrtab中的索引，该string table中存储着所有 session 的名字
        //通过shdr -> sh_name 在str字符串中索引，与.mytext进行字符串比较，如果不匹配，继续读取
        if (strcmp(shstr + shdr.sh_name, target_section) == 0) {
            //找到对应的我们要加密的session， 通过shdr -> sh_offset 和 shdr -> sh_size字段，将.mytext内容读取并保存在content中。
            base = shdr.sh_offset;
            length = shdr.sh_size;
            LOGD("Find section %s\n", target_section);
            break;
        }
    }

    //定位到查找到session的位置
    lseek(fd, base, SEEK_SET);
    //分配内存空间，存储要加密的内容
    content = (char *) malloc(length);
    if (content == NULL) {
        LOGD("Malloc space for content failed");
        goto _error;
    }
    //读取内容
    if (read(fd, content, length) != length) {
        LOGD("Read section .text failed");
        goto _error;
    }

    nblock = length / block_size;
    nsize = base / 4096 + (base % 4096 == 0 ? 0 : 1);
    LOGD("base = %d, length = %d\n", base, length);
    LOGD("nblock = %d, nsize = %d\n", nblock, nsize);

    //为了验证第二节中关于section 字段可以任意修改的结论，这里，将shdr -> addr 写入ELF头e_shoff，将shdr -> sh_size 和 addr 所在内存块写入e_entry中，
    //即ehdr.e_entry = (length << 16) + nsize。当然，这样同时也简化了解密流程，还有一个好处是：如果将so文件头修正放回去，程序是不能运行的。
    ehdr.e_entry = (length << 16) + nsize;
    //ndroid7.0后JNI库必须保留Section Headers。由于加密时修改了shoff值，导致加载so库值解析Section Headers 解析不了，
    //包.dynamic section header was not found。修改测量：shoff和entry目的是为了存储加密的偏移大小和加密的大小。我们可以使用entry高低16位来分别存储着两个值。即可解决该问题。
    ehdr.e_flags = base;
    //ehdr.e_entry = (length >> 16) + base;

    //  为了便于理解，不使用复杂的加密算法。这里，只将content的所有内容取反，即 *content = ~(*content);
    for (i = 0; i < length; i++) {
        content[i] = ~content[i];
    }

    //最后将修改的内容写回去
    lseek(fd, 0, SEEK_SET);
    if (write(fd, &ehdr, sizeof(Elf64_Ehdr)) != sizeof(Elf64_Ehdr)) {
        LOGD("Write ELFhead to .so failed");
        goto _error;
    }

    lseek(fd, base, SEEK_SET);
    if (write(fd, content, length) != length) {
        LOGD("Write modified content to .so failed");
        goto _error;
    }

    LOGD("Completed");
    _error:
    free(content);
    free(shstr);
    close(fd);
    return 0;
}
```
加密流程：
> 1)  从so文件头读取section偏移shoff、shnum和shstrtab
2)  读取shstrtab中的字符串，存放在str空间中
3)  从shoff位置开始读取section header, 存放在shdr
4)  通过shdr -> sh_name 在str字符串中索引，与.mytext进行字符串比较，如果不匹配，继续读取
5)  通过shdr -> sh_offset 和 shdr -> sh_size字段，将.mytext内容读取并保存在content中。
6)  为了便于理解，不使用复杂的加密算法。这里，只将content的所有内容取反，即 *content = ~(*content);
7)  将content内容写回so文件中
8)  为了验证第二节中关于section 字段可以任意修改的结论，这里，将shdr -> addr 写入ELF头e_shoff，将shdr -> sh_size 和 addr 所在内存块写入e_entry中，即  ehdr.e_entry = (length << 16) + nsize。当然，这样同时也简化了解密流程，还有一个好处是：如果将so文件头修正放回去，程序是不能运行的。

这里要注意，对于第八点来说，在Android 7.0以上是不允许修改 e_shoff 的，具体可以参考下面的连接，这里为了方便将addr放在了 flags字段中,还有要注意是64位还是32位的，这个可以通过头部的class字段知道
[ELF学习日记-Android SO库文件头分析](https://blog.micblo.com/2018/02/10/Android-SO%E5%BA%93%E6%96%87%E4%BB%B6%E5%A4%B4%E5%88%86%E6%9E%90/)

解密的实现
```java
#include <jni.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>
#include <elf.h>
#include <sys/mman.h>
#include <stdio.h>
#include <fcntl.h>
#include <elf.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include "PrintLog.h"

jstring getString(JNIEnv*) __attribute__((section (".textdemo")));
jstring getString(JNIEnv* env){
	return (*env)->NewStringUTF(env, "Native method return 123!");
};

//解密时，需要保证解密函数在so加载时被调用，所以函数的声明为  void init_getString() __attribute__((constructor));
void init_getString() __attribute__((constructor));
unsigned long getLibAddr();

void init_getString(){
    char name[15];
    unsigned int nblock;
    unsigned int nsize;
    unsigned long base;
    unsigned long text_addr;
    unsigned int i;
    Elf64_Ehdr *ehdr;
    Elf64_Shdr *shdr;

    //首先调用 getLibAddr方法，得到so 文件在内存中的开地址
    base = getLibAddr();

    //读取前 52个字节，即 ELF 头，通过 e_shoff 获得 .mytext内存加载地址，endr_entry获取 .mytext大小和所在内存块
    //这是因为你存的时候，就是这样存储的
    ehdr = (Elf64_Ehdr *)base;

    text_addr = ehdr->e_flags + base;

    // ehdr.e_entry = (length << 16) + nsize;
    nblock = ehdr->e_entry >> 16;
    nsize = ehdr->e_entry & 0xffff;

    LOGD("nblock = %d nsize == %d \n", nblock,nsize);

    //修改.mytext所在内存块的读写权限
    if(mprotect((void *) base, 4096 * nsize, PROT_READ | PROT_EXEC | PROT_WRITE) != 0){
        LOGD("mem privilege change failed");
    }

    //将 [e_shoff,e_shoff+size]内存区域数据解密，即取反操作
    for(i=0;i< nblock; i++){
        char *addr = (char*)(text_addr + i);
        *addr = ~(*addr);
    }

    //修改会内存区域的读写权限
    if(mprotect((void *) base, 4096 * nsize, PROT_READ | PROT_EXEC) != 0){
        LOGD("mem privilege change failed");
    }
    LOGD("Decrypt success");
}

unsigned long getLibAddr(){
    unsigned long ret = 0;
    char name[] = "libnative-lib.so";
    char buf[4096], *temp;
    int pid;
    FILE *fp;
	//当前当前进程下面的所有的文件描述符
    pid = getpid();
    sprintf(buf, "/proc/%d/maps", pid);
    fp = fopen(buf, "r");
    if(fp == NULL)
    {
        LOGD("open failed");
        goto _error;
    }
    while(fgets(buf, sizeof(buf), fp)){
		//找到  libnative-lib.so 的内存映射地址
        if(strstr(buf, name)){
            temp = strtok(buf, "-");
            ret = strtoul(temp, NULL, 16);
            break;
        }
    }
    _error:
    fclose(fp);
    return ret;
}

JNIEXPORT jstring JNICALL
Java_com_example_androidelf_MainActivity_stringFromJNI( JNIEnv* env,jobject thiz )
{
    return getString(env);
}
```

在这里重点解释这个解密函数：
首先看到的是getLibAddr()这个函数：在介绍这个函数之前首先了解一个内存映射问题：
和Linux一样，Android提供了基于/proc的“伪文件”系统来作为查看用户进程内存映像的接口(cat /proc/pid/maps)。可以说，这是Android系统内核层开放给用户层关于进程内存信息的一扇窗户。通过它，我们可以查看到当前进程空间的内存映射情况，模块加载情况以及虚拟地址和内存读写执行（rwxp）属性等。
![结果显示](/uploads/so加密/进程下面打开的文件描述符.png)

因为根据前面加载so的流程可以知道，其实也是调用open函数打开一个文件，所以肯定可以这里面找到
接下来包括内存权限的修改以及函数的解密算法，最后包括内存权限的修改回去，应该都比较好理解。ok，以上编写完以后就编译生成.so文件


解密时，需要保证解密函数在so加载时被调用，那函数声明为：init_getString __attribute__((constructor))。(也可以使用c++构造器实现， 其本质也是用attribute实现)
解密流程：
>1)  动态链接器通过call_array调用init_getString
2)  Init_getString首先调用getLibAddr方法，得到so文件在内存中的起始地址
3)  读取前52字节，即ELF头。通过 e_flags 获得.textdemo内存加载地址，ehdr.e_entry获取.mytext大小和所在内存块
4)  修改.textdemo 所在内存块的读写权限
5)  将[e_flags, e_flags + size]内存区域数据解密，即取反操作：*content = ~(*content);
6)  修改回内存区域的读写权限
(这里是对代码段的数据进行解密，需要写权限。如果对数据段的数据解密，是不需要更改权限直接操作的)


这里要注意的是，我们的这个解密算法是肯定写在我们的so里面的，第一次运行的时候，我们只能执行安装操作，而不能直接使用，因为我们使用了解密函数，而一开始我们的so是没有加密的，那肯定会崩溃所以一开始我们我们可以将这个原始的so打包出来，然后放到sd卡下面，接着 程序运行的时候，加载  System.loadLibrary("encoder-lib"); 完成加密的操作

加密的结果，我们可以使用  readelf -S libnative-lib.so ,查看session信息，可以看到，我们加的session  textdemo 是在里面的
![结果显示](/uploads/so加密/添加的session.png)

拿到加密后的so使用  IDA 打开，可以看到出现了错误
![结果显示](/uploads/so加密/加密后的so使用ID.png)

接着拿到加密后的so重新加载，此时就可以解析出来
![结果显示](/uploads/so加密/解码成功.png)

### LLVM
LLVM可以做到达到混淆native层代码的效果，增大破解的难度，可以参考下面的文章
[手把手编译OLLVM（obfuscator-llvm）](https://blog.csdn.net/tabactivity/article/details/81262570)
[利用OLLVM混淆Android Native代码篇一](https://blog.csdn.net/mergerly/article/details/60962140?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromBaidu-1.channel_param&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromBaidu-1.channel_param)
也可以参考我之前写的关于LLVM的搭建过程,其实有些目前市面上的sdk，有些就是直接使用了llvm来做的
[Window下OOLLVM Android编译环境搭建](https://aheadsnail.github.io/2018/06/17/Window%E4%B8%8BOOLLVM-Android%E7%BC%96%E8%AF%91%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA/)

### 参考链接
1. [ELF学习日记-Android SO库文件头分析](https://blog.micblo.com/2018/02/10/Android-SO%E5%BA%93%E6%96%87%E4%BB%B6%E5%A4%B4%E5%88%86%E6%9E%90/#%E5%81%B7%E5%B0%B1%E5%AE%8C%E4%BA%8B%E4%BA%86)
2. [Android SO库文件头分析](https://www.matools.com/blog/190347556)
3. [android so加固之section加密](https://xz.aliyun.com/t/5328)
4. [ELF中可以被修改又不影响执行的区域](https://cloud.tencent.com/developer/article/1193471)
5. [ELF文件解析和加载(附代码)](https://blog.csdn.net/qq_40239482/article/details/100132259)
6. [Android SO文件保护加固——加密篇](https://blog.csdn.net/zhangmiaoping23/article/details/80016399)
7. [ELF文件格式分析](https://shuwoom.com/?p=286)
8. [SO加载解析过程](https://shuwoom.com/?p=351)
9. [ELF 解析.dynamic 节](https://blog.csdn.net/ylcangel/article/details/37997853)
10. [ELF文件格式修复](https://www.jianshu.com/p/27bed97a3896)
11. [无文件的ELF执行](http://www.polaris-lab.com/index.php/2019/09/)
12. [【腾讯Bugly干货分享】Android Linker 与 SO 加壳技术](https://zhuanlan.zhihu.com/p/22652847)
13. [Android ELF系列:ELF文件格式简析到linker的链接so文件原理分析](https://bbs.pediy.com/thread-249589.htm)
14. [Windows下的ELF文件解析代码C++](https://blog.csdn.net/helloworld_ptt/article/details/79575783)
15. [elfloader](https://github.com/elemeta/elfloader)
16. [ReflectiveELFLoader](https://github.com/nsxz/ReflectiveELFLoader)




