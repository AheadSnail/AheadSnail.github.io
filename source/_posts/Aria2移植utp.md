---
layout: pager
title: Aria2移植utp
date: 2018-10-18 10:27:13
tags: [Android,NDK,Aria2]
description:  Aria2移植utp
---

 Aria2移植utp
<!--more-->
****简介****
===
```
先来了解下打洞的基础知识:

点对点穿透，需要实现的是对NAT的穿透。想实现NAT的穿透，当然要先了解NAT到底是什么，以及NAT是用来干什么的。
NAT全称Network Address Translation，意思是网络地址转换，在1994年提出。它可以对不同的IP及端口进行映射，将一个网络地址转换为另一个。NAT的主要用途，大家可以看路由器。
路由器具有一个WAN口及多个LAN口；WAN口对外，连接因特网，拥有公网IP；LAN口对内，构建本地网络，分配的是私网IP。当处于LAN网下的本地主机想要访问因特网的时候，路由器就会通过NAT技术，
将LAN 口的私网IP映射到WAN口的公网IP，实现网络地址的转换，这样本地主机就可以访问因特网了。

NAT的工作需要LAN网下的本地设备主动发起网络连接，然后NAT服务才会将这个连接映射到WAN口的公网IP完成转换。也就是说，如果本地主机没有主动发起连接，那么这个映射就不会存在，
那么公网上的机器就无法访问到私网上的机器。也就是说，只能是私网机器主动连接公网机器，而不能是公网机器主动连接私网机器。而为了实现公网机器主动连接私网机器，我们就需要穿透NAT，
这就是NAT穿透的由来。

目前比较好实现NAT穿透的方式是采用UDP连接对NAT进行打洞，然后完成连接。何为打洞呢？就是为了使NAT产生一个可用的映射。具体步骤就是在私网机器上用UDP向某台公网机器发起连接，
使得NAT产生一个可以使用的映射（洞）。然后通过这个映射（洞），就可以穿透NAT。至于为什么要用UDP，这是由于UDP的某些特性。UDP通信需要先绑定本地机器的端口，完成后就可以从这个端口收发数据，
至于从哪里收，发到哪里，可以在收发数据的时候再决定，这也就意味着我可以用这一个端口同时和多个对象通信，只要我收发数据的时候指定不同的对象即可。当本地机器用UDP向英特网上的某个服务器
发送数据的时候，这个映射不但能用来和这个服务器进行数据交互，也能用来接收其他主机发来的数据。NAT穿透就是本地主机向公网上的某台服务器发送数据，这时服务器就可以获得NAT对这台主机的映射，
在之前举得例子中就是55.66.77.88:5000这个地址。由于NAT会将55.66.77.88:5000收到的数据转发至本地主机，所以公网上的其他机器可以从服务器获取到55.66.77.88:5000这个网络地址，
然后通过这个网络地址向私网下的机器发出数据。而至于为什么不用TCP，也是由于TCP的某些特性。TCP通信的步骤与UDP不同，它需要先在两个对象之间建立一个专用通道，再用这个通道收发数据。
也就是说外人无法插手。这样一来，虽然其他机器可以通过服务器获取到NAT的映射对象，也没办法利用它向私网下的机器发出数据。

由于我们这个采用的是Aria2这个开源的p2p项目，当然如果人家本身就支持的化，那当然是最好的，当然也有人去提问，但是看情况的化，近期之内是不会添加的
```
![结果显示](/uploads/Aria2Utp/Aria2官网目前不支持utp.png)
![结果显示](/uploads/Aria2Utp/Aria2不支持utp.png)

****utp的使用****
===
```java
可以看出来，Aria2在2015年就已经提出了，但是直到现在都没有任何的进展，当然我们不能等他支持，现在我要尝试的引进这个utp的协议

utp这个库本质就是udp的协议，只是他做了一层封装，对于使用者来说，不用去管理丢包等问题,接下来先看看utp的使用
utp下载源码地址为  https://github.com/bittorrent/libutp

将源码下载下来，放在linux下面是可以直接编译的,这里为了看代码的快速，这里推荐使用Eclipse for c++,这个专门为c/c++定制的Eclipse还是挺好用的,由于他本身就包含了Makefile文件的，所以
在使用Eclipse的时候，直接File 选项 选择Makefile project with Exiting Code 导进项目，点击make就可以编译了,编译完成之后，进入到源码有一个ucat可执行程序，这个就是提供的demo
```
使用方法
![结果显示](/uploads/Aria2Utp/utp使用.png)

服务端监听端口
![结果显示](/uploads/Aria2Utp/Utp客户端监听端口.png)

客户端连接
![结果显示](/uploads/Aria2Utp/Utp客户端连接.png)

utp连接上互相发送消息
![结果显示](/uploads/Aria2Utp/utp连接上互相发送消息.jpg)

```java
源码分析，首先是解析传递过来的参数
int main(int argc, char *argv[])
{
	int i;

	o_local_address = "0.0.0.0";

	while (1) {
		int c = getopt (argc, argv, "hdlp:B:s:n");
		if (c == -1) break;
		switch(c) {
			case 'h': usage(argv[0]);				break;
			case 'd': o_debug++;					break;
			case 'l': o_listen++;					break;//标识是否是服务端
			case 'p': o_local_port = optarg;		break;//服务端监听的端口号
			case 'B': o_buf_size = atoi(optarg);	break;
			case 's': o_local_address = optarg;		break;
			case 'n': o_numeric++;					break;
			//case 'w': break;	// timeout for connects and final net reads
			default:
				die("Unhandled argument: %c\n", c);
		}
	}
	...
	setup();
	...
}
void setup(void)
{
	//首先客户端跟服务端都要绑定一个端口号，要不然怎么能收到数据
	if (o_numeric)
		hints.ai_flags |= AI_NUMERICHOST;

	if ((error = getaddrinfo(o_local_address, o_local_port, &hints, &res)))
		die("getaddrinfo: %s\n", gai_strerror(error));

	//udp监听端口
	if (bind(fd, res->ai_addr, res->ai_addrlen) != 0)
		pdie("bind");

	freeaddrinfo(res);
	..
	//执行utp的初始化
	ctx = utp_init(2);

	//配置utp的函数指针
	utp_set_callback(ctx, UTP_LOG,				&callback_log); //utp日志回调
	utp_set_callback(ctx, UTP_SENDTO,			&callback_sendto);//utp发送数据的回调
	utp_set_callback(ctx, UTP_ON_ERROR,			&callback_on_error);//utp错误的回调
	utp_set_callback(ctx, UTP_ON_STATE_CHANGE,	&callback_on_state_change);//utp 状态的回调
	utp_set_callback(ctx, UTP_ON_READ,			&callback_on_read);//utp接收到数据之后，解析返回原始数据的回调
	utp_set_callback(ctx, UTP_ON_FIREWALL,		&callback_on_firewall);
	utp_set_callback(ctx, UTP_ON_ACCEPT,		&callback_on_accept);//服务端收到utp连接的回调
	...
	
	//o_listen 是解析传递过来的参数来确定是否是服务端，这里!代表是客户端，也即是客户端此时应该要触发连接操作
	if (! o_listen) {
		//客户端触发连接操作
		s = utp_create_socket(ctx);

		if ((error = getaddrinfo(o_remote_address, o_remote_port, &hints, &res)))
			die("getaddrinfo: %s\n", gai_strerror(error));

		utp_connect(s, res->ai_addr, res->ai_addrlen);
		freeaddrinfo(res);
	}
}

由于utp不管你数据的接收跟发送，所以当你发送数据的之前，你先将数据丢给utp，当utp处理完之后，其实就是添加数据头，然后他会调用 callback_sendto 函数，来完成真正的发送数据
uint64 callback_sendto(utp_callback_arguments *a)
{
	struct sockaddr_in *sin = (struct sockaddr_in *) a->address;
	//真正的发送数据
	sendto(fd, a->buf, a->len, 0, a->address, a->address_len);
	return 0;
}

//客户端监听标准输入，也即是命令行，获取到用户输入的内容
if ((p[0].revents & POLLIN) == POLLIN) {
	//读取用户输入的内容
	len = read(STDIN_FILENO, buf+buf_len, o_buf_size-buf_len);
	if (len < 0 && errno != EINTR)
		pdie("read stdin");
	if (len == 0) {
		debug("EOF from file\n");
		eof_flag = 1;
		close(STDIN_FILENO);
	}
	else {
		//将当前要发送的内容存储到缓冲区中，至于为什么要有缓冲区，是因为当utp还没有连接或者处于UTP_STATE_WRITABLE的时候，是不能发送数据的，这时候，就要将内容缓存起来
		buf_len += len;
		debug("Read %d bytes, buffer now %d bytes long\n", len, buf_len);
	}
	//发送用户的内容
	write_data();
}

//发送内容
void write_data(void)
{
	if (! s)
		goto out;

	while (p < buf+buf_len) {
		size_t sent;

		//返回值代表写入了多少内容,如果为0可能utp还没有连接上，或者对方处于UTP_STATE_WRITABLE 状态，此时应该退出，下次再来尝试的写
		sent = utp_write(s, p, buf+buf_len-p);
		if (sent == 0) {
			debug("socket no longer writable\n");
			return;
		}

		//如果写成功,将当前写的内容从缓存中移除
		p += sent;

		if (p == buf+buf_len) {
				debug("wrote %zd bytes; buffer now empty\n", sent);
				p = buf;
				buf_len = 0;
			}
		else
			debug("wrote %zd bytes; %d bytes left in buffer\n", sent, buf+buf_len-p);
	}
	...
}


//接下来分析下utp状态改变的回调函数，
uint64 callback_on_state_change(utp_callback_arguments *a)
{
	debug("state %d: %s\n", a->state, utp_state_names[a->state]);
	utp_socket_stats *stats;

	switch (a->state) {
		case UTP_STATE_CONNECT://utp连接上可以发送数据了
		case UTP_STATE_WRITABLE://出现这种状态是当utp发现对方的滑动窗口太小，就会处于这种状态
			write_data();
			break;

		case UTP_STATE_EOF://这是当收到了另一方的断开连接的状态,要关闭连接
			debug("Received EOF from socket\n");
			utp_eof_flag = 1;
			if (utp_shutdown_flag) {
				utp_close(a->socket);
			}
			break;
		....
	}

	return 0;
}


下面看看服务端的接收数据
if ((p[1].revents & POLLIN) == POLLIN) {
	while (1) {
		//接收数据
		len = recvfrom(fd, socket_data, sizeof(socket_data), MSG_DONTWAIT, (struct sockaddr *)&src_addr, &addrlen);
		//没有读取到内容
		if (len < 0) {
			//要注意这个，当没有收到内容的时候，要调用这个utp_issue_deferred_acks 方法，这是发送ack，这样对方才能知道你这边的情况，到底是丢包了还是收到了等
			if (errno == EAGAIN || errno == EWOULDBLOCK) {
				utp_issue_deferred_acks(ctx);
				break;
			}
			else
			pdie("recv");
		}

		//读取到了内容
		debug("Received %zd byte UDP packet from %s:%d\n", len, inet_ntoa(src_addr.sin_addr), ntohs(src_addr.sin_port));

		//交给utp处理
		if (! utp_process_udp(ctx, socket_data, len, (struct sockaddr *)&src_addr, addrlen))
			debug("UDP packet not handled by UTP.  Ignoring.\n");
	}
}

当utp解析客户度发送的内容之后，如果发现 客户度发送的连接为utp连接，utp库会内部自动的创建一个utpSocket，并且回调执行,通知服务端可以发送数据了
uint64 callback_on_accept(utp_callback_arguments *a)
{
	assert(!s);
	s = a->socket;
	debug("Accepted inbound socket %p\n", s);
	write_data();
	return 0;
}

当utp解析客户度发送的内容之后，如果发现 客户度发送的内容不为连接请求，那么将会回调执行  callback_on_read ，那么我们就能得到了对方发送的原始的数据了
uint64 callback_on_read(utp_callback_arguments *a)
{
	const unsigned char *p;
	ssize_t len, left;

	left = a->len;
	p = a->buf;

	//将内容写到标准的输出控制台
	while (left) {
		len = write(STDOUT_FILENO, p, left);
		left -= len;
		p += len;
		debug("Wrote %d bytes, %d left\n", len, left);
	}
	utp_read_drained(a->socket);
	return 0;
}

下面说下大概的实现细节:
首先调用下面的这个方法，得到一个UtpContext对象，这个方法只能调用一次，UtpContext对象非常重要，基本上所有的api都要使用这个对象
ctx = utp_init(2);

创建UtpSocket对象,要注意的是可以创建多个UtpSocket对象，但是UtpContext对象只能有一个，UtpContext实现是内部会有一个集合，会根据对应的ip和端口号，找到对应的UtpSocket对象
而UtpSocket对象就相当是一个连接
s = utp_create_socket(ctx);

对应的这些回调的设置也只需要设置一次就好了，即使存在多个UtpSocket对象，内部也会找到当前对应的UtpSocket对象返回给你
utp_set_callback(ctx, UTP_LOG,				&callback_log);
utp_set_callback(ctx, UTP_SENDTO,			&callback_sendto);
utp_set_callback(ctx, UTP_ON_ERROR,			&callback_on_error);
utp_set_callback(ctx, UTP_ON_STATE_CHANGE,	&callback_on_state_change);
utp_set_callback(ctx, UTP_ON_READ,			&callback_on_read);
utp_set_callback(ctx, UTP_ON_FIREWALL,		&callback_on_firewall);
utp_set_callback(ctx, UTP_ON_ACCEPT,		&callback_on_accept);

给UtpContext对象设置userData，这相当于是设置公有的配置
void*			utp_context_set_userdata		(utp_context *ctx, void *userdata);
获取到UtpConext对象设置的userData
void*			utp_context_get_userdata		(utp_context *ctx);

给当前的UtpSocket设置Userdata。这相当于是给每一个UtpSocket设置自己的UserData,这相当于是自己独有的
void*			utp_set_userdata				(utp_socket *s, void *userdata);
获取到UtpSocket对象设置的userData
void*			utp_get_userdata				(utp_socket *s);
```

****Aria2 移植utp****
===
```java
当然一开始我并不知道怎么去移植utp库，我是参考transmission的做法，只是transmission使用的utp库不是官网提供的，估计是早先的版本，他有做了部分的修改，但是只要了解他为什么要这样做
就其实也没有多大的关系，所以最终我采用的是系统的utp库，并且到了最后面也没有动到这个库的一行代码，关于transmission的源码分析，前面一篇文章有介绍,下面开始移植


1.首先找到初始化UtpContext的对象，而且要考虑这个对象只能初始化一次，也即是单例的存在,最终放到了Dht的初始化部分
DHTSetup::setup(DownloadEngine* e, int family)
{
	...
	//这里要注意，  DHTRegistry::isInitialized() 这个取的是静态对象值，所以对于多个种子来说，如果之前dht已经初始化过了，就不会再继续往下执行了，直接return 处理,那这样也可以保证我们的
	//utpContext  的创建也是只会有一份 ,也即是后面的这些对象关于Dht的创建都是只有一份的，也即是所有的种子文件公用的
	if ((family != AF_INET && family != AF_INET6) || (family == AF_INET && DHTRegistry::isInitialized()) || (family == AF_INET6 && DHTRegistry::isInitialized6())) {
		return {};
	}
	
	//创建Utp的 context对象,下面的这个选项是自己创建的，让他变成一个可控的形式
    if(e->getOption()->getAsBool(PREF_ENABLE_UTP))
    {
        utp_context* utpContext = utp_init(2);
        assert(utpContext);
        A2_LOG_DEBUG(fmt("UTP context create"));
        //构建一个公有的对象记得回收处理
        UtpContextUserData * utpContextUserData = new UtpContextUserData();
        //赋值UserData
        utpContextUserData->connection = connection.get();
        utpContextUserData->downloadEngine = e;
        //设置公有的回调函数执行的时候，传递的参数
        utp_context_set_userdata(utpContext,utpContextUserData);
        //保存utp Context对象,这个对象是全局的唯一的
        e->getBtRegistry()->setUtpContext(utpContext);

        //添加对应的回调函数
        utp_set_callback(utpContext, UTP_SENDTO,			&callback_sendto);
        utp_set_callback(utpContext, UTP_ON_ERROR,			&callback_on_error);
        utp_set_callback(utpContext, UTP_ON_STATE_CHANGE,	&callback_on_state_change);
        utp_set_callback(utpContext, UTP_ON_READ,			&callback_on_read);
        utp_set_callback(utpContext, UTP_ON_ACCEPT,		    &callback_on_accept);
    }	
	...
}

transmission中utp和dht都是使用同一个端口号，所以这边也是参考transmission的做法，所以在上边设置UtpContext的UserData的时候设置了一个成员为connnection对象，这个connection对象用来
后面真正的发送消息

//Utp 发送的触发函数
uint64 callback_sendto(utp_callback_arguments *a)
{
    struct sockaddr_in *sin = (struct sockaddr_in *) a->address;
	//获取到UtpContext 公有的UserData
    UtpContextUserData *contextUserData = static_cast<UtpContextUserData *>(utp_context_get_userdata(a->context));
    //发送消息
    size_t port = ntohs(sin->sin_port);
    //最终都是使用socketfd发送的，所以可以这样写
    contextUserData->connection->sendMessage(a->buf, a->len,inet_ntoa(sin->sin_addr), port);
    return 0;
}

//看下utp的消息的接收
bool DHTInteractionCommand::execute()
{
	...
	while (1) {
      //尝试的读取是否接受到了内容
      ssize_t length = connection_->receiveMessage(data.data(), data.size(), remoteAddr, remotePort);
      //如果没有接受到内容
      if (length <= 0) {
          utp_context * utpContext = e_->getBtRegistry()->getUtpContext();
          if(utpContext)
          {
              //发送ack
              if (errno == EAGAIN || errno == EWOULDBLOCK) {
                  utp_issue_deferred_acks(utpContext);
              }
          }
          break;
      }

      if (data[0] == 'd') {  //这是dht接收的内容
        // udp tracker response does not start with 'd', so assume
        // this message belongs to DHT. nothrow.
        receiver_->receiveMessage(remoteAddr, remotePort, data.data(), length);
        //LOGD("dht receiveMessage");
      }
      else if (length >= 8 && data.data()[0] == 0 && data.data()[1] == 0 && data.data()[2] == 0 && data.data()[3] <= 3) {//udp tracker
          std::shared_ptr<UDPTrackerRequest> req;
          if (udpTrackerClient_->receiveReply(req, data.data(), length, remoteAddr, remotePort, global::wallclock()) == 0) {
              if (req->action == UDPT_ACT_ANNOUNCE) {
                  auto c = static_cast<TrackerWatcherCommand *>(req->user_data);
                  if (c) {
                      c->setStatus(Command::STATUS_ONESHOT_REALTIME);
                      e_->setNoWait(true);
                  }
              }
          }
      }
      else {
         //utp  消息的转发处理
         utp_context * utpContext = e_->getBtRegistry()->getUtpContext();
         if(utpContext)
         {
             int error;
             struct addrinfo hints,*res;
             memset(&hints, 0, sizeof(hints));
             hints.ai_family = AF_INET;
             hints.ai_socktype = SOCK_DGRAM;
             hints.ai_protocol = IPPROTO_UDP;
             //转成地址
             if((error = getaddrinfo(remoteAddr.c_str(), util::uitos(remotePort).c_str(),&hints, &res)))
             {
                 //出现异常
                 if (error) {
                     LOGE("DHTInteractionCommand::execute getaddrinfo error break");
                     throw DL_ABORT_EX(fmt(EX_RESOLVE_HOSTNAME, remoteAddr.c_str(), gai_strerror(error)));
                 }
             }

             //将接受的内容转到utp
             if (!utp_process_udp(utpContext, data.data(), length, res->ai_addr, res->ai_addrlen))
             {
                 LOGD("UDP packet not handled by UTP.  Ignoring.\n");
             }
             //释放res
             freeaddrinfo(res);
         }
      }
    }
	...
}
在原有的消息类型的基础上，添加了一个utp类型的消息接收，这个也是参照transmission的做法

创建UtpSocket执行连接操作
bool PeerInitiateConnectionCommand::executeInternal()
{
	...
	//每个peer都要创建对应的Utp_Sokcet 对象
    utp_context * utpContext = getDownloadEngine()->getBtRegistry()->getUtpContext();
    //当前peer 如果么有创建对应的utpSocket对象，则构建一个
    utp_socket * utpSocket = getPeer()->getUtpSocket();
    if(utpContext && !utpSocket)
    {
        A2_LOG_DEBUG(fmt(MSG_CONNECTING_TO_SERVER, getCuid(), getPeer()->getIPAddress().c_str(), getPeer()->getPort()));
        LOGD(MSG_CONNECTING_TO_SERVER, getCuid(), getPeer()->getIPAddress().c_str(), getPeer()->getPort());
        int error;
        struct addrinfo hints,*res;
        memset(&hints, 0, sizeof(hints));
        hints.ai_family = AF_INET;
        hints.ai_socktype = SOCK_DGRAM;
        hints.ai_protocol = IPPROTO_UDP;
        //创建utp
        utpSocket = utp_create_socket(utpContext);
        assert(utpSocket);
        A2_LOG_DEBUG(fmt("CUID#%" PRId64 " UTP socket create Success",getCuid()));
        LOGD("CUID#%" PRId64 " UTP socket create Success",getCuid());

        //保存当前种子对象 对应的utpSocket对象
        getPeer()->setUtpSocket(utpSocket);
        //转成地址
        if((error = getaddrinfo(getPeer()->getIPAddress().c_str(), util::uitos(getPeer()->getPort()).c_str(),&hints, &res)))
        {
            //出现异常,抛出
            if (error) {
                A2_LOG_DEBUG(fmt("utp_connect getAddrinof error"));
                LOGD("utp_connect getAddrinof error");
                throw DL_ABORT_EX(fmt(EX_RESOLVE_HOSTNAME, getPeer()->getIPAddress().c_str(), gai_strerror(error)));
            }
        }

        if(utpSocket)
        {
            //这里也会重置掉之前的设置的userData
            utp_set_userdata(utpSocket,this);
        }
        //执行utp的连接
        utp_connect(utpSocket, res->ai_addr, res->ai_addrlen);
        //释放res
        freeaddrinfo(res);
	}
	...
}

这里要注意对于原有的Aria2的源码来说，都是使用command来执行对应的类的，对应原有的tcp的化，他每个都会创建一个连接，但是对于utp的化，我们因为是在同一个地方执行数据的发送，同一个对方
执行数据的接收，所以我们就要设计到对应的分发操作，所以设置了 utp_set_userdata(utpSocket,this); 这样当utp接收到了数据，触发callback_on_read 的时候，就可以分发数据了

//utp 将接受的数据包，处理之后执行的回调函数
uint64 callback_on_read(utp_callback_arguments *a)
{
    //LOGD("utp callback_on_read len %d",a->len);
    //对应的utpSocket对象
    utp_socket * utpSocket = a->socket;
	//获取到UtpSocket对应的UserData对象
    void* utp_socket_userData = utp_get_userdata(utpSocket);
    //我们在超时的时候，手动的置为了NULL
    if(utp_socket_userData)
    {
        PeerAbstractCommand * peerAbstractCommand = static_cast<PeerAbstractCommand *>(utp_socket_userData);
        if(peerAbstractCommand)
        {
            //对应的子类实现 分发消息
            peerAbstractCommand->dispatchUtpMessage(a->buf,a->len);
        }
    }
    utp_read_drained(a->socket);
    return 0;
}

由于这些command都是PeerAbstractCommand的子类，所以子类只要重写了这个方法，就能将消息转发过来，比如
//utp分发消息
void PeerReceiveHandshakeCommand::dispatchUtpMessage(const unsigned char *buff,size_t buffLen)
{
    A2_LOG_DEBUG(fmt("CUID#%" PRId64 " PeerReceiveHandshakeCommand dispatchUtpMessage buffLen %d",getCuid(),buffLen));
    LOGD("CUID#%" PRId64 " PeerReceiveHandshakeCommand dispatchUtpMessage buffLen %d",getCuid(),buffLen);
    //第三个参数默认为true
    peerConnection_->PeerUtpRecvDataProcess(buff,buffLen);
}

我们采用的一个思想就是尽量的少改动他tcp原有的逻辑，让utp也走tcp一样的逻辑，所以对应utp来说的，当把消息接受过来之后，我们就将消息的内容填充到对应的buff中，剩余的逻辑就走他原本tcp
的逻辑，我们可以实时的改变对应的UtpSocket对象的UserData,然后将对应的数据转到对应的对象来处理


utp的连接以及对应的超时重试处理:
如果当没有收到utp的状态变成UTP_STATE_CONNECT时，就执行数据的发送，此时utp的处理是直接抛去这个数据，参照他的demo也可知道，此时正确的做法是应该等他处于连接状态才能允许发送数据，还有
后面由于打洞的需要，我们要将utp能实现超时重连的机制，增大打洞的机制，这些操作，我们都可以直接在第一个Command 也即是PeerInitiateConnectionCommand 中处理

我们在utp状态改变回调的时候，通知这个对象，让他做相应的原本的逻辑
//utp 状态的变化回调
uint64 callback_on_state_change(utp_callback_arguments *a)
{
	...
		case UTP_STATE_CONNECT://通知utp连接上了，可以往下走了
            if(utp_socket_userData)
            {
                PeerAbstractCommand * peerAbstractCommand = static_cast<PeerAbstractCommand *>(utp_socket_userData);
                if(peerAbstractCommand)
                {
                    peerAbstractCommand->utpConnected();
                }
            }
            break;
        case UTP_STATE_EOF://收到了对方的断开连接的消息，比如下载完成的时候，或者其他的错误的时候，此时应该要关闭掉utp的连接,
             A2_LOG_DEBUG(fmt("Received EOF from socket"));
            if(utp_socket_userData)
            {
                PeerAbstractCommand * peerAbstractCommand = static_cast<PeerAbstractCommand *>(utp_socket_userData);
                if(peerAbstractCommand)
                {
                    //这里简单，直接传递一个utp错误
                    peerAbstractCommand->setUtpErrorCode(1);
                }
            }
            break;
	...
}

对应utp的连接通知，只有第一个command才需要知道，对应后面的command都是在当前处于连接状态才往后面走，所以在PeerAbstractCommand中可以添加默认的虚函数实现 
//新增utp连接上的回调,默认不处理
virtual void utpConnected(){};

然后让PeerInitiateConnectionCommand 实现这个方法，这样消息就能转发到这里
//第一个command应该要确保utp连接上之后，才让继续往下走，如果没有收到当前的回调函数，如果采用utp的化，应该要继续重试连接，直到peer的超时为止
void PeerInitiateConnectionCommand::utpConnected()
{
    A2_LOG_DEBUG(fmt("CUID#%" PRId64 " - utpConnected",getCuid()));
    LOGD("CUID#%" PRId64 " - utpConnected",getCuid());
    //返回true，代表当前这个command可以回收内存了
    isUtpConnected = true;
    //获取到当前peer保存的utpSocket对象,设置对应的SocketCore对象，然后继续往下走
    utp_socket * utpSocket = getPeer()->getUtpSocket();
    UtpContextUserData *contextUserData = static_cast<UtpContextUserData *>(utp_context_get_userdata(getDownloadEngine()->getBtRegistry()->getUtpContext()));
    //为每一个peer的连接都创建对应的SocketCore对象，但是由于是utp，所以这里的fd要共用一个 socketCore = contextUserData->connection->getSocket();
    auto socketCore = std::make_shared<SocketCore>(contextUserData->connection->getSocket()->getSockfd(),contextUserData->connection->getSocket()->getSocketType());
    //赋值utpSocket
    socketCore->setUtpSocket(utpSocket);
    //mseHandshakeEnabled_ 默认为true
    if (mseHandshakeEnabled_) {
        //构建一个InitiatorMSEHandshakeCommand 对象，调用对应的构造函数完成初始化,注意这里传递了socket对象 getSocket为对应的tcp 对象
        auto c = make_unique<InitiatorMSEHandshakeCommand>(getCuid(), requestGroup_, getPeer(), getDownloadEngine(), btRuntime_, socketCore);
        c->setPeerStorage(peerStorage_);
        c->setPieceStorage(pieceStorage_);
        getDownloadEngine()->addCommand(std::move(c));
    }
    else {
        //为false代表当前peer发送的交换秘钥的消息没有回应,直接走这里
        A2_LOG_DEBUG(fmt("PeerInitiateConnectionCommand mseHandshakeEnabled_ false"));
        LOGD("PeerInitiateConnectionCommand mseHandshakeEnabled_ false");
        //状态为INITIATOR_SEND_HANDSHAKE 代表客户端
        auto c = make_unique<PeerInteractionCommand>(getCuid(), requestGroup_, getPeer(), getDownloadEngine(), btRuntime_, pieceStorage_, peerStorage_, socketCore,
                                                     PeerInteractionCommand::INITIATOR_SEND_HANDSHAKE);
        getDownloadEngine()->addCommand(std::move(c));
    }
}

当前这个command收到了utp连接的状态才会往下执行构建其他的command执行剩余的操作，还有一点就是由于我们utp跟dht都是采用了同一个socket的fd，做到了消息的统一发送，消息的统一接受
但是每一个command内部都有自己的数据，如果不区分的化，那么数据就混在一起了，对应的也即是SocketCore 对象，我们这里采用的是保持他原本的逻辑只是我们构建SocketCore的时候，都是使用
同一个socket的fd，这样每一个command都有自己的socketCore，就可以将数据区分开来

//为每一个peer的连接都创建对应的SocketCore对象，但是由于是utp，所以这里的fd要共用一个 socketCore = contextUserData->connection->getSocket();
auto socketCore = std::make_shared<SocketCore>(contextUserData->connection->getSocket()->getSockfd(),contextUserData->connection->getSocket()->getSocketType());

当然对应SocketCore原有的写数据的方法，要换成我们utp的写方法,由于我们前面处理了utp处于连接状态的问题，所以后面就可以随便的发送数据，不用再考虑这个问题
ssize_t SocketCore::writeVector(a2iovec* iov, size_t iovcnt)
{
  ssize_t ret = 0;
  wantRead_ = false;
  wantWrite_ = false;
  if (!secure_) {
#ifdef __MINGW32__
    DWORD nsent;
    int rv = WSASend(sockfd_, iov, iovcnt, &nsent, 0, 0, 0);
    if (rv == 0) {
      ret = nsent;
    }
    else {
      ret = -1;
    }
#else  // !__MINGW32__
    //转成utp
    if(sockType_ == SOCK_DGRAM && utpSocket_)
    {
        //这里要增加返回值，要不然外层会抛出异常,0表示套接字不再可写,这个要注意utp的这个状态不是错误
        ret = utp_writev(utpSocket_,(struct utp_iovec *)iov,iovcnt);
        int errNum = SOCKET_ERRNO;
        if (ret == -1) {
            if (!A2_WOULDBLOCK(errNum)) {
                LOGD(EX_SOCKET_SEND, errorMsg(errNum).c_str());
                throw DL_RETRY_EX(fmt(EX_SOCKET_SEND, errorMsg(errNum).c_str()));
            }
            //出现了错误，如果为-1的化
            wantWrite_ = true;
            LOGD("utp_writev return -1");
            ret = 0;
        }
    }else{
        while ((ret = writev(sockfd_, iov, iovcnt)) == -1 && SOCKET_ERRNO == A2_EINTR);
#endif // !__MINGW32__
        int errNum = SOCKET_ERRNO;
        if (ret == -1) {
            if (!A2_WOULDBLOCK(errNum)) {
                throw DL_RETRY_EX(fmt(EX_SOCKET_SEND, errorMsg(errNum).c_str()));
            }
            wantWrite_ = true;
            ret = 0;
        }
    }
  }
  else {
    // For SSL/TLS, we could not use writev, so just iterate vector
    // and write the data in normal way.
    for (size_t i = 0; i < iovcnt; ++i) {
      ssize_t rv = writeData(iov[i].A2IOVEC_BASE, iov[i].A2IOVEC_LEN);
      if (rv == 0) {
        break;
      }
      ret += rv;
    }
  }
  return ret;
}

当然对应SocketCore原本的逻辑，当这个对象被销毁的时候，会关闭掉这个socket的fd，我们由于是共用的，不能关闭

//关闭连接,这里要处理，由于当前的utp共有一个fd，但是含有多个SocketCore对象，所以对应的虚构函数，就不能关闭掉这个fd，要不然会导致所有的都会失败
//最终关闭的地方应该交给最原始的对象DHTConnectionImpl 这个对象中的socketCore来关闭
void SocketCore::closeConnection()
{
  ...
  //tcp 不干预处理
  if(isNeedCloseFd || sockType_ != SOCK_DGRAM)
  {
    if (sockfd_ != (sock_t)-1) {
        shutdown(sockfd_, SHUT_WR);
        CLOSE(sockfd_);
        sockfd_ = -1;
        LOGD("SocketCore isNeedCloseFd shutDown sockfd");
    }
  }
}

而utp的超时回调会在错误的回调中告知,相应的我们也要在这个方法中告知对应的消息
//utp 错误回调，超时也是一个错误回调,所以这里要区分
uint64 callback_on_error(utp_callback_arguments *a)
{
    utp_socket * utpSocket = a->socket;
    void* utp_socket_userData = utp_get_userdata(utpSocket);
    //我们在超时的时候，手动的置为了NULL
    if(utp_socket_userData)
    {
        PeerAbstractCommand * peerAbstractCommand = static_cast<PeerAbstractCommand *>(utp_socket_userData);
        if(peerAbstractCommand)
        {
            A2_LOG_DEBUG(fmt("CUID#%" PRId64 " -- Error: %s Close Utp ", peerAbstractCommand->getCuid(),utp_error_code_names[a->error_code]));
            //设置对应的command为禁止写的，对于握手的command，有可能出现第一步utp的连接件就失败了，对于现有的逻辑握手的消息只有等utp连接上才会执行发送数据，导致
            //缓冲区中有数据，让command一直的在执行，不会销毁这个连接，而其实这边会收到当前utp的连接错误，如果对方不支持utp的化，那么就要这边来触发对应的command超时
            peerAbstractCommand->setUtpErrorCode(a->error_code);
        }
    }
    return 0;
}

对应utp的超时，其实也只要在第一个command也即是PeerInitateConnectionCommand中处理就好了，这个超时的处理是写在父类的 PeerAbstractCommand中的,下面新增这个方法
void PeerAbstractCommand::setUtpErrorCode(int utpErrorCode)
{
    A2_LOG_DEBUG(fmt("PeerAbstractCommand setError Code %d",utpErrorCode));
    utpErrorCode_ = utpErrorCode;
}

这样当这个command执行的时候，就要事实的去判断是否处于超时，是否有必要重新连接
bool PeerAbstractCommand::execute()
{
  //exitBeforeExecute() 由具体的子类来实现， 可以用来检查种子是否下载完成，如果完成，回收对象使用，btRuntime_->isHalt(); 默认为false，当下载完成的时候由SeedCheckCommand::execute()触发，设置为true
  if (exitBeforeExecute()) {
    //下载完成的时候也会触发onAbort函数
    onAbort();
    return true;
  }
  try {
    //readEventEnabled() 是检查socketfd 的是否可以读的结果       writeEventEnabled() 是检查socketfd 是否可以写的结果
    if (noCheck_ || (checkSocketIsReadable_ && readEventEnabled()) || (checkSocketIsWritable_ && writeEventEnabled()) || hupEventEnabled()) {
       checkPoint_ = global::wallclock();
    }
    else if (errorEventEnabled()) {
      A2_LOG_DEBUG(fmt(MSG_NETWORK_PROBLEM, socket_->getSocketError().c_str()));
      throw DL_ABORT_EX(fmt(MSG_NETWORK_PROBLEM, socket_->getSocketError().c_str()));
    }
    //检查是否超时了,如果超时了抛出异常
    if (checkPoint_.difference(global::wallclock()) >= timeout_) {
       LOGD(EX_TIME_OUT);
       throw DL_ABORT_EX(EX_TIME_OUT);
    }

    //如果当前utp的错误码为超时
    if(utpErrorCode_ == UTP_ETIMEDOUT)
    {
       A2_LOG_DEBUG(fmt("CUID#%" PRId64 " utp timeout error retry Connect",getCuid()));
       //utp超时了，当peer还没有超时，utp 应该再次触发连接，这个utp的超时重新连接应该发生在第一个command上，默认为空实现
       bool isNeedProcess = onUtpTimeOutNextConnect();
       if(isNeedProcess)
       {
           A2_LOG_DEBUG(fmt("CUID#%" PRId64 " utp timeout error recycle data",getCuid()));
           throw DL_ABORT_EX(EX_TIME_OUT);
       }
    } else if(utpErrorCode_ != -1){//默认为-1
        //如果是其他的错误，就直接抛出异常终止连接
        A2_LOG_DEBUG(fmt("CUID#%" PRId64 " utp other error recycle data",getCuid()));
        throw DL_ABORT_EX(EX_TIME_OUT);
    }
    //变为默认值
    utpErrorCode_ = -1;
    //执行子类的executeInternal函数 ,相应的也即是执行每一个commamand的execute方法
    return executeInternal();
  }
  catch (DownloadFailureException& err) {//下载异常会导致返回true，这样当前对应的对象就会被释放掉了
    //A2_LOG_ERROR_EX(EX_DOWNLOAD_ABORTED, err);
    onAbort();
    onFailure(err);
    A2_LOG_DEBUG(fmt("CUID#%" PRId64 " PeerAbstractCommand Download Failure return true ",getCuid()));
    return true;
  }
  catch (RecoverableException& err) {//超时异常的响应
    A2_LOG_DEBUG_EX(fmt(MSG_TORRENT_DOWNLOAD_ABORTED, getCuid()), err);
    A2_LOG_DEBUG(fmt("CUID#%" PRId64 " - PeerAbstractCommand 超时或者出现了异常了或者下载完成了..... ",getCuid()));
    //LOGD(MSG_PEER_BANNED, getCuid(), peer_->getIPAddress().c_str(), peer_->getPort());
    onAbort();
    //子类实现
    return prepareForNextPeer(0);
  }
}

我们的utp超时是跟peer超时关联在一起的，也即是当peer还没有超时，而utp超时的化，此时utp应该要重试连接,所以我们把超时的逻辑放到了peer检查超时的后面，这样对应peer的超时我们就不要处理任何的
逻辑，走原本的逻辑就好，超时的时候会触发onUtpTimeOutNextConnect 操作,这个函数默认是返回true的，返回true的化，就代表不用执行重新连接

//当前收到了utp的超时回调，默认返回true，代表是否需要中断连接，执行销毁操作
bool PeerAbstractCommand::onUtpTimeOutNextConnect()
{
    return true;
}

除了我们前面所的第一个command  PeerInitiateConnectionCommand,要重写这个方法，实现重新连接,由于返回了false，就不会抛出异常
//utp的超时重连实现,只针对第一个command才要增加，如果utp连接上了，后面就不用管了
bool PeerInitiateConnectionCommand::onUtpTimeOutNextConnect()
{
    A2_LOG_DEBUG(fmt("CUID#%" PRId64 " - onUtpTimeOutNextConnect",getCuid()));
    LOGD("CUID#%" PRId64 " - onUtpTimeOutNextConnect",getCuid());
    //显示的调用父类的函数，完成utp销毁操作
    PeerAbstractCommand::onAbort();
    return false;
}

PeerAbstractCommand::onAbort(); 用来统一的销毁，所以写在父类 异常终止的时候会触发这个方法，或者下载完成也会触发这个方法，在这里执行对应的utp的清理操作
void PeerAbstractCommand::onAbort()
{
    //如果这里返回了true，DownLoadEngine就会执行释放当前的对象，那么我们绑定的userData就为非法的所以再返回true之前，应该手动的清理userData
    utp_socket * utpSocket = getPeer()->getUtpSocket();
    if(utpSocket)
    {
        //我们在清除utpSocket的UserData的时候，要清除掉用户的数据，还有这里要手动的释放关闭掉utp的连接操作，要不然可能会一直收到对方peer的信息，但是我们是不需要的,比如收到了对方peer数据不完整的时候，此时
        //我们是需要断开的，
        //utp的关闭操作处理，由于有些地方我们手动的调用了close操作，所以这里可能会导致二次close会出现异常,下面的代码来自utp
        if(utp_get_status(utpSocket) != 0 && utp_get_status(utpSocket) != 7)
        {
            utp_close(utpSocket);
        }
        utp_set_userdata(utpSocket,NULL);
        //清除peer的 utpSocket对象引用
        getPeer()->clearUtpSocket();
    }
    A2_LOG_DEBUG(fmt("CUID#%" PRId64 " -- PeerAbstractCommand::onAbort clear utp",getCuid()));
}

//command命令的执行，由于在超时的方法中清空掉了内容所以 getPeer()->getUtpSocket();就为空，所以会再创建一个utpSocket执行连接
bool PeerInitiateConnectionCommand::executeInternal()
{
   if(requestGroup_->getOption()->getAsBool(PREF_ENABLE_UTP))//utp 的处理，默认为true
   {
       //如果当前utp连接上了，可以销毁这个对象了
       if(isUtpConnected)
       {
           return true;
       }
       //每个peer都要创建对应的Utp_Sokcet 对象
       utp_context * utpContext = getDownloadEngine()->getBtRegistry()->getUtpContext();
       //当前peer 如果么有创建对应的utpSocket对象，则构建一个
       utp_socket * utpSocket = getPeer()->getUtpSocket();
       if(utpContext && !utpSocket)
       {
           A2_LOG_DEBUG(fmt(MSG_CONNECTING_TO_SERVER, getCuid(), getPeer()->getIPAddress().c_str(), getPeer()->getPort()));
           LOGD(MSG_CONNECTING_TO_SERVER, getCuid(), getPeer()->getIPAddress().c_str(), getPeer()->getPort());
           int error;
           struct addrinfo hints,*res;
           memset(&hints, 0, sizeof(hints));
           hints.ai_family = AF_INET;
           hints.ai_socktype = SOCK_DGRAM;
           hints.ai_protocol = IPPROTO_UDP;
           //创建utp
           utpSocket = utp_create_socket(utpContext);
           assert(utpSocket);
           A2_LOG_DEBUG(fmt("CUID#%" PRId64 " UTP socket create Success",getCuid()));
           LOGD("CUID#%" PRId64 " UTP socket create Success",getCuid());

           //保存当前种子对象 对应的utpSocket对象
           getPeer()->setUtpSocket(utpSocket);
           //转成地址
           if((error = getaddrinfo(getPeer()->getIPAddress().c_str(), util::uitos(getPeer()->getPort()).c_str(),&hints, &res)))
           {
               //出现异常,抛出
               if (error) {
                   A2_LOG_DEBUG(fmt("utp_connect getAddrinof error"));
                   LOGD("utp_connect getAddrinof error");
                   throw DL_ABORT_EX(fmt(EX_RESOLVE_HOSTNAME, getPeer()->getIPAddress().c_str(), gai_strerror(error)));
               }
           }

           if(utpSocket)
           {
               //这里也会重置掉之前的设置的userData
               utp_set_userdata(utpSocket,this);
           }
           //执行utp的连接
           utp_connect(utpSocket, res->ai_addr, res->ai_addrlen);
           //释放res
           freeaddrinfo(res);
       }
       //添加自己到commands集合中 ,下次继续的检查
       addCommandSelf();
       return false;
   }
}

//服务端收到了utp的消息，服务端进行后续的操作，比如初始化 握手对象等,这里就能看出来为什么我们将engine存储在userData中
uint64 callback_on_accept(utp_callback_arguments *a)
{
    struct sockaddr_in *sin = (struct sockaddr_in *) a->address;
    //构建一个Peer 对象，调用对应的构造函数完成初始化，注意这里的最后一个参数为true
    size_t port = ntohs(sin->sin_port);
    auto peer = std::make_shared<Peer>(inet_ntoa(sin->sin_addr), port, true);

    //获取到utpContext对象中的userData
    UtpContextUserData *contextUserData = static_cast<UtpContextUserData *>(utp_context_get_userdata(a->context));
    DownloadEngine * engine = contextUserData->downloadEngine;
    cuid_t cuid = engine->newCUID();

    //为每一个peer的连接都创建对应的SocketCore对象，但是由于是utp，所以这里的fd要共用一个 socketCore = contextUserData->connection->getSocket();
    //注意这里不能使用contextUserData中的connection中的socketCore，那个SocketCore是公有的，只负责utp的发数据，其他的都应该重新的构建一个SokcetCore，只是共用一个fd
    auto socketCore = std::make_shared<SocketCore>(contextUserData->connection->getSocket()->getSockfd(),contextUserData->connection->getSocket()->getSocketType());
    //保存当前的UtpSockt,方便后面获取到更改对应的userData
    peer->setUtpSocket(a->socket);

    LOGD("CUID#%" PRId64 " service receive  the connection from %s:%u.", cuid,peer->getIPAddress().c_str(), peer->getPort());

    //设置当前的socket对应的utp_socket对象，后面要用到
    socketCore->setUtpSocket(a->socket);
    //构建服务端的握手对象
    auto c = make_unique<ReceiverMSEHandshakeCommand>(cuid, peer, engine, socketCore);
    //服务端 构建一个ReceiverMSEHandshakeCommand 对象，调用对应的构造函数完成初始化，也即是服务端要初始化对应的捂手对象等，用于和客户端的连接,这里还传递了当前连接创建的peerSocket对象
    engine->addCommand(std::move(c));

    A2_LOG_DEBUG(fmt("CUID#%" PRId64 " service receive  the connection from %s:%u.", cuid,peer->getIPAddress().c_str(), peer->getPort()));
    return 0;
}

下面再说几个细节问题，对应buff的接受，不是直接将内容接受过来，就好了，由于采用了utp 他会将多个数据包同时返回给你，所以对于一些细节就要注意：比如
//utp 接受分发消息
void InitiatorMSEHandshakeCommand::dispatchUtpMessage(const unsigned char *buff,size_t buffLen)
{
    //这里要注意，如果处于当前的这个状态的化，对于原有的tcp接收数据，socket_->readData(rbuf_ + rbufLength_, len); 这里是有最大的长度限制的，如果传递的数据大于当前可以接收的buff
    //会留到下一次再来接受，所以我们采用utp也要有这样的逻辑，要不然会出现的偶现失败  !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
    mseHandshake_->utpRecvDataProcess(buff,buffLen);
}

最后说下Utp的销毁操作,由于BtRegister是跟DownloadEngine共生死的，而我们的UtpContext也是类似的，所以将UtpContext放到了BtRegister中，所以在此处来执行销毁的操作
//增加一个虚构函数，移除utpContext对象
BtRegistry::~BtRegistry()
{
    if(utpContext_)
    {
        //释放之前，先释放我们设置的对象也即是 UtpContextUserData
        UtpContextUserData * userData = static_cast<UtpContextUserData *>(utp_context_get_userdata(utpContext_));
        if(userData)
        {
            delete userData;
            utp_context_set_userdata(utpContext_,NULL);
        }
        utp_destroy(utpContext_);
        utpContext_ = NULL;
        LOGD("BtRegistry 虚构函数执行，清理UtpContext 以及 UtpContextUserData内存");
    }
}
```

