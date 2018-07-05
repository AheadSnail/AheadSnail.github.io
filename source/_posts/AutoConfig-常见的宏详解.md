---
layout: pager
title: AutoConfig 常见的宏详解
date: 2018-04-11 11:15:12
tags: [AutoConf,AutoMake,Libtool]
description: AutoConfig 常见的宏详解
---

AutoConfig 常见的宏详解
<!--more-->


configure.ac由一些宏组成（如果已经有源代码，你可以运行autoscan来产生一个configure.scan文件，在此基础修改成configure.ac将更加方便） 
最基本的组成可以是下面的

```
AC_INIT([PACKAGE], [VERSION], [BUG-REPORT-ADDRESS])
# Checks for programs.
# Checks for libraries.
# Checks for header les.
# Checks for typedefs, structures, and compiler characteristics.
# Checks for library functions.
# Output les.
AC_CONFIG_FILES([FILES])
AC_OUTPUT

基本含义已经在上篇文章中介绍了，这里不再叙述。

AC_INIT(PACKAGE, VERSION, BUG-REPORT-ADDRESS)
autoconf 强制性初始化。告诉autoconf包名称，版本，一个bug报告emall。 
例如：

1
AC_INIT([hello], [1.0], [bug-report@address])
并且这些名称将出现在config.h(如果你有在config.ac中这样写的化 AC_CONFIG_HEADERS([config.h]))，你可以在程序直接引用这些宏。

创建头文件的HEADER.in，HEADERS包含使用AC_DEFINE的定义。  
AC_CONFIG_HEADER 宏用于生成config.h文件，以便 autoheader 命令使用,这样我们定义的宏都会生成在config.h文件里面
AC_CONFIG_HEADERS([config.h])

/* Define to the address where bug reports for this package should be sent. */
#define PACKAGE_BUGREPORT "BUG-REPORT-ADDRESS"
 
/* Define to the full name of this package. */
#define PACKAGE_NAME "FULL-PACKAGE-NAME"
 
/* Define to the full name and version of this package. */
#define PACKAGE_STRING "FULL-PACKAGE-NAME VERSION"
 
/* Define to the one symbol short name of this package. */
#define PACKAGE_TARNAME "full-package-name"
 
/* Define to the version of this package. */
#define PACKAGE_VERSION "VERSION"

需要的最低autoconf版本，如：AC_PREREQ([2.65])
AC_PREREQ([2.67])

设置系统信息host变量。 只执行AC_CANONICAL_SYSTEM中关于主机类型功能的子集。 对于不是编译工具链（compiler toolchain）一部分的程序，这就是所需要的全部功能。
AC_CANONICAL_HOST

设置了系统信息target变量，也即是会得到当前系统的target信息
AC_CANONICAL_TARGET

初始化Automake 这个选项中的subdir-objects会为生成的中间文件自动创建与源码一致的目录结构，而不是统一放在同一目录
AM_INIT_AUTOMAKE([subdir-objects])

指定libTool需要的版本号
LT_PREREQ([2.2.6])

为使用m4注释，使用m4内置的dnl； 它使m4放弃本行中其后的所有文本。因为在调用AC_INIT之前，所
dnl See versioning rule:
dnl  http://www.gnu.org/software/libtool/manual/html_node/Updating-version-info.html

定义变量，最后会在生成的MakeFile文件中可以使用(如果你的输出有多个makeFIle文件的化，就会在每一个makeFIle文件中都会定义这样的变量)
从一个shell变量创建一个输出变量。让AC_OUTPUT把变量variable替换到输出文件中（通常是一个或多个`Makefile'） 就像这样 LT_CURRENT = 0    
AC_SUBST(LT_CURRENT, 0)  

使用一个 m4 /目录存储本地宏在解释当地的宏,源码目录下面确实有这个目录,也即是设置自定义的宏所在的目录，这里是m4目录里面
m4目录里面是用户自定义的宏,这样可以保证自定义的宏在同一个文件夹里面，而不会随便放置
AC_CONFIG_MACRO_DIR([m4])

一个安全的检查。FILE将是一个发布的源文件。这让configure脚本确保自己运行在正确的目录中
AC_CONFIG_SRCDIR([src/a2io.h])

用来定义一个这样的宏，在你的config.h 中 如果你有写（AC_CONFIG_HEADERS([config.h])） 也即是在config.h头文件中定义这样的宏 #define HAVE_OPENSSL 1 最后一个参数为他的注释
AC_DEFINE([HAVE_OPENSSL], [1], [Define to 1 if you have openssl.])

#如果已经调用了AC_CONFIG_HEADER，那么就不是创建DEFS，而是由AC_OUTPUT 创建一个头文件，这是通过在一个暂时文件中把正确的值替换到#define语句中来实现的
#类似于AC_DEFINE，但还要对variable和value进行三种shell替换（每种替换只进行一次）： 变量扩展（`$'），命令替换（``'），以及反斜线传义符（`/'）
#相当于定义变量BUILD = $build的值放在config.h文件里面,而之前的build，host，target可以根据前面的AC_CANONICAL_HOST 获取到

也即是AC_DEFINE_UNQUOTED 也是用来定义宏，在config.h文件中，跟AC_DEFINE 功能是一样的。。。只是他可以在里面使用变量，比如$build,他可以取的他的值，
下面的类似这样#define BUILD "x86_64-pc-linux-gnu"
AC_DEFINE_UNQUOTED([BUILD], ["$build"], [Define build-type])  

#Autoconf自定义宏是用宏AC_DEFUN定义的，该宏与m4的内置define宏相似。 除了定义一个宏 
AC_DEFUN (name, [body])

AC_ARG_WITH (package, help-string, [action-if-given], [action-if-not-given]) 
这个宏可以给configure增加–-with-package这样模式的参数。很多软件都有可选项用来打开扩展功能，AC_ARG_WITH就是干这个的。它的第一参数给出扩展包的名称，
出现在–with-后面。第二个参数给出一个参数说明，用在./configure –help中。[action-if-given]如果有该选项就被执行，[action-if-not-given]如果没有加这个选项就执行
果然在configure -help中看到了 --with-libuv   Use libuv   
AS_HELP_STRING([–with-militant], 为一个帮助字符串 将在configure –help中被显示出来就跟 echo类似
shell命令中对于echo命令的封装，也即是会在命令行中输出  $have_getrandom_interface 的值，这里为false
AC_MSG_RESULT([$have_getrandom_interface])
  
下面是一个列子 在configure.ac中这样使用  ARIA2_ARG_WITH([libuv])
AC_DEFUN([ARIA2_ARG_WITH],
[AC_ARG_WITH([$1],
	AS_HELP_STRING([--with-$1], [Use $1.]),
	[with_$1_requested=$withval with_$1=$withval], [with_$1=no])]
)
上面的意思就是说定义一个宏ARIA2_ARG_WITH，后面代表的为参数， 所以$1 就可以获取到传递的参数 这里为libuv ，上面也即是用来在config.h文件中 添加--with-libuv Use libuv 


#另外使用AC_ARG_ENABLE宏可以为configure增加–enable-feature 或者 –disable-feature这样的选项。 
#AC_ARG_ENABLE (feature, help-string, [action-if-given], [action-if-not-given]) 
#如果configure中加了给定的选项，就执行action-if-given，否则执行action-if-not-given。 
AC_DEFUN([ARIA2_ARG_ENABLE],
[AC_ARG_ENABLE([$1],
	AS_HELP_STRING([--enable-$1], [Enable $1 support.]),
	[enable_$1_requested=$enableval enable_$1=$enableval], [enable_$1=no])]
)

相当于是echo，是用来帮助输出显示的。。 用来告知用户当前正在干什么  这里就是显示为 --with-ca-bundle=FILE Use FILE as default CA bundle....
AS_HELP_STRING([--with-ca-bundle=FILE],[Use FILE as default CA bundle.]),
相当于是echo，只是用来输出错误的信息,而且会退出 上面的宏也是用来打印的，但是他不会退出
AC_MSG_FAILURE([ar is not found in the system.])

声明一个变量，变量值的时候配置被启动的内容保存在缓存中，包括如果它没有在命令行上指定，但是通过环境然后当运行./configure --help的时候会出现关于他的描述
也即是相当于是一个环境变量的意思，也即是一个configure中的一个选项 这里既是为 ARIA2_STATIC  Set 'yes' to build a statically linked aria2
AC_ARG_VAR([ARIA2_STATIC], [Set 'yes' to build a statically linked aria2])

通过测试来定义一个变量USE_GNUTLS_SYSTEM_CRYPTO_POLICY=1。 如果测试通过就会在config.h文件中定义这样的宏 #define USE_GNUTLS_SYSTEM_CRYPTO_POLICY 1
AS_IF([test "x$enable_gnutls_system_crypto_policy" = "xyes"], [
  AC_DEFINE([USE_GNUTLS_SYSTEM_CRYPTO_POLICY], [1], [Define to 1 if using gnutls system wide crypto policy .])
])

AC_PROG_CXX                      自动检测要使用的C++编译器 检查是否设置了环境变量 CXX或CCC（按该顺序）; 如果是，则将输出变量CXX设置为其值。
AC_PROG_CC						 自动检测要使用的C编译器
AC_PROG_INSTALL					 生成安装脚本 install-sh 	
AC_PROG_MKDIR_P				     将输出变量设置MKDIR_P为一个程序，以确保对于每个参数
AC_PROG_YACC					 如果bison找到，将输出变量设置YACC为'野牛 - 年”。否则，如果byacc找到，请设置YACC为'byacc”。否则设置YACC为'YACC

AC_LANG_PROGRAM 扩展成由序言组成的源文件，然后将主体作为主函数的主体（例如，main在C中）。由于它使用AC_LANG_SOURCE，后者的功能是可用的。
AC_LANG_PROGRAM (prologue, body)

AC_COMPILE_IFELSE 用来检车input的值，在成功时运行shell命令 action-if-true，否则运行if-false
AC_COMPILE_IFELSE (input, [action-if-true], [action-if-false])

下面为一个列子
AC_COMPILE_IFELSE([AC_LANG_PROGRAM([[
]],
[[
int *a = nullptr;       
]])],
[],
#如果不是就输出错误的信息
[AC_MSG_FAILURE([C++ compiler does not understand nullptr, perhaps C++ compiler is too old.  Try again with new one (gcc >= 4.8.3 or clang >= 3.4)])])
首先一开始用来检测输入，这里为[AC_LANG_PROGRAM([[ ，而这个是用来检测程序的， 第一个参数为函数头部，第二个参数为函数的body部分,用来检测当前的程序执行body的时候会不会出现问题
]],
[[
int *a = nullptr;       
]])]
后面的俩个参数 即为检测成功之后，还有检测失败之后的action，这里为[],[AC_MSG_FAILURE([C++ compiler does not understand nullptr, perhaps C++ compiler is too old.Try again with new one (gcc >= 4.8.3 or clang >= 3.4)])]


如果后面的条件满足，就定义HAVE_LIBUV 然后可以在Makefile.am 中 有这样的使用 if HAVE_LIBUV ，就表示如果有定义。可以用来条件编译
AM_CONDITIONAL([HAVE_LIBUV], [test "x$have_libuv" = "xyes"])

用来检测头文件是不是存在 ,对于每个在以空格分隔的参数列表header-file出现的头文件，如果存在，就定义 HAVE_header-file 定义在config.h文件中（全部大写）。如果给出了action-if-found，
它就是在找到一个头文件的时候执行的附加shell代码。你可以把`break'作为它的值 以便在第一次匹配的时候跳出循环。如果给出了action-if-not-found，它就在找不到 某个头文件的时候被执行。
宏： AC_CHECK_HEADERS (header-file... [, action-if-found [, action-if-not-found]])

下面为一个列子
    AC_CHECK_HEADERS([windows.h \  	检测的头文件列表， 如果存在就会在config.h文件中定义对应的宏 比如  #define HAVE_WINDOWS_H 1
                      winsock2.h \
                      ws2tcpip.h \
                      mmsystem.h \
                      io.h \
                      iphlpapi.h\
                      winioctl.h \
                      share.h], [], [],
                      [[
#ifdef HAVE_WS2TCPIP_H
# include <ws2tcpip.h>
#endif
#ifdef HAVE_WINSOCK2_H
# include <winsock2.h>
#endif
#ifdef HAVE_WINDOWS_H
# include <windows.h>
#endif
                      ]])

检测是否有这样的头文件，这个跟上面的AC_CHECK_HEADERS 唯一不同的地方就是，如果存在 也不会在config.h文件中定义这样的宏，而是如果存在就执行action-true,不存在就执行action-no-found
AC_CHECK_HEADER([wincrypt.h], [have_wintls_headers=yes], [have_wintls_headers=no], [[
#ifdef HAVE_WINDOWS_H
# include <windows.h>
#endif
  ]])
  
AC_CHECK_FUNCS：检查C标准库中是否存在函数。 如果找到，则定义预处理器宏HAVE_ [function]。它生成一个测试程序，声明一个具有相同名称的函数，然后编译并链接它。
  
检测是否有这样的库，存在就执行aciton-true，否则执行action-no-found
AC_HAVE_LIBRARY([crypt32],[have_wintls_libs=yes],[have_wintls_libs=no])

# Checks for header files.
检测如何获得alloca。本宏试图通过检查`alloca.h'或者预定义C预处理器宏 __GNUC__和_AIX来获得alloca的内置（builtin）版本。 如果本宏找到了`alloca.h'，它就定义HAVE_ALLOCA_H。
在config.h文件中确实有这样的宏定义 #define HAVE_ALLOCA_H 1
AC_FUNC_ALLOCA

如果含有标准C（ANSI C）头文件，就定义STDC_HEADERS。 特别地，本宏检查`stdlib.h'、`stdarg.h'、`string.h'和`float.h'； 如果系统含有这些头文件
在config.h 文件中确实有这样的宏定义 #define STDC_HEADERS 1
AC_HEADER_STDC

检查是否 stdbool.h存在并符合C99，并将结果缓存在ac_cv_header_stdbool_h变量中。如果类型 _Bool已定义，则定义HAVE__BOOL为1。在config.h文件中是这样的 #undef HAVE__BOOL */
AC_HEADER_STDBOOL

如果C编译器不能完全支持关键字const，就把const定义成空 在config.h文件中是这样定义的 #undef const
AC_C_CONST

如果C编译器支持关键字inline，就什么也不作。如果C编译器可以接受__inline__或者__inline，就把inline定义成可接受的关键字，否则就把inline定义为空。在config.h头文件中是这样的 #undef inline
AC_C_INLINE

如果c编译器支持int16_t 就定义，如果不支持就定义为空， 这里config.h 头文件中是这样定义的  #undef int16_t 
AC_TYPE_INT16_T

如果c编译器支持int32_t 就定义，如果不支持就定义为空， 这里config.h 头文件中是这样定义的  #undef int32_t 
AC_TYPE_INT32_T

如果c编译器支持int64_t 就定义，如果不支持就定义为空， 这里config.h 头文件中是这样定义的  #undef int64_t 
AC_TYPE_INT64_T

如果c编译器支持int8_t 就定义，如果不支持就定义为空， 这里config.h 头文件中是这样定义的  #undef int8_t 
AC_TYPE_INT8_T

如果没有定义mode_t，就把mode_t定义成int。 这里config.h 头文件中是这样定义的   #undef mode_t
AC_TYPE_MODE_T

如果没有定义off_t，就把off_t定义成long。 这里config.h头文件中是这样定义的 #undef off_t 
AC_TYPE_OFF_T

如果没有定义size_t，就把size_t定义成unsigned。 这里的config.h头文件中是这样定义的  #undef size_t 
AC_TYPE_SIZE_T

size_t如果标准头文件没有定义它， 定义一个合适的类型。 这里的config.h头文件中是这样定义的 #undef ssize_t
AC_TYPE_SSIZE_T

如果程序可能要同时引入`time.h'和`sys/time.h'，就定义TIME_WITH_SYS_TIME  这里的config.h头文件中是这样定义的 #define TIME_WITH_SYS_TIME 1
AC_HEADER_TIME

如果`time.h'没有定义struct tm，就定义TM_IN_SYS_TIME 这里的config.h头文件中是这样定义的 #undef TM_IN_SYS_TIME
AC_STRUCT_TM

如果 stdint.h 要么 inttypes.h没有定义类型 uint16_t，定义uint16_t为一个无符号整数类型  这里的config.h头文件中是这样定义的 #undef uint16_t
AC_TYPE_UINT16_T

这里的config.h头文件中是这样定义的 #undef uint32_t
AC_TYPE_UINT32_T

这里的config.h头文件中是这样定义的 #undef uint64_t
AC_TYPE_UINT64_T

这里的config.h头文件中是这样定义的 #undef uint8_t
AC_TYPE_UINT8_T

如果没有定义pid_t，就把pid_t定义成int。 这里的config.h头文件中是这样定义的  #undef pid_t
AC_TYPE_PID_T

如果C编译器不理解关键字volatile，则将其定义volatile为空。程序可以简单地使用 volatil 这里的config.h头文件中是这样定义的 #undef volatile
AC_C_VOLATILE

对于定义的每种类型的类型，定义 HAVE_类型（全部大写）。每种类型都必须遵守规则AC_CHECK_TYP   这里的config.h头文件中是这样定义的 #define HAVE_PTRDIFF_T 1
AC_CHECK_TYPES([ptrdiff_t])

检查类型是否定义。它可能是一个编译器内置类型或由includes定义。 包括一系列包含指令，默认为AC_INCLUDES_DEFAULT（参见默认包含），这些指令在被测试的类型之前使用。
在C中，type必须是一个类型名称，这样表达式'sizeof（类型）'是有效的（但'sizeof（（type ））' 不是）。
在编译C ++时应用相同的测试，这意味着在C ++ 类型中应该是类型标识并且不应该是匿名的struct 或者是 union

如果存在这个type就会在config.h头文件里面定义HAVE_TYPE 的宏，这里config.h头文件中有这样的宏 #define HAVE_A2_STRUCT_TIMESPEC 1
AC_CHECK_TYPE([struct timespec], [have_timespec=yes], [have_timespec=no])

如果字（word）按照最高位在前的方式储存（比如Motorola和SPARC，但不包括Intel和VAX，CPUS），就定义 WORDS_BIGENDIAN
AC_C_BIGENDIAN

安排64位文件偏移量，称为 大文件支持。在某些主机上，必须使用特殊的编译器选项来构建可以访问大文件的程序。
AC_SYS_LARGEFILE

AC_LINK_IFELSE(input,action-if-true,action-if-false) 也即是使用shell命令来检测input，如果成功执行action—if-true，如果失败执行action-if-false
下面为一个列子
AC_LINK_IFELSE([AC_LANG_PROGRAM([[
#include <sys/syscall.h>      这个相当于是函数的头部分
#include <linux/random.h>
]],
[[
int x = GRND_NONBLOCK;        这个相当于是函数的body部分
int y = (int)SYS_getrandom;
]])],
  [have_getrandom_interface=yes
   AC_DEFINE([HAVE_GETRANDOM_INTERFACE], [1], [Define to 1 if getrandom linux syscall interface is available.])], 如果成功就会在config.h头文件中定义 目前是失败#undef HAVE_GETRANDOM_INTERFACE
  [have_getrandom_interface=no])		检测失败之后执行的action
这里的input为要检测的函数部分AC_LANG_PROGRAM();这个检测执行的结果

这个宏用来检查成员，在给定的path中查找，如果找到了，执行action-if-found,否则执行 action-if-not-found
AC_CHECK_MEMBER (aggregate.member, [action-if-found], [action-if-not-found], [includes = ‘AC_INCLUDES_DEFAULT’])

下面为一个列子
AC_CHECK_MEMBER([struct sockaddr_in.sin_len],
                [AC_DEFINE([HAVE_SOCKADDR_IN_SIN_LEN],[1], 如果找到了 就在config.h 中定义这样的宏 目前config.h的定义为 #undef HAVE_SOCKADDR_IN_SIN_LEN 代表没有找到
                  [Define to 1 if struct sockaddr_in has sin_len member.])],
                [], 如果没有找到
                [[
#include <sys/types.h>
#include <sys/socket.h>  为includes
#include <netinet/in.h>
]])

也即是找struct sockaddr_in.sin_len 结构体，在[[ #include <sys/types.h> #include <sys/socket.h> #include <netinet/in.h>]] 里面查找，如果找到了就执行action-found,这里的action-found
为[AC_DEFINE([HAVE_SOCKADDR_IN_SIN_LEN],[1],[Define to 1 if struct sockaddr_in has sin_len member.])]，
如果找到了 就在config.h 中定义这样的宏 目前config.h的定义为 #undef HAVE_SOCKADDR_IN_SIN_LEN 代表没有找到执行 action-if-not-found 这里为 []

设置要输出的目录,也即是说上面可以用来输出在makefile中的变量，下面的输出文件里面都会有一份的存在
AC_CONFIG_FILES([Makefile
                src/Makefile
                src/libaria2.pc
                src/includes/Makefile
                test/Makefile
                po/Makefile.in
                lib/Makefile
                doc/Makefile
                doc/manual-src/Makefile
                doc/manual-src/en/Makefile
                doc/manual-src/en/conf.py
                doc/manual-src/ru/Makefile
                doc/manual-src/ru/conf.py
                doc/manual-src/pt/Makefile
                doc/manual-src/pt/conf.py
                deps/Makefile])
AC_OUTPUT
```

下面是对应的参考链接
[AutoConf](https://www.gnu.org/software/autoconf/manual/autoconf.html)
[AutoMake](https://www.gnu.org/software/automake/manual/automake.html)
[libTool](https://www.gnu.org/software/libtool/manual/html_node/LT_005fINIT.html)
[AutoConf中文版](https://blog.csdn.net/romandion/article/details/1688234#SEC29)