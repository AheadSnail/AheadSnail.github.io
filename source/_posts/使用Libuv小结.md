---
layout: pager
title: 使用Libuv小结
date: 2020-06-29 10:26:32
tags: [bt,libuv]
description:  使用Libuv小结
---

### 概述

> 使用Libuv小结

<!--more-->

### 简介
重写Bt项目在Linux下面已经差不多已经接近尾声，后续在Linux下面测试稳定后，就要移植到Android上面，这篇文章主要介绍重写的使用Libuv中要注意的问题

关于libuv的介绍，可以查看这个链接
https://github.com/libuv/libuv
libuv是一个支持多平台的异步IO库。它主要是为了node.js而开发的，但是也可以用于Luvit, Julia, pyuv及其他软件。 

1.libuv是使用C标准写的
2.tcp的接收新连接是共用一个loop的
3.udp处理完数据的时候，每次返回值都为0
4.发送数据的时候是不提供缓冲区
5.退出的时候如何正确的清理释放资源
6.libuv线程间的通信问题


### libuv是使用C标准写的
原本打算整个项目使用C++11标准使用智能指针的方式来写，这样可以减少内存泄露的问题，而且可以使用C++11中的异常等的新特性，由于libuv使用了C标准，所以导致接口的回调部分都要用C的方式,
虽然尽量使用new,delete来代替malloc,free，但是相比智能指针还是存在很多不安全的地方，而且我们知道在C中是不存在异常的方式，是否执行成功都是通过返回值的方式来代替的，更此次的是有些
地方直接使用assert的方式，一旦assert不通过，就直接程序崩溃


### tcp的接收新连接是共用一个loop的
我们之前项目是使用Aria2的单线程版本强制改成多线程的方式而且加了很多的支持，导致很不稳定，我们重写的目的也就是要达到稳定，所以我们的方案是对于每一个连接都是一个新的线程，不管是tcp
还是udp，每个线程里面都有一个loop，每个线程的变量尽量的做到线程独有，这样就会稳定的多，但是原本libuv的tcp默认accept是共用一个looper的，其实这个问题，官网上也有人提出，下面是修改的方案


```C
//记得在socket不需要后，调用uv_close。如果你不需要接受连接，你甚至可以在uv_listen的回调函数中调用uv_close
void UVSocketCore::tcp_on_connection(uv_stream_t *server, int status) {
	auto socketCore = static_cast<UVSocketCore*>((server->data));
	//新连接到来的回调函数
	if (status < 0) {
		G_LOG_DEBUG("New connection error %s\n", uv_strerror(status));
		//错误回调
		socketCore->onTcpServerConnectionCallback_(nullptr);
		return;
	}
	if(socketCore->acceptConnect(server)){
		//错误回调
		socketCore->onTcpServerConnectionCallback_(nullptr);
	}
}

int UVSocketCore::acceptConnect(uv_stream_t *server){
	//创建一个uv_tcp_t的客户端
	uv_tcp_t *client = new uv_tcp_t;
	//执行初始化
	uv_tcp_init(loop_.get(), client);
	int ret = uv_accept(server, (uv_stream_t*) client);
	if (ret == 0) {
		G_LOG_DEBUG("server_on_new_connection client fd %d",client->io_watcher.fd);

		int fds[2];
		//成功之后，创建sockfd 对
		int rv = uv_get_socketpair(fds,0);
		CHECK_UV_ASSERT(rv);

		//初始化当前的pipet
		uv_pipe_t* pipet = new uv_pipe_t;
		//初始化主线程的main_pipet　句柄
		uv_pipe_init(loop_.get(), pipet, 1);
		//设置当前关联，监听的fd
		uv_pipe_open(pipet, fds[0]);

		//封装要传递的数据
		PipeThreadRunStruct* threadRunData = new PipeThreadRunStruct;
		threadRunData->socketCore = this;
		threadRunData->fd = fds[1];

		//成功之后，再创建线程
		std::thread connectThread(&UVSocketCore::connect_thread_run,this,threadRunData);
		// 分离线程，让线程自己回收资源。
		connectThread.detach();

		PipeWriteStruct * writeData = new PipeWriteStruct;
		//执行发送
		char data[10] = {"a"};
		uv_write_t * req = new uv_write_t;
		uv_buf_t buf = uv_buf_init(data, 1);
		writeData->client = client;
		writeData->pipe = pipet;
		//存储携带的数据
		req->data = writeData;
		uv_write2(req, (uv_stream_t*) pipet, &buf, 1,(uv_stream_t*) client, pipe_write_finish);

		G_LOG_DEBUG("UVSocketCore::acceptConnect clear childThread finish");
	}else{
		G_LOG_DEBUG("uv_accept error %s\n", uv_strerror(ret));
		//释放操作
		uv_close((uv_handle_t *)client,NULL);
		delete(client);
		return SocketAcceptError;
	}
	return SUCCESS;
}

void UVSocketCore::connect_thread_run(void *arg){
	PipeThreadRunStruct* threadData = static_cast<PipeThreadRunStruct*>(arg);
	UVSocketCore* socketCore = threadData->socketCore;
	int fd = threadData->fd;
	//释放
	delete(threadData);

	//改用智能指针
	auto thread_pipe = std::make_shared<uv_pipe_t>();
	auto threadLoop = std::make_shared<uv_loop_t>();
	uv_loop_init(threadLoop.get());
	//执行管道的初始化
	uv_pipe_init(threadLoop.get(), thread_pipe.get(), 1 /* ipc */);
	//打开管道 打开一个已存在的文件描述符或者句柄作为一个管道。
	uv_pipe_open(thread_pipe.get(), fd);

	AcceptUvData acceptObj;
	//默认值
	acceptObj.error_code = SUCCESS;
	acceptObj.client = nullptr;
	acceptObj.current_loop = threadLoop;
	thread_pipe->data = static_cast<void *>(&acceptObj);
	//执行监听操作
	uv_read_start((uv_stream_t*) (thread_pipe.get()), alloc_buffer,pipe_on_new_connection);
	//开始事件循环
	uv_run(threadLoop.get(), UV_RUN_DEFAULT);

	//引擎退出了,我们要判断错误码,是否执行对应的清理操作
	if(acceptObj.error_code){
		//非成功,失败的时候才要清理,正常退出,清理操作
		uv_loop_close(threadLoop.get());
		//错误
		socketCore->onTcpServerConnectionCallback_(nullptr);
	}else{
		//成功
		if (socketCore->onTcpServerConnectionCallback_) {
			socketCore->onTcpServerConnectionCallback_(acceptObj.client);
		}
	}
	G_LOG_DEBUG("UVSocketCore::connect_thread_run finish");
}

//子线程管道收到了数据
void UVSocketCore::pipe_on_new_connection(uv_stream_t *q, ssize_t nread, const uv_buf_t *buf){
	AcceptUvData *pAcceptObj = static_cast<AcceptUvData*>(q->data);
	if (nread < 0) {
		G_LOG_DEBUG("Pipe Read error %s\n", uv_err_name(nread));
		//带回错误码
		pAcceptObj->error_code = LibUvPipeError;
		//错误的化,由于当前只有一个handle,我们直接执行关闭操作,由libuv来执行退出操作
		uv_close((uv_handle_t*) q, NULL);
		delete[](buf->base);
		return;
	}
	//正常读取到数据
	uv_pipe_t *pipe = (uv_pipe_t*) q;
	if (!uv_pipe_pending_count(pipe)) {
		G_LOG_DEBUG("No pending count\n");
		delete[](buf->base);
		return;
	}
	uv_handle_type pending = uv_pipe_pending_type(pipe);
	assert(pending == UV_TCP);
	//创建对应的tcp SocketCore
	auto tcpSocket = std::make_shared<UVSocketCore>(pAcceptObj->current_loop,SOCK_STREAM);
	int result = tcpSocket->acceptPipe(pipe);
	pAcceptObj->error_code = result;
	if(!result){
		//正常执行，我们让loop退出，从而子线程退出，但是当前的loop不能执行销毁操作，要给后面的客户端使用,这里由于 tcpSocket 内部会含有一个handle所以要手动调用uv_stop
		uv_stop(q->loop);
		pAcceptObj->client = tcpSocket;
	}
	//不管成功,还是失败都要调用close 方法,方便引擎
	uv_close((uv_handle_t*) q, NULL);
	delete[](buf->base);
}
int UVSocketCore::acceptPipe(uv_pipe_t* pipe){
	enSureTcpHandle();
	int r = uv_accept((uv_stream_t* )pipe, (uv_stream_t*) tcp_handle_);
	if (r == 0) {
		//之后客户端的fd执行监听操作
		uv_read_start((uv_stream_t *)tcp_handle_, alloc_buffer, tcp_on_read);
		//当前连接的tcp　handle中得到当前连接的地址信息
		struct sockaddr addr = uvTcpGetAddr(tcp_handle_);
		connectAddr_ = *(struct sockaddr_in *) &addr;
		return SUCCESS;
	}
	//失败，执行关闭
	closeTcp();
	return LibUvPipeError;
}

int uv_accept(uv_stream_t* server, uv_stream_t* client) {
  int err;

  assert(server->loop == client->loop);

  if (server->accepted_fd == -1)
    return UV_EAGAIN;

  switch (client->type) {
    case UV_NAMED_PIPE:
    case UV_TCP:
      err = uv__stream_open(client,
                            server->accepted_fd,
                            UV_HANDLE_READABLE | UV_HANDLE_WRITABLE);
      if (err) {
        /* TODO handle error */
        uv__close(server->accepted_fd);
        goto done;
      }
      break;

    case UV_UDP:
      err = uv_udp_open((uv_udp_t*) client, server->accepted_fd);
      if (err) {
        uv__close(server->accepted_fd);
        goto done;
      }
      break;

    default:
      return UV_EINVAL;
  }

  client->flags |= UV_HANDLE_BOUND;
  ...
}
int uv__stream_open(uv_stream_t* stream, int fd, int flags) {
#if defined(__APPLE__)
  int enable;
#endif

  if (!(stream->io_watcher.fd == -1 || stream->io_watcher.fd == fd))
    return UV_EBUSY;

  assert(fd >= 0);
  stream->flags |= flags;

  if (stream->type == UV_TCP) {
    if ((stream->flags & UV_HANDLE_TCP_NODELAY) && uv__tcp_nodelay(fd, 1))
      return UV__ERR(errno);

    /* TODO Use delay the user passed in. */
    if ((stream->flags & UV_HANDLE_TCP_KEEPALIVE) &&
        uv__tcp_keepalive(fd, 1, 60)) {
      return UV__ERR(errno);
    }
  }

#if defined(__APPLE__)
  enable = 1;
  if (setsockopt(fd, SOL_SOCKET, SO_OOBINLINE, &enable, sizeof(enable)) &&
      errno != ENOTSOCK &&
      errno != EINVAL) {
    return UV__ERR(errno);
  }
#endif

  stream->io_watcher.fd = fd;
  return 0;
}


```
大致就是先调用原本的libuv,accept的操作，获取到对应的fd,此时是共用一个loop的，原后通过创建一个管道，管道的一端自己保留，另一端传给另一个线程，另一个线程在自己的线程里面创建
当前线程的loop，然后创建uv_pipe_t 事件，设置关联我们传递进来的管道的另一端，并且设置监听事件，之后，主线程往这个管道写内容，写的内容可以随便，那么这个线程就可以收到这个事件
那么就会触发pipe_on_new_connection 回调，在这个回调函数里面我们利用当前线程的Loop对象 创建一个 UVSocketCore 对象，然后调用acceptPipe完成转换，其实内部也是通过调用uv_accept
方式来完成转换的,而uv_accept 其实也没做什么，其实就是将原本tcp的fd转到我们新的UVSocketCore里面,stream->io_watcher.fd = fd; 就是最根本实现,这里还要注意的一点就是当转换完成之后
我们要让这个线程退出，而loop不能销毁，因为这个loop后续要放到另一个线程里面完成后续的通信，所以我们要调用uv_stop(q->loop);这个内部就是退出事件循环，而不会清理资源,那么下一个线程
可以继续使用这个looper


###  udp处理完数据的时候，每次返回值都为0
```C
//要监听的udp的io事件发生了变化
static void uv__udp_io(uv_loop_t* loop, uv__io_t* w, unsigned int revents) {
  uv_udp_t* handle;

  handle = container_of(w, uv_udp_t, io_watcher);
  assert(handle->type == UV_UDP);

  //如果当前的事件为数据到来,执行数据的接收
  if (revents & POLLIN)
    uv__udp_recvmsg(handle);

  //如果当前的事件为写数据,执行写数据
  if (revents & POLLOUT) {
	//执行发送数据包
    uv__udp_sendmsg(handle);
    //处理已经发送的内容
    uv__udp_run_completed(handle);
  }
}

static void uv__udp_recvmsg(uv_udp_t* handle) {
  struct sockaddr_storage peer;
  struct msghdr h;
  ssize_t nread;
  uv_buf_t buf;
  int flags;
  int count;

  assert(handle->recv_cb != NULL);
  assert(handle->alloc_cb != NULL);

  /* Prevent loop starvation when the data comes in as fast as (or faster than)
   * we can read it. XXX Need to rearm fd if we switch to edge-triggered I/O.
   */
  count = 32;

  do {
    buf = uv_buf_init(NULL, 0);
    handle->alloc_cb((uv_handle_t*) handle, 64 * 1024, &buf);
    if (buf.base == NULL || buf.len == 0) {
      handle->recv_cb(handle, UV_ENOBUFS, &buf, NULL, 0);
      return;
    }
    assert(buf.base != NULL);

    memset(&h, 0, sizeof(h));
    memset(&peer, 0, sizeof(peer));
    h.msg_name = &peer;
    h.msg_namelen = sizeof(peer);
    h.msg_iov = (void*) &buf;
    h.msg_iovlen = 1;

    do {
      nread = recvmsg(handle->io_watcher.fd, &h, 0);
    }
    while (nread == -1 && errno == EINTR);

    if (nread == -1) {
      if (errno == EAGAIN || errno == EWOULDBLOCK)
        handle->recv_cb(handle, 0, &buf, NULL, 0);
      else
        handle->recv_cb(handle, UV__ERR(errno), &buf, NULL, 0);
    }
    else {
      flags = 0;
      if (h.msg_flags & MSG_TRUNC)
        flags |= UV_UDP_PARTIAL;

      handle->recv_cb(handle, nread, &buf, (const struct sockaddr*) &peer, flags);
    }
  }
  /* recv_cb callback may decide to pause or close the handle */
  while (nread != -1
      && count-- > 0
      && handle->io_watcher.fd != -1
      && handle->recv_cb != NULL);
}
```
我们通过 man 2 recvmsg 查看函数的介绍，通过介绍我们可以知道如果当前没有数据可以读取，比如出现了 EAGAIN 或者 EWOULDBLOCK错误，就会返回 -1，而且处理的回调函数返回值为0
handle->recv_cb(handle, 0, &buf, NULL, 0);正常来说，对于recvmsg 返回值为0代表正在关闭
![结果显示](/uploads/libuv优化/recvmsg.png)


### 发送数据的时候是不提供缓冲区
对于这个限制，官网也是有说的，在api中也是有说明，但是比较坑的是tcp的发送是有说明的，但是对于udp的发送函数并没有说明,关于没有提供缓冲区的，我上一篇文章已经有体现

```C
//tcp发送函数
/* The buffers to be written must remain valid until the callback is called.
 * This is not required for the uv_buf_t array.
 */
int uv_write(uv_write_t* req,
             uv_stream_t* handle,
             const uv_buf_t bufs[],
             unsigned int nbufs,
             uv_write_cb cb) {
  return uv_write2(req, handle, bufs, nbufs, NULL, cb);
}

//udp发送函数
int uv_udp_send(uv_udp_send_t* req,
                uv_udp_t* handle,
                const uv_buf_t bufs[],
                unsigned int nbufs,
                const struct sockaddr* addr,
                uv_udp_send_cb send_cb) {
  int addrlen;

  addrlen = uv__udp_check_before_send(handle, addr);
  if (addrlen < 0)
    return addrlen;

  return uv__udp_send(req, handle, bufs, nbufs, addr, addrlen, send_cb);
}
```
这里主要大致说一下对于tcp和udp的差异，tcp其实是一个发送完再发送一个的，而udp可以做到统一集中发送，下面是源码的体现
```C
tcp的发送函数
static void uv__write(uv_stream_t* stream) {
  
  ...
  
  if (QUEUE_EMPTY(&stream->write_queue))
    return;

  q = QUEUE_HEAD(&stream->write_queue);
  req = QUEUE_DATA(q, uv_write_t, queue);
  assert(req->handle == stream);

  /*
   * Cast to iovec. We had to have our own uv_buf_t instead of iovec
   * because Windows's WSABUF is not an iovec.
   */
  assert(sizeof(uv_buf_t) == sizeof(struct iovec));
  iov = (struct iovec*) &(req->bufs[req->write_index]);
  iovcnt = req->nbufs - req->write_index;

  iovmax = uv__getiovmax();
 
  ...
  
  do
    n = uv__writev(uv__stream_fd(stream), iov, iovcnt);
  while (n == -1 && RETRY_ON_WRITE_ERROR(errno));
  

  if (n == -1 && !IS_TRANSIENT_WRITE_ERROR(errno, req->send_handle)) {
    err = UV__ERR(errno);
    goto error;
  }

  if (n >= 0 && uv__write_req_update(stream, req, n)) {
    uv__write_req_finish(req);
    return;  /* TODO(bnoordhuis) Start trying to write the next request. */
  }
  
  ...
}

//使用udp发送数据
static void uv__udp_sendmsg(uv_udp_t* handle) {
  uv_udp_send_t* req;
  QUEUE* q;
  struct msghdr h;
  ssize_t size;

  //一次就将当前handle的 write_queue 队列的内容,执行发送
  while (!QUEUE_EMPTY(&handle->write_queue)) {
	//获取到队列的首元素
    q = QUEUE_HEAD(&handle->write_queue);
    assert(q != NULL);

    req = QUEUE_DATA(q, uv_udp_send_t, queue);
    assert(req != NULL);

    //往下走的就是都要发送的内容
    memset(&h, 0, sizeof h);
    if (req->addr.ss_family == AF_UNSPEC) {
      h.msg_name = NULL;
      h.msg_namelen = 0;
    } else {
      h.msg_name = &req->addr;
      if (req->addr.ss_family == AF_INET6)
        h.msg_namelen = sizeof(struct sockaddr_in6);
      else if (req->addr.ss_family == AF_INET)
        h.msg_namelen = sizeof(struct sockaddr_in);
      else if (req->addr.ss_family == AF_UNIX)
        h.msg_namelen = sizeof(struct sockaddr_un);
      else {
        assert(0 && "unsupported address family");
        abort();
      }
    }
    //要发送数据的赋值
    h.msg_iov = (struct iovec*) req->bufs;
    h.msg_iovlen = req->nbufs;

    //执行发送
    do {
      size = sendmsg(handle->io_watcher.fd, &h, 0);
    } while (size == -1 && errno == EINTR);


    if (size == -1) {
      //如果当前是写缓冲区满了,直接break掉,下次触发
      if (errno == EAGAIN || errno == EWOULDBLOCK || errno == ENOBUFS)
        break;
    }

    req->status = (size == -1 ? UV__ERR(errno) : size);

    /* Sending a datagram is an atomic operation: either all data
     * is written or nothing is (and EMSGSIZE is raised). That is
     * why we don't handle partial writes. Just pop the request
     * off the write queue and onto the completed queue, done.
     */
    //发送数据报是一项原子操作：要么所有数据被写或什么都不写（并且EMSGSIZE被引发）。那是为什么我们不处理部分写入。只是弹出请求从写队列移到完成队列，完成。
    QUEUE_REMOVE(&req->queue);
    QUEUE_INSERT_TAIL(&handle->write_completed_queue, &req->queue);
    uv__io_feed(handle->loop, &handle->io_watcher);
  }
}

可以发现tcp是一个发完就处理完成的回调，而udp是一个循环，直到发到缓冲区满才停止发送
```

### 退出的时候如何正确的清理释放资源
其实关于libuv的文档已经官方的demo，甚至网上的资料都非常的少，甚至官方的demo写的都很不规范，比如一大堆的资源没有清理，内存没有释放，我发现这个问题还是在模拟android上面频繁的
点击暂停恢复的时候出现的，下面是对应的代码模拟
```C
while(true){
#ifdef CLIENT
	sleep(1);
	BtManager::getInstance()->pauseDownload(model);
	sleep(1);
	BtManager::getInstance()->resumeDownload(model);
#else
	sleep(1);
#endif
}
```
下面出现的错误，
![结果显示](/uploads/libuv优化/文件描述符.png)
要想查看linux下面能支持的最大文件描述符的数量，可以执行 ulimit -n 的命令输出，我当前的电脑是显示为1024个，出现了这个问题说明肯定是资源释放的问题，要想查看当前进程打开的文件描述符
可以使用 lsof -p 进程号 得到当前进程的所有打开的文件描述符，通过查看发现是libuv的释放有问题,libuv的释放函数为 uv_loop_close

```C
int uv_loop_close(uv_loop_t* loop) {
  QUEUE* q;
  uv_handle_t* h;
#ifndef NDEBUG
  void* saved_data;
#endif

  if (uv__has_active_reqs(loop)){
	  //printf("uv_loop_close uv__has_active_reqs \n");
	  return UV_EBUSY;
  }

  QUEUE_FOREACH(q, &loop->handle_queue) {
    h = QUEUE_DATA(q, uv_handle_t, handle_queue);
    if (!(h->flags & UV_HANDLE_INTERNAL)){
    	 //printf("uv_loop_close h->flags & UV_HANDLE_INTERNAL \n");
    	 return UV_EBUSY;
    }
  }

  uv__loop_close(loop);

#ifndef NDEBUG
  saved_data = loop->data;
  memset(loop, -1, sizeof(*loop));
  loop->data = saved_data;
#endif
  if (loop == default_loop_ptr)
    default_loop_ptr = NULL;

  //printf("uv_loop_close close finish \n");
  return 0;
}

void uv__loop_close(uv_loop_t* loop) {
  uv__signal_loop_cleanup(loop);
  uv__platform_loop_delete(loop);
  uv__async_stop(loop);

  if (loop->emfile_fd != -1) {
    uv__close(loop->emfile_fd);
    loop->emfile_fd = -1;
  }

  if (loop->backend_fd != -1) {
    uv__close(loop->backend_fd);
    loop->backend_fd = -1;
  }

  uv_mutex_lock(&loop->wq_mutex);
  assert(QUEUE_EMPTY(&loop->wq) && "thread pool work queue not empty!");
  assert(!uv__has_active_reqs(loop));
  uv_mutex_unlock(&loop->wq_mutex);
  uv_mutex_destroy(&loop->wq_mutex);

  /*
   * Note that all thread pool stuff is finished at this point and
   * it is safe to just destroy rw lock
   */
  uv_rwlock_destroy(&loop->cloexec_lock);

#if 0
  assert(QUEUE_EMPTY(&loop->pending_queue));
  assert(QUEUE_EMPTY(&loop->watcher_queue));
  assert(loop->nfds == 0);
#endif

  uv__free(loop->watchers);
  loop->watchers = NULL;
  loop->nwatchers = 0;
}
```
通过上面可以发现释放最重要的函数为 uv__loop_close 函数，在之前还有俩个判断，如果满足了前面俩个判断就不会执行到uv__loop_close，前面俩个判断的意思是如果当前含有正在执行的request，或者
还有正在执行的Handle数量不为0，那就会执行return，导致这次的释放操作，并没有执行到，故导致资源的泄露，还有如果是下面的这样写也会有问题
```C
uv_close((uv_handle_t*) time_handle_.get(), NULL);
uv_loop_close(uvSocketCore_->getUvLoop().get());

在调用uv_close方法的时候，并不会立马将当前的handle移除出去，而是要等到下一次的轮询才会移除，下面是代码的实现
void uv_close(uv_handle_t* handle, uv_close_cb close_cb) {
  assert(!uv__is_closing(handle));

  handle->flags |= UV_HANDLE_CLOSING;
  handle->close_cb = close_cb;

  switch (handle->type) {
  case UV_NAMED_PIPE:
    uv__pipe_close((uv_pipe_t*)handle);
    break;

  case UV_TTY:
    uv__stream_close((uv_stream_t*)handle);
    break;

  case UV_TCP:
    uv__tcp_close((uv_tcp_t*)handle);
    break;

  case UV_UDP:
    uv__udp_close((uv_udp_t*)handle);
    break;

  case UV_PREPARE:
    uv__prepare_close((uv_prepare_t*)handle);
    break;

  case UV_CHECK:
    uv__check_close((uv_check_t*)handle);
    break;

  case UV_IDLE:
    uv__idle_close((uv_idle_t*)handle);
    break;

  case UV_ASYNC:
    uv__async_close((uv_async_t*)handle);
    break;

  case UV_TIMER:
    uv__timer_close((uv_timer_t*)handle);
    break;

  case UV_PROCESS:
    uv__process_close((uv_process_t*)handle);
    break;

  case UV_FS_EVENT:
    uv__fs_event_close((uv_fs_event_t*)handle);
    break;

  case UV_POLL:
    uv__poll_close((uv_poll_t*)handle);
    break;

  case UV_FS_POLL:
    uv__fs_poll_close((uv_fs_poll_t*)handle);
    /* Poll handles use file system requests, and one of them may still be
     * running. The poll code will call uv__make_close_pending() for us. */
    return;

  case UV_SIGNAL:
    uv__signal_close((uv_signal_t*) handle);
    /* Signal handles may not be closed immediately. The signal code will
     * itself close uv__make_close_pending whenever appropriate. */
    return;

  default:
    assert(0);
  }

  uv__make_close_pending(handle);
}

void uv__make_close_pending(uv_handle_t* handle) {
  assert(handle->flags & UV_HANDLE_CLOSING);
  assert(!(handle->flags & UV_HANDLE_CLOSED));
  handle->next_closing = handle->loop->closing_handles;
  handle->loop->closing_handles = handle;
}
```
上面的代码可以发现最终要的是uv__make_close_pending 函数，内部会将当前的handle放到了当前loop的closing_handles队列中,那这个队列什么时候才会处理
```C
static void uv__run_closing_handles(uv_loop_t* loop) {
  uv_handle_t* p;
  uv_handle_t* q;

  p = loop->closing_handles;
  loop->closing_handles = NULL;

  while (p) {
    q = p->next_closing;
    uv__finish_close(p);
    p = q;
  }
}
static void uv__finish_close(uv_handle_t* handle) {
  assert(handle->flags & UV_HANDLE_CLOSING);
  assert(!(handle->flags & UV_HANDLE_CLOSED));
  handle->flags |= UV_HANDLE_CLOSED;

  switch (handle->type) {
    case UV_PREPARE:
    case UV_CHECK:
    case UV_IDLE:
    case UV_ASYNC:
    case UV_TIMER:
    case UV_PROCESS:
    case UV_FS_EVENT:
    case UV_FS_POLL:
    case UV_POLL:
    case UV_SIGNAL:
      break;

    case UV_NAMED_PIPE:
    case UV_TCP:
    case UV_TTY:
      uv__stream_destroy((uv_stream_t*)handle);
      break;

    case UV_UDP:
      uv__udp_finish_close((uv_udp_t*)handle);
      break;

    default:
      assert(0);
      break;
  }

  uv__handle_unref(handle);
  QUEUE_REMOVE(&handle->handle_queue);

  if (handle->close_cb) {
    handle->close_cb(handle);
  }
}

int uv_run(uv_loop_t* loop, uv_run_mode mode) {
  int timeout;
  int r;
  int ran_pending;

  r = uv__loop_alive(loop);
  if (!r)
    uv__update_time(loop);

  while (r != 0 && loop->stop_flag == 0) {
    uv__update_time(loop);
    uv__run_timers(loop);
    ran_pending = uv__run_pending(loop);
    uv__run_idle(loop);
    uv__run_prepare(loop);

    timeout = 0;
    if ((mode == UV_RUN_ONCE && !ran_pending) || mode == UV_RUN_DEFAULT)
      timeout = uv_backend_timeout(loop);

    uv__io_poll(loop, timeout);
    uv__run_check(loop);
    uv__run_closing_handles(loop);

    if (mode == UV_RUN_ONCE) {
      /* UV_RUN_ONCE implies forward progress: at least one callback must have
       * been invoked when it returns. uv__io_poll() can return without doing
       * I/O (meaning: no callbacks) when its timeout expires - which means we
       * have pending timers that satisfy the forward progress constraint.
       *
       * UV_RUN_NOWAIT makes no guarantees about progress so it's omitted from
       * the check.
       */
      uv__update_time(loop);
      uv__run_timers(loop);
    }

    r = uv__loop_alive(loop);
    if (mode == UV_RUN_ONCE || mode == UV_RUN_NOWAIT)
      break;
  }

  /* The if statement lets gcc compile it to a conditional store. Avoids
   * dirtying a cache line.
   */
  if (loop->stop_flag != 0)
    loop->stop_flag = 0;

  return r;
}
```
其实整个libuv就是个无限循环，内部使用了epoll来监听对应的事件，而我们的关闭操作的处理是在uv__run_closing_handles函数触发的,在这个函数里面执行了 QUEUE_REMOVE(&handle->handle_queue);
才移除出去，所以如果你在调用了stop方法后立马调用close方法也是不会清理的，也即是一定要等到他自动的终止，而不能有任何的干扰操作,这样写也是有目的，因为像tcp或者udp是支持异步写操作的
用户提供的写请求，如果当前不能写的话，是会立马返回，libuv内部通过监听可写的事件，当可写事件再次触发，他内部会自动的帮你发送出去，所以内部难免会积累一下要写的内容，所以当你要退出的
时候，就要完成这些的清理操作，防止内存泄露，比如udp的关闭操作,内部在移除handle之前会先调用下面的函数，将当前的写请求都释放，回调回去
```C
void uv__udp_finish_close(uv_udp_t* handle) {
  uv_udp_send_t* req;
  QUEUE* q;

  assert(!uv__io_active(&handle->io_watcher, POLLIN | POLLOUT));
  assert(handle->io_watcher.fd == -1);

  while (!QUEUE_EMPTY(&handle->write_queue)) {
    q = QUEUE_HEAD(&handle->write_queue);
    QUEUE_REMOVE(q);

    req = QUEUE_DATA(q, uv_udp_send_t, queue);
    req->status = UV_ECANCELED;
    QUEUE_INSERT_TAIL(&handle->write_completed_queue, &req->queue);
  }

  uv__udp_run_completed(handle);

  assert(handle->send_queue_size == 0);
  assert(handle->send_queue_count == 0);

  /* Now tear down the handle. */
  handle->recv_cb = NULL;
  handle->alloc_cb = NULL;
  /* but _do not_ touch close_cb */
}
```
根据前面的分析，既然我们不能中断libuv的退出操作，那么程序中就不能使用异常，其实我们在线程中使用libuv，其实我们的代码都是跑在libuv的回调事件中的，如果我们在它的回调里面抛出了一个异常
那么就不能按照libuv的原本的逻辑执行退出会导致更严重的问题，可能你会想到发生异常退出的时候，后续再次执行uv_run函数，让他自动的终止，不好意思，这是不可能实现的，下面是分析
比如udp当数据发送完成之后，会执行uv__udp_run_completed函数，在这个函数的一开始就会执行  assert(!(handle->flags & UV_HANDLE_UDP_PROCESSING));操作，然后设置这个flag
handle->flags |= UV_HANDLE_UDP_PROCESSING;，在程序的后面执行  handle->flags &= ~UV_HANDLE_UDP_PROCESSING;将这个变量重置回去,
假设 我们在执行   req->send_cb(req, req->status);回调函数里面抛出了异常，导致没有重置会这个flag，那么下次重新进来的时候，就会直接导致闪退，因为 assert(!(handle->flags & UV_HANDLE_UDP_PROCESSING));
```C
static void uv__udp_run_completed(uv_udp_t* handle) {
  uv_udp_send_t* req;
  QUEUE* q;

  //标识当前正在处理结果的队列
  assert(!(handle->flags & UV_HANDLE_UDP_PROCESSING));
  handle->flags |= UV_HANDLE_UDP_PROCESSING;

  while (!QUEUE_EMPTY(&handle->write_completed_queue)) {
    q = QUEUE_HEAD(&handle->write_completed_queue);
    QUEUE_REMOVE(q);

    req = QUEUE_DATA(q, uv_udp_send_t, queue);

    //从handle中移除当前的req
    uv__req_unregister(handle->loop, req);

    handle->send_queue_size -= uv__count_bufs(req->bufs, req->nbufs);
    handle->send_queue_count--;

    //正常的udp请求
    if (req->bufs != req->bufsml)
      uv__free(req->bufs);
    req->bufs = NULL;

    if (req->send_cb == NULL)
      continue;

    /* req->status >= 0 == bytes written
     * req->status <  0 == errno
     */
    if (req->status >= 0)
      req->send_cb(req, 0);
    else
      req->send_cb(req, req->status);
  }

  //如果当前handle 要发送的数据队列为空,我们停止写事件的监听
  if (QUEUE_EMPTY(&handle->write_queue)) {
    /* Pending queue and completion queue empty, stop watcher. */
    uv__io_stop(handle->loop, &handle->io_watcher, POLLOUT);
    if (!uv__io_active(&handle->io_watcher, POLLIN))
      uv__handle_stop(handle);
  }

  //重置当前没有正在处理完成队列
  handle->flags &= ~UV_HANDLE_UDP_PROCESSING;
}
```
所以最后面，我们的项目由原本使用异常的项目变成了一个无异常的项目，代替的实现就是通过返回值的方式来实现，在知乎各种关于C++是否使用异常好像存在很大的争议，我的观点就是使用异常代码简单清晰
如果使用返回值，代码将变得冗余，啰嗦，比如在多层函数里面，最里面的那层函数发生了错误，你就得一层一层的返回，每一层都要检查上一层的函数调用是否出现了问题，不管怎么样，项目里面去掉了
异常，也做到了libuv的正常清理操作


### libuv线程间的通信问题
在看libuv源码的时候，发现了他内部一种线程间的通信机制，使用了uv_async_t，下面是代码的实现
```C
void testThread(){
	//初始化默认的loop对象
	uv_loop_t * loop = uv_loop_new();
	uv_async_init(loop, &asyncMain, main_sync_callback);
	uv_thread_t hare_id;
	uv_thread_t tortoise_id;
	uv_thread_create(&tortoise_id, thradA, NULL);
	//创建两个线程,第三个参数是传递的参数
	uv_thread_create(&hare_id, threadB, NULL);
	uv_run(loop, UV_RUN_DEFAULT);
}

//主线程收到了事件的回调函数
void main_sync_callback(uv_async_t *handle) {
	struct sendData* senddata = (struct sendData*) handle->data;
	printf("mainThread callback recv %s send data \n", senddata->data);
	strcpy(senddata->data,"MMMM");

	senddata->async->data = senddata;
	//发送给发送方
	uv_async_send(senddata->async);
}

//第二个线程执行的函数
void thradA(void *arg) {
	//初始化默认的loop对象
	uv_loop_t * loop = uv_loop_new();
	uv_async_init(loop, &asyncA, threadA_sync_callback);
	//设置一个定时器，用于后续的发送事件到另一个线程
	uv_timer_t timer_req;
	uv_timer_init(loop, &timer_req);
	uv_timer_start(&timer_req, threadA_time_callback, 2000, 1000);
	uv_run(loop, UV_RUN_DEFAULT);
}

//线程Ａ　时间到了的回调函数
void threadA_time_callback(uv_timer_t* handle){
	printf("ThreadA time_callback send data \n");
	struct sendData* data = (struct sendData *)malloc(sizeof(struct sendData));
	data->data  = (char *)malloc(20);
	memset(data->data,0,20);
	strcpy(data->data,"AAAA");
	//将当前的Sync对象发给对方
	data->async = &asyncA;
	asyncMain.data = data;
	uv_async_send(&asyncMain);
	//关闭定时器
	uv_close((uv_handle_t*)handle,NULL);
}

//主线程收到了事件的回调函数
void threadA_sync_callback(uv_async_t *handle) {
	struct sendData* senddata = (struct sendData*) handle->data;
	printf("ThreadA callback recv %s send data \n", senddata->data);
	strcpy(senddata->data,"AAAA");
	senddata->async->data = senddata;
	//发送给发送方
	uv_async_send(senddata->async);
}

也即是通过拿到对方的 uv_async_t 对象使用uv_async_send方法，通知对方，对方在回调函数里面处理发送的内容,我们先分析下代码的实现

int uv_async_send(uv_async_t* handle) {
  /* Do a cheap read first. */
  if (ACCESS_ONCE(int, handle->pending) != 0)
    return 0;

  /* Tell the other thread we're busy with the handle. */
  if (cmpxchgi(&handle->pending, 0, 1) != 0)
    return 0;

  /* Wake up the other thread's event loop. */
  uv__async_send(handle->loop);

  /* Tell the other thread we're done. */
  if (cmpxchgi(&handle->pending, 1, 2) != 1)
    abort();

  return 0;
}

真正的发送函数其实是在   uv__async_send(handle->loop); 内部其实就是使用了管道，写一个字节，另一端在监听这个管道，所以能收到这个事件
static void uv__async_send(uv_loop_t* loop) {
  const void* buf;
  ssize_t len;
  int fd;
  int r;

  buf = "";
  len = 1;
  fd = loop->async_wfd;

#if defined(__linux__)
  if (fd == -1) {
    static const uint64_t val = 1;
    buf = &val;
    len = sizeof(val);
    fd = loop->async_io_watcher.fd;  /* eventfd */
  }
#endif

  do
    r = write(fd, buf, len);
  while (r == -1 && errno == EINTR);

  if (r == len)
    return;

  if (r == -1)
    if (errno == EAGAIN || errno == EWOULDBLOCK)
      return;

  abort();
}
这里先假设第一次的事件发送出去了，我们看看他的处理函数

static void uv__async_io(uv_loop_t* loop, uv__io_t* w, unsigned int events) {
  char buf[1024];
  ssize_t r;
  QUEUE queue;
  QUEUE* q;
  uv_async_t* h;

  assert(w == &loop->async_io_watcher);

  for (;;) {
    r = read(w->fd, buf, sizeof(buf));

    if (r == sizeof(buf))
      continue;

    if (r != -1)
      break;

    if (errno == EAGAIN || errno == EWOULDBLOCK)
      break;

    if (errno == EINTR)
      continue;

    abort();
  }

  QUEUE_MOVE(&loop->async_handles, &queue);
  while (!QUEUE_EMPTY(&queue)) {
    q = QUEUE_HEAD(&queue);
    h = QUEUE_DATA(q, uv_async_t, queue);

    QUEUE_REMOVE(q);
    QUEUE_INSERT_TAIL(&loop->async_handles, q);

    if (0 == uv__async_spin(h))
      continue;  /* Not pending. */

    if (h->async_cb == NULL)
      continue;

    h->async_cb(h);
  }
}
大致就是这里只接收一个字节的内容，因为发送的时候，其实也是发送了一个字节，所以这里就接收一个字节，之后调用了async_cb 回调我们，处理完成之后有一个这样的函数(0 == uv__async_spin(h)）

/* Only call this from the event loop thread. */
static int uv__async_spin(uv_async_t* handle) {
  int rc;

  for (;;) {
    /* rc=0 -- handle is not pending.
     * rc=1 -- handle is pending, other thread is still working with it.
     * rc=2 -- handle is pending, other thread is done.
     */
    rc = cmpxchgi(&handle->pending, 2, 0);

    if (rc != 1)
      return rc;

    /* Other thread is busy with this handle, spin until it's done. */
    cpu_relax();
  }
}

#define ACCESS_ONCE(type, var)                                                
  (*(volatile type*) &(var))

UV_UNUSED(static int cmpxchgi(int* ptr, int oldval, int newval)) {
#if defined(__i386__) || defined(__x86_64__)
  int out;
  __asm__ __volatile__ ("lock; cmpxchg %2, %1;"
                        : "=a" (out), "+m" (*(volatile int*) ptr)
                        : "r" (newval), "0" (oldval)
                        : "memory");
  return out;
#elif defined(__MVS__)
  unsigned int op4;
  if (__plo_CSST(ptr, (unsigned int*) &oldval, newval,
                (unsigned int*) ptr, *ptr, &op4))
    return oldval;
  else
    return op4;
#elif defined(__SUNPRO_C) || defined(__SUNPRO_CC)
  return atomic_cas_uint((uint_t *)ptr, (uint_t)oldval, (uint_t)newval);
#else
  return __sync_val_compare_and_swap(ptr, oldval, newval);
#endif
}

其实上面的这几个函数都是为了保证一次的有效传输，他是怎么保证的呢，首先使用了 volatile大体的检查pending 是否为0，然后使用cmpxchgi也即是使用cpu的加锁指令，这其实就是java的自旋锁
的实现，将这个值改为1，代表当前正在占用，然后执行发送函数，发送完之后，再次调用cmpxchgi将他改为2代表我们处理完成了，那什么时候重置回去呢，也即当这次处理完的时候，
调用了uv__async_spin 函数，这个函数的内部使用了无限循环保证后续不能再次进入，当它退出的时候，比如为0，或者2的时候就可以退出，代表当前这次传输处理完成，
那么下次就可以继续发送了，所以就导致了一个问题，如果有俩个线程同时往一个线程的 发送数据的时候，就会导致同时只能处理一个，而另一个直接被抛弃了，导致消息的丢失
```




























