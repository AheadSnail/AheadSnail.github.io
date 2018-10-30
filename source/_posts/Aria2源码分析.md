---
layout: pager
title: Aria2源码分析
date: 2018-10-17 09:50:26
tags: [Android,NDK,Aria2]
description: Aria2源码分析
---

### 概述

>  Aria2源码分析

<!--more-->


### 简介

之前文章分析过，怎么样使用Aria2，以及对应的编译，如果只是做为简单的使用的化，估计早就已经解决了问题，先说说使用这个项目的原由吧，就是公司想节省带宽，比如在下载游戏，下载视频等方面
所以可以使用p2p技术，而Aria2是p2p开源中最火的一个，也是最多start数的一个，所以采用了这个开源库，但是由于Aria2 不具备打洞的方案，他原本是采用tcp的方式来传输内容，而像有些种子工具
比如Utorrent是具备这种打洞技术的，采用了utp的协议，而这个utp的库也是由Utorrent开源的，不足的是，这个库并没有开源，但是linux下面的p2p工具 transmission是具备utp，tcp协议的，而且是开源的
所以就要去阅读transmission的源码，查看他对应的utp以及tcp有什么不同点，在发的数据方面，这里就直接告诉大家答案，对于这俩种协议在发数据的时候，并没有任何的区别,由于要设计到修改源码，就有
必要对Aria2的源码有一定深度的认识，要不然是改不动的

### P2P技术

`p2p`也叫做变态下载，何为变态就是同一时刻当下载的人越多，速度就越快， p2p为什么能节省带宽，也就是本身你要从服务器下载的资源，可以直接从其他的用户那里获取到，由其他的用户提供带宽，
下面说下p2p大致的工作流程
1.首先客户端启动要监听一个端口号，用来接受其他客户端的连接，就相当于是服务端的角色，因为对于任何的一个用户来说，即是客户端又是服务端，但你没有资源的时候，相当于是一个客户端，从别人
下载内容，当你有内容之后，就变成了服务端，就可以响应其他用户的连接，共享资源
2.客户端要连接tracker 服务器，连接的时候，要告知本地客户端监听的端口号，下载的情况，比如下载了多少内容，还有多少内容没有下载等，不过最重要的是这个监听的端口号
3.tracker服务器收到了之后，就会返回给你，当前同时在线的所用用户的外网的ip和端口号，有几个用户下载完了，有几个用户还没有下载完，这个可以通过客户端连接tracker服务器的时候告知的下载情况可以得知
4.客户端收到了tracker服务器返回的当前在线用户的ip和端口号，此时客户端就可以尝试的跟他们连接，大致的连接内容有，说先公布下加密的情况，然后是握手的消息，最后是对应要下载内容的信息，比如告知他当前我有什么资源，我要什么资源等，对方得到回应之后，就会传递对应的资源

当然种子技术设计到了很多的协议，比如Dht等，具体的可以查看官网的介绍 http://bittorrent.org/beps/bep_0000.html

### Aria2源码分析

由于涉及到的源码比较多，这里不做详细的介绍,这里主要介绍下对应的类是干什么用的，哪些对象是共用的，哪些是自己独有的

```C++
1.DownloadEngine  首先这个类是所有种子公有的一个类，也即是只会有一个对象的存在 
他内部含有这俩个成员，这里面存放各种command，DownLoadEngine负责执行他们
std::deque<std::unique_ptr<Command>> routineCommands_;
std::deque<std::unique_ptr<Command>> commands_;
DownloadEngine 负责执行各种command的函数
int DownloadEngine::run(bool oneshot)
{
  //如果  commands_ 或者 routineCommands_ 队列不为空，commands_一开始就有添加了一个保持事件响应的引用对象KeepRunningCommand  所以不为空
  while (!commands_.empty() || !routineCommands_.empty()) {
	...
    //判断是否达到了刷新界面的时间，constexpr auto A2_DELTA_MILLIS = std::chrono::milliseconds(10);
    if (lastRefresh_.difference(global::wallclock()) + A2_DELTA_MILLIS >= refreshInterval_) {
      //刷新的间隔为1秒
      refreshInterval_ = DEFAULT_REFRESH_INTERVAL;
      //保存上一次刷新的时间
      lastRefresh_ = global::wallclock();
      //执行命令 ,状态为 Command::STATUS_ALL
      executeCommand(commands_, Command::STATUS_ALL);
    }
    else {
      //如果还没有到刷新的时间，执行命令，状态为  Command::STATUS_ACTIVE
      executeCommand(commands_, Command::STATUS_ACTIVE);
    }
    //执行命令
    executeCommand(routineCommands_, Command::STATUS_ALL);
    afterEachIteration();
    if (!noWait_ && oneshot) {
        return 1;
    }
  }
  //如果到了这里，就说明引擎已经开始停止
  onEndOfRun();
  return 0;
}
他内部还有一个成员为 std::unique_ptr<RequestGroupMan> requestGroupMan_; 这个类为请求的管理类，也即是相当于请求的集合,至于为什么要持有这个引用，当引擎要退出的时候，可以将
要退出的动作传递过去
//标识引擎要退出
void DownloadEngine::requestHalt()
{
  haltRequested_ = std::max(haltRequested_, 1);
  //标识所有的requestGroup退出
  requestGroupMan_->halt();
}
//标识引擎要强制退出
void DownloadEngine::requestForceHalt()
{
  haltRequested_ = std::max(haltRequested_, 2);
  requestGroupMan_->forceHalt();
}

他内部含有一个成员为 std::unique_ptr<BtRegistry> btRegistry_; 这个对象存储了当前所有正在运行的种子文件集合,下面看看BtRegister的成员以及作用

//map值的引用,key为当前任务的gid，value为BtObject对象引用   ,因为BtRegistry 是公有的，这个集合就是当前所有的种子对象, 所以根据gid从多个种子任务中获取到对应的种子对象的封装
std::map<a2_gid_t, std::unique_ptr<BtObject>> pool_;
以及对应的提供了添加还有获取对应的对象
void BtRegistry::put(a2_gid_t gid, std::unique_ptr<BtObject> obj)
{
  pool_[gid] = std::move(obj);
}
//根据gid从pool   Map集合中获取到对应的BtObject对象 也即是对应的种子对象的封装
BtObject* BtRegistry::get(a2_gid_t gid) const
{
  auto i = pool_.find(gid);
  if (i == std::end(pool_)) {
    return nullptr;
  }
  else {
    return (*i).second.get();
  }
}

//至于BtObject 这个结构体代表当前种子下载的所有信息
struct BtObject {
  //当前种子下载的上下文引用
  std::shared_ptr<DownloadContext> downloadContext;
  std::shared_ptr<PieceStorage> pieceStorage;
  std::shared_ptr<PeerStorage> peerStorage;
  std::shared_ptr<BtAnnounce> btAnnounce;
  //当前种子下载的BtRuntime引用
  std::shared_ptr<BtRuntime> btRuntime;
  std::shared_ptr<BtProgressInfoFile> btProgressInfoFile;

  BtObject(const std::shared_ptr<DownloadContext>& downloadContext,
           const std::shared_ptr<PieceStorage>& pieceStorage,
           const std::shared_ptr<PeerStorage>& peerStorage,
           const std::shared_ptr<BtAnnounce>& btAnnounce,
           const std::shared_ptr<BtRuntime>& btRuntime,
           const std::shared_ptr<BtProgressInfoFile>& btProgressInfoFile);
  BtObject();
};

一般在使用的时候是通过先获取到DownloadEngine对象，然后获取到他里面的成员BtRegister成员，然后根据gid 获取到对应的BtObject对象，这个存放了当前种子文件的所有信息

3.RequestGroupMan 这个类为请求的管理类，也即是相当于请求的集合
首先内部有俩个集合，一个代表当前正在请求的集合，另一个代表正要执行的请求集合
RequestGroupList requestGroups_;//当前正在请求的集合
RequestGroupList reservedGroups_;//正要执行的请求的集合

//标识当前最多支持多少个同时并发的下载
int maxConcurrentDownloads_;
以及提供了对应的让所有请求都退出的方法，本身这个方法的调用由DownloadEngine触发，前面分析过
void RequestGroupMan::halt()
{
  for (auto& elem : requestGroups_) {
      elem->setHaltRequested(true);
  }
}

void RequestGroupMan::forceHalt()
{
  for (auto& elem : requestGroups_) {
      elem->setForceHaltRequested(true);
  }
}

4.RequestGroup  代表当前种子请求对应的所有的内容，含有各种的对象
//标识是否停止
bool haltRequested_;
//标识是否强制停止
bool forceHaltRequested_;
//标识是否为暂停的请求
bool pauseRequested_;
上面这些成员可以用来标识当前请求的状态，比如是退出，还是暂停等

//DownloadContext 引入对象  当前种子的下载上下文对象,存放了当前种子的文件的信息
std::shared_ptr<DownloadContext> downloadContext_;
//BtRuntime 指针引用    当前种子对应的下载运行时对象
BtRuntime* btRuntime_;
//PeerStorage 指针引用    当前种子对应的存储Peer的对象
PeerStorage* peerStorage_;

BtRuntime 对象 代表当前种子下载的运行时对象
//是否停止状态，每一个RequestGroup 对应的一个BtRuntime对象
bool halt_;
//标识当前的连接数  
int connections_;
//标识是否已经准备好
bool ready_;
//最大支持存储的Peer数量
int maxPeers_;
//最小支持存储的Peer数量
int minPeers_;

一般要想知道当前种子对应的下载状态，可以通过当前对象的下面这俩个方法获取到
//获取当前的RequetGroup是否为停止状态
bool isHalt() const { return halt_; }
//设置当前的RequetGroup为停止状态
void setHalt(bool halt) { halt_ = halt; }
而这俩个方法的调用，其实是有对应的RequstGroup调用的,而RequestGroup又可以由RequestGroupMan调用，RequestGroupMan又可以由DownloadEngine调用
//设置当前requestGroup停止状态
void RequestGroup::setHaltRequested(bool f, HaltReason haltReason)
{
  haltRequested_ = f;
  if (haltRequested_) {
    pauseRequested_ = false;
    haltReason_ = haltReason;
  }
#ifdef ENABLE_BITTORRENT
  if (btRuntime_) {
     //设置停止
     btRuntime_->setHalt(f);
  }
#endif // ENABLE_BITTORRENT
}


//这里注意一下dht的初始化，这个只会初始化一次
//配置DHT,解析dht指定的文件等,返回值为command Map ,这里是针对每一个种子都对应的初始化DhtSetup操作,但是里面会有限制，当前面的种子已经初始化过，后面的就不会再进行初始化
std::tie(c, rc) = DHTSetup().setup(e, AF_INET);	
//在dht协议中，bt客户端使用“distributed sloppy hash table”（DHT的全称）来存储没有tracker地址的种子文件所对应的peer节点的信息，
//在这种情况下，每一个peer节点变成了一个tracker服务器，dht协议是在udp通信协议的基础上使用Kademila（俗称Kad算法）算法实现。
DHTSetup::setup(DownloadEngine* e, int family)
{
	...
	//这里要注意，  DHTRegistry::isInitialized() 这个取的是静态对象值，所以对于多个种子来说，如果之前dht已经初始化过了，就不会再继续往下执行了，直接return 处理,那这样也可以保证我们的
	//utpContext  的创建也是只会有一份 ,也即是后面的这些对象关于Dht的创建都是只有一份的，也即是所有的种子文件公用的
	if ((family != AF_INET && family != AF_INET6) || (family == AF_INET && DHTRegistry::isInitialized()) || (family == AF_INET6 && DHTRegistry::isInitialized6())) {
		return {};
	}
	...
	//标识初始化完成,下次另一个种子文件，就不会再次的执行初始化了
    DHTRegistry::setInitialized(true);
	...
}	

所以对应dht后面初始化的对象也都是独有的是为所有的种子文件共享的,以及生成的各种command也是独有的，是不会销毁的，只有当引擎退出的时候才会退出，比如

//构建一个  DHTInteractionCommand 对象，调用对应的构造函数完成初始化 , 这个command用于执行udp tracker ，dht ,以及 utp
auto command = make_unique<DHTInteractionCommand>(e->newCUID(), e);
bool DHTInteractionCommand::execute()
{
  // needs this. 这里要注意这里的退出条件，如果当前全部的种子都下载完成了，或者引擎退出 的时候这个对象才会销毁
  if (e_->getRequestGroupMan()->downloadFinished() || (e_->isHaltRequested() && udpTrackerClient_->getNumWatchers() == 0)) {
    LOGD("DHTInteractionCommand exiting");
    return true;
  }
  else if (e_->isForceHaltRequested()) {//如果当前引擎是强制退出的化
    udpTrackerClient_->failAll();
    LOGD("DHTInteractionCommand exiting");
    return true;
  }
  ...
}  

```

### 源码执行流程

下面是做这个项目的时候整理的源码过程

[BT(带中心Tracker)通信协议的分析](http://note.youdao.com/noteshare?id=0c030a62ab1994e3539ccbec25c28a20&sub=3B8250E73A1F44E7B35CE099B116383E)

[Aria2 Tracker连接过程](http://note.youdao.com/noteshare?id=6b622065253105bb9ec1e742272f621b&sub=5B58733856A447ED814D75346D2598B3)

[Aria2 客户端Peer 逻辑](http://note.youdao.com/noteshare?id=a0c82267bb9bd047fb75a3531a14243d&sub=EFA1737AB5784AFDA622CB939DBE0E79)

[Aria2 服务端Peer逻辑](http://note.youdao.com/noteshare?id=54aba8c5205c0458d6fc133c03bc305e&sub=53E793F9F728480CB4E31214F928ADCD)

当然由于要参考transmission的源码，对应的源码分析过程

[Transmission Tracker连接过程](http://note.youdao.com/noteshare?id=6996c89f4c794afb8157aa5a060b468d&sub=A86A1FBFA4FD4F5F962E6D77D6211418)

[Transmission Peer连接](http://note.youdao.com/noteshare?id=2f856b2d03517b3e845e88585a16e26e&sub=3C36D35980224DFDAC2A924D2D888AA2)


