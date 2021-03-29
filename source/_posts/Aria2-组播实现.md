---
layout: pager
title: Aria2 组播实现
date: 2018-10-22 15:37:43
tags: [Android,NDK,Aria2]
description:  Aria2 组播实现
---
### 组播

>  组播协议允许将一台主机发送的数据通过网络路由器和交换机复制到多个加入此组播的主机，是一种一对多的通讯方式。组播协议与现在广泛使用的单播协议的不同之处在于，一个主机用单播协议向n个主机发送相同的数据时，发送主机需要分别向n个主机发送，共发送n次。一个主机用组播协议向n个主机发送相同的数据时，只要发送1次，其数据由网络中的路由器和交换机逐级进行复制并发送给各个接收方，这样既节省服务器资源也节省网络主干的带宽资源。与广播协议相比，只有组播接收方向路由器发出请求后，网络路由器才复制一份数据给接收方，从而节省接收方的带宽。而广播方式无论接收方是否需要，网络设备都将所有广播信息向所有设备发送，从而大量占据接收方的接入带宽。

大概的工作方式类似于

![结果显示](/uploads/Aria2Utp/组播.png)

就是在一个网络组里面，当有一个用户发送消息给特定的组播地址，其他在这个网络组里面的用户也能收到这条消息,而不用逐一的发送,当然并不是每一个网络都支持组播，这要看路由器的设置，可以通过下面的方式判断当前的这个网络是否支持组播

判断是否支持组播协议:

![结果显示](/uploads/Aria2Utp/判断是否支持组播.png)

### 本地发现

而Bt协议中有一个协议叫做本地发现，在Bt协议中，tracker服务器返回的都是外网的ip和对应的端口号，如果当前俩个用户同处于Nat后的网络，那么这俩个用户直接使用tracker服务器返回的ip和端口号
是没有办法直接连接的，此时如果要连接的化，就应该用本地ip和端口号，而不应该使用tracker服务器返回的ip和端口号，在Bt协议中的本地发现就是使用了组播的技术，做到了这一点，下面是对应的官网介绍

> http://bittorrent.org/beps/bep_0014.html

Bt组播协议内容

![结果显示](/uploads/Aria2Utp/Bt组播协议.png)


可以看出Bt本地发现协议发送的内容中有俩个比较重要的字段，一个是hash值，代表当前种子文件的hash值，可以根据这个hash值判断是否是自己需要的 port: 对方bt 监听的端口号，得到了这个端口号，就可以直接的跟对方通信了

### 源码实现

首先要开启这个选项，默认是关闭的，如果要打开，可以动态的设置这个选项 gloableOptions.push_back(std::pair<std::string,std::string> ("bt-enable-lpd","true"));

```cpp
//设置是否本地发现,如果设置为tru的化，就可以在同一个局域网下载，默认是关闭的,还有不能为私有的种子
if (option->getAsBool(PREF_BT_ENABLE_LPD) && btReg->getTcpPort() && (metadataGetMode || !torrentAttrs->privateTorrent)) {
    //如果还没有构建，进行初始化,这里要注意的是对应LpdMessageReciver的对象是全局的，是单例的存在，他作用于所有的种子
    if (!btReg->getLpdMessageReceiver()) {
      LOGD("Initializing LpdMessageReceiver.");
      auto receiver = std::make_shared<LpdMessageReceiver>(LPD_MULTICAST_ADDR, LPD_MULTICAST_PORT);
      bool initialized = false;
      //默认是为空的，
      const std::string& lpdInterface = e->getOption()->get(PREF_BT_LPD_INTERFACE);
      //如果没有指定本地发现的ip和端口号
      if (lpdInterface.empty()) {
        if (receiver->init("")) {
           initialized = true;
        }
    }
    else {
        //如果有指定的化，获取指定的ip和端口号
        auto ifAddrs = SocketCore::getInterfaceAddress(lpdInterface, AF_INET, AI_NUMERICHOST);
        for (const auto& soaddr : ifAddrs) {
          char host[NI_MAXHOST];
          if (inetNtop(AF_INET, &soaddr.su.in.sin_addr, host, sizeof(host)) == 0 &&
            receiver->init(host)) {
            initialized = true;
            break;
            }
        }
    }
    //如果初始化完成
    if (initialized) {
        //将当前的LpdMessageReciever保存到btReg中，这个对象是共享的是所有种子共享的
        btReg->setLpdMessageReceiver(receiver);
        LOGD("LpdMessageReceiver initialized. multicastAddr=%s:%u,"" localAddr=%s", LPD_MULTICAST_ADDR, LPD_MULTICAST_PORT, receiver->getLocalAddress().c_str());
        //构建一个LpdReceiveMessageCommand 对象，用来接受消息，同时添加到engine中
        e->addCommand(make_unique<LpdReceiveMessageCommand>(e->newCUID(), receiver, e));
    }
    else {
        LOGD("LpdMessageReceiver not initialized.");
    }
}

```

上面的逻辑就是，判断是否支持本地发现，如果支持，当前还没有进行初始化的化，执行对应的初始化，会构建LpdMessageReceiver ，用于接受所有种子文件对应的本地发现消息

```cpp
auto receiver = std::make_shared<LpdMessageReceiver>(LPD_MULTICAST_ADDR, LPD_MULTICAST_PORT);
constexpr const char LPD_MULTICAST_ADDR[] = "239.192.152.143";
constexpr uint16_t LPD_MULTICAST_PORT = 6771;
也就刚好对应bt协议中组播的ip和端口号的设置

LpdMessageReceiver 的初始化，会加入这个组播
bool LpdMessageReceiver::init(const std::string& localAddr)
{
  try {
    //构建了一个udp的socket对象
    socket_ = std::make_shared<SocketCore>(SOCK_DGRAM);
#ifdef __MINGW32__
    // Binding multicast address fails under Windows.
    socket_->bindWithFamily(multicastPort_, AF_INET);
#else  // !__MINGW32__  绑定ip和端口
    socket_->bind(multicastAddress_.c_str(), multicastPort_, AF_INET);
#endif // !__MINGW32__
    //将当前的ip地址加入multicastAddress_ 指定的组
    LOGD("Joining multicast group. %s:%u, localAddr=%s", multicastAddress_.c_str(), multicastPort_, localAddr.c_str());
    socket_->joinMulticastGroup(multicastAddress_, multicastPort_, localAddr);
    //设置不阻塞
    socket_->setNonBlockingMode();
    localAddress_ = localAddr;
    LOGD("Listening multicast group (%s:%u) packet", multicastAddress_.c_str(), multicastPort_);
    return true;
  }
  catch (RecoverableException& e) {
    A2_LOG_ERROR_EX("Failed to initialize LPD message receiver.", e);
  }
  return false;
}
```

当前主机加入组播

```cpp
void SocketCore::joinMulticastGroup(const std::string& multicastAddr, uint16_t multicastPort, const std::string& localAddr)
{
  //将  multicastAddr 转成 in_addr
  in_addr multiAddr;
  //只支持ipv4
  if (inetPton(AF_INET, multicastAddr.c_str(), &multiAddr) != 0) {
     throw DL_ABORT_EX(fmt("%s is not valid IPv4 numeric address", multicastAddr.c_str()));
  }
  //如果localAddr为空，则随机的获取一个
  in_addr ifAddr;
  if (localAddr.empty()) {
      ifAddr.s_addr = htonl(INADDR_ANY);
  }
  else if (inetPton(AF_INET, localAddr.c_str(), &ifAddr) != 0) {//如果localAddr不为空，则要判断是否为ipv4的地址
     throw DL_ABORT_EX(fmt("%s is not valid IPv4 numeric address", localAddr.c_str()));
  }

  //当前的ip地址加入组播
  struct ip_mreq mreq;
  memset(&mreq, 0, sizeof(mreq));
  mreq.imr_multiaddr = multiAddr;
  mreq.imr_interface = ifAddr;
  //3．选项IP_ADD_MEMBERSHIP和IP_DROP_MEMBERSHIP 加入或者退出一个组播组，通过选项IP_ADD_MEMBERSHIP和IP_DROP_MEMBER- SHI
  //选项IP_ADD_MEMBERSHIP用于加入某个广播组，之后就可以向这个广播组发送数据或者从广播组接收数据。此选项的值为mreq结构，成员imn_multiaddr是需要加入的广播组IP地址，
  // 成员imr_interface是本机需要加入广播组的网络接口IP地址。例如：
  setSockOpt(IPPROTO_IP, IP_ADD_MEMBERSHIP, &mreq, sizeof(mreq));
}

//如果初始化完成
if (initialized) {
    //将当前的LpdMessageReciever保存到btReg中，这个对象是共享的是所有种子共享
    btReg->setLpdMessageReceiver(receiver);
    LOGD("LpdMessageReceiver initialized. multicastAddr=%s:%u,"" localAddr=%s", LPD_MULTICAST_ADDR, LPD_MULTICAST_PORT, receiver->getLocalAddress().c_str());
    //构建一个LpdReceiveMessageCommand 对象，用来接受消息，同时添加到engine中
    e->addCommand(make_unique<LpdReceiveMessageCommand>(e->newCUID(), receiver, e));
}
else{
    LOGD("LpdMessageReceiver not initialized.");
}
```
如果初始化完成，则将LpdMessageReciever对象保存到BtRegister中，这个对象是唯一的,前面有介绍,之后构建LpdReceiveMessageCommand 对象，这个command 是用来接受Lpd的消息的，这个对象也是
作用于所有的种子，并且这个对象会一直处于运行中，也即是一直处于接受lpd的消息
```cpp
bool LpdReceiveMessageCommand::execute()
{
  //只有引擎退出的时候，这个对象才会被销毁
  if (e_->getRequestGroupMan()->downloadFinished() || e_->isHaltRequested()) {
      return true;
  }
  ....
}

前面介绍的这几个对象都是公有的，那么对于特定的种子就要构建对应的对象
//获取到当前btReg 中的LpdMessageReceiver对象，这个对象是共享的，不一定是当前种子构建的对象,但是对应的每一个种子都会执行下面的逻辑，构建对应的LpdMessageDispatcher对象等
if (btReg->getLpdMessageReceiver()) {
    //获取到当前种子的hash值
    const unsigned char* infoHash = bittorrent::getInfoHash(requestGroup->getDownloadContext());
    LOGD("Initializing LpdMessageDispatcher.");

    //默认为tcp的 Bt监听端口，如果当前支持utp的化，那就为udp Bt的监听端口
    uint16_t port ;
    if(btReg->getUtpContext())
    {
        port = btReg->getUdpPort();
        LOGD("LPD Message change port udp %d",port);
    } else{
        port = btReg->getTcpPort();
    }
    //构建当前种子对应的LpdMessageDispatcher 对象  这里应该要修改，因为  btReg->getTcpPort()  代表当前服务端监听的tcp端口号，如果是utp实现的化，这里要换成我们的udp端口号
    auto dispatcher = std::make_shared<LpdMessageDispatcher>(std::string(&infoHash[0], &infoHash[INFO_HASH_LENGTH]), port, LPD_MULTICAST_ADDR, LPD_MULTICAST_PORT);
    //进行初始化操作
    if (dispatcher->init(btReg->getLpdMessageReceiver()->getLocalAddress(),/*ttl*/ 1, /*loop*/ 1)) {
        LOGD("LpdMessageDispatcher initialized.");
        //初始化成功，构建  LpdDispatchMessageCommand 对象，然后添加到engine中 这也是对应的每一个种子文件对应的一个对象
        auto cmd = make_unique<LpdDispatchMessageCommand>(e->newCUID(), dispatcher, e);
        cmd->setBtRuntime(btRuntime);
        e->addCommand(std::move(cmd));
    }
    else {
        LOGD("LpdMessageDispatcher not initialized.");
    }
}

```
首先获取到当前种子对应的hash值，然后获取到对应的bt端口号，由于这里我做了utp的支持，所以相应的这个端口号也要设置成udp监听的那个端口号
```cpp
//构建当前种子对应的LpdMessageDispatcher 对象  这里应该要修改，因为  btReg->getTcpPort()  代表当前服务端监听的tcp端口号，如果是utp实现的化，这里要换成我们的udp端口号
auto dispatcher = std::make_shared<LpdMessageDispatcher>(std::string(&infoHash[0], &infoHash[INFO_HASH_LENGTH]), port, LPD_MULTICAST_ADDR, LPD_MULTICAST_PORT);
constexpr const char LPD_MULTICAST_ADDR[] = "239.192.152.143";
constexpr uint16_t LPD_MULTICAST_PORT = 6771;

之后执行初始化操作 if (dispatcher->init(btReg->getLpdMessageReceiver()->getLocalAddress(),/*ttl*/ 1, /*loop*/ 1)) 
//初始化操作
bool LpdMessageDispatcher::init(const std::string& localAddr, unsigned char ttl, unsigned char loop)
{
  try {
    //创建一个ipv4  UDP的SocketCore 对象
    socket_ = std::make_shared<SocketCore>(SOCK_DGRAM);
    socket_->create(AF_INET);

    //选项IP_MULTICAST_IF用于设置组播的默认默认网络接口，会从给定的网络接口发送，另一个网络接口会忽略此数据。例如：
    LOGD("Setting multicast outgoing interface=%s", localAddr.c_str());
    socket_->setMulticastInterface(localAddr);

    //选项IP_MULTICAST_TTL允许设置超时TTL，范围为0～255之间的任何值，例如：
    LOGD("Setting multicast ttl=%u", static_cast<unsigned int>(ttl));
    socket_->setMulticastTtl(ttl);

    //选项IP_MULTICAST_LOOP用于控制数据是否回送到本地的回环接口。例如：
    LOGD("Setting multicast loop=%u", static_cast<unsigned int>(loop));
    socket_->setMulticastLoop(loop);
    return true;
  }
  catch (RecoverableException& e) {
    A2_LOG_ERROR_EX("Failed to initialize LpdMessageDispatcher.", e);
  }
  return false;
}
```
首先构建一个udp的soketCore，然后加入这个组播,然后针对当前的这个种子文件，构建一个LpdDispatchMessageCommand 消息对象
//初始化成功，构建  LpdDispatchMessageCommand 对象，然后添加到engine中 这也是对应的每一个种子文件对应的一个对象
auto cmd = make_unique<LpdDispatchMessageCommand>(e->newCUID(), dispatcher, e);
这个对象主要用来针对当前的种子文件，发送对应的组播消息，组播协议中每5分钟就要发送对应消息，这样其他刚加入的主机才有机会收到这个消息，当然也不能发送的太频繁，会引起阻塞,检验是5分钟
```cpp
//command命令的执行
bool LpdDispatchMessageCommand::execute()
{
  //如果当前的种子是退出状态，返回true，回收这个对象，注意这里不是引擎退出，而是当前的种子
  if (btRuntime_->isHalt()) {
    return true;
  }
  //判断当前主机是否需要发送LPD消息，LPD协议规定默认为5分钟发送一次消息到组 ，这样其他在这个组的主机就可以接受到当前主机发送的消息了
  if (dispatcher_->isAnnounceReady()) {
    try {
      A2_LOG_INFO(fmt("Dispatching LPD message for infohash=%s", util::toHex(dispatcher_->getInfoHash()).c_str()));
      //如果当前主机发送LPD消息成功,重置时间，下次再歌5分钟再发送一次
      if (dispatcher_->sendMessage()) {
        A2_LOG_INFO("Sending LPD message is complete.");
        dispatcher_->resetAnnounceTimer();
        tryCount_ = 0;
      }
      else {
        //如果当前LPD消息发送失败，这里是重试机制
        ++tryCount_;
        if (tryCount_ >= 5) {
          A2_LOG_INFO(fmt("Sending LPD message %u times but all failed.", tryCount_));
          dispatcher_->resetAnnounceTimer();
          tryCount_ = 0;
        }
        else {
          A2_LOG_INFO("Could not send LPD message, retry shortly.");
        }
      }
    }
    catch (RecoverableException& e) {
      A2_LOG_INFO_EX("Failed to send LPD message.", e);
      dispatcher_->resetAnnounceTimer();
      tryCount_ = 0;
    }
  }
  e_->addCommand(std::unique_ptr<Command>(this));
  return false;
}

//检查是否需要发送lpd消息了，只要客户端参与群组，它就应该在每个接口上发送一个LSD，每隔5分钟就会发出一个LSD
bool LpdMessageDispatcher::isAnnounceReady() const
{
  return timer_.difference(global::wallclock()) >= interval_;
}

```
上面的逻辑就是判断是否有必要再次发送lpd消息，当时间间隔到达了5分钟就发送一个lpd消息，就会触发  dispatcher_->sendMessage()
```cpp
//发送消息 ，将创建的LPD消息发送到 组指定的  multicastAddress_   multicastPort_ 上 这样在这个组的其他主机就可以接收到消息了
bool LpdMessageDispatcher::sendMessage()
{
  return socket_->writeData(request_.c_str(), request_.size(), multicastAddress_, multicastPort_) == (ssize_t)request_.size();
}

而这个request的内容为 request_(bittorrent::createLpdRequest(multicastAddress_, multicastPort_, infoHash_, port_))

/*LPD 消息长这样
A LSD announce is formatted as follows:
BT-SEARCH * HTTP/1.1\r\n
Host: <host>\r\n
Port: <port>\r\n       当前种子监听的bt端口
Infohash: <ihash>\r\n  当前种子的对应的infoHash 唯一值
cookie: <cookie (optional)>\r\n
\r\n
\r\n*/

//创建lpd消息
std::string createLpdRequest(const std::string& multicastAddress, uint16_t multicastPort, const std::string& infoHash, uint16_t port)
{
  return fmt("BT-SEARCH * HTTP/1.1\r\n"
             "Host: %s:%u\r\n"
             "Port: %u\r\n"
             "Infohash: %s\r\n"
             "\r\n\r\n",
             multicastAddress.c_str(), multicastPort, port,
             util::toHex(infoHash).c_str());
}
```
这样当前种子的lpd消息就发送出去了，可以看出来协议的内容跟bt协议的一样,携带了当前种子对应的hash值，已经本地监听的端口号,所以如果要支持utp的化，我们只要改这个值为我们udp监听的端口号就好
对于本地发现消息的接受端，就是前面介绍的LpdReceiveMessageCommand 来统一的接受所有的本地发现的消息,
```cpp
bool LpdReceiveMessageCommand::execute()
{
  //只有引擎退出的时候，这个对象才会被销毁
  if (e_->getRequestGroupMan()->downloadFinished() || e_->isHaltRequested()) {
      return true;
  }

  for (size_t i = 0; i < 20; ++i) {
    //接收消息，创建对应的LPDMessage消息
    auto m = receiver_->receiveMessage();
    //如果没有接收到内容，或者内容不合法，m会为null
    if (!m) {
      break;
    }

    //解析得到了LPDMessage消息
    auto& reg = e_->getBtRegistry();
    //根据接受到的LPD 消息中传递种子的infoHash值，判断是否是我们需要的
    auto& dctx = reg->getDownloadContext(m->infoHash);
    //当前的种子不是我需要的，直接跳过
    if (!dctx) {
      LOGD("Download Context is null for infohash=%s.", util::toHex(m->infoHash).c_str());
      continue;
    }
    //还要判断是否为私有的种子
    if (bittorrent::getTorrentAttrs(dctx)->privateTorrent) {
      LOGD("Ignore LPD message because the torrent is private.");
      continue;
    }

    //到了这里就说明,当前收到的这个LPD消息是符合我们的
    RequestGroup* group = dctx->getOwnerRequestGroup();
    assert(group);
    //根据gid获取到对应的BtObject 对象，这个对象为当前种子的所有的信息
    auto btobj = reg->get(group->getGID());
    assert(btobj);
    //获取到当前peerStorage对象
    auto& peerStorage = btobj->peerStorage;
    assert(peerStorage);
    auto& peer = m->peer;
    //添加当前peer
    if (peerStorage->addPeer(peer)) {
        LOGD("LPD peer %s:%u local=%d added.", peer->getIPAddress().c_str(), peer->getPort(), peer->isLocalPeer() ? 1 : 0);
    }
    else {
      LOGD("LPD peer %s:%u local=%d not added.", peer->getIPAddress().c_str(), peer->getPort(), peer->isLocalPeer() ? 1 : 0);
    }
  }
  e_->addCommand(std::unique_ptr<Command>(this));
  return false;
}

上面的逻辑也是比较简单的。首先判断是否有接收到本地发现的消息，这里是通过是否能解析成一个LpdMessage对象，这个对象也很简单的，就是包含了对方传递的消息内容
struct LpdMessage {
  std::shared_ptr<Peer> peer;//其中peer封装了对方的ip和端口号，端口号可以通过lpd消息中获取到,注意这里的ip是本地局域网的ip
  std::string infoHash;
}

//下面是接受本地消息，以及解析本地本地消息的方法实现
//判断是否有本地消息到来，如果有解析传递的内容
std::unique_ptr<LpdMessage> LpdMessageReceiver::receiveMessage()
{
  while (1) {
    unsigned char buf[200];
    Endpoint remoteEndpoint;
    ssize_t length;
    try {
      //读取内容
      length = socket_->readDataFrom(buf, sizeof(buf), remoteEndpoint);
      //如果没有读取到，直接返回nullptr
      if (length == 0) {
        return nullptr;
      }
    }
    catch (RecoverableException& e) {
      LOGD("Failed to receive LPD message.");
      return nullptr;
    }

    //读取到了内容
    HttpHeaderProcessor proc(HttpHeaderProcessor::SERVER_PARSER);
    try {
      if (!proc.parse(buf, length)) {
        // UDP packet must contain whole HTTP header block.
        continue;
      }
    }
    catch (RecoverableException& e) {
      LOGD("Failed to parse LPD message.");
      continue;
    }

    //获取到处理之后的结果
    auto header = proc.getResult();
    //找到LPD 中包含的infohash 字段的内容
    const std::string& infoHashString = header->find(HttpHeader::INFOHASH);
    //找到对应的端口号，然后检查这个端口号是否合法
    uint32_t port = 0;
    if (!util::parseUIntNoThrow(port, header->find(HttpHeader::PORT)) || port > UINT16_MAX || port == 0) {
      LOGD("Bad LPD port=%u", port);
      continue;
    }
    LOGD("LPD message received infohash=%s, port=%u from %s", infoHashString.c_str(), port, remoteEndpoint.addr.c_str());

    //校验hashInfo长度是否合法
    std::string infoHash;
    if (infoHashString.size() != 40 || (infoHash = util::fromHex(infoHashString.begin(), infoHashString.end())).empty()) {
      LOGD("LPD bad request. infohash=%s", infoHashString.c_str());
      continue;
    }
    //如果都满足 则构建 Peer ,这里的port为对方在lpd消息中告知的bt 监听的端口号，所以对于接收端可以将这个当做一个peer
    auto peer = std::make_shared<Peer>(remoteEndpoint.addr, port, false);
    //判断是否为本地的ip地址
    if (util::inPrivateAddress(remoteEndpoint.addr)) {
        //如果是本地的设置当前的peer来自本地
        peer->setLocalPeer(true);
    }
    //然后构建一个LpdMessage 消息
    return make_unique<LpdMessage>(peer, infoHash);
  }
}

//根据接受到的LPD 消息中传递种子的infoHash值，判断是否是我们需要的
auto& dctx = reg->getDownloadContext(m->infoHash);
//当前的种子不是我需要的，直接跳过
if (!dctx) {
    LOGD("Download Context is null for infohash=%s.", util::toHex(m->infoHash).c_str());
    continue;
}
//还要判断是否为私有的种子
if (bittorrent::getTorrentAttrs(dctx)->privateTorrent) {
    LOGD("Ignore LPD message because the torrent is private.");
    continue;
}

//到了这里就说明,当前收到的这个LPD消息是符合我们的
RequestGroup* group = dctx->getOwnerRequestGroup();
assert(group);
//根据gid获取到对应的BtObject 对象，这个对象为当前种子的所有的信息
auto btobj = reg->get(group->getGID());
assert(btobj);
//获取到当前peerStorage对象
auto& peerStorage = btobj->peerStorage;
assert(peerStorage);
auto& peer = m->peer;
//添加当前peer
if (peerStorage->addPeer(peer)) {
   LOGD("LPD peer %s:%u local=%d added.", peer->getIPAddress().c_str(), peer->getPort(), peer->isLocalPeer() ? 1 : 0);
}
else {
   LOGD("LPD peer %s:%u local=%d not added.", peer->getIPAddress().c_str(), peer->getPort(), peer->isLocalPeer() ? 1 : 0);
}

```

最后通过比较当前接收到的lpd消息的infoHash值是否我们当前有，没有就代表这个种子我们不关心，所以直接省略掉，如果这个lpd消息满足条件，则添加到PeerStorage对象中，后面会触发对应的连接
操作,那这样俩者就可以通信了，

下面是组播的日志，可以看出能得到对方的本地ip和端口号,这里要注意的是，这个组播是否能支持要看路由器是否能够支持，所以有些情况下组播得到对方的组播信息，那估计就是路由器不支持了

![结果显示](/uploads/Aria2Utp/Aria2组播日志.png)


