---
layout: pager
title: Fba 街机对战研究的准备
date: 2018-10-10 10:06:56
tags: [Android,NDK,Fba]
description: Fba 街机对战研究的准备
---

### 概述

> Fba 街机对战研究的准备

<!--more-->


### Kaillera

前面一篇文章已经分析过，Fba街机的编译过程，像Window下的编译已经完成，Fba本身是支持对战的，只是对战是采用第三方的库实现的，window下的街机实现，只是在使用的时候链接进来这个
外部库，同时使用这个开源库提供的接口，下面来看看window下的街机对战 入口在哪里

![结果显示](/uploads/Fba/Window对战的入口.png)

但是如果本地文件夹里面如果没有这个对战库的化，就会出现这个错误，

![结果显示](/uploads/Fba/本地缺少对战库.png)

根据错误提示 kailleraclient.dll ，所以就去对应的查找 http://www.kaillera.com/download.php 这个网站就有对应的dll 的下载,这里下载一个客户端,
甚至在 Wiki 都有介绍  https://en.wikipedia.org/wiki/Kaillera  说明这个开源库还是有点意义的

![结果显示](/uploads/Fba/Wikipedia显示Kaillera.jpg)

下载包含的内容
![结果显示](/uploads/Fba/对战库包含的内容.png)

将下载的dll文件放到对应的fba源码目录里面，再次的打开对战选项，出现了这个结果，代表对战库引入成功

![结果显示](/uploads/Fba/使用对战库成功.jpg)

下面大致分析下，这个对战库提供了什么功能，首先他会列出当前所有的服务器列表，上图有体现，点击其中的一个服务器就会进入到类似于房间

![结果显示](/uploads/Fba/kaillera对战库功能.png)

右上角是当前玩家的列表，你可以在这里面跟他们聊天，聊天窗口体现在左侧的位置，下面是当前房间里面创建的游戏，如果想进入这个游戏里面，前提是你要有这个游戏才可以进入,否则会提示你没有对应
的rom，当然你也可以自己创建游戏，当然如果是自己创建的游戏当然是立刻就可以进入到这个游戏房间里面

![结果显示](/uploads/Fba/游戏房间.png)

由于我是房主，所以这里面我可以踢出用户的权限，而且有startGame的权限，点击StartGame就可以了游戏

![结果显示](/uploads/Fba/联网开始游戏.jpg)

上面就是这个对战库大致的功能，当然这个dll还是只涉及到了客户端的实现，还有一端是服务端，对应的应用程序  http://www.kaillera.com/download.php 也可以从这里面下载体验，大致就是下载下来一个exe的形式

![结果显示](/uploads/Fba/kaillera服务端应用程序.jpg)

如果在本地的化，就连接这个服务器监听的端口27888 就可以连接上服务器了，客户端就可以对战了,当然这并不是源码，但是幸运的是，这个对战库是开源的，具体的链接为 https://sourceforge.net/projects/okai/ 这里面可以下载到对应的客户端，以及服务端，但是这里面的代码并不是像之前那个网站所呈现的结果，但是大体的功能是可以运行的,包括服务端的代码,下面是使用visualStudio 加载打开项目，运行的结果显示，包括服务端，客户端

![结果显示](/uploads/Fba/kaillera运行结果.png)

虽然界面可能显示的不太一样，但是这没有关系，最主要的是将这客户端以及服务端的代码，放到fba中是可以对战的，说明是没有问题的，由于这对战库的代码比较多，所以需要有对应的阅读代码的工具
这里选择使用 c/c++ 开发最喜欢的 VisualStuido ，要想快速的看懂代码，调试代码是必不可以少的，对于服务端的代码来说，调试代码是本身就可以的，但是对于客户端的代码，因为他编译出来的是一个 dll，所以调试代码比较麻烦，这里介绍下，怎么调试别人提供的dll 代码

### VisualStudio配置

1.首先新建一个VisualStudio 项目，可以按照下面的方式来创建

![结果显示](/uploads/Fba/visualStudio创建项目.png)

2.在创建的工程里面，点击解决方案的右键，添加新建项目，将另一个dll的工程包含进来,此时就存在俩个项目了，将第一个项目设置为启动项目

![结果显示](/uploads/Fba/MyKailleraDemo.png)

3.点击MyKaileraClient 右键，打开属性-〉配置属性-〉C/C++-〉常规-〉调试信息格式，这里不能为“禁用”。这里选择这个

![结果显示](/uploads/Fba/visualStudio配置禁用.png)

4.在代码生成这一选项，改为下面的值

![结果显示](/uploads/Fba/VisualStdio配置二.png)

5.点击n02项目 右键，打开属性-〉配置属性-〉链接器-〉调试-〉生成调试信息，这里设为“是”。这里不能为“禁用”。这里选择这个

![结果显示](/uploads/Fba/visualStudio配置三.png)

6.在代码生成这一选项，改为下面的值

![结果显示](/uploads/Fba/visualStdio配置四.png)

7.点击确定，应用，点击全部重新生成解决方案，生成完之后，将生成的dll文件的文件全部拷贝到测试项目里面

![结果显示](/uploads/Fba/visualStudio配置五.png)

8.在测试项目的右键，通过点击属性，找到下面的选项页面，找到输入，在附加依赖项里面，输入我们生成dll，的lib文件，然后我们就可以在测试项目里面使用它们了

![结果显示](/uploads/Fba/visualStdio配置6.png)

9.在测试工程里面添加断点，然后通过F11进入到dll项目里面

![结果显示](/uploads/Fba/visualStdio配置8.png)

接下来就是分析下Fba的源码是怎么样使用这个开源的对战库的，由于这个源码编译下来是采用makefile编译的，并没有像类似VC6,VisualStdio 所能识别的文件，比如VisualStdio识别的文件为.sln后缀
这里推荐使用另一款的IDE，叫做VSCode，这是一款比较轻巧的IDE，安装包非常小，但是功能也是挺强大的 有点类似阉割版的VisualStdio版，最主要的他可以直接的调试exe，下面介绍下怎么使用

### VSCode配置

首先安装完VSCode 之后，打开应用程序，安装下面的插件

![结果显示](/uploads/Fba/VSCode安装插件.png)

插件安装完之后，选择文件选项，找到打开文件夹，找到我们编译好的fba目录，将整个目录加载进来,加载进来之后，在调试选项列表里面，选择添加配置,修改program 的内容我们要调试的exe文件,这里要注意的是这个exe在编译的时候是有加上调试信息选项的，这一点在官网也有体现，具体就是通过  DEBUG = 1 来控制的 ，这样编译出来的exe才是可以调试的，才能打上断点

![结果显示](/uploads/Fba/VSCode添加调试配置.png)

接下来找到Window下的main函数的入口，在里面打上断点既可，主函数的路径为 /src/burner/win32/main.cpp  接着在调试选项里面，点击启动调试

![结果显示](/uploads/Fba/VSCode启动调试.jpg)

按下F11 进入代码里面
![结果显示](/uploads/Fba/进入代码里面.png)

至此，我们可以愉快的看代码了，接下来一篇会大致分析下Window下的Fba 源码过程






