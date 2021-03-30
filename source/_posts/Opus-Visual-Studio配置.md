---
layout: pager
title: Opus Visual Studio配置
date: 2018-06-25 22:44:15
tags: [Opus,Visual Studio]
description:  Opus Visual Studio配置
---
### 什么是Opus
> opus是完全开放的,免版税的,高度通用的音频编解码器。作品是无与伦比的交互式语音和音乐在互联网上传播,但也用于存储和流媒体的应用程序。标准化是因特网工程任务组(IETF)为RFC 6716,从Skype编解码器和Xiph整合技术。Org的编解码器。

opus可以处理广泛的音频应用程序,包括IP电话、视频会议、游戏内聊天,甚至远程现场音乐表演。它可以规模从低比特率窄带语音非常高质量的立体声音乐。支持功能:
> 比特率从6 kb / s到510 kb / s
采样率从8 kHz(窄带)48千赫(fullband)
帧大小从2.5毫秒到60毫秒
支持两个恒定比特率(CBR)和可变比特率(VBR)
从窄带fullband音频带宽
支持语音和音乐
支持单声道和立体声
支持多达255个频道的节目(多流道帧)
动态可变比特率、音频带宽,和帧大小
良好的鲁棒性和包丢失隐藏损失(PLC)
浮点和定点实现

Opus官网 http://www.opus-codec.org/


### 为什么要在visual studio 中配置opus
> 做为一名android开发人员，要将这个opus的库移植到android上，最好的就是官方的列子了，而且这些例子都是默认有支持visual Studio的，使用过visual Studio的人都知道，
visual Studio也是一个非常牛逼，非常方便的一个软件，提供了很多方便的功能，当然我们不是没事干配置Visual Studio 的项目，这么做的目的还是最终为了移植到android上
通过了解他提供的demo，或者尝试的去修改demo代码，修改完之后，再移植到android上面,下面会介绍怎么配置visual Studoiio 

### visual studio中配置opus

关于Opus的配置介绍，可以查看官网的介绍
> Opus development http://www.opus-codec.org/development/

我们可以通过上面的连接将要下载的内容，克隆下来，依次执行

opus库为主要的编解码库
```bash
git clone https://git.xiph.org/opus.git
```

Opus-tools编码/解码 opus到wav,或者wav到opus的实现
```bash
git clone https://git.xiph.org/opus-tools.git
```

Opusfile API提供了一个高层次的解码和寻求在.opus文件类似libvorbisfile Vorbis提供。
```bash
git clone https://git.xiph.org/opusfile.git
```

libopusenc提供高级API创建.opus文件和流。
```bash
git clone https://git.xiph.org/libopusenc.git
```

#### opus库生成

在克隆下来的opus库，目录下面会有一个win32目录，里面会有一个vs2015目录，在这个目录里面会有一个opus.ls文件，在安装好了visual Studio 之后，是可以直接打开这个文件的visual Sutdio打开之后，点击生成，重新生成解决方案，会产生下面的结果

![](/uploads/Opus Window/opus生成.png)

上图所示，生成了5个结果，对应的项目为左边的刚好5个,那这样opus库配置完成,opus生成的结果，生成的目录以及文件为
![](/uploads/Opus Window/opus生成结果.png)


#### libopusenc库生成

在克隆下来的libopusenc库，目录下面会有一个win32目录，里面会有一个vs2015目录，在这个目录里面会有一个opusenc.ls文件，在安装好了visual Studio 之后，是可以直接打开这个文件的
visual Sutdio打开之后，点击生成，重新生成解决方案，会产生下面的结果

![](/uploads/Opus Window/libopusenc.png)

上图所示，生成了1个结果，对应的项目为左边的刚好1个,那这样libopusenc库配置完成,libopusenc生成的结果，生成的目录以及文件为
![](/uploads/Opus Window/libopusenc生成结果.png)

在libopusenc 项目右键选项中的c/c++一览，常规选项，有一个选项为包含目录中有一个..\..\..\opus\include ，这个刚好对应的是我们的opus库中，所以要先编译opus库
![](/uploads/Opus Window/libopus生成需知.jpg)

#### Opusfile库生成

在克隆下来的opusfile库，目录下面会有一个win32目录，里面会有一个vs2015目录，在这个目录里面会有一个opusfile.ls文件，在安装好了visual Studio 之后，是可以直接打开这个文件的
visual Sutdio打开之后，点击生成，重新生成解决方案，会出现错误，大致就是说缺少相应的文件比如ogg/ogg.h文件等,我们通过点击项目opusfile右键查看属性

![](/uploads/Opus Window/opusfile依赖.jpg)

上图所示，我们缺少ogg ，以及opensssl文件,所以我们必须要先编译对应的文件 通过上面的图片可以知道ogg,openssl的目录必须要跟opusfile同一级的目录，而且文件夹名必须为ogg,openssl，要不然对应不上

#### Ogg库生成
首先克隆ogg的代码
```bash
git clone -q https://github.com/xiph/ogg.git
```
进入ogg的目录，里面也有一个win32目录，在win32里面有一个VS2015目录，这个目录里面存在一个名为libogg_static.sln ，这个就是我们visual Studio可以打开的文件，
双击打开这个工程,然后通过生成， 重新生成解决方案，会产生下面的结果

![](/uploads/Opus Window/ogg生成解决方案.png)

ogg生成的结果，生成的目录以及文件为
![结果显示](/uploads/Opus Window/ogg生成文件目录.png)

#### openssl库生成
具体的编译过程可以参考这篇文章 openssl编译 https://www.cnblogs.com/lpxblog/p/5382653.html
![](/uploads/Opus Window/openssl编译结果.png)

我们通过点击openfile_example 项目的右键 在链接器一览 输入中可以看出，这个项目需要的外部库
![](/uploads/Opus Window/opusfile_example 依赖.png)

我们可以直接将openssl 编译的文件中找到对应的lib，然后拷贝到当前的目录,或者修改lib库文件的依赖，我们采用前者 拷贝之后的目录为
![](/uploads/Opus Window/openssl文件拷贝.png)

最后我们点击生成，重新生成解决方案，会产生下面的结果
![](/uploads/Opus Window/opusfile生成结果.png)

opusfile 生成结果的目录
![](/uploads/Opus Window/opusfile生成目录.png)

#### Opustool库生成

在克隆的项目中，在VS2015目录中存在一个opus-tools.sln文件，这个就是我们visual Studio可以打开的文件，双击打开这个工程,然后通过生成， 重新生成解决方案，
会产生错误，缺少flac文件,缺少libFLAC_static.lib

![](/uploads/Opus Window/open-tool依赖.jpg)

所以我们必须要先编译对应的文件我们通过执行
```bash
git clone -q https://github.com/xiph/flac.git，
```
将flac的代码克隆下来，要注意的是，通过上图可知，flac的文件目录要跟opus目录处于同一级别，而且文件名 必须要为flac，要不然就要修改对应的依赖配置

目录文件结构为
![](/uploads/Opus Window/opus目录文件.png)

#### 编译flac

在克隆的代码目录中有一个FLAC.sln文件，双击使用Visual Studio打开,如果直接使用生成，重新生成解决方案，会出现无法打开libFLAC_static.lib之类的，这是因为要提前编译
对应的lib库文件，然后再去编译其他的，下面是要先去编译的项目，通过点击对应的项目，右键然后执行生成，就可以生成对应的库文件

![](/uploads/Opus Window/flac库生成优先级.png)

在生成libFLAC_static.lib的文件的时候，会出现找不到对应的ogg/ogg.h之类的文件，在flac对应的右键属性中可以找到对应的依赖
![](/uploads/Opus Window/flac 依赖ogg.jpg)


所以我们要将编译好的对应的ogg拷贝到对应的文件里面，首先拷贝ogg的头文件,在ogg的工程目录中，有一个inlcude目录里面有一个ogg的目录，将这个拷贝到flac目录中的include目录
下面是拷贝之后的结果

![](/uploads/Opus Window/flac ogg头文件的拷贝.png)

然后拷贝 ogg生成的文件 libogg_static.lib 到flac 中的 flac\objs\debug\lib 目录中

然后重新生成,这些库文件生成之后，然后点击生成，生成解决方案，就可以将全部的文件生成，注意这里不能点击重新生成解决方案，要不然又出现上面的问题，生成的目录文件为

![](/uploads/Opus Window/flac生成的文件内容.png)

点击项目右键属性查看依赖
![](/uploads/Opus Window/opus-tools依賴文件.jpg)

可以看出来，opus-tools需要依赖很多的lib，比如opus.lib,opus_file.lib等，所以我们要将opus-tools的编译放在最后面，从这里还知道opus-tools也需要openssl
我们可以参考上面生成opus-file的时候，怎么引进openssl的方式拷贝内容


最终生成的结果为：
![](/uploads/Opus Window/opus-tools生成的结果.png)

以上就是Opus 在window工程的配置，之后我们就可以在Window下面方便的查看代码,然后修改代码，最后就修改之后的代码，转移到Android来编译

