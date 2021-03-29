---
layout: pager
title: 'Aria2 添加 natpmp,upnp 协议支持'
date: 2018-11-16 11:31:11
tags: [NDK,Aria2,natpmp,upnp]
description: Aria2 添加 natpmp,upnp 协议支持
---
什么是UPnp
> 所谓 UPnP ，就是“通用的即插即用” ，注意是通用的，虽然很容易和 Windows 的即插即用混淆，但这肯定不是微软的专利！现在大部分的路由器都支持这个功能，只是默认情况下没有打开而已(基于安全考虑)。请管理员手动打开这个支持选项。

可以用来干嘛?
> 如果我们要写 P2P 软件，那就用的着了，电骡不是有所谓的 LowID 和 highID 吗？ 为了提高自己的共享能力(我为别人共享，别人也为我共享)，我们(软件)要使用公网 IP 地址监听和建立连接，但是我们(软件)不是路由器，如何监听？ 只好请路由器帮我们做一个端口映射，然后我们(软件)在内网监听，效果跟在公网上监听一样，也就是所谓 电骡的 HighID 了。 现在越来越多的用户都是内网用户的上网形式(NAT)，如网吧。能够把自己的 LowID 提升为 HighID ，那么肯定会有更多的备选数据源啊，这样下载就被加速了!

natp,upnp,nat-pmp区别
> natp是内部机器通过路由器也就是网关向外部发送网络请求时，路由器记住内部机器的ip和端口，同时跟真正发送数据的外网端口绑定，产生一个临时映射表，当收到外网数据以后通过这个映射表将数据转发给内部机器。nat的多种映射类型之前说过而 upnp和nat-pmp差不多，就是在路由器和内部机器提供一个中间服务，内部机器请求upnp将其使用到的端口跟某个外网端口绑定，这样当路由器收到外网请求时先去upnp里查找是否此外网端口已经被upnp映射，如果被映射则将数据转发到内部机器对应的端口。

总结
> 动态NAPT的映射关系是内网数据包来触发的，如果外网有主动进来的数据包，因为查询不到映射关系的存在，就会被丢弃掉。 而 natpmp,upnp 就可以做到 让外网能够直接访问内网,当然这个要你的路由器支持才能添加这个映射的关系，但是目前的情况来看，估计很多路由器都是有支持的


### Utorrent Upnp,pmp支持
Utorrent 中就有对Upnp,pmp协议做了支持
![结果显示](/uploads/upnp协议支持/utorrent upnp协议支持.png)

可以看出这里我们指定的端口为60924 ,之后我们可以登录路由器查看是否存在这样的隐射关系 ,具体在系统工具，端口转发这一栏
![结果显示](/uploads/upnp协议支持/utorrent端口隐射成功.png)


### Upnp 使用
Upnp下载地址为
> http://miniupnp.free.fr/files/

这里选择一个稳定的版本
![结果显示](/uploads/upnp协议支持/upnp下载.jpg)
下载完之后，直接进入源码目录，由于本身是含有MakeFile文件的，直接使用make install 既可以编译出来
![结果显示](/uploads/upnp协议支持/upnp编译结果.png)
upnp具体的使用
![结果显示](/uploads/upnp协议支持/upnp具体的使用.png)
首先填的是你的本地的ip地址，后面是你要映射的本地的端口，后面是要映射的公网的端口，最后一个是协议，这里可以选择TCP，或者UDP
![结果显示](/uploads/upnp协议支持/upnp隐射成功.png)
查看路由器
![结果显示](/uploads/upnp协议支持/upnp编译隐射成功.png)
我们也可以使用tcpdump监听我们的本地端口，之后再通过访问我们的外网ip地址跟映射的外网端口号是否有反应 ,注意这里监听是我们的本地的端口
![结果显示](/uploads/upnp协议支持/upnp验证是否隐射成功.jpg)
我们可以通过在浏览器中输入 218.66.157.234:33333  ,可以看到监听到了内容
![结果显示](/uploads/upnp协议支持/tcpDump监听upnp成功.png)

### natPmp 使用
nat-pmp 下载地址为
> http://miniupnp.free.fr/files/

natPmp下载地址
![结果显示](/uploads/upnp协议支持/natPmp下载地址.png)
下载完之后，直接进入源码目录，由于本身是含有MakeFile文件的，直接使用make install 既可以编译出来
![结果显示](/uploads/upnp协议支持/pmp编译成功.jpg)
pmp 编译选项支持
![结果显示](/uploads/upnp协议支持/pmp使用.png)
首先你要映射的本地的端口，后面是要映射的公网的端口，最后一个是协议，这里可以选择TCP，或者UDP
![结果显示](/uploads/upnp协议支持/natPmp执行成功.png)
查看路由器
![结果显示](/uploads/upnp协议支持/natPmp查看路由结果.png)
当然我们也可以使用tcpdump监听我们的本地端口，之后再通过访问我们的外网ip地址跟映射的外网端口号是否有反应,注意这里监听是我们的本地的端口
![结果显示](/uploads/upnp协议支持/pmp监听本地端口.jpg)
我们可以通过在浏览器中输入 218.66.157.234:1130 ,可以看到监听到了内容
![结果显示](/uploads/upnp协议支持/pmp监听端口结果.jpg)

### transmission upnp,pmp分析
首先transmission 是有对upnp,pmp协议支持的，所以我们可以分析他的使用，首先要打开端口隐射支持
![结果显示](/uploads/upnp协议支持/transmission支持upnp选项配置.png)

查看路由器，映射的结果为,可以看到这里既有tcp，也有udp 
![结果显示](/uploads/upnp协议支持/transmission端口隐射的结果.png)

参数选项解析
![结果显示](/uploads/upnp协议支持/参数选项解析.png)

tr_sessionSetPortForwardingEnabled函数实现
![结果显示](/uploads/upnp协议支持/SetPortForwardEnabled函数实现.png)

```cpp
这里的shared是在前面初始化session的时候就执行了初始化，下面是调用的过程

//构建一个tr_shared 对象，赋值默认的成员变量,赋值给session中的shared,这个shared成员保存了upnp,pmp对应的对象
session->shared = tr_sharedInit(session);

/***
 **** 分配 tr_shared 内存空间，赋初值
 ***/
tr_shared *
tr_sharedInit(tr_session * session) {
	//分配一块内存空间tr_shared
	tr_shared * s = tr_new0(tr_shared, 1);
	//赋初值
	s->session = session;
	//默认是不允许的
	s->isEnabled = false;
	//开始的状态为 TR_PORT_UNMAPPED 为没有 mapped
	s->upnpStatus = TR_PORT_UNMAPPED;
	s->natpmpStatus = TR_PORT_UNMAPPED;
	..
	return s;
}

而tr_shared 结构体定义为

//保存了端口隐射的状态跟对象
struct tr_shared {
	//是否可以用
	bool isEnabled;
	//标识是否正在关闭
	bool isShuttingDown;
	//标识是否要做端口检查
	bool doPortCheck;

	// 对应的 pmp 协议状态
	tr_port_forwarding natpmpStatus;
	// 对应的 upnc 协议状态
	tr_port_forwarding upnpStatus;

	//对应的 upnp的协议的 对象指针
	tr_upnp * upnp;
	//对应的 pmp 协议的 对象指针
	tr_natpmp * natpmp;
	//session引用
	tr_session * session;
	//定时器对象
	struct event * timer;
};

继续回到这个函数
//设置port Forward 是否允许 设置是否允许内网端口隐射
void tr_sessionSetPortForwardingEnabled(tr_session * session, bool enabled) {
	struct port_forwarding_data * d;
	//构建一个port_forwarding_data 对象 分配内存空间
	d = tr_new0(struct port_forwarding_data, 1);
	//完成赋值操作 ，
	d->shared = session->shared;
	//默认初始的时候为false
	d->enabled = enabled;
	//切换到libEvent线程执行 回调函数setPortForwardingEnabled,这里的d为传递的参数
	tr_runInEventThread(session, setPortForwardingEnabled, d);
}

tr_runInEventThread 这个会执行现成的切换，当切换成功之后会执行setPortForwardingEnabled 函数，d为传递的参数

//函数是在libEvent中执行, vdata的类型为 struct port_forwarding_data
static void setPortForwardingEnabled(void * vdata) {
	struct port_forwarding_data * data = vdata;
	//将data enable值赋值给 shared 的 enable ,如果是为true的话，就创建一个定时器，赋值给shared 中的time成员
	tr_sharedTraversalEnable(data->shared, data->enabled);
	//释放data内存
	tr_free(data);
}

//如果当前isEnable为真，赋值给shared 中的isEnabled成员，而且创建一个定时器，赋值
void tr_sharedTraversalEnable(tr_shared * s, bool isEnabled) {
    //如果当前isEnabled为真，完成赋值操作赋值给  isEnabled ，并且开启定时器
    if ((s->isEnabled = isEnabled))
    {
        //开启定时器
        start_timer(s);
    }
	else
    {
        //如果不可以用停止 端口隐射
        stop_forwarding(s);
    }
}

//开启定时器
static void start_timer(tr_shared * s) {
    //构建一个定时器，同时赋值给timer成员,当定时器触发的时候，会回调执行 onTimer 函数
    s->timer = evtimer_new(s->session->event_base, onTimer, s);
	
    //根据tr_shared 状态来设置定时器的时间间隔
    set_evtimer_from_status(s);
}

//端口隐射定时器触发函数 ,也即是这个函数会被多次的触发，就看这个定时器设置的时间,而关键点就是在 natPulse
static void onTimer(evutil_socket_t fd UNUSED, short what UNUSED,void * vshared) {
    tr_shared * s = vshared;

    //参数校验
    assert(s);
    assert(s->timer);

    /* do something */
    natPulse(s, s->doPortCheck);

    //将doPortCheck置位false
    s->doPortCheck = false;

    /* set up the timer for the next pulse */
    //根据不同的状态决定定时器触发的时间
    set_evtimer_from_status(s);
}

static void natPulse(tr_shared * s, bool do_check) {
    int oldStatus;
    int newStatus;
    tr_port public_peer_port;
    //获取到局域网的端口，也即是Bt指定的监听端口
    const tr_port private_peer_port = s->session->private_peer_port;
    //判断是否允许,当前是要允许，并且没有正在关闭
    const int is_enabled = s->isEnabled && !s->isShuttingDown;

    //如果为空，创建对应的pmp对象
    if (s->natpmp == NULL)
    {
       s->natpmp = tr_natpmpInit();
    }

    //如果为空，创建对应的upnp 对象
    if (s->upnp == NULL)
    {
       s->upnp = tr_upnpInit();
    }

    //获取到老的状态
    oldStatus = tr_sharedTraversalStatus(s);

    //pmp 完成内网的端口隐射， 隐射完成之后， public_peer_port 接受公网的端口
    s->natpmpStatus = tr_natpmpPulse(s->natpmp, private_peer_port, is_enabled,&public_peer_port);

    //如果pmp 隐射成功
    if (s->natpmpStatus == TR_PORT_MAPPED)
    {
        //session保存pmp 隐射完成之后外网的端口号
        s->session->public_peer_port = public_peer_port;
    }
    //执行 upnp的隐射
    s->upnpStatus = tr_upnpPulse(s->upnp, private_peer_port, is_enabled,	do_check);
    //获取到最新的状态
    newStatus = tr_sharedTraversalStatus(s);

    //发生状态的变化，打印
    if (newStatus != oldStatus)
    {
       printf("Port Forwarding State changed \n");
    }
}

第一次 upnp为空，执行 tr_natpmpInit();

//pmp 协议初始化
struct tr_natpmp*
tr_natpmpInit (void)
{
    //分配内存空间
    struct tr_natpmp * nat;
    nat = tr_new0 (struct tr_natpmp, 1);
    //默认的状态为 pmp 发现状态
    nat->state = TR_NATPMP_DISCOVER;
    nat->public_port = 0;
    nat->private_port = 0;
    //对应的socket为空
    nat->natpmp.s = TR_BAD_SOCKET; /* socket */
    return nat;
}

而tr_natpmp 结构体定义为
struct tr_natpmp
{
    //标识是否已经发现
    bool              has_discovered;
    //标识是否已经mapped
    bool              is_mapped;
    //隐射完成之后 得到的外网端口号
    tr_port           public_port;
    //隐射的内网端口号
    tr_port           private_port;
    time_t            renew_time;
    //下一个command 发送的时间
    time_t            command_time;
    //pmp 当前的状态
    tr_natpmp_state   state;
    //真正的pmp 库对象
    natpmp_t          natpmp;
};

所以初始化，只是分配块内存空间，变为默认值

tr_upnpInit();

/**
 *** 初始化操作
 **/
tr_upnp*
tr_upnpInit(void) {
	tr_upnp * ret = tr_new0(tr_upnp, 1);

	ret->state = TR_UPNP_DISCOVER;
	ret->port = -1;
	return ret;
}
struct tr_upnp {
	//标识是否发现
	bool hasDiscovered;
	struct UPNPUrls urls;
	struct IGDdatas data;
	int port;
	char lanaddr[16];
	//标识是否隐射完成
	unsigned int isMapped;
	//upnp的状态
	tr_upnp_state state;
};
所以初始化，只是分配块内存空间，变为默认值

而 这个才是关键 tr_natpmpPulse(s->natpmp, private_peer_port, is_enabled,&public_peer_port);
//pmp 完成隐射
int tr_natpmpPulse (struct tr_natpmp * nat, tr_port private_port, bool is_enabled, tr_port * public_port)
{
    int ret;

    //如果is_enabled 为true，当前的状态处于TR_NATPMP_DISCOVER
    if (is_enabled && (nat->state == TR_NATPMP_DISCOVER))
    {
    	//首先执行pmp初始化操作
        int val = initnatpmp (&nat->natpmp, 0, 0);
        printf("initnatpmp %d \n",val);
        //logVal ("initnatpmp", val);
        val = sendpublicaddressrequest (&nat->natpmp);
        //logVal ("sendpublicaddressrequest", val);
        printf("sendpublicaddressrequest %d \n", val);
        nat->state = val < 0 ? TR_NATPMP_ERR : TR_NATPMP_RECV_PUB;
        //标识发现完成
        nat->has_discovered = true;
        //赋值 command_time ，也即是下一个command发送的时间
        setCommandTime (nat);
    }
	//如果上面一个步骤没有出现错误 默认状态为 TR_NATPMP_RECV_PUB ，再判断当前的时间是否到了可以发送下一个command的时间
    if ((nat->state == TR_NATPMP_RECV_PUB) && canSendCommand (nat))
    {
        natpmpresp_t response;
        //调用pmp函数，得到结果
        const int val = readnatpmpresponseorretry (&nat->natpmp, &response);
        //logVal ("readnatpmpresponseorretry", val);
        //printf ("readnatpmpresponseorretry %d \n", val);
        //如果结果大于0，打印的得到的外网的ip
        if (val >= 0)
        {
            char str[128];
            evutil_inet_ntop (AF_INET, &response.pnu.publicaddress.addr, str, sizeof (str));
            //tr_logAddNamedInfo (getKey (), _("Found public address \"%s\""), str);
            printf ("Port Forwarding (NAT-PMP)  Found public address %s \n", str);
            //同时状态为 TR_NATPMP_IDLE
            nat->state = TR_NATPMP_IDLE;
        }
        else if (val != NATPMP_TRYAGAIN)
        {
            //出现了错误
            nat->state = TR_NATPMP_ERR;
        }
    }

    //判断是否需要删除之前的隐射
    if ((nat->state == TR_NATPMP_IDLE) || (nat->state == TR_NATPMP_ERR))
    {
    	//is_mapped 代表之前已经隐射成功过，如果是这样，就要先删除之前的隐射
        if (nat->is_mapped && (!is_enabled || (nat->private_port != private_port)))
        {
            nat->state = TR_NATPMP_SEND_UNMAP;
        }
    }

    //判断是否需要删除之前的隐射 并且当前的时间达到了下一个command执行的时间
    if ((nat->state == TR_NATPMP_SEND_UNMAP) && canSendCommand (nat))
    {
    	//调用pmp库 发起端口隐射
        const int val = sendnewportmappingrequest (&nat->natpmp, NATPMP_PROTOCOL_TCP,
                                                   nat->private_port,
                                                   nat->public_port,
                                                   0);
        //logVal ("sendnewportmappingrequest", val);
        printf("sendnewportmappingrequest %d \n", val);
        //赋值状态
        nat->state = val < 0 ? TR_NATPMP_ERR : TR_NATPMP_RECV_UNMAP;
        //标识下一个command 执行的时间
        setCommandTime (nat);
    }
    ...
}

tr_upnpPulse(s->upnp, private_peer_port, is_enabled,	do_check);
//doPortCheck 代表是否需要隐射，如果为false代表不需要隐射，执行验证
int tr_upnpPulse(tr_upnp * handle, int port, int isEnabled, int doPortCheck) {
	int ret;

	//默认的状态 为 TR_UPNP_DISCOVER
	if (isEnabled && (handle->state == TR_UPNP_DISCOVER)) {
		struct UPNPDev * devlist;

		//执行upnp的发现,主要用来搜索局域网中所有的UPNP设备
		devlist = tr_upnpDiscover(2000);

		errno = 0;
		//获取有效的IGD
		if (UPNP_GetValidIGD(devlist, &handle->urls, &handle->data,handle->lanaddr, sizeof(handle->lanaddr)) == UPNP_IGD_VALID_CONNECTED) {
			//tr_logAddNamedInfo(getKey(),	_( "Found Internet Gateway Device \"%s\""),handle->urls.controlURL);
			printf("Port Forwarding (UPnP) Found Internet Gateway Device %s \n",handle->urls.controlURL);
			//tr_logAddNamedInfo(getKey(), _( "Local Address is \"%s\""),handle->lanaddr);
			printf("Port Forwarding (UPnP) Local Address is %s \n",handle->lanaddr);
			//赋值状态为TR_UPNP_IDLE
			handle->state = TR_UPNP_IDLE;
			//当前已经执行了发现
			handle->hasDiscovered = true;
		} else {
			//出现了错误
			handle->state = TR_UPNP_ERR;
			//获取无效的IGD,也即是你的路由器不支持
			//tr_logAddNamedDbg(getKey(),	"UPNP_GetValidIGD failed (errno %d - %s)", errno,	tr_strerror (errno));
			printf("Port Forwarding (UPnP) UPNP_GetValidIGD failed (errno %d - %s) \n", errno,	tr_strerror (errno));
			//tr_logAddNamedDbg(getKey(),"If your router supports UPnP, please make sure UPnP is enabled!");
			printf("Port Forwarding (UPnP) If your router supports UPnP, please make sure UPnP is enabled! \n");
		}
		freeUPNPDevlist(devlist);
	}

	//判断当前是否需要关掉之前隐射的关系
	if (handle->state == TR_UPNP_IDLE) {
		if (handle->isMapped && (!isEnabled || (handle->port != port)))
		{
			handle->state = TR_UPNP_UNMAP;
		}
	}
	//如果当前已经完成了隐射，判断是否需要检查端口
	if (isEnabled && handle->isMapped && doPortCheck) {
		//upnp 端口隐射，这里有tcp，udp,检查tcp 或者 udp
		if ((tr_upnpGetSpecificPortMappingEntry(handle, "TCP") != UPNPCOMMAND_SUCCESS) || (tr_upnpGetSpecificPortMappingEntry(handle, "UDP") != UPNPCOMMAND_SUCCESS)) {
			//出现了错误，标识没有隐射完成
			//tr_logAddNamedInfo(getKey(), _("Port %d isn't forwarded"),	handle->port);
			printf("Port Forwarding (UPnP) Port %d isn't forwarded \n",handle->port);
			//如果tcp，udp检查都有问题，标识没有隐射成功
			handle->isMapped = false;
		}
	}

	//删除隐射
	if (handle->state == TR_UPNP_UNMAP) {
		//删除tcp ，udp隐射
		tr_upnpDeletePortMapping(handle, "TCP", handle->port);
		tr_upnpDeletePortMapping(handle, "UDP", handle->port);

		//tr_logAddNamedInfo(getKey(),_("Stopping port forwarding through \"%s\", service \"%s\""),handle->urls.controlURL, handle->data.first.servicetype);
		printf("Port Forwarding (UPnP) Stopping port forwarding through %s service %s \n",handle->urls.controlURL, handle->data.first.servicetype);
		//删除隐射，通知状态，比如port为-1
		handle->isMapped = 0;
		handle->state = TR_UPNP_IDLE;
		handle->port = -1;
	}
	...
}

这俩个函数涉及到的内容比较多，具体去看源码的实现,下面看set_evtimer_from_status ，这个函数会设计到定时器的时间设置

//根据状态来决定定时器的时间间隔
static void set_evtimer_from_status(tr_shared * s) {
	int sec = 0, msec = 0;

	/* when to wake up again */
	switch (tr_sharedTraversalStatus(s)) {
	case TR_PORT_MAPPED:
		//如果当前我们已经隐射成功了，那么20分钟之后再来检查
		/* if we're mapped, everything is fine... check back in 20 minutes
		 * to renew the port forwarding if it's expired */
		//将doPortCheck置位true
		s->doPortCheck = true;
		sec = 60 * 20;
		break;

	case TR_PORT_ERROR:
		//如果出现了错误，那么60秒之后再来
		/* some kind of an error. wait 60 seconds and retry */
		sec = 60;
		break;

	default:
		/* in progress. pulse frequently. */
		msec = 333000;
		break;
	}

	//添加定时器
	if (s->timer != NULL)
	{
		tr_timerAdd(s->timer, sec, msec);
	}
}
```

### Aria2 移植upnp,pmp
```cpp
首先我们需要将这俩个库引进来，对应的CmakeList中添加要编译的内容

#添加头文件的查找目录
include_directories(${CMAKE_SOURCE_DIR}/src/main/cpp/src/includes
                    ${CMAKE_SOURCE_DIR}/src/main/cpp/src/includes/libutp
                    ${CMAKE_SOURCE_DIR}/src/main/cpp/src/includes/libnatpmp
                    ${CMAKE_SOURCE_DIR}/src/main/cpp/src/includes/miniupnp
                    ${CMAKE_SOURCE_DIR}/src/main/cpp/third/includes )

...
set(libnatpmp
        ${My_ARIC_SRC}/libnatpmp/getgateway.c
        ${My_ARIC_SRC}/libnatpmp/natpmp.c
        ${My_ARIC_SRC}/libnatpmp/wingettimeofday.c )

set(libupnp
        ${My_ARIC_SRC}/miniupnp/connecthostport.c
        ${My_ARIC_SRC}/miniupnp/igd_desc_parse.c
        ${My_ARIC_SRC}/miniupnp/minisoap.c
        ${My_ARIC_SRC}/miniupnp/minissdpc.c
        ${My_ARIC_SRC}/miniupnp/miniupnpc.c
        ${My_ARIC_SRC}/miniupnp/miniwget.c
        ${My_ARIC_SRC}/miniupnp/minixml.c
        ${My_ARIC_SRC}/miniupnp/portlistingparse.c
        ${My_ARIC_SRC}/miniupnp/receivedata.c
        ${My_ARIC_SRC}/miniupnp/upnpcommands.c
        ${My_ARIC_SRC}/miniupnp/upnpreplyparse.c )
		
#之后，添加编译		

add_library( # Sets the name of the library.
             Aria

             # Sets the library as a shared library.
             SHARED

            ${AriaSrc}
            ${am__append_1_4}
            ${am__append_6_22}
            ${am__append_23_29}
            ${am__append_30}
            ${am__append_31}
            ${wslay_Src}
            ${libutp}
            ${libnatpmp}
            ${libupnp}

            #自己的源码文件
            src/main/cpp/Aria2AndroidJni.cpp
            )
```
CmakeList目录结构
![结果显示](/uploads/upnp协议支持/CmakeList目录结构.png)
transmission中代码为C风格，这里要改成C++11的方式  ，接下来就是对应的移植了
```cpp
首先 我们也要搞一个开关配置，具体怎么添加，可以参照源码原本的添加过程
//设置允许端口隐射
gloableOptions.push_back(std::pair<std::string,std::string> ("bt-enable-port-forward","true"));

对于这种端口映射，我们并不是针对单个种子，而是针对所有的种子，这样都可以共用这个隐射,所以我们的初始化，放在Dht初始化

//判断是否允许端口隐射
if(e->getOption()->getAsBool(PREF_ENABLE_PORT_FORWARD))
{
    //构建一个共享的智能指针对象交给BtRegister保存，这样当BtRegistry销毁的时候，这个对象就会被销毁了
    auto portForward = std::make_shared<BtPortForward>();
    //将BtPortForward 对象保存在 BtRegister中，这样当BtRegister销毁的时候，这个对象才会被销毁 由于 make_shared 引用为0 才会销毁
    e->getBtRegistry()->setBtPortForward(portForward);

    //构建BtPortForwardCommand对象，调用对应的构造函数完成初始化 用于监听端口隐射变化
    auto c = make_unique<BtPortForwardCommand>(e->newCUID(), e);
    c->setBtPortForward(portForward);
    //添加到commands集合中 ,move之后，这个对象就转移了不可以用了
    tempCommands.push_back(std::move(c));
}

我们的BtPortForward 就相当于transmission 中的tr_shared对象
 //端口隐射
class BtPortForward {
private:
    //标识是否允许
    bool isEnabled;

    //标识是否正在关闭
    bool isShuttingDown;

    //标识是否需要端口检查
    bool doPortCheck;

    //pmp 协议 状态
    BtPortForwarding natpmpStatus_;

    //upnp 协议状态
    BtPortForwarding upnpStatus_;

    //对应的 upnp的协议的 对象指针
    std::shared_ptr<BtUpnp> _upnp;
    //对应的 pmp 协议的 对象指针
    std::shared_ptr<BtNatpmp> _natpmp;
}

对于transmission的 定时器我们采用 Command
class BtPortForwardCommand : public Command {
private:
    DownloadEngine* e_;
    std::shared_ptr<BtPortForward> portForward_;
    //检查的时间点
    Timer checkPoint_;
    //刷新的时间点
    std::chrono::milliseconds refreshInterval_;

    int tr_sharedTraversalStatus();
    void natPulse(bool status);
    void set_evtimer_from_status();
public:

    BtPortForwardCommand(cuid_t cuid, DownloadEngine* e);

    virtual bool execute() CXX11_OVERRIDE;

    void setBtPortForward(const std::shared_ptr<BtPortForward>& portForward);
};

所以我们的command execute代码可以这样写

//command命令的执行,用来执行校验端口隐射
bool BtPortForwardCommand::execute() {
    //当前引擎退出的还是才会回收这个command对象
    if (e_->getRequestGroupMan()->downloadFinished() || e_->isHaltRequested()) {
        //LOGD("BtPortForwardCommand return true");
        return true;
    }

    //检查是否到了刷新的时间点
    if(checkPoint_.difference(global::wallclock()) >= refreshInterval_)
    {
        //重置刷新的时间
        checkPoint_ = global::wallclock();

        LOGD("BtPortForwardCommand::execute time to exec");

        /* do something */
        natPulse(portForward_->getDoPortCheckStatus());

        //将doPortCheck置位false
        portForward_->setDoPortCheckStatus(false);

        //根据不同的状态决定定时器触发的时间
        set_evtimer_from_status();
    }
    //下次再触发检查
    e_->addRoutineCommand(std::unique_ptr<Command>(this));
    return false;
}

/**
 * 依据情况更改检查端口的时间点
*/
void BtPortForwardCommand::set_evtimer_from_status() {

    int msec = 0;
    /* when to wake up again */
    switch (tr_sharedTraversalStatus()) {
        case TR_PORT_MAPPED:
            //如果当前我们已经隐射成功了，那么20分钟之后再来检查
            /* if we're mapped, everything is fine... check back in 20 minutes
             * to renew the port forwarding if it's expired */
            //将doPortCheck置位true
            portForward_->setDoPortCheckStatus(true);
            msec = 60 * 20 * 1000;
            break;

        case TR_PORT_ERROR:
            //如果出现了错误，那么60秒之后再来
            /* some kind of an error. wait 60 seconds and retry */
            msec = 60 * 1000;
            break;

        default:
            /* in progress. pulse frequently. */
            msec = 333000;
            break;
    }
    LOGD("BtPortForwardCommand::set_evtimer_from_status msec %d",msec);
    //看情况的更改下次检查的刷新时间点
    refreshInterval_ = std::chrono::milliseconds(msec);
}

BtNatpmp 对象就跟 transmission 中的 struct tr_natpmp,只不过我们把他封装起来，比如 natPmpPulse

class BtNatpmp {
private:
    //标识是否已经发现
    bool has_discovered;
    //标识是否已经mapped
    bool is_mapped;
    //隐射完成之后 得到的外网端口号
    int public_port_;
    //隐射的内网端口号
    int private_port_;

    unsigned int renew_time;
    //下一个command 发送的时间
    unsigned int  command_time;
    //pmp 当前的状态
    tr_natpmp_state state;
    //真正的pmp 库对象
    natpmp_t natpmp;

    void setCommandTime ();

    bool canSendCommand ();

public:

    BtNatpmp();

    ~BtNatpmp();

    int natPmpPulse(int private_port,bool is_enable,int * public_port);.
}

BtUpnp 对象就跟 transmission 中的struct tr_upnp ,只不过我们把他封装起来，比如 tr_upnpPulse

class BtUpnp {
private:
    //标识是否发现
    bool hasDiscovered;

    struct UPNPUrls urls;

    struct IGDdatas data;
    //上一次upnp 隐射的本地局域网
    int port_;

    char lanaddr[16];

    //标识是否隐射完成
    bool isMapped;

    //upnp的状态
    tr_upnp_state state;

    struct UPNPDev * tr_upnpDiscover(int msec);

    int tr_upnpGetSpecificPortMappingEntry(const char * proto);

    int tr_upnpAddPortMapping(const char * proto, int port, const char * desc);

    void tr_upnpDeletePortMapping( const char * proto, int port);

public:
    BtUpnp();

    ~BtUpnp();

    int tr_upnpPulse(int port, int isEnabled, int doPortCheck);
};

再次回到初始化的地方
//判断是否允许端口隐射，端口隐射是针对所有的种子，并不是针对单个种子
if(e->getOption()->getAsBool(PREF_ENABLE_PORT_FORWARD))
{
    //构建一个共享的智能指针对象交给BtRegister保存，这样当BtRegistry销毁的时候，这个对象就会被销毁了
    auto portForward = std::make_shared<BtPortForward>();
    //将BtPortForward 对象保存在 BtRegister中，这样当BtRegister销毁的时候，这个对象才会被销毁 由于 make_shared 引用为0 才会销毁
    e->getBtRegistry()->setBtPortForward(portForward);
    ...
}
```
### Aria2 释放操作
> 由于我们采用的是智能指针，所以不用关心对象的销毁动作，比如 BtPortForward ，我们把他保存到了 BtRegistery中，这样当BtRegistery销毁的时候，这个BtPortForward的引用才会变成0,进而完成它的销毁操作,又因为  BtPortForward 中以智能指针的方式持有 BtUpnp,BtNatpmp 的引用，所以对应 BtUpnp,BtNatpmp 只要以智能指针的方式初始化，之后设置到这个对象中，对应这俩个对象也不用管理销毁的操作

```cpp
//对应的 upnp的协议的 对象指针
std::shared_ptr<BtUpnp> _upnp;
//对应的 pmp 协议的 对象指针
std::shared_ptr<BtNatpmp> _natpmp;

//如果为空，创建对应的pmp对象
if (!portForward_->getNatPmp())
{
    portForward_->setBtNatpmp(std::make_shared<BtNatpmp>());
}

//如果为空，创建对应的upnp 对象
if (!portForward_->getBtUpnp())
{
    portForward_->setBtUpnp(std::make_shared<BtUpnp>());
}

相应的我们可以在这俩个对象的虚构函数中，完成销毁的操作，比如退出的时候，删除之前的隐射关系

BtNatpmp::~BtNatpmp()
{
    if(is_mapped)
    {
        //判断是否需要删除之前的隐射 并且当前的时间达到了下一个command执行的时间
        const int val = sendnewportmappingrequest (&natpmp, NATPMP_PROTOCOL_TCP, private_port_, public_port_, 0);
        LOGD("is_mapped BtNatpmp::~BtNatpmp delete Port Forward", val);
    }
    closenatpmp (&natpmp);
    //LOGD("BtNatpmp 虚构函数执行");
}

BtUpnp::~BtUpnp()
{
    if(isMapped)
    {
        //删除隐射
        tr_upnpDeletePortMapping("TCP", port_);
        tr_upnpDeletePortMapping("UDP", port_);
        //tr_logAddNamedInfo(getKey(),_("Stopping port forwarding through \"%s\", service \"%s\""),handle->urls.controlURL, handle->data.first.servicetype);
        LOGD("BtUpnp::~BtUpnp Stopping port forwarding through %s service %s \n", urls.controlURL, data.first.servicetype);
        //删除隐射，通知状态，比如port为-1
        isMapped = false;
        state = TR_UPNP_IDLE;
        port_ = -1;
    }
    if (hasDiscovered)
    {
        FreeUPNPUrls(&urls);
    }
    //LOGD("BtUpnp 虚构函数执行");
}
```
Aria2 移植成功
![结果显示](/uploads/upnp协议支持/Aria2 移植成功.png)
比如当我们程序退出的时候查看释放的过程
![结果显示](/uploads/upnp协议支持/Aria2删除隐射.png)
在路由器上查看是否释放成功
![结果显示](/uploads/upnp协议支持/查看Aria2释放映射是否成功.png)

