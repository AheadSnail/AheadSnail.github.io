---
layout: pager
title: scrcpy 源码分析
date: 2018-12-01 10:07:07
tags: [scrcpy,NDK]
description: scrcpy 源码分析
---

### 概述

> scrcpy 源码分析

<!--more-->

### 简介
> 前面一篇文章介绍了scrcpy 的使用，以及对应的编译，可以实现在Ubuntu 编译调试的客户端代码，这篇文章会介绍 简要的介绍下scrcpy的源码实现，在介绍scrcpy源码之前，可以查看 https://github.com/Genymobile/scrcpy/blob/master/DEVELOP.md 关于scrcpy的大概实现

比如客户端包含了三个线程，每个线程负责的工作
![结果显示](/uploads/scrcpy源码实现/客户端线程数量.png)

服务端包含了俩个线程，每个线程负责的工作
![结果显示](/uploads/scrcpy源码实现/服务端线程数量.png)

### scrcpy的原理
> 从上面的链接也可以大概的知道 scrcpy 的实现原理,服务端负责发送界面上的数据到客户端，客户端使用ffmpeg,sdl 显示出来，客户端要监听界面上的操作，如果有操作触发的时候，要将操作发送给服务端，服务端，利用发射的原理 模拟用户调用的过程


### 源码分析
```C
首先是客户端启动，也即是对应的main方法运行, 这里启动运行的时候，并没有传递任何的参数，所以 args代表默认的参数 

int main(int argc, char *argv[]) {
#ifdef __WINDOWS__
    // disable buffering, we want logs immediately
    // even line buffering (setvbuf() with mode _IOLBF) is not sufficient
    setbuf(stdout, NULL);
    setbuf(stderr, NULL);
#endif
    //默认的参数选项
    struct args args = {
        .serial = NULL,
        .crop = NULL,
        .record_filename = NULL,
        .help = SDL_FALSE,
        .version = SDL_FALSE,
        .show_touches = SDL_FALSE,
        .port = DEFAULT_LOCAL_PORT, // #define DEFAULT_LOCAL_PORT 27183
        .max_size = DEFAULT_MAX_SIZE, //#define DEFAULT_MAX_SIZE 0
        .bit_rate = DEFAULT_BIT_RATE, // #define DEFAULT_BIT_RATE 8000000
    };
    //解析参数,当前我们没有传递参数进来
    if (!parse_args(&args, argc, argv)) {
        return 1;
    }
    
    ...
	
#if LIBAVCODEC_VERSION_INT < AV_VERSION_INT(58, 9, 100)
    //注册FFmpeg
    av_register_all();
#endif

    //FFmpeg 注册网络
    if (avformat_network_init()) {
        return 1;
    }
	
    //将解析之后的选项 赋值到options 中，这个代表最终的选项,也即是后面要应用的选项结构体
    struct scrcpy_options options = {
        .serial = args.serial,
        .crop = args.crop,
        .port = args.port,
        .record_filename = args.record_filename,
        .max_size = args.max_size,
        .bit_rate = args.bit_rate,
        .show_touches = args.show_touches,
        .fullscreen = args.fullscreen,
    };
	
    //启动
    int res = scrcpy(&options) ? 0 : 1;

    avformat_network_deinit(); // ignore failure
    ...
}

客户端main函数做的操作，主要是解析传递过来的参数，当然如果你没有传递参数的时候，也有默认的实现，当参数解析完之后，将最终的参数结果保存到 options 中，之后调用 
scrcpy(&options),这才是关键的实现

//启动scrcpy
SDL_bool scrcpy(const struct scrcpy_options *options) {
	//判断是否需要录制文件
    SDL_bool send_frame_meta = !!options->record_filename;

    //1 使用adb连接手机，发送命令 ，如果返回值为false，代表出现了错误
    if (!server_start(&server, options->serial, options->port,options->max_size, options->bit_rate, options->crop, send_frame_meta)) {
        return SDL_FALSE;
    }

    process_t proc_show_touches = PROCESS_NONE;
    SDL_bool show_touches_waited;
    //2 是否有设置显示touch
    if (options->show_touches) {
        LOGI("Enable show_touches");
        proc_show_touches = set_show_touches_enabled(options->serial, SDL_TRUE);
        show_touches_waited = SDL_FALSE;
    }

    //下面初始化SDL
    SDL_bool ret = SDL_TRUE;
    //3 初始化SDL 激活视频子模块
    if (!sdl_init_and_configure()) {
        ret = SDL_FALSE;
        goto finally_destroy_server;
    }

    //4 连接server端 ,返回连接的socket
    socket_t device_socket = server_connect_to(&server);
    //验证socket是否合法
    if (device_socket == INVALID_SOCKET) {
        server_stop(&server);
        ret = SDL_FALSE;
        goto finally_destroy_server;
    }

    //#define DEVICE_NAME_FIELD_LENGTH 64  ,用来存储设备的名称
    char device_name[DEVICE_NAME_FIELD_LENGTH];
    struct size frame_size;
	
    //5 当连接上的时候，服务端会告知客户端设备的名称，以及视频的大小
    if (!device_read_info(device_socket, device_name, &frame_size)) {
    	//读取失败，停止服务
        server_stop(&server);
        ret = SDL_FALSE;
        goto finally_destroy_server;
    }

    //6 根据客户端告知的视频的大小，初始化SDL 视频显示的界面
    if (!frames_init(&frames)) {
        server_stop(&server);
        ret = SDL_FALSE;
        goto finally_destroy_server;
    }

    //7 初始化file_handler
    if (!file_handler_init(&file_handler, server.serial)) {
        ret = SDL_FALSE;
        server_stop(&server);
        goto finally_destroy_frames;
    }

    struct recorder *rec = NULL;
    //8 如果选项里面有制定录制文件
    if (options->record_filename) {
        if (!recorder_init(&recorder, options->record_filename, frame_size)) {
            ret = SDL_FALSE;
            server_stop(&server);
            goto finally_destroy_file_handler;
        }
        rec = &recorder;
    }

    //9 初始化解码的结构体
    decoder_init(&decoder, &frames, device_socket, rec);

    // now we consumed the header values, the socket receives the video stream
    // start the decoder

    //10 创建解码的线程,用来事实的将客户端的界面渲染出来
    if (!decoder_start(&decoder)) {
        ret = SDL_FALSE;
        server_stop(&server);
        goto finally_destroy_recorder;
    }

    //11 初始化control 结构体
    if (!controller_init(&controller, device_socket)) {
        ret = SDL_FALSE;
        goto finally_stop_decoder;
    }

    //12 初始化 controller 线程，这个线程的作用就是发送界面上的点击事件给服务端
    if (!controller_start(&controller)) {
        ret = SDL_FALSE;
        goto finally_destroy_controller;
    }

    //13 初始化SDL 显示窗口 device_name 为 客户端告知的设备号， fame_size 为当前设备
    if (!screen_init_rendering(&screen, device_name, frame_size)) {
        ret = SDL_FALSE;
        goto finally_stop_and_join_controller;
    }

    //是否要显示触摸点
    if (options->show_touches) {
        wait_show_touches(proc_show_touches);
        show_touches_waited = SDL_TRUE;
    }

    //是否要显示全屏
    if (options->fullscreen) {
        screen_switch_fullscreen(&screen);
    }

    //14 事件循环
    ret = event_loop();

    LOGD("quit...");
    screen_destroy(&screen);
	
    ...
    return ret;
}

可以看到这个函数每一步的操作都是非常关键的，下面会分点的方式依次来解析 ,首先是

//使用adb连接手机，发送命令
if (!server_start(&server, options->serial, options->port,options->max_size, options->bit_rate, options->crop, send_frame_meta)) {
   return SDL_FALSE;
}

//server_start函数实现
SDL_bool server_start(struct server *server, const char *serial,
                      Uint16 local_port, Uint16 max_size, Uint32 bit_rate,
                      const char *crop, SDL_bool send_frame_meta) {
    server->local_port = local_port;

    //serial不为空，代表在多个手机的时候选择一个手机
    if (serial) {
        server->serial = SDL_strdup(serial);
        if (!server->serial) {
            return SDL_FALSE;
        }
    }

    //启动adb push 将服务端的jar推送到手机上,如果返回false，代表出现了错误,返回false
    if (!push_server(serial)) {
        SDL_free((void *) server->serial);
        return SDL_FALSE;
    }

    //标识推送jar成功
    server->server_copied_to_device = SDL_TRUE;

    //执行 adb reverse ，adb forward 隐射，也即是至少有一个成功，要不然都会返回false
    if (!enable_tunnel(server)) {
        SDL_free((void *) server->serial);
        return SDL_FALSE;
    }

    //如果到了这里，并且tunnel_forward 为fale，代表执行 adb reverse 成功 ,也即是将远端的手机端口隐射到了pc端口上
    if (!server->tunnel_forward) {
    	//远端的手机端口映射到了本地的pc端口上，监听这个端口
        server->server_socket = listen_on_port(local_port);
        //判断是否出现了错误，出现了错误，删除端口映射 ，释放资源
        if (server->server_socket == INVALID_SOCKET) {
            LOGE("Could not listen on port %" PRIu16, local_port);
            disable_tunnel(server);
            SDL_free((void *) server->serial);
            return SDL_FALSE;
        }
    }

    //服务端将会连接我们的server_socket ,这一步骤就是启动我们服务端的服务了
    server->process = execute_server(serial, max_size, bit_rate,
                                     server->tunnel_forward, crop,
                                     send_frame_meta);

    //如果执行不成功,返回false ，关闭socket
    if (server->process == PROCESS_NONE) {
        if (!server->tunnel_forward) {
            close_socket(&server->server_socket);
        }
        disable_tunnel(server);
        SDL_free((void *) server->serial);
        return SDL_FALSE;
    }

    server->tunnel_enabled = SDL_TRUE;
    return SDL_TRUE;
}

执行 push_server 函数,这个函数主要是将我们的服务端jar 使用adb 命令发送给服务器,下面是对应的函数实现

//推送jar
static SDL_bool push_server(const char *serial) {
    //启动adb 推送服务端的jar到  /data/local/tmp/scrcpy-server.jar 中  process 为执行的结果
    process_t process = adb_push(serial, get_server_path(), DEVICE_SERVER_PATH);
    //检查结果 如果不为PROCESS_SUCCESS 返回false
    return process_check_success(process, "adb push");
}

//获取到jar存放的目录
static const char *get_server_path(void) {
    //如果环境变量中有设置,就取环境变量的值，否则取默认的
    const char *server_path = getenv("SCRCPY_SERVER_PATH");
    if (!server_path) {
    	//PREFIX PREFIXED_SERVER_PATH   /home/yuhui/WorkSpace/scrcpy/app/scrcpy-server.jar 这个路径就是我存放jar的路径
        server_path = DEFAULT_SERVER_PATH;
    }
    return server_path;
}

而PREFIX，PREFIXED_SERVER_PATH 是在编译scrcpy的时候生成的config.h文件中定义的

//服务端jar的路径
#define PREFIX "/home/yuhui/WorkSpace/scrcpy/app"
#define PREFIXED_SERVER_PATH "/scrcpy-server.jar"

//执行 adb push命令 ,对于 push_server 调用来说， local 为 我本地存放这个jar的路径 /home/yuhui/WorkSpace/scrcpy/app/scrcpy-server.jar remote为服务端存放jar的地方
//也即是 /data/local/tmp/scrcpy-server.jar
process_t adb_push(const char *serial, const char *local, const char *remote) {
    ...
    //local为本地的server的jar remote为服务端jar存放的路径
    const char *const adb_cmd[] = {"push", local, remote};
    //执行adb push
    process_t proc = adb_execute(serial, adb_cmd, ARRAY_LEN(adb_cmd));
    ...
    //返回执行的结果
    return proc;
}

//执行adb的命令
process_t adb_execute(const char *serial, const char *const adb_cmd[], int len) {
    const char *cmd[len + 4];
    int i;
    process_t process;
    //cmd[0] = adb
    cmd[0] = get_adb_command();
    //serial代表是否是在多个手机中选择一个连接
    if (serial) {
        cmd[1] = "-s";
        cmd[2] = serial;
        i = 3;
    } else {
        i = 1;
    }

    //将adb要执行的参数，填充到cmd 中
    memcpy(&cmd[i], adb_cmd, len * sizeof(const char *));
    //字符串的结束标识
    cmd[len + i] = NULL;
    //执行adb的命令,r 代表执行的结果
    enum process_result r = cmd_execute(cmd[0], cmd, &process);
    //如果出现了错误
    if (r != PROCESS_SUCCESS) {
        show_adb_err_msg(r);
        return PROCESS_NONE;
    }
    return process;
}

这个函数是主要完成adb 命令的拼接，对于pursh_server 来说，当前的命令就为 adb push /home/yuhui/WorkSpace/scrcpy/app/scrcpy-server.jar  /data/local/tmp/scrcpy-server.jar
如果当前只有连接一个手机的化,当然如果当前有多个手机拼接的命令就变成了 adb -s "手机的编号" push ....    之后执行   cmd_execute() 函数

//执行adb的命令
enum process_result cmd_execute(const char *path, const char *const argv[], pid_t *pid) {
    int fd[2];

    //首先创建一个管道
    if (pipe(fd) == -1) {
        perror("pipe");
        return PROCESS_ERROR_GENERIC;
    }

    enum process_result ret = PROCESS_SUCCESS;

    //fork 进程， 注意fork会被执行俩次，第一次执行fork大于0 代表为父进程，第二次执行fork 为子进程,所以对于管道来说，子进程父进程都是有的
    *pid = fork();
    //fork失败
    if (*pid == -1) {
        perror("fork");
        ret = PROCESS_ERROR_GENERIC;
        goto end;
    }

    //父进程
    if (*pid > 0) {
        // parent close write side  关闭掉写的一断
        close(fd[1]);
        fd[1] = -1;
        // wait for EOF or receive errno from child  父进程监听读的一端，这个函数会阻塞，这里父进程在等到子进程的执行结果,这里父进程监听到结果之后，将内容赋值到ret中
        if (read(fd[0], &ret, sizeof(ret)) == -1) {
            perror("read");
            ret = PROCESS_ERROR_GENERIC;
            goto end;
        }
    } else if (*pid == 0) {//子进程
        // child close read side  子进程关闭掉读的一段
        close(fd[0]);
        //fcntl是计算机中的一种函数，通过fcntl可以改变已打开的文件性质。f
        if (fcntl(fd[1], F_SETFD, FD_CLOEXEC) == 0) {
        	//execvp（执行文件） 如果执行成功则函数不会返回，执行失败则直接返回-1，失败原因存于errno中。 也即是在子进程中执行adb的命令
            execvp(path, (char *const *)argv);
            //ENOENT 2 无此文件或目录
            if (errno == ENOENT) {
                ret = PROCESS_ERROR_MISSING_BINARY;
            } else {
                ret = PROCESS_ERROR_GENERIC;
            }
            //出现了错误
            perror("exec");
        } else {
            perror("fcntl");
            ret = PROCESS_ERROR_GENERIC;
        }

        // send ret to the parent 发送执行的结果给父进程
        if (write(fd[1], &ret, sizeof(ret)) == -1) {
            perror("write");
        }

        // close write side before exiting 之后子进程退出
        close(fd[1]);
        _exit(1);
    }
end:
	//关闭管道
    if (fd[0] != -1) {
        close(fd[0]);
    }
    if (fd[1] != -1) {
        close(fd[1]);
    }
    return ret;
}

这个函数通过创建一个管道用于子进程跟父进程通信，在子进程中调用了 execvp函数，执行命令 ，将执行的结果 发送给父进程，之后子进程退出，这里要注意，如果父进程先退出的化，
子进程就变成了孤儿进程了,执行 execvp函数的时候，如果当前你并没有设置adb的环境变量，这里会出现错误的,最终将结果一层层返回,最终回到调用  push_server 函数，
如果返回值为false 就代表出现了错误

if (!push_server(serial)) {
   SDL_free((void *) server->serial);
   return SDL_FALSE;
}

接着 执行 adb reverse ，adb forward 隐射，也即是至少有一个成功，要不然都会返回false
if (!enable_tunnel(server)) {
   SDL_free((void *) server->serial);
   return SDL_FALSE;
}

static SDL_bool enable_tunnel(struct server *server) {
    //执行adb reverse命令 将手机指定的端口隐射到pc 的local_port,如果允许 就直接返回true了
    if (enable_tunnel_reverse(server->serial, server->local_port)) {
        return SDL_TRUE;
    }
    //那到了这里就是不允许反向隐射 下面执行正向的隐射
    LOGW("'adb reverse' failed, fallback to 'adb forward'");
    //标示为 使用adb forward 替换 adb referse
    server->tunnel_forward = SDL_TRUE;
    //执行 adb forward命令 完成端口的隐射  将本地PC指定Port端口,映射到设备手机指定Port端口上.
    return enable_tunnel_forward(server->serial, server->local_port);
}

static SDL_bool enable_tunnel_reverse(const char *serial, Uint16 local_port) {
    process_t process = adb_reverse(serial, SOCKET_NAME, local_port);
    //检查adb reverse执行的结果
    return process_check_success(process, "adb reverse");
}

//执行正向隐射
static SDL_bool enable_tunnel_forward(const char *serial, Uint16 local_port) {
    process_t process = adb_forward(serial, local_port, SOCKET_NAME);
    return process_check_success(process, "adb forward");
}

对于enable_tunnel函数，主要是执行 adb reverse 判断是否能完成反向映射，如果成功，就直接返回，如果不成功，执行 adb forward 完成端口的正向隐射,对于adb命令的执行，这里就不分析了，
前面已经分析过,接着判断是否反向映射成功，如果成功的化，我们就要执行tcp的端口监听

//如果到了这里，并且tunnel_forward 为fale，代表执行 adb reverse 成功 ,也即是将远端的手机端口隐射到了pc端口上
if (!server->tunnel_forward) {
    //远端的手机端口映射到了本地的pc端口上，监听这个端口
    server->server_socket = listen_on_port(local_port);
    //判断是否出现了错误，出现了错误，删除端口映射 ，释放资源
    if (server->server_socket == INVALID_SOCKET) {
       LOGE("Could not listen on port %" PRIu16, local_port);
       disable_tunnel(server);
       SDL_free((void *) server->serial);
       return SDL_FALSE;
    }
}

//监听端口
static socket_t listen_on_port(Uint16 port) {
    return net_listen(IPV4_LOCALHOST, port, 1);
}
socket_t net_listen(Uint32 addr, Uint16 port, int backlog) {
    //SOCK_STREAM 代表当前 创建的是一个tcp的socket
    socket_t sock = socket(AF_INET, SOCK_STREAM, 0);
    if (sock == INVALID_SOCKET) {
        perror("socket");
        return INVALID_SOCKET;
    }
    int reuse = 1;
    //设置socket的属性 SO_REUSEADDR 使得这个监听的端口变成可以复用的。
    if (setsockopt(sock, SOL_SOCKET, SO_REUSEADDR, (const void *) &reuse, sizeof(reuse)) == -1) {
        perror("setsockopt(SO_REUSEADDR)");
    }
    SOCKADDR_IN sin;
    sin.sin_family = AF_INET;
    sin.sin_addr.s_addr = htonl(addr); // htonl() harmless on INADDR_ANY
    sin.sin_port = htons(port);
    //绑定端口
    if (bind(sock, (SOCKADDR *) &sin, sizeof(sin)) == SOCKET_ERROR) {
        perror("bind");
        return INVALID_SOCKET;
    }
    //坚听端口
    if (listen(sock, backlog) == SOCKET_ERROR) {
        perror("listen");
        return INVALID_SOCKET;
    }
    //返回当前的socket对象
    return sock;
}

接着客户端发送指令 启动服务端的jar  服务端将会连接我们的server_socket ,这一步骤就是启动我们服务端的服务了
server->process = execute_server(serial, max_size, bit_rate,
                                     server->tunnel_forward, crop,
                                     send_frame_meta);
									 
//执行服务端的jar tunnel_forward 代表是否执行 adb forward ，如果为false为 执行了 adb referse
static process_t execute_server(const char *serial,
                                Uint16 max_size, Uint32 bit_rate,
                                SDL_bool tunnel_forward, const char *crop,
                                SDL_bool send_frame_meta) {
    char max_size_string[6];
    char bit_rate_string[11];
    //C 语言中的 格式化输出
    sprintf(max_size_string, "%"PRIu16, max_size);
    sprintf(bit_rate_string, "%"PRIu32, bit_rate);

    //在android系统中运行jar操作步骤：
    //1.       打包编译jar包
    //2.       将jar包导入android设备中
    //adb push test.jar  /data/local/tmp     //将PC端编译好的jar包push到android设备中的/data/local/tmp目录下
    //3.       设置CLASSPATH
    //export CLASSPATH=/data/local/tmp/apt.jar
    //4.       启动jar
    //app_process /data/local/tmp  com.app.process.test.Test      // data/local/apt.jar包所在路径，com.app.process.test.Test含有main方法的类名

    //要执行的命令
    const char *const cmd[] = {
        "shell",
        "CLASSPATH=/data/local/tmp/scrcpy-server.jar",
        "app_process",
        "/", // unused
        "com.genymobile.scrcpy.Server",
        max_size_string,
        bit_rate_string,
        tunnel_forward ? "true" : "false",
        crop ? crop : "''",
        send_frame_meta ? "true" : "false",
    };
    //adb shell 执行命令，这样手机端指定路径下的jar就会被启动，执行main函数
    return adb_execute(serial, cmd, sizeof(cmd) / sizeof(cmd[0]));
}

客户端通过执行 adb shell 命令将服务端jar启动起来,这样服务端的程序就启动起来了，此时我们就有必要分析下服务端的代码了

//客户端通过adb shell 执行命令，让服务端的jar启动起来， 也即是会执行这个main函数，args为传递的参数
public static void main(String... args) throws Exception {
    Thread.setDefaultUncaughtExceptionHandler(new Thread.UncaughtExceptionHandler() {
        @Override
        public void uncaughtException(Thread t, Throwable e) {
           Ln.e("Exception on thread " + t, e);
        }
    });

    //解析传递过来的参数，配置运行
    Options options = createOptions(args);

    //启动服务端的逻辑，根据客户端传递过来的参数
    scrcpy(options);
}

createOptions 函数完成客户端传递参数的解析，并将最终的参数保存到  Options ，这里有一个参数比较重要  tunnelForward ，代表客户端是否要监听端口
private static Options createOptions(String... args) {
   Options options = new Options();
   ...
   //解析mas_size属性
   int maxSize = Integer.parseInt(args[0]) & ~7; // multiple of 8
   options.setMaxSize(maxSize);
   //解析
   int bitRate = Integer.parseInt(args[1]);
   options.setBitRate(bitRate);

   // use "adb forward" instead of "adb tunnel"? (so the server must listen)
   //解析客户端传递的 tunnel_forward 标识，这个标识了 客户端是执行了 adb forward 还是执行了 adb referse 之后的结果，如果为true 代表 客户端执行了 adb forward
   //也即是客户端将pc端口映射到了 手机的的端口上, 这个时候，服务端要完成监听端口的操作
   boolean tunnelForward = Boolean.parseBoolean(args[2]);
   //标识客户端是否执行了 adb forward
   options.setTunnelForward(tunnelForward);
   ...
   return options;
}

之后服务端执行 scrcpy(options) 函数
private static void scrcpy(Options options) throws IOException {
   //构建Device 对象，确定屏幕的大小，视频的大小，注册屏幕旋转监听
   final Device device = new Device(options);

   //判断服务端是否需要监听端口
   boolean tunnelForward = options.isTunnelForward();

   //获取跟客户端的连接对象，并且发送当前手机的型号，屏幕的大小
   try (DesktopConnection connection = DesktopConnection.open(device, tunnelForward)) {
   
        //从客户端传递过来的参数中，初始化ScreenEncoder 对象
        ScreenEncoder screenEncoder = new ScreenEncoder(options.getSendFrameMeta(), options.getBitRate());
		
        // asynchronous 服务端开启一个控制同步线程，用于同步客户端发送的按键内容
        startEventController(device, connection);
         try {
                // synchronous  主线程主要用来压缩屏幕的视频界面发送给客户端显示
                screenEncoder.streamScreen(device, connection.getFd());
            } catch (IOException e) {
                // this is expected on close
                Ln.d("Screen streaming stopped");
            }
     }
}

public Device(Options options)
{
   //根据客户端传递过来的参数，确定屏幕的大小
   screenInfo = computeScreenInfo(options.getCrop(), options.getMaxSize());

   //给WindowManagerSerice 中的 watchRotation 方法，注册这个AIDL回调监听，所以当rotation 改变的时候，这里会接收到内容
   registerRotationWatcher(new IRotationWatcher.Stub()
   {
      @Override
      public void onRotationChanged(int rotation) throws RemoteException
      {
          //当WindowMangerService 监听到屏幕发生了旋转的时候，会触发这个这个方法
          synchronized (Device.this)
          {
             screenInfo = screenInfo.withRotation(rotation);

             // notify
             if (rotationListener != null)
             {
                rotationListener.onRotationChanged(rotation);
             }
         }
      }
   });
}

接着服务端执行 DesktopConnection connection = DesktopConnection.open(device, tunnelForward)

//tunnelForward 为服务端是否需要监听端口，还是连接端口
public static DesktopConnection open(Device device, boolean tunnelForward) throws IOException {
   LocalSocket socket;
   //监听端口
   if (tunnelForward) {
       //内部会调用accept函数，会阻塞直到有客户端连接上。才会返回
       socket = listenAndAccept(SOCKET_NAME);
	   
       // send one byte so the client may read() to detect a connection error
       //所以如果到了这里，就说明客户端已经连接上了，服务端发送一个字节告知他们连接成功
      socket.getOutputStream().write(0);
   } else {
      //如果为false 代表 服务端直接执行连接，客户端在监听
      socket = connect(SOCKET_NAME);
   }

   //到了这里就说明俩者已经连接 , 构建一个DesktopConnection 代表俩者的连接对象
   DesktopConnection connection = new DesktopConnection(socket);

   //服务端得到手机的视频大小
   Size videoSize = device.getScreenInfo().getVideoSize();

   //服务端告知客户端 当前设备的名称 以及 当前屏幕的大小
   connection.send(Device.getDeviceName(), videoSize.getWidth(), videoSize.getHeight());
   return connection;
}

//listenAndAccept 函数实现 服务端注册一个tcp的sokcet 等待连接
private static LocalSocket listenAndAccept(String abstractName) throws IOException {
   //创建一个Socket
   LocalServerSocket localServerSocket = new LocalServerSocket(abstractName);
   try {
        //等待连接，这里要注意这个函数会阻塞的，直到有客户端连接上
        return localServerSocket.accept();
    } finally {
       localServerSocket.close();
   }
}

如果服务端要负责监听端口的化，会执行 localServerSocket.accept()，阻塞直到客户端有连接，要不然是不会往下执行的，此时回到客户端代码,此时客户端的server_start函数就执行完毕了，回到scrcpy 函数，执行第二点

process_t proc_show_touches = PROCESS_NONE;
SDL_bool show_touches_waited;
//是否有设置显示touch
if (options->show_touches) {
   LOGI("Enable show_touches");
   proc_show_touches = set_show_touches_enabled(options->serial, SDL_TRUE);
   show_touches_waited = SDL_FALSE;
}

//执行 adb shell settings put system show_touches true
static process_t set_show_touches_enabled(const char *serial, SDL_bool enabled) {
    const char *value = enabled ? "1" : "0";
    const char *const adb_cmd[] = {
        "shell", "settings", "put", "system", "show_touches", value
    };
    return adb_execute(serial, adb_cmd, ARRAY_LEN(adb_cmd));
}
判断是否要显示touch 对应的也是调用adb 命令

接着执行  下面初始化SDL
SDL_bool ret = SDL_TRUE;

//初始化SDL 激活视频子模块
if (!sdl_init_and_configure()) {
   ret = SDL_FALSE;
   goto finally_destroy_server;
}

//SDL初始化
SDL_bool sdl_init_and_configure(void) {
    //分析SDL的初始化函数SDL_Init()。该函数可以确定希望激活的子系统。SDL_Init()函数原型如下  SDL_INIT_VIDEO：视频
    if (SDL_Init(SDL_INIT_VIDEO)) {
        LOGC("Could not initialize SDL: %s", SDL_GetError());
        return SDL_FALSE;
    }

    //atexit()用来设置一个程序正常结束前调用的函数. 当程序通过调用exit()或从main 中返回时, 参数function 所指定的函数会先被调用, 然后才真正由exit()结束程序. 可以用来释放
    atexit(SDL_Quit);

    // Use the best available scale quality
    if (!SDL_SetHint(SDL_HINT_RENDER_SCALE_QUALITY, "2")) {
        LOGW("Could not enable bilinear filtering");
    }
    return SDL_TRUE;
}

接着 连接server端 ,返回连接的socket
socket_t device_socket = server_connect_to(&server);
//验证socket是否合法
if (device_socket == INVALID_SOCKET) {
   server_stop(&server);
   ret = SDL_FALSE;
   goto finally_destroy_server;
}

//客户端连接服务端
socket_t server_connect_to(struct server *server) {
    //如果是反向连接，也即是将远端手机端的端口隐射到pc上
    if (!server->tunnel_forward) {
    	//等待服务端的连接,如果成功，返回一个socket
        server->device_socket = net_accept(server->server_socket);
    } else {
    	//如果是正向连接，将pc端的端口隐射到远端的手机端
        Uint32 attempts = 100;
        Uint32 delay = 100; // ms
        server->device_socket = connect_to_server(server->local_port, attempts, delay);
    }
    //判断连接socket是否合法
    if (server->device_socket == INVALID_SOCKET) {
        return INVALID_SOCKET;
    }
    //由于服务端已经连接上我们监听的端口，并且创建了一个新的socket 所以这个监听的socket就没有用了,关闭
    if (!server->tunnel_forward) {
        // we don't need the server socket anymore
        close_socket(&server->server_socket);
    }

    // the server is started, we can clean up the jar from the temporary folder  由于这个服务端的代码jar已经运行了，此时移除推送的jar
    remove_server(server->serial); // ignore failure
    //标识服务端的jar已经移除
    server->server_copied_to_device = SDL_FALSE;

    // we don't need the adb tunnel anymore
    disable_tunnel(server); // ignore failure
    //移除之后，将端口映射改为false
    server->tunnel_enabled = SDL_FALSE;

    //返回这个连接的socket
    return server->device_socket;
}

这样客户端就跟服务端连接上了，服务端也不会继续阻塞了,此时回到服务端代码
//tunnelForward 为服务端是否需要监听端口，还是连接端口
public static DesktopConnection open(Device device, boolean tunnelForward) throws IOException {
      
   //内部会调用accept函数，会阻塞直到有客户端连接上。才会返回
   socket = listenAndAccept(SOCKET_NAME);

   // send one byte so the client may read() to detect a connection error
   //所以如果到了这里，就说明客户端已经连接上了，服务端发送一个字节告知他们连接成功
   socket.getOutputStream().write(0);
      
   //到了这里就说明俩者已经连接 , 构建一个DesktopConnection 代表俩者的连接对象
   DesktopConnection connection = new DesktopConnection(socket);

   //服务端得到手机的视频大小
   Size videoSize = device.getScreenInfo().getVideoSize();

   //服务端告知客户端 当前设备的名称 以及 当前屏幕的大小
   connection.send(Device.getDeviceName(), videoSize.getWidth(), videoSize.getHeight());
   return connection;
}

也即是客户端会先发送一个字节，用来确认是否已经连接，然后发送设备的名称，以及当前屏幕的大小,回到客户端的代码，前面已经分析了，此时客户端跟服务端已经连接上了
接着客户端解析服务端传递过来的内容

char device_name[DEVICE_NAME_FIELD_LENGTH];
struct size frame_size;

//当连接上的时候，服务端会告知客户端设备的名称，以及视频的大小
if (!device_read_info(device_socket, device_name, &frame_size)) {
   //读取失败，停止服务
   server_stop(&server);
   ret = SDL_FALSE;
   goto finally_destroy_server;
}

//读取设备的名称，以及视频的大小，
SDL_bool device_read_info(socket_t device_socket, char *device_name, struct size *size) {
    unsigned char buf[DEVICE_NAME_FIELD_LENGTH + 4];
    //socket接受内容
    int r = net_recv_all(device_socket, buf, sizeof(buf));

    if (r < DEVICE_NAME_FIELD_LENGTH + 4) {
        LOGE("Could not retrieve device information");
        return SDL_FALSE;
    }
    //填充device_name ，以及 size
    buf[DEVICE_NAME_FIELD_LENGTH - 1] = '\0'; // in case the client sends garbage
    // strcpy is safe here, since name contains at least DEVICE_NAME_FIELD_LENGTH bytes
    // and strlen(buf) < DEVICE_NAME_FIELD_LENGTH
    strcpy(device_name, (char *) buf);
    size->width = (buf[DEVICE_NAME_FIELD_LENGTH] << 8) | buf[DEVICE_NAME_FIELD_LENGTH + 1];
    size->height = (buf[DEVICE_NAME_FIELD_LENGTH + 2] << 8) | buf[DEVICE_NAME_FIELD_LENGTH + 3];
    return SDL_TRUE;
}

接着客户端执行  根据客户端告知的视频的大小，初始化 frames 结构体，创建对应frame
if (!frames_init(&frames)) {
   server_stop(&server);
   ret = SDL_FALSE;
   goto finally_destroy_server;
}

//初始化frames 的成员变量
SDL_bool frames_init(struct frames *frames) {

    //初始化解码帧
    if (!(frames->decoding_frame = av_frame_alloc())) {
        goto error_0;
    }

    //初始化渲染帧
    if (!(frames->rendering_frame = av_frame_alloc())) {
        goto error_1;
    }

    //初始化锁
    if (!(frames->mutex = SDL_CreateMutex())) {
        goto error_2;
    }
    ...
}

接着客户端 初始化解码的结构体
decoder_init(&decoder, &frames, device_socket, rec);

//初始化decoder ,注意这里的 video_socket 为客户端跟服务端通信的socket 对象 ,frames 为  执行 frames_init 初始化的对象
void decoder_init(struct decoder *decoder, struct frames *frames,
                  socket_t video_socket, struct recorder *recorder) {
    decoder->frames = frames;
    decoder->video_socket = video_socket;
    decoder->recorder = recorder;
}

接着客户端开启这个视频编解码线程  创建解码的线程,用来事实的将客户端的界面渲染出来
if (!decoder_start(&decoder)) {
   ret = SDL_FALSE;
   server_stop(&server);
   goto finally_destroy_recorder;
}

//开启解码的线程
SDL_bool decoder_start(struct decoder *decoder) {
    LOGD("Starting decoder thread");

    //sdl 创建一个线程,线程启动的时候会执行 run_decoder 方法，decoder为传递的参数
    decoder->thread = SDL_CreateThread(run_decoder, "video_decoder", decoder);
    if (!decoder->thread) {
        LOGC("Could not start decoder thread");
        return SDL_FALSE;
    }
    return SDL_TRUE;
}

所以关键的函数就在  run_decoder 函数指针里面 decoder 解码视频线程启动
static int run_decoder(void *data) {
    struct decoder *decoder = data;

    //找到 h264解码Code
    AVCodec *codec = avcodec_find_decoder(AV_CODEC_ID_H264);
    if (!codec) {
        LOGE("H.264 decoder not found");
        goto run_end;
    }

    //创建解码对应的Context对象
    AVCodecContext *codec_ctx = avcodec_alloc_context3(codec);
    if (!codec_ctx) {
        LOGC("Could not allocate decoder context");
        goto run_end;
    }

    //打开h264 解码器
    if (avcodec_open2(codec_ctx, codec, NULL) < 0) {
        LOGE("Could not open H.264 codec");
        goto run_finally_free_codec_ctx;
    }

    //初始化格式化的结构体成员
    AVFormatContext *format_ctx = avformat_alloc_context();
    if (!format_ctx) {
        LOGC("Could not allocate format context");
        goto run_finally_close_codec;
    }

    unsigned char *buffer = av_malloc(BUFSIZE);
    if (!buffer) {
        LOGC("Could not allocate buffer");
        goto run_finally_free_format_ctx;
    }

    // initialize the receiver state 初始化 接受者的状态
    decoder->receiver_state.frame_meta_queue = NULL;
    decoder->receiver_state.remaining = 0;

    // if recording is enabled, a "header" is sent between raw packets 如果当前允许录制，服务端 要将 头部的内容要提前发送，后面再发送视频的内容
    int (*read_packet)(void *, uint8_t *, int) =  decoder->recorder ? read_packet_with_meta : read_raw_packet;

    //read_packet 函数将赋值给read_packet，当调用avcodec_send_packet函数，将会从ReadInputData读取指定的的BUF_SIZE,也即是后面读取解析的内容，要从read_packet 中读取
    //decoder为 函数运行的时候，函数的参数
    AVIOContext *avio_ctx = avio_alloc_context(buffer, BUFSIZE, 0, decoder, read_packet, NULL, NULL);
    if (!avio_ctx) {
        LOGC("Could not allocate avio context");
        // avformat_open_input takes ownership of 'buffer'
        // so only free the buffer before avformat_open_input()
        av_free(buffer);
        goto run_finally_free_format_ctx;
    }

    //将avio_alloc_context产生的AVIOContext设置到AVFormatContext的pb变量。
    format_ctx->pb = avio_ctx;

    if (avformat_open_input(&format_ctx, NULL, NULL, NULL) < 0) {
        LOGE("Could not open video stream");
        goto run_finally_free_avio_ctx;
    }

    //如果有设置录制的化，打开录制
    if (decoder->recorder && !recorder_open(decoder->recorder, codec)) {
        LOGE("Could not open recorder");
        goto run_finally_close_input;
    }

    AVPacket packet;
    av_init_packet(&packet);
    packet.data = NULL;
    packet.size = 0;

    while (!av_read_frame(format_ctx, &packet)) {
// the new decoding/encoding API has been introduced by:
// <http://git.videolan.org/?p=ffmpeg.git;a=commitdiff;h=7fc329e2dd6226dfecaa4a1d7adf353bf2773726>
#if LIBAVCODEC_VERSION_INT >= AV_VERSION_INT(57, 37, 0)
        int ret;
        //发送给解码器
        if ((ret = avcodec_send_packet(codec_ctx, &packet)) < 0) {
            LOGE("Could not send video packet: %d", ret);
            goto run_quit;
        }
        //接受解码之后的内容
        ret = avcodec_receive_frame(codec_ctx, decoder->frames->decoding_frame);
        if (!ret) {
            // a frame was received 将解码之后的内容，推送给SDL 显示
            push_frame(decoder);
        } else if (ret != AVERROR(EAGAIN)) {
            LOGE("Could not receive video frame: %d", ret);
            av_packet_unref(&packet);
            goto run_quit;
        }

        if (decoder->recorder) {
            // we retrieve the PTS in order they were received, so they will
            // be assigned to the correct frame
            uint64_t pts = receiver_state_take_meta(&decoder->receiver_state);
            packet.pts = pts;
            packet.dts = pts;

            // no need to rescale with av_packet_rescale_ts(), the timestamps
            // are in microseconds both in input and output
            if (!recorder_write(decoder->recorder, &packet)) {
                LOGE("Could not write frame to output file");
                av_packet_unref(&packet);
                goto run_quit;
            }
        }

        av_packet_unref(&packet);

        if (avio_ctx->eof_reached) {
            break;
        }
    }
    LOGD("End of frames");
    ...
    return 0;
}

这里面就跟FFmpeg函数使用有关了，这里不介绍怎么使用， 这里主要介绍下他是怎么样从服务端获取到视频数据，主要是使用到了 avio_alloc_context 函数 可以指定数据源从哪里获取，这里指定
int (*read_packet)(void *, uint8_t *, int) =  decoder->recorder ? read_packet_with_meta : read_raw_packet; 如果不录制视频的化，默认为 read_raw_packet 

//如果没有指明录制视频，那么服务端就会直接发送视频的数据
static int read_raw_packet(void *opaque, uint8_t *buf, int buf_size) {
    struct decoder *decoder = opaque;
    ssize_t r = net_recv(decoder->video_socket, buf, buf_size);
    if (r == -1) {
        return AVERROR(errno);
    }
    if (r == 0) {
        return AVERROR_EOF;
    }
    return r;
}

最终调用 recv 函数完成数据的接收
ssize_t net_recv(socket_t socket, void *buf, size_t len) {
    return recv(socket, buf, len, 0);
}

现在看看服务端是怎么样发送视频数据的,服务端在确认了跟客户端连接之后,开启一个

//从客户端传递过来的参数中，初始化ScreenEncoder 对象
ScreenEncoder screenEncoder = new ScreenEncoder(options.getSendFrameMeta(), options.getBitRate());

// synchronous  主线程主要用来压缩屏幕的视频界面发送给客户端显示
screenEncoder.streamScreen(device, connection.getFd());

//获取到界面的视频内容，发送给客户端
public void streamScreen(Device device, FileDescriptor fd) throws IOException
{
   //根据制定的 bitRate, frameRate, iFrameInterval   创建MediaFormat对象
   MediaFormat format = createFormat(bitRate, frameRate, iFrameInterval);
   //设置屏幕的旋转监听
   device.setRotationListener(this);
   boolean alive;
   try
   {
      do
      {
          //创建Codec
          MediaCodec codec = createCodec();
          IBinder display = createDisplay();
          //获取到屏幕显示的范围
          Rect contentRect = device.getScreenInfo().getContentRect();
          //获取到视频显示的范围
          Rect videoRect = device.getScreenInfo().getVideoSize().toRect();
          //设置大小为视频的宽高
          setSize(format, videoRect.width(), videoRect.height());
          configure(codec, format);
          //Requests a Surface to use as the input to an encoder, in place of input buffers.  This
          //may only be called after {@link #configure} and before {@link #start}
          Surface surface = codec.createInputSurface();
          setDisplaySurface(display, surface, contentRect, videoRect);
          codec.start();
          try
          {
              //获取到屏幕的数据，并且执行发送
              alive = encode(codec, fd);
          }
          finally
          {
              //释放操作
              codec.stop();
              destroyDisplay(display);
              codec.release();
              surface.release();
          }
       }
       while (alive);//无限循环
   }
   finally
   {
      device.setRotationListener(null);
   }
}

//创建对应格式的MediaCodec
private static MediaCodec createCodec() throws IOException
{
    return MediaCodec.createEncoderByType("video/avc");
}

//根据bitRate  frameRate   iFrameInterval 创建对应的MediaFormat 对象
private static MediaFormat createFormat(int bitRate, int frameRate, int iFrameInterval) throws IOException
{
   MediaFormat format = new MediaFormat();
   format.setString(MediaFormat.KEY_MIME, "video/avc");
   //设置比特率
   format.setInteger(MediaFormat.KEY_BIT_RATE, bitRate);
   //设置帧率
   format.setInteger(MediaFormat.KEY_FRAME_RATE, frameRate);
   format.setInteger(MediaFormat.KEY_COLOR_FORMAT, MediaCodecInfo.CodecCapabilities.COLOR_FormatSurface);
   //设置帧率的延时时间
   format.setInteger(MediaFormat.KEY_I_FRAME_INTERVAL, iFrameInterval);
   // display the very first frame, and recover from bad quality when no new frames
   format.setLong(MediaFormat.KEY_REPEAT_PREVIOUS_FRAME_AFTER, MICROSECONDS_IN_ONE_SECOND * REPEAT_FRAME_DELAY / frameRate); // µs
   return format;
}

//编码视频数据
private boolean encode(MediaCodec codec, FileDescriptor fd) throws IOException
{
   boolean eof = false;
   MediaCodec.BufferInfo bufferInfo = new MediaCodec.BufferInfo();

   //如果屏幕的方向发生了变化，并且没有收到流的结尾，就一直循环获取
   while (!consumeRotationChange() && !eof)
   {
      //将输出缓冲区出列，最多阻塞“timeoutUs”微秒。返回已成功输出缓冲区的索引  解码或其中一个INFO_ *常量
      int outputBufferId = codec.dequeueOutputBuffer(bufferInfo, -1);
      //判断是否到了结尾
      eof = (bufferInfo.flags & MediaCodec.BUFFER_FLAG_END_OF_STREAM) != 0;
      try
      {
          //如果屏幕的方向发生了变化，必须重新的启动压缩编码
          if (consumeRotationChange())
          {
             // must restart encoding with new size
             break;
          }

          //已成功输出缓冲区的索引
          if (outputBufferId >= 0)
          {
             //获取到outputBufferId 中对应的buff
             ByteBuffer codecBuffer = codec.getOutputBuffer(outputBufferId);

             //是否需要发送元数据，如果是要录制视频的时候需要发送
             if (sendFrameMeta)
             {
                 writeFrameMeta(fd, bufferInfo, codecBuffer.remaining());
             }
             //发送视频数据
             IO.writeFully(fd, codecBuffer);
          }
       }
       finally
       {
           if (outputBufferId >= 0)
           {
              codec.releaseOutputBuffer(outputBufferId, false);
           }
        }
   }
   return !eof;
}

//发送视频数据 ,这里的 fd  为 socket的描述符,这样就将数据发送给了客户端
public static void writeFully(FileDescriptor fd, ByteBuffer from) throws IOException {
   int remaining = from.remaining();
   while (remaining > 0) {
     try {
          int w = Os.write(fd, from);
          if (BuildConfig.DEBUG && w < 0) {
             // w should not be negative, since an exception is thrown on error
             throw new AssertionError("Os.write() returned a negative value (" + w + ")");
          }
          remaining -= w;
        } catch (ErrnoException e) {
           if (e.errno != OsConstants.EINTR) {
               throw new IOException(e);
           }
       }
   }
}

回到客户端,服务端发送了视频数据之后，客户端相应的收到了数据，利用FFmpeg将视频解码成原始的数据，交给SDL显示,这个函数后面再来分析，先看看SDL的初始化
// a frame was received 将解码之后的内容，推送给SDL 显示
push_frame(decoder);


//客户端继续往下执行 初始化 controller 线程，这个线程的作用就是发送界面上的点击事件给服务端
if (!controller_start(&controller)) {
    ret = SDL_FALSE;
   goto finally_destroy_controller;
}

//启动一个control 线程，负责收集界面上的操作，发送给服务端
SDL_bool controller_start(struct controller *controller) {
    LOGD("Starting controller thread");

    //创建一个线程,线程启动的时候，会执行run_controller 函数，controller为函数执行的参数
    controller->thread = SDL_CreateThread(run_controller, "controller", controller);
    if (!controller->thread) {
        LOGC("Could not start controller thread");
        return SDL_FALSE;
    }

    return SDL_TRUE;
}

//control 线程，负责收集界面上的操作，同步给服务端 
static int run_controller(void *data) { 
    struct controller *controller = data;

    //保证线程不会退出，一直做收集操作
    for (;;) {
    	//加锁
        mutex_lock(controller->mutex);
        //如果当前controller线程没有停止，并且queue队列没有内容，那么该线程处于等待
        while (!controller->stopped && control_event_queue_is_empty(&controller->queue)) {
        	cond_wait(controller->event_cond, controller->mutex);
        }

        //如果处于停止状态，解锁，退出这个循环，线程执行结束
        if (controller->stopped) {
            // stop immediately, do not process further events
            mutex_unlock(controller->mutex);
            break;
        }

        //那如果到了这里就说明，不处于停止状态，并且queue中有内容，那么获取队列里的内容
        struct control_event event;
        //获取queue中的下一个内热
        SDL_bool non_empty = control_event_queue_take(&controller->queue, &event);
        //确保不为空，解锁
        SDL_assert(non_empty);
        mutex_unlock(controller->mutex);

        //处理这个事件，发送给服务端
        SDL_bool ok = process_event(controller, &event);
        control_event_destroy(&event);
        if (!ok) {
            LOGD("Cannot write event to socket");
            break;
        }
    }
    return 0;
}

从这个线程的run方法可知，他主要是监控 controller中的 queue 队列是否有内容，如果没有内容，使用 cond_wait 阻塞，如果有内容就获取到队列中的内容,最后使用process_event()函数将操作发送给服务端
这个函数先不分析，先假设当前没有事件，那么就会阻塞，客户端继续往下执行

//初始化SDL 显示窗口 device_name 为 客户端告知的设备号， fame_size 为当前设备
if (!screen_init_rendering(&screen, device_name, frame_size)) {
    ret = SDL_FALSE;
    goto finally_stop_and_join_controller;
}

//初始化显示的窗口，
SDL_bool screen_init_rendering(struct screen *screen, const char *device_name, struct size frame_size) {
    screen->frame_size = frame_size;

    //根据服务端传递的界面大小，跟当前的pc界面的大小，算出一个合适的大小
    struct size window_size = get_initial_optimal_size(frame_size);
    Uint32 window_flags = SDL_WINDOW_HIDDEN | SDL_WINDOW_RESIZABLE;
    //创建这个window,同时保存在window这个成员上 ,窗口的标题为 device_name
    screen->window = SDL_CreateWindow(device_name, SDL_WINDOWPOS_UNDEFINED, SDL_WINDOWPOS_UNDEFINED,
                                      window_size.width, window_size.height, window_flags);
    //窗口创建失败
    if (!screen->window) {
        LOGC("Could not create window: %s", SDL_GetError());
        return SDL_FALSE;
    }

    //创建纹理渲染器 SDL_Renderer对象
    screen->renderer = SDL_CreateRenderer(screen->window, -1, SDL_RENDERER_ACCELERATED);
    if (!screen->renderer) {
        LOGC("Could not create renderer: %s", SDL_GetError());
        screen_destroy(screen);
        return SDL_FALSE;
    }

    if (SDL_RenderSetLogicalSize(screen->renderer, frame_size.width, frame_size.height)) {
        LOGE("Could not set renderer logical size: %s", SDL_GetError());
        screen_destroy(screen);
        return SDL_FALSE;
    }

    //读取android 小图标
    SDL_Surface *icon = read_xpm(icon_xpm);
    if (!icon) {
        LOGE("Could not load icon: %s", SDL_GetError());
        screen_destroy(screen);
        return SDL_FALSE;
    }
    //设置界面图标
    SDL_SetWindowIcon(screen->window, icon);
    SDL_FreeSurface(icon);

    LOGI("Initial texture: %" PRIu16 "x%" PRIu16, frame_size.width, frame_size.height);
    //创建纹理对象 SDL_Texture
    screen->texture = create_texture(screen->renderer, frame_size);
    //创建纹理失败
    if (!screen->texture) {
        LOGC("Could not create texture: %s", SDL_GetError());
        screen_destroy(screen);
        return SDL_FALSE;
    }
    return SDL_TRUE;
}

这样SDL默认的界面就显示出来了，最终客户端调用   ret = event_loop();  来轮训的监听界面上的操作

//SDL 事件循环
static SDL_bool event_loop(void) {
    SDL_Event event;
    //在处理SDL的事件有两种模式，一种是等待 SDL_WaitEvent,另一种是轮询SDL_PollEvent 目前，在网上可以查到的文章，大多数都使用了轮询，还有人指出等待有时会导致事件处理延迟。
    //但是我在实际coding中，使用SDL_PollEvent，一运行就风扇不停的转，用top看了下，cpu占用到了99%。
    //由于SDL_WaitEvent语句，在用户未发生任何操作前，程序处于等待状态，不会进入下一次循环，这样就避免了cpu占用的问题。 也即是用户有操作的时候才会触发
    while (SDL_WaitEvent(&event)) {
        switch (event.type) {
            case EVENT_DECODER_STOPPED://视频渲染停止
                LOGD("Video decoder stopped");
                return SDL_FALSE;
            case SDL_QUIT://SDL 退出
                LOGD("User requested to quit");
                return SDL_TRUE;
            case EVENT_NEW_FRAME://当客户端收到了服务端发送的视频数据，并且解码之后，会发送这个事件，通知SDL可以渲染界面了
            	//如果当前屏幕还没有界面，也即是第一帧，显示当前的window
                if (!screen.has_frame) {
                    screen.has_frame = SDL_TRUE;
                    // this is the very first frame, show the window
                    screen_show_window(&screen);
                }
                //更新界面的内容
                if (!screen_update_frame(&screen, &frames)) {
                    return SDL_FALSE;
                }
                break;
            case SDL_WINDOWEVENT://Windwow事件，比如界面大小发生变化
                switch (event.window.event) {
                    case SDL_WINDOWEVENT_EXPOSED:
                    case SDL_WINDOWEVENT_SIZE_CHANGED:
                        screen_render(&screen);
                        break;
                }
                break;
            case SDL_TEXTINPUT://监听到了键盘的输入事件
                input_manager_process_text_input(&input_manager, &event.text);
                break;
            ...
        }
    }
    return SDL_FALSE;
}

前面分析过服务端发送视频数据，客户端收到之后，通过FFmpeg解码成原始的视频数据之后,会调用 push_frame(decoder); 推送给SDL，让SDL完成显示,现在就可以分析这个函数的实现了
//将视频界面之后的原始数据通知SDL，显示界面
static void push_frame(struct decoder *decoder) {
    SDL_bool previous_frame_consumed = frames_offer_decoded_frame(decoder->frames);
    if (!previous_frame_consumed) {
        // the previous EVENT_NEW_FRAME will consume this frame
        return;
    }
    //构建一个EVENT_NEW_FRAME 事件，利用SDL发送这个事件
    static SDL_Event new_frame_event = {
        .type = EVENT_NEW_FRAME,
    };
    SDL_PushEvent(&new_frame_event);
}

可以看到这里是通过构建一个EVENT_NEW_FRAME 事件，使用SDL_PushEvent 来通知SDL，那么SDL就能接收到这个事件，完成对应的渲染
case EVENT_NEW_FRAME://当客户端收到了服务端发送的视频数据，并且解码之后，会发送这个事件，通知SDL可以渲染界面了
   //如果当前屏幕还没有界面，也即是第一帧，显示当前的window
   if (!screen.has_frame) {
        screen.has_frame = SDL_TRUE;
        // this is the very first frame, show the window
        screen_show_window(&screen);
   }
   //更新界面的内容
   if (!screen_update_frame(&screen, &frames)) {
       return SDL_FALSE;
   }
break;

//渲染屏幕 根据解码之后的frame
SDL_bool screen_update_frame(struct screen *screen, struct frames *frames) {
    mutex_lock(frames->mutex);
    const AVFrame *frame = frames_consume_rendered_frame(frames);
    //新的大小
    struct size new_frame_size = {frame->width, frame->height};
    if (!prepare_for_frame(screen, new_frame_size)) {
        mutex_unlock(frames->mutex);
        return SDL_FALSE;
    }

    //填充数据
    update_texture(screen, frame);
    mutex_unlock(frames->mutex);

    //更新纹理，显示界面
    screen_render(screen);
    return SDL_TRUE;
}

//更新纹理，显示界面
void screen_render(struct screen *screen) {
    SDL_RenderClear(screen->renderer);
    SDL_RenderCopy(screen->renderer, screen->texture, NULL, NULL);
    SDL_RenderPresent(screen->renderer);
}

这样服务端发送的视频数据，客户端就能接受到了，并且通过SDL显示出来, 下面分析下客户端界面的点击操作，这里先分析下服务端开启controller线程监控事件的接收

// asynchronous 服务端开启一个控制同步线程，用于同步客户端发送的按键内容
startEventController(device, connection);


//服务端开启一个控制同步线程，用于同步客户端发送的按键内容
private static void startEventController(final Device device, final DesktopConnection connection) {
    new Thread(new Runnable() {
       @Override
       public void run() {
          try {
                 new EventController(device, connection).control();
              } catch (IOException e) {
                  // this is expected on close
                  Ln.d("Event controller stopped");
              }
         }
    }).start();
}

public EventController(Device device, DesktopConnection connection) {
    this.device = device;
    //俩者连接的对象
    this.connection = connection;
    initPointer();
}

//开始同步
public void control() throws IOException {
    // on start, turn screen on 开始的时候，打开屏幕
    turnScreenOn();

    //无限循环，同步客户端发送的按键操作
    while (true) {
        handleEvent();
    }
}

//打开屏幕
private boolean turnScreenOn() {
    //injectKeycode(KeyEvent.KEYCODE_POWER); 也即是模仿 按下电源键
    return device.isScreenOn() || injectKeycode(KeyEvent.KEYCODE_POWER);
}

//由于要模拟某个按键 keyCode，所以对应的就要有 按下以及抬起的动作
private boolean injectKeycode(int keyCode) {
    //模拟keyCode 对应的按下 , 模拟 keyCode 对应的抬起
    return injectKeyEvent(KeyEvent.ACTION_DOWN, keyCode, 0, 0) && injectKeyEvent(KeyEvent.ACTION_UP, keyCode, 0, 0);
}

//根据传递过来的内容，构建一个KeyEvent  ，deviceId 为VIRTUAL_KEYBOARD ,source 为 SOURCE_KEYBOARD
private boolean injectKeyEvent(int action, int keyCode, int repeat, int metaState) {
    long now = SystemClock.uptimeMillis();
    KeyEvent event = new KeyEvent(now, now, action, keyCode, repeat, metaState, KeyCharacterMap.VIRTUAL_KEYBOARD, 0, 0, InputDevice.SOURCE_KEYBOARD);
    //发射调用执行这个按键
    return injectEvent(event);
}

//发射调用按键
private boolean injectEvent(InputEvent event) {
    return device.injectInputEvent(event, InputManager.INJECT_INPUT_EVENT_MODE_ASYNC);
}

public boolean injectInputEvent(InputEvent inputEvent, int mode)
{
    return serviceManager.getInputManager().injectInputEvent(inputEvent, mode);
}
public InputManager(IInterface manager) {
    this.manager = manager;
    try {
        // 也即是获取到 InputManager 中的 injectInputEvent    public boolean injectInputEvent(InputEvent event, int mode)
        injectInputEventMethod = manager.getClass().getMethod("injectInputEvent", InputEvent.class, int.class);
    } catch (NoSuchMethodException e) {
        throw new AssertionError(e);
    }
}

//反射调用这个按键
public boolean injectInputEvent(InputEvent inputEvent, int mode) {
   try {
         return (Boolean) injectInputEventMethod.invoke(manager, inputEvent, mode);
     } catch (InvocationTargetException | IllegalAccessException e) {
         throw new AssertionError(e);
     }
}

现在看下客户端的收集按键的操作,对应的实现就是通过SDL 的event_looper实现的，通过SDL可以捕获到界面上的操作,比如

case SDL_TEXTINPUT://监听到了键盘的输入事件
     input_manager_process_text_input(&input_manager, &event.text);
break;


//监听到了键盘的输入事件，
void input_manager_process_text_input(struct input_manager *input_manager,const SDL_TextInputEvent *event) {

    char c = event->text[0];
    if (isalpha(c) || c == ' ') {
        SDL_assert(event->text[1] == '\0');
        // letters and space are handled as raw key event
        return;
    }
    //创建一个control_event 对象,填充对象的内容
    struct control_event control_event;
    //类型
    control_event.type = CONTROL_EVENT_TYPE_TEXT;
    //文本
    control_event.text_event.text = SDL_strdup(event->text);
    if (!control_event.text_event.text) {
        LOGW("Cannot strdup input text");
        return;
    }
    //将当前的control_event 对象，添加到 input_manager 中的 controller 成员的队列
    if (!controller_push_event(input_manager->controller, &control_event)) {
        SDL_free(control_event.text_event.text);
        LOGW("Cannot send text event");
    }
}

//将当前的control_event 事件， 添加到 controller 的queue队列中
SDL_bool controller_push_event(struct controller *controller, const struct control_event *event) {
    SDL_bool res;
    //加锁
    mutex_lock(controller->mutex);
    //判断是否为空
    SDL_bool was_empty = control_event_queue_is_empty(&controller->queue);
    //添加到队列中
    res = control_event_queue_push(&controller->queue, event);
    //如果当前为空，那么另一边control线程就会处于阻塞的状态，所以这边要发送一个信号，control线程才不会阻塞
    if (was_empty) {
        cond_signal(controller->event_cond);
    }
    //解锁
    mutex_unlock(controller->mutex);
    return res;
}

通过SDL捕获到事件之后，这里通过构建一个control_event 事件，然后添加到 control_event 中的queue队列中， 不知道前面还有没有印象，客户端启动了一个controll线程用来收集用户的行为操作，监控的
就是这个control_event中的queue 队列,也即是下面的代码实现

//control 线程，负责收集界面上的操作，同步给服务端
static int run_controller(void *data) {
    struct controller *controller = data;

    //保证线程不会退出，一直做收集操作
    for (;;) {
    	//加锁
        mutex_lock(controller->mutex);
        //如果当前controller线程没有停止，并且queue队列没有内容，那么该线程处于等待
        while (!controller->stopped && control_event_queue_is_empty(&controller->queue)) {
        	cond_wait(controller->event_cond, controller->mutex);
        }

        //如果处于停止状态，解锁，退出这个循环，线程执行结束
        if (controller->stopped) {
            // stop immediately, do not process further events
            mutex_unlock(controller->mutex);
            break;
        }

        //那如果到了这里就说明，不处于停止状态，并且queue中有内容，那么获取队列里的内容
        struct control_event event;
        //获取queue中的下一个内热
        SDL_bool non_empty = control_event_queue_take(&controller->queue, &event);
        //确保不为空，解锁
        SDL_assert(non_empty);
        mutex_unlock(controller->mutex);

        //处理这个事件，发送给服务端
        SDL_bool ok = process_event(controller, &event);
        control_event_destroy(&event);
        if (!ok) {
            LOGD("Cannot write event to socket");
            break;
        }
    }
    return 0;
}

由于队列中含有元素，这样就不会进行阻塞，进而执行process_event()函数

//处理control 事件，发送给服务端
static SDL_bool process_event(struct controller *controller, const struct control_event *event) {
	//#define TEXT_MAX_LENGTH 300 #define SERIALIZED_EVENT_MAX_SIZE 3 + TEXT_MAX_LENGTH 也即是长度为303
    unsigned char serialized_event[SERIALIZED_EVENT_MAX_SIZE];
    //将对应的事件，填充内容到serialized_event 中
    int length = control_event_serialize(event, serialized_event);
    if (!length) {
        return SDL_FALSE;
    }
    //发送事件
    int w = net_send_all(controller->video_socket, serialized_event, length);
    return w == length;
}

这样客户端就完成了收集界面上的操作，然后相应的发送给了服务端，服务端就可以接收到,通过判断事件的种类，然后客户端相应的通过反射来模拟用户的点击操作，从而达到操作同步
private void handleEvent() throws IOException {
   ControlEvent controlEvent = connection.receiveControlEvent();
   switch (controlEvent.getType()) {
       case ControlEvent.TYPE_KEYCODE:
            injectKeycode(controlEvent.getAction(), controlEvent.getKeycode(), controlEvent.getMetaState());
			break;
       case ControlEvent.TYPE_TEXT:
            injectText(controlEvent.getText());
            break;
       case ControlEvent.TYPE_MOUSE:
            injectMouse(controlEvent.getAction(), controlEvent.getButtons(), controlEvent.getPosition());
            break;
       case ControlEvent.TYPE_SCROLL:
            injectScroll(controlEvent.getPosition(), controlEvent.getHScroll(), controlEvent.getVScroll());
            break;
      case ControlEvent.TYPE_COMMAND:
           executeCommand(controlEvent.getAction());
           break;
      default:
           // do nothing
    }
}
```

### 总结
> 客户端通过adb命令将服务端的jar推送到手机，执行通过执行 adb reverse, adb forward 的结果来决定应该由哪端来监听tcp的端口,通过adb shell 启动服务端jar，并且发送对应的参数，比如是否应该服务端开启端口的监听，这样服务端的main函数启动起来， 接着服务端等待客户端的连接，连接通过之后发送设备的信息，比如手机的型号，视频的大小等，客户端接收到之后完成对应的SDL窗口的设置，紧接着客户端会创建一个video encoder 线程，专门用来接受服务端发送的视频数据，服务端会在主线程通过MediaCodec 完成视频的编码，客户端收到之后，通过FFmpeg 解码成原始的数据，通过SDL 将界面渲染出来同时客户端还会启动一个Controller  线程，负责收集界面上的操作，通过SDL 捕获到界面上的操作,然后 发送给服务端，服务端也会开启一个线程专门用于同步客户端发送的界面操作，然后通过反射的方式，模拟用户的操作

