---
layout: pager
title: JNI双进程守护
date: 2017-11-04 19:33:08
tags: [Android,JNI双进程守护]
description: JNI双进程守护
---

JNI双进程守护
<!--more-->

**JNI双进程守护**
===


**怎么样防止服务不被杀死**
===
```
1.提高优先级
这个办法对普通应用而言，
应该只是降低了应用被杀死的概率，但是如果真的被系统回收了，还是无法让应用自动重新启动！
2 让service.onStartCommand返回START_STICKY
START_STICKY是service被kill掉后自动重启
通过实验发现，如果在adb shell当中kill掉进程模拟应用被意外杀死的情况(或者用360手机卫士进行清理操作)，
如果服务的onStartCommand返回START_STICKY，
在进程管理器中会发现过一小会后被杀死的进程的确又会出现在任务管理器中，貌似这是一个可行的办法。
但是如果在系统设置的App管理中选择强行关闭应用，
这时候会发现即使onStartCommand返回了START_STICKY，应用还是没能重新启动起来！
3.android:persistent="true"
网上还提出了设置这个属性的办法，通过实验发现即使设置了这,应用程序被kill之后还是不能重新启动起来的！
4.让应用成为系统应用
实验发现即使成为系统应用，被杀死之后也不能自动重新启动。
但是如果对一个系统应用设置了persistent="true"，情况就不一样了
。实验表明对一个设置了persistent属性的系统应用，即使kill掉会立刻重启。
一个设置了persistent="true"的系统应用，
android中具有core service优先级，这种优先级的应用对系统的low memory killer是免疫的！

应用优先级
Android中的进程是托管的，当系统进程空间紧张的时候，会依照优先级自动进行进程的回收
Android将进程分为5个等级,它们按优先级顺序由高到低依次是：
  ● 空进程 Empty process
  ● 可见进程 Visible process
  ● 服务进程 Service process
  ● 后台进程 Background process
  ● 前台进程 Foreground process

```


```
我们知道通过在linux命令中，可以通过ps来得到当前运行的进程信息，所以我们可以通过监听fork一个子进程，fork有一个特点就是子进程的ppid为父进程的pid，
linux进程里面，1为init进程，当父进程被杀死掉之后，子进程的ppid就变成了1，所以我们可以通过轮询监听，子进程的ppid来判断是否为1
```

```java
java里面声明了一个servcice
public class ProcessService extends Service {

    private static final String TAG = "tuch";
    int i=0;
    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }

    @Override
    public void onCreate() {
        super.onCreate();
        Wathcer wathcer=new Wathcer();
        wathcer.createWatcher(Process.myUid());直接调用JNI native方法
        Timer timer = new Timer();
        //定时器
        timer.scheduleAtFixedRate(
                new TimerTask() {
                    public void run() {
                        Log.i(TAG, "服务  "+i);
                        i++;
                    }
                }, 0, 1000 * 3);
    }
}
JNI的调用
int user_id;
void create_child()
{
    //调用了fork之后，会创建一个子进程，fork代码之后的代码都会被执行俩次，一次是由父进程执行的
    //一次是由子进程执行的，我们要在子进程里面创建一个线程来轮询监听 子进程的ppid是否为1，如果为1，代表
    pid_t  pid = fork();
    if(pid < 0)
    {
        LOGE("创建进程失败\n");
    }
    else if(pid > 0)
    {
        LOGE("pid 大于0，代表的是父进程执行的");
    } else{
        LOGE("pid 小于0 代表的是子进程执行的");
        child_start_monitor();
    }
}

void * MythreadRunMethod(void * args)
{
    pid_t pid;
    while((pid = getpid()) != 1)
    {
        sleep(2);
        LOGE("循环 %d",pid);
    }

    LOGE("重启父进程");
    //利用am 命令重启服务
    execlp("am","am","startservice","--user",user_id,"com.example.administrator.my360unistall/com.example.administrator.my360unistall.ProcessService",(char *)NULL);
}


void child_start_monitor()
{
    pthread_t tid;
    //创建一个线程,第一个参数为指向线程标识符的指针
    //第二个参数用来设置线程的属性
    //第三个参数是线程运行函数的起始的地址
    //最后一个参数是函数运行的参数
    //int pthread_create(pthread_t *thread, pthread_attr_t const * attr void *(*start_routine)(void *), void * arg);
    int res = pthread_create(&tid,NULL,MythreadRunMethod,NULL);
    //函数在执行错误时的错误信息将作为返回值返回，并不修改系统全局变量errno，当然也无法使用perror()打印错误信息。
    if(res !=0 )
    {
        LOGE("cannot create thread:%s\n",strerror(res));
        return;
    }
}

JNIEXPORT void JNICALL Java_com_example_administrator_my360unistall_Wathcer_createWatcher
        (JNIEnv * env, jobject obj, jint userId)
{
    user_id = userId;
    create_child();
}
```

运行结果为：
![结果显示](/uploads/前台服务.png)
调用cmd，进入到adb shell命令中，调用ps 得到进程信息为
![结果显示](/uploads/360轮询监听pid实现.png)
此时执行ps -9 kill 6458 注意6458为父进程的pid，也为子进程的ppid也就是父进程的pid
前台服务再次打印信息
![结果显示](/uploads/前台进程开始打印.png)
再次调用ps查看，发现我们的父进程的pid已经发生了变化，而子进程的pid也已经发生了变化，对于此前的那个子进程ppid变为1之后被回收了
![结果显示](/uploads/监听pid实现双进程.png)

```
虽然这种方法可以实现，守护进程的方式，但是这个是通过在子进程里面轮询监听子进程的ppid是否为1，来达到的，比较浪费资源
下面采用另一种方式来实现 通过socket的方式来实现
```

```java
java里面声明了一个servcice
public class ProcessService extends Service {

    private static final String TAG = "tuch";
    int i=0;
    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }

    @Override
    public void onCreate() {
        super.onCreate();
        Wathcer wathcer=new Wathcer();
        wathcer.createWatcher(String.valueOf(Process.myUid()));
        wathcer.connectToMonitor();
        Timer timer = new Timer();
        //定时器
        timer.scheduleAtFixedRate(
                new TimerTask() {
                    public void run() {
                        Log.i(TAG, "服务  "+i);
                        i++;
                    }
                }, 0, 1000 * 3);
    }
}
public class Wathcer {
    // Used to load the 'native-lib' library on application startup.
    static {
        System.loadLibrary("native-lib");
    }

    public native void createWatcher(String userId);
    public native void connectToMonitor();
}
JNI代码的实现
int m_child;

const  char * user_id;
const char *PATH = "/data/data/com.dongnao.socketprocess/my.sock";
extern "C"{
JNIEXPORT void JNICALL
Java_com_dongnao_socketprocess_Wathcer_createWatcher(JNIEnv *env, jobject instance, jstring userId) {
    //
    user_id = (const char *) env->GetStringChars(userId, 0);
    create_child();
}

JNIEXPORT void JNICALL
Java_com_dongnao_socketprocess_Wathcer_connectToMonitor(JNIEnv *env, jobject instance) {
//    子进程1    父进程2
    int sockfd;
    struct sockaddr_un  addr;
    while (1) {
        LOGE("客户端  父进程开始连接");
        //调用socket之后，会得到 socket的套接字，之后可以通过这个套接字直接来发送内容
        sockfd=socket(AF_LOCAL, SOCK_STREAM, 0);
        //sockfd 小于0，代表创建socket失败
        if (sockfd < 0)
        {
            return;
        }
        memset(&addr, 0, sizeof(sockaddr_un));
        addr.sun_family = AF_LOCAL;
        //设置用来监听的path
        strcpy(addr.sun_path, PATH);
        //connect函数的第一个参数即为客户端的socket描述字，第二参数为服务器的socket地址，
        //第三个参数为socket地址的长度。客户端通过调用connect函数来建立与TCP服务器的连接
        //返回值如果小于0代表连接失败
        if (connect(sockfd, (const sockaddr *) &addr, sizeof(addr)) < 0) {
            LOGE("连接失败  休眠");
//            连接失败
            close(sockfd);
            sleep(1);
//            再来继续下一次尝试
            continue;
        }
        LOGE("连接成功  父进程跳出循环");
        break;
    }

}

}

//创建守护进程
void create_child() {
//
    pid_t pid = fork();
    if (pid < 0) {
        LOGE("fork 进程失败 ");
        return;
    } else if (pid > 0) {
        LOGE("父进程 ");
    } else if (pid == 0){
        LOGE("子进程开启 ");
        child_do_work();
    }
}

void child_do_work() {
//    守护进程
//   1 建立socket服务
//    2读取消息
    if(child_create_channel()) {
        child_listen_msg();
    }

}

//监听消息
void child_listen_msg() {
    fd_set rfds;
    while (1) {
        //清空端口号
        FD_ZERO(&rfds);
        //添加要监听的socket套接字
        FD_SET(m_child,&rfds);
//        设置超时时间
        struct timeval timeout={3,0};
        int r=select(m_child + 1, &rfds, NULL, NULL, &timeout);
        LOGE("读取消息前  %d  ",r);
        if (r > 0) {
            char pkg[256] = {0};
//            确保读到的内容是制定的端口号
            if (FD_ISSET(m_child, &rfds)) {
//                阻塞式函数  客户端写到内容  apk进程  没有进行任何写入    连接
                int result = read(m_child, pkg, sizeof(pkg));
//                读到内容的唯一方式 是客户端断开
                LOGE("重启父进程  %d ",result);
                LOGE("读到信息  %d    userid  %d ",result,user_id);

                //执行am命令，重启服务
                execlp("am", "am", "startservice", "--user",user_id,
                       "com.dongnao.socketprocess/com.dongnao.socketprocess.ProcessService", (char*)NULL);
                break;
            }
        }



    }
}

int child_create_channel() {
//    创建socket  listenfd 对象 ,返回一个socket 套接字，唯一的
    int listenfd=socket(AF_LOCAL, SOCK_STREAM, 0);
    if(listenfd < 0)
    {
        LOGE("守护进程 创建scoket 失败");
        return 1;
    }
//    取消之前进程文件连接
    unlink(PATH);
    struct sockaddr_un  addr;
//    清空内存
    memset(&addr, 0, sizeof(sockaddr_un));
    addr.sun_family = AF_LOCAL;
    //给addr中的sun_path赋值
    strcpy(addr.sun_path, PATH);
    int connfd=0;
    LOGE("绑定端口号");
    //sockfd：即socket描述字，它是通过socket()函数创建了，唯一标识一个socket。bind()函数就是将给这个描述字绑定一个名字。
    //addr：一个const struct sockaddr *指针，指向要绑定给sockfd的协议地址。这个地址结构根据地址创建socket时的地址协议族的不同而不同，如ipv4对应的是
    //第三个为第二个参数的长度
    if(bind(listenfd, (const sockaddr *) &addr, sizeof(addr))<0)
    {
        LOGE("绑定错误");
        return 0;
    }
    //listen函数的第一个参数即为要监听的socket描述字，第二个参数为相应socket可以排队的最大连接个数。
    //socket()函数创建的socket默认是一个主动类型的，listen函数将socket变为被动类型的，等待客户的连接请求。
    listen(listenfd, 5);
    while (1) {
        LOGE("子进程循环等待连接  %d ",m_child);
            //不断接受客户端请求的数据
            //等待 客户端连接  accept阻塞式函数
        //accept函数的第一个参数为服务器的socket描述字，第二个参数为指向struct sockaddr *的指针，用于返回客户端的协议地址，
        // 第三个参数为协议地址的长度。如果accpet成功，那么其返回值是由内核自动生成的一个全新的描述字，代表与返回客户的TCP连接。
        if ((connfd = accept(listenfd, NULL, NULL)) < 0) {
            if (errno == EINTR) {
                continue;
            } else{
                LOGE("读取错误");
                return 0;
            }
        }
        //apk 进程连接上了
        m_child = connfd;
        LOGE("apk 父进程连接上了  %d ",m_child);
        break;
    }
    LOGE("返回成功");
    return 1;
}

```

运行结果为：
![结果显示](/uploads/socket前台进程打印.png)
调用cmd，进入到adb shell命令中，调用ps 得到进程信息为
![结果显示](/uploads/socket执行ps进程信息.png)
此时执行ps -9 kill 23049 注意23049为父进程的pid，也为子进程的ppid也就是父进程的pid
前台服务再次打印信息
![结果显示](/uploads/socket前台进程再次打印.png)
再次调用ps查看，发现我们的父进程的pid已经发生了变化，而子进程的pid也已经发生了变化，对于此前的那个子进程ppid变为1之后被回收了
![结果显示](/uploads/socket再次执行ps.png)
