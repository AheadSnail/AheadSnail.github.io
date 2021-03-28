---
layout: pager
title: Bt项目总结
date: 2020-08-24 20:48:53
tags: [bt]
description:  Bt项目总结
---

### 简介
Bt项目已经完成的差不多了，后续的功能也陆陆续续的加上去了，整体是不会有多大的变化，趁此来总结下，以免后续忘了，先来看看整体的方案
![结果显示](/uploads/Bt项目总结/Bt项目总结.png)

### 整体思路
整体的思路是采用One Thread One Loop + ThreadPool来实现的，貌似也是目前主流的C++服务端高性能的开发模式，下面依次来介绍下这些关键的类

GlobalListener
> 完成全局的Bt端口的监听，通过上面的图可以知道，当前类主要完成跟当前Bt端口有关的操作，比如发起连接，响应连接，由于我们想做到类似Tcp的负载均衡，我们使用了SO_REUSEADDR、SO_REUSEPORT,来实现Udp的负载均衡，具体的原理可以查看下面的这篇文章
https://cloud.tencent.com/developer/article/1004555
为了保证正确性，我们就要确保对于当前Bt端口的 connect，bind操作都要在当前监听线程中完成，包括GlobalTracker中用来通信的SocketCore，也要在当前线程中完成bind，connect操作，这俩部操作完成之后，后续的通信就可以转发到你当前的这个SocketCore了，而不会由GlobalListener收到,从而可以实现负载均衡，由于是通过四元组确定来由内核来维护这种关系的话，如果对于一个用户多个资源的下载，会分配到同一个Socket的问题，因为四元组是一样的，内核会直接使用同一个Socket来处理，为了杜绝一个文件描述符在多个线程中使用的关系，我们自己维护了一个集合来保证同一时刻只能有一个资源下载,当执行完连接connect之后，四元组确定下来了，就可以交给线程池来处理了，后续的通信就实现了分离了

GlobalTracker
> 主要管理当前下载资源的Tracker，比如定时的连接tracker服务器，响应tracker服务器返回的peer节点，解析到对应的实体中，后续通过发送一条消息给GlobalListener，由GlobalListener来发起主动的连接操作,响应服务端的主动打洞的逻辑，暂停，恢复时等操作

BtManager 
> 管理全局的资源下载，单例存在，主要通过维护一个下载实体的集合，响应用户的操作 添加下载，暂停下载，恢复下载，删除下载等的操作，还有一些类似全局的东西，比如Dns缓存，Cookie缓存,用来保证同一时刻只能同一个四元组只有一个连接存在，全局的网络监测等

DownloadModel 
> 主要用来管理当前资源的下载状态，状态发生改变的时候，回调通知对应的实体，为了方便后续对于当前资源的任务管理，维护了一个队列 std::deque<DownloadRequest*> runningRequest_;为了快速的响应用户的暂停，恢复操作，每个下载请求DownloadRequest维护一个 std::atomic<bool> isCancelled_; 状态，当用户暂停的时候，将队列的每个请求，设置为cancel状态，DownloadRequest内部会尽快的响应这个取消的状态，然后清空runningRequest_，为了下一次下载请求的收集，对于恢复的操作，我们就重新创建对应的DownloadRequest，从而实现上一个操作跟当前的操作的无关性，也可以方便的快速的响应用户的暂停，恢复

HttpHeader 
> 为了方便管理HttpRunnable，内部会等待收集HttpRunnable的错误信息，比如下载劫持，412，404，HttpDns切换，等各种错误实现统一的处理，HttpHeader完成速度的统计，下载进度的保存下载错误等的统一处理

TrafficManager
> 主要用来统计当前的下载情况的管理，针对每个资源，每隔一段时间，或者用户触发操作的时候，上传统计的情况,比如Bt下载了多少，http下载了多少，用户共享了多少，从Cdn节点下载了多少，为线程独有，使用了多生产者单消费者模式,

LogManager
> 主要为了记录当前下载的打点记录，也是线程独有的，也是采用多消费者单生产者模式


上面就是一些主要的类，最核心的还是要解决Udp的负载均衡，如果这个可以解决的话，就可以将下载按用户做到线程的隔离，开发的初衷也就是为了尽量的减少加锁的操作，从而减少开发的难度，当前需要共享的对象主要有全局的片管理对象(BitfieldMan) 用来记录当前哪些片下载完，哪些片正在下载，哪些片是可以使用的，DefaultPieceStorage完成片的获取，管理，记录哪些片正在下载，还有Bt需要共享的对象DefaultPeerStorage完成对Peer的管理，当前哪些Peer正在连接，哪些Peer已经连接过，下一个要连接的Peer是哪个，其他的对象就是线程独有的了，比如写文件对象，socketfd等,下面来看看高性能的Muduo网络库跟我们的差别

### Muduo 网络库
> 这几天看了本 Linux多线程服务端编程：使用muduo C++网络库》是C++高性能编程 陈硕大牛写的关于其写的开源库 muduo 的介绍,下面是github网站的地址
https://github.com/chenshuo/muduo
有点可惜的是写完了才来看，好在整体的思路没有多大的变化，比如 他也是推荐 One Thread One Loop + ThreadPool来实现的,只不过对于Loop来说他是自己使用Epoll/Poll来实现的，而我是通过直接使用libuv 来实现的，像锁方面，使用的是互斥锁，C++对于重入锁当做是一种设计的错误等

下面主要来介绍下不同的点
1.线程间通信
> 不管是libuv还是muduo都有类似的线程间通信，不过libuv的线程间通信会有一个问题，他为了实现同一时刻只能处理一个任务，而且这个任务在处理的时候，不会被打扰中断，内部会抛弃同时触发的其他的任务，内部是使用pipe管道来实现的，在项目中我是通过任务队列加锁的方式来实现的，然后由当前线程的定时器来触发检查这个任务队列，也可以做到将当前的操作转到当前线程来执行，只是可能会存在一定的延时，毕竟是由定时器来触发的，而 muduo的线程间通信

```java
首先也是定义一个集合用来保存当前需要在当前线程中触发的操作
typedef std::function<void()> Functor;
std::vector<Functor> pendingFunctors_;

但是他为了更快的触发响应，通过eventfd来做到快速响应的，也即是一上来就检测创建的eventfd的读写事件
int createEventfd()
{
  // eventfd是Linux 2.6提供的一种系统调用，它可以用来实现事件通知。eventfd包含一个由内核维护的64位无符号整型计数器，创建eventfd时会返回一个文件描述符，
  //进程可以通过对这个文件描述符进行read/write来读取/改变计数器的值，从而实现进程间通信。
  int evtfd = ::eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC);
  if (evtfd < 0)
  {
    LOG_SYSERR << "Failed in eventfd";
    abort();
  }
  return evtfd;
}

下面看看他是怎么做的
void EventLoop::queueInLoop(Functor cb)
{
  {
  MutexLockGuard lock(mutex_);
  pendingFunctors_.push_back(std::move(cb));
  }

  //如果当前不在环境不在Loop的线程,或者当前在Loop的线程，但是已经执行完了 PendingFunctors集合中的function了,执行唤醒操作
  //要不然，你添加的这个回调函数，可能没有机会执行，因为没有事件的发生，既然要有事件的发生就要通过往 eventfd 中写内容，那么就会很快的响应了
  if (!isInLoopThread() || callingPendingFunctors_)
  {
    wakeup();
  }
}

/**
 * 如果当前不在Loop的线程，执行唤醒
 */
void EventLoop::wakeup()
{
  uint64_t one = 1;
  ssize_t n = sockets::write(wakeupFd_, &one, sizeof one);
  if (n != sizeof one)
  {
    LOG_ERROR << "EventLoop::wakeup() writes " << n << " bytes instead of 8";
  }
}

//下面看看Loop的实现
void EventLoop::loop()
{
  assert(!looping_);
  assertInLoopThread();
  looping_ = true;
  quit_ = false;  // FIXME: what if someone calls quit() before loop() ?
  LOG_TRACE << "EventLoop " << this << " start looping";

  while (!quit_)
  {
	//执行poll事件， activeChannels_ 用于接收 有事件改变的 Channel集合
    activeChannels_.clear();
    pollReturnTime_ = poller_->poll(kPollTimeMs, &activeChannels_);
    ++iteration_;
    if (Logger::logLevel() <= Logger::TRACE)
    {
      printActiveChannels();
    }

    // TODO sort channel by priority  将改变的事件，依次执行响应处理
    eventHandling_ = true;
    for (Channel* channel : activeChannels_)
    {
      currentActiveChannel_ = channel;
      currentActiveChannel_->handleEvent(pollReturnTime_);
    }

    //标识当前没有正在执行 poll的事件响应
    currentActiveChannel_ = NULL;
    eventHandling_ = false;

    //执行需要在Loop线程中执行的Function列表
    doPendingFunctors();
  }

  LOG_TRACE << "EventLoop " << this << " stop looping";
  looping_ = false;
}
```
> 所以相比我们当前的做法来说，muduo做的更加优雅，如果我们想实现这个功能，就可以使用libuv原本的线程间通信，然后加上当前的任务队列也可以实现一样的效果，这样就可以巧妙的使用这个操作比如多个线程都往当前线程的任务队列添加任务，然后每个线程都通过这个libuv线程间通信，虽然只能同一时刻处理一个请求，但是没关系，反正可以确保会有一个操作会触发，之后就转到了当前的线程就可以执行之前多个线程添加的任务队列了，或许这才是libuv线程间通信的正确使用方式

2.定时器
```java
接下来看看libuv中的定时器跟muduo定时器的差别，先看libuv的实现，下面是关键的插入时间的实现

HEAP_EXPORT(void heap_insert(struct heap* heap,
                             struct heap_node* newnode,
                             heap_compare_fn less_than)) {
  struct heap_node** parent;
  struct heap_node** child;
  unsigned int path;
  unsigned int n;
  unsigned int k;

  newnode->left = NULL;
  newnode->right = NULL;
  newnode->parent = NULL;

  /* Calculate the path from the root to the insertion point.  This is a min
   * heap so we always insert at the left-most free node of the bottom row.
   */
  path = 0;
  for (k = 0, n = 1 + heap->nelts; n >= 2; k += 1, n /= 2)
    path = (path << 1) | (n & 1);

  /* Now traverse the heap using the path we calculated in the previous step. */
  parent = child = &heap->min;
  while (k > 0) {
    parent = child;
    if (path & 1)
      child = &(*child)->right;
    else
      child = &(*child)->left;
    path >>= 1;
    k -= 1;
  }

  /* Insert the new node. */
  newnode->parent = *parent;
  *child = newnode;
  heap->nelts += 1;

  /* Walk up the tree and check at each node if the heap property holds.
   * It's a min heap so parent < child must be true.
   */
  while (newnode->parent != NULL && less_than(newnode, newnode->parent))
    heap_node_swap(heap, newnode->parent, newnode);
}
可以看出他是根据定时器触发的时间来维护一个小顶堆,堆顶的元素就是当前要最先触发的时间,之后Loop每次触发的时候都会检测堆顶的元素，判断是否到了执行的时间
void uv__run_timers(uv_loop_t* loop) {
  struct heap_node* heap_node;
  uv_timer_t* handle;

  for (;;) {
    heap_node = heap_min(timer_heap(loop));
    if (heap_node == NULL)
      break;

    handle = container_of(heap_node, uv_timer_t, heap_node);
    if (handle->timeout > loop->time)
      break;

    uv_timer_stop(handle);
    uv_timer_again(handle);
    handle->timer_cb(handle);
  }
}
也就是libuv需要每次loop轮询的时候都要去检查一次，判断是否需要执行,现在看看muduo的做法

/**
 * timerfd是Linux为用户程序提供的一个定时器接口。这个接口基于文件描述符，通过文件描述符的可读事件进行超时通知，所以能够被用于select/poll的应用场景。
 * 它是用来创建一个定时器描述符timerfd
 * 第一个参数：clockid指定时间类型，有两个值：
 * CLOCK_REALTIME :Systemwide realtime clock. 系统范围内的实时时钟
 * CLOCK_MONOTONIC:以固定的速率运行，从不进行调整和复位 ,它不受任何系统time-of-day时钟修改的影响
 * 第二个参数：flags可以是0或者O_CLOEXEC/O_NONBLOCK。
 * 返回值：timerfd（文件描述符）
 */
int createTimerfd()
{
  int timerfd = ::timerfd_create(CLOCK_MONOTONIC,TFD_NONBLOCK | TFD_CLOEXEC);
  if (timerfd < 0)
  {
    LOG_SYSFATAL << "Failed in timerfd_create";
  }
  return timerfd;
}

//使用 timerfd_settime 来设置定时触发的时间
void resetTimerfd(int timerfd, Timestamp expiration)
{
  // wake up loop by timerfd_settime()
  struct itimerspec newValue;
  struct itimerspec oldValue;
  memZero(&newValue, sizeof newValue);
  memZero(&oldValue, sizeof oldValue);
  newValue.it_value = howMuchTimeFromNow(expiration);
  //第二个结构体itimerspec就是timerfd要设置的超时结构体，它的成员it_value表示定时器第一次超时时间，it_interval表示之后的超时时间即每隔多长时间超时
  int ret = ::timerfd_settime(timerfd, 0, &newValue, &oldValue);
  if (ret)
  {
    LOG_SYSERR << "timerfd_settime()";
  }
}

之后就可以像使用普通的文件描述符一样，监听读事件

/**
 * 定时器触发,响应定时操作，然后找到下一个需要定时的时间，重置定时器
 */
void TimerQueue::handleRead()
{
  loop_->assertInLoopThread();
  //获取到当前的时间
  Timestamp now(Timestamp::now());
  readTimerfd(timerfd_, now);

  //返回当前已经到期的Entry集合,之后依次执行回调
  std::vector<Entry> expired = getExpired(now);

  callingExpiredTimers_ = true;
  cancelingTimers_.clear();

  // safe to callback outside critical section
  for (const Entry& it : expired)
  {
    it.second->run();
  }
  callingExpiredTimers_ = false;

  //重新设置下一个定时器
  reset(expired, now);
}
当定时器触发的时候，再重新的添加下一个定时器，由于定时器的响应操作是通过事件来触发，所以不用每次都执行轮询操作,当然内部肯定会维护时间的有序性，排序什么的

我们不能直接用map<Timestamp,Timer*>因为这样无法处理两个Timer到期时间相同的情况，有两个解决方案，一是用multimap或multiset，二是无法区分
key，muduo现在采用的是第二种的做法，这样可以避免使用不常见的multimap class, 具体来说，以pair<Timestamp,Timer*>为key，这样即便两个Timer的到期时间
相同，他们的地址也必定不同
typedef std::pair<Timestamp, Timer*> Entry;
typedef std::set<Entry> TimerList;
typedef std::pair<Timer*, int64_t> ActiveTimer;
typedef std::set<ActiveTimer> ActiveTimerSet;

// Timer list sorted by expiration
TimerList timers_;
ActiveTimerSet activeTimers_;
ActiveTimerSet cancelingTimers_;

所以对于时间的插入，查找是通过set来操作的，set内部是通过二分搜索树来实现的，所以对于查找添加时间是非常快速的
```

3.缓冲区
```java
关于缓冲区方面来说的话，由于要实现高性能，所以对于异步操作来说的话，使用缓冲区是非常有必要的，让上层不用关心数据有没有发送完成，
libuv是没有提供自己的缓冲区的，不管是在tcp还是udp方面，所以现看看muduo关于缓冲区的实现

class Buffer : public muduo::copyable
{
 public:
  //缓冲区前面预留了8个字节，方便后续有需求往这里添加head之类的内容
  static const size_t kCheapPrepend = 8;
  //默认缓冲区的大小为 1kB
  static const size_t kInitialSize = 1024;

  //Buffer里有两个常数 kChecapPrepend和kInitialSize，定义了prependable的初始大小和writable的初始大小
  explicit Buffer(size_t initialSize = kInitialSize)
    : buffer_(kCheapPrepend + initialSize),
      readerIndex_(kCheapPrepend),
      writerIndex_(kCheapPrepend)//一旦将 readerIndex_ =  writerIndex_ ，就代表当前没有可以发送的内容
  {
    assert(readableBytes() == 0);
    assert(writableBytes() == initialSize);
    assert(prependableBytes() == kCheapPrepend);
  }
  ...
private:
  //拥有存储数据的缓冲区
  std::vector<char> buffer_;
  //当前读的索引
  size_t readerIndex_;
  //当前写的索引
  size_t writerIndex_;
} 
内部是通过分配定长的缓冲区大小来做的默认大小为   1kB ，当然内部是允许扩容的，而且是自动的扩容
/**
* 执行扩容操作
*/
void makeSpace(size_t len)
{
  if (writableBytes() + prependableBytes() < len + kCheapPrepend)
  {
    // FIXME: move readable data
    buffer_.resize(writerIndex_+len);
  }
  else
  {
    // move readable data to the front, make space inside buffer
    assert(kCheapPrepend < readerIndex_);
    size_t readable = readableBytes();
    std::copy(begin()+readerIndex_,
                begin()+writerIndex_,
                begin()+kCheapPrepend);
    readerIndex_ = kCheapPrepend;
    writerIndex_ = readerIndex_ + readable;
    assert(readable == readableBytes());
  }
}

对于异步操作来说，当有消息可读的时候，都是尽可能的一次性读取完，那么就会造成缓冲区不够的问题，分配太大会造成浪费，分配太小不够使用，看看muduo的处理

/**
 * 读取消息 在非阻塞网络编程中，如何设计并使用缓冲区，一方面我们希望减少系统调用，一次读的数据越多越划算，那么似乎应该准备一个大的缓冲区，另一方面
 * 希望减少内存占用，如果有10000个 并发连接，每个连接一建立就分配50kB的读写缓冲区的话，将占用1GB内存，而大多数这些缓冲区的使用率很低
 * 在栈上准备一个 65536子节的 extrabuf,然后利用readv()来读取数据，iovec有俩快，第一块只想muduo Buffer中的writeable字节，另一块执行栈上的
 * extrabuf，这样如果读入的数据不多，那么全部读取到Buffer中，如果长度超过了Buffer的Writable字节，就会读到extrabu里，然后程序再把 extrabuf
 * 里的数据append()到Buffer中
 *
 * 这么做利用了临时栈上空间，避免每个连接的初始Buffer过大造成的内存浪费，也避免反复调用read的系统开销
 */
ssize_t Buffer::readFd(int fd, int* savedErrno)
{
  // saved an ioctl()/FIONREAD call to tell how much to read
  char extrabuf[65536];
  struct iovec vec[2];
  //返回当前缓冲区可写的大小
  const size_t writable = writableBytes();
  //防止覆盖掉
  vec[0].iov_base = begin()+writerIndex_;
  vec[0].iov_len = writable;
  //在准备一块栈上的空间
  vec[1].iov_base = extrabuf;
  vec[1].iov_len = sizeof extrabuf;
  // when there is enough space in this buffer, don't read into extrabuf.
  // when extrabuf is used, we read 128k-1 bytes at most.
  const int iovcnt = (writable < sizeof extrabuf) ? 2 : 1;
  const ssize_t n = sockets::readv(fd, vec, iovcnt);
  //读出现了问题
  if (n < 0)
  {
    *savedErrno = errno;
  }
  else if (implicit_cast<size_t>(n) <= writable)
  {
	//如果读取的长度小于缓冲区的长度，说明全部写到了当前缓冲区中
    writerIndex_ += n;
  }
  else
  {
	//如果读取的大小大于缓冲区的可写的大小，那么说明有一部分数据写到了栈上，我们要将这部分的数据，保存到缓冲区中
    writerIndex_ = buffer_.size();
    append(extrabuf, n - writable);
  }
  return n;
}
muduo巧妙的使用了 readv 函数 通过提供 struct iovec vec[2] ，一部分用数据可以允许写到栈的空间上，在readv确定了读取的内容长度，后续再将这部分的内容写到缓冲区中

下面看看我们项目中使用的 发送 缓冲区,我们使用的是队列，所以不存在扩容的操作
std::deque<std::unique_ptr<BufEntry>> bufq_;

对应的消息的内容添加
void SocketBuffer::pushBytes(std::vector<unsigned char> bytes,std::unique_ptr<ProgressUpdate> progressUpdate) {
	if (!bytes.empty()) {
		bufq_.push_back(make_unique<ByteArrayBufEntry>(std::move(bytes), std::move(progressUpdate)));
	}
}
void SocketBuffer::pushStr(std::string data,std::unique_ptr<ProgressUpdate> progressUpdate) {
	if (!data.empty()) {
		bufq_.push_back(make_unique<StringBufEntry>(std::move(data), std::move(progressUpdate)));
	}
}

下面来看看当libuv收到数据回调通知的时候，我们没有扩容这个缓冲区，而是采用先消耗当前缓冲区的内容，再添加的
int PeerBtInteraction::processBtMessage(const unsigned char *buff, size_t length){
	ssize_t currentWriteIndex = 0;
	//这里要考虑当前接收的内容长度是否大于剩余的缓冲区,如果大于的化,要提前消耗掉
	while ((btInteractive_->getBufferLength() + length) > btInteractive_->getBufferCapacity()) {
		//最大的拷贝长度
		size_t writeLength = btInteractive_->getBufferCapacity() - btInteractive_->getBufferLength();
		//先处理当前的内容
		btInteractive_->recvDataProcess(buff + currentWriteIndex,writeLength);
		//当前已经处理到了哪里
		currentWriteIndex += writeLength;
		//当前剩余的没有处理的长度
		length -= writeLength;
		//根据步骤处理对应的逻辑,检查错误
		int code = process_data();
		if (code) {
			return code;
		}
	}
	//如果到了这里,说明缓冲区的大小是足够的
	btInteractive_->recvDataProcess(buff + currentWriteIndex,length);
	//根据步骤处理对应的逻辑
	return process_data();
}
```

4.信号处理
```java
在多线程时代，signal 的语义更为复杂，信号分为俩类，发送给某一线程，发送给进程中的任一线程，还要考虑掩码对信号的屏蔽等，特别是在signal handle中不能
调用任何Pthreads函数，不能通过condition variable来通知其他的线程，所以在muduo 中这样写到 在多线程程序中，使用 signal的第一原则是不要使用signal

但是有一个例外， SIGPIPE，服务端程序通常的做法是忽略此信号，否则如果对方断开连接，而本机继续write的话，会导致程序意外终止，看看muduo的处理

#pragma GCC diagnostic ignored "-Wold-style-cast"
class IgnoreSigPipe
{
 public:
  IgnoreSigPipe()
  {
	//忽略SIGPIPE 信号，这个是tcp不知道对方已经断开的时候，而往socket写数据的时候产生的信号，默认不用处理，直接忽略
	//SIGPIPE的默认行为是终止进场，在命令行程序中这是合理的，但是在网络编程中，这意味着如果对方断开而本地继续写入的话，会造成进场意外退出
    //假如服务进程繁忙，没有及时处理对对方断开连接的事件，就有可能出现在连接断开之后继续发送数据的情况
    ::signal(SIGPIPE, SIG_IGN);
    // LOG_TRACE << "Ignore SIGPIPE";
  }
};
#pragma GCC diagnostic error "-Wold-style-cast"
IgnoreSigPipe initObj;
}  // namespace


而我们程序中也是直接忽略了这个信号的，但是很惭愧，当初出现了 SIGPIPE的 时候，由于第一次接触，导致我找了一个星期的错误，后续才发现是这个引起的

BtManager::BtManager()
	  : httpThreadPool_(ThreadPool::create(3)),//默认Http线程池数量为３个
	  exit_(false),
      isInit_(false),
	  globalBtListenerUdpPort_(0),
	  globalBtLisgenerTcpPort_(0),
	  cookieStorage_(make_unique<CookieStorage>()),
	  maxOverallHttpDownloadSpeedLimit_(0),
	  maxOveralBtDownloadSpeedLimit_(0),maxOverallUploadSpeedLimit_(0),
	  httpStat_(make_unique<NetStat>()),btStat_(make_unique<NetStat>()), dnsCache_(make_unique<DNSCache>()),isCancelAll_(false),
	  changeDnsCallback_(nullptr),
	  failNoticeCallback_(nullptr)
{
	mDownloads.clear();
	connectionPeer_.clear();
	runningRpcRequest_.clear();
	//https://www.cnblogs.com/jingzhishen/p/3453727.html
	signal(SIGPIPE, SIG_IGN);
}
```

内存检测
```java
内存检测方面通过不断的暂停，恢复下载来模拟更多的操作，也为了更好的暴露问题,比如闪退，资源清理等，实际看起来确实挺有用的
while(true){
	if(index == 200){
		sleep(5);
		BtManager::getInstance()->exit();
		BtManager::release();
		G_LOG_DEBUG("clear finish model use_count  %d",model.use_count());
		model = nullptr;
		break;
	}else{
		index++;
		sleep(5);
		BtManager::getInstance()->pauseDownload(model);
		sleep(1);
		BtManager::getInstance()->resumeDownload(model);
	}
}
```

内存检测方面使用  valgrind 来实现，执行的检测命令为
> valgrind --tool=memcheck --leak-check=full --show-leak-kinds=all ./main
有一点要注意的是，为了让他更好的检测，我们在推出的时候，我们要释放掉当前应用程序的所有的资源，其中也包括静态资源，要不然valgrind 也会把这些资源当做泄露,通过内存检测的时候也发现了一些问题，比如循环引用，父类持有子类的成员，子类又持有父类的成员，而且都是通过 shared_ptr来持有的，导致资源不会释放，解决这个方案可以通过父类持有子类的shared_ptr成员，子类持有父类的weak_ptr成员，或者直接让子类持有父类的 指针，如果你能确保子类对象会优先父类对象释放的话

下面项目中内存检测的结果，可以看出没有任何的内存泄露
![结果显示](/uploads/Bt项目总结/内存检测.png)


### 总结
当使用Libuv的时候，一定要确保让libuv的Loop正常退出，任何打乱退出的操作都会导致不正常的效果  !!


















