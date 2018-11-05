---
layout: pager
title: Aria2 RPC 实现机制
date: 2018-11-05 15:01:43
tags: [NDK,Aria2,RPC]
description: Aria2 RPC 实现机制
---

### 概述

> Aria2 RPC 实现机制

<!--more-->

### 简介
RPC(Remote Procedure Call Protocol)--远程过程调用协议
> 它是一种通过网络从远程计算机程序上请求服务，而不需要了解底层网络技术的协议。RPC协议假定某些传输协议的存在，如TCP或UDP，为通信程序之间携带信息数据。在OSI网络通信模型中，
RPC跨越了传输层和应用层。RPC使得开发包括网络分布式多程序在内的应用程序更加容易。

RPC的工作方式
> RPC采用客户机/服务器模式。请求程序就是一个客户机，而服务提供程序就是一个服务器。首先，客户机调用进程发送一个有进程参数的调用信息到服务进程，然后等待应答信息。在服务器端，
进程保持睡眠状态直到调用信息的到达为止。当一个调用信息到达，服务器获得进程参数，计算结果，发送答复信息，然后等待下一个调用信息，最后，客户端调用进程接收答复信息，获得进程结果，然后调用执行继续进行。

个人理解
> RPC就是一种进程间通信的一种方式，类似于Android使用Sokcet实现进程间通信一样，使用Tcp或者udp来传输通信，为通信程序之间携带信息数据，本文主要介绍下Aria2 中Rpc的实现，在介绍之前先看别人使用Aria2 Rpc的支持做的一个下载工具，这个工具的优点是支持直接下载网盘的内容，不限速 官网地址为 https://pandownload.com/

具体的界面是长这样的
![结果显示](/uploads/Aria2Rpc实现/网盘界面.png)

直接使用Aria2库的
![结果显示](/uploads/Aria2Rpc实现/使用Aria2库.png)

Aria2 Rpc的实现，进程一 
![结果显示](/uploads/Aria2Rpc实现/网盘进程一.png)

进程二
![结果显示](/uploads/Aria2Rpc实现/网盘进程二.png)

工作原理是:
> 开启俩个进程，一个是Aria2下载库的进程，这个库开启对rpc的支持，具体的实现就是内部会有一个服务端开启一个tcp的端口监听，用来监听其他端的请求，另一个进程是pandownload 封装了界面的支持,当触发下载的时候，会发送一个类似于http的请求，包含了请求的内容 ，Aria2收到请求，Aria2完成下载，当然也提供了下载界面的操作，下载速度等

目前使用这个工具来下载的人不是少数，支持window，mac 在github上面的start数量达到了6k，人家只是简单的做了一层封装就可以做到这层的效果，下面分析下Aria2 Rpc的实现

### Aria2 Rpc的使用
首先打开Aria2 rpc的支持 配置的参数为 --enbale-rpc 此时就会处于监听连接请求的状态
![结果显示](/uploads/Aria2Rpc实现/Aria2Rpc的配置.jpg)

对应另一端我们可以使用Postman来模拟rpc的通信
![结果显示](/uploads/Aria2Rpc实现/postMan的配置.png)

Aria收到rpc的请求开始响应对应的内容
![结果显示](/uploads/Aria2Rpc实现/Aria2收到rpc请求.png)

至于我怎么知道rpc要发送的内容 可以查看官网 http://aria2.github.io/manual/en/html/aria2c.html
![结果显示](/uploads/Aria2Rpc实现/官网rpc支持的内容.png)

当然这里包括了很多的rpc的请求，反正对于一个下载器来说，这些api是完全够用的，下面来分析下具体的源码实现

### Rpc源码分析
```C++
#### 首先要开启Rpc的支持

//设置允许rpc访问
gloableOptions.push_back(std::pair<std::string,std::string> ("enable-rpc","true"));
//设置rpc的监听端口
gloableOptions.push_back(std::pair<std::string,std::string> ("rpc-listen-port","7800"));
//设置监听所有，这样如果不是在同一个局域网的时候，可以通过对方的ip来连接
gloableOptions.push_back(std::pair<std::string,std::string> ("rpc-listen-all","true"));
```

#### 创建Tcp Socket监听端口
```C++
//构建对应的ipv4,ipv6
static int families[] = {AF_INET, AF_INET6};
size_t familiesLength = op->getAsBool(PREF_DISABLE_IPV6) ? 1 : 2;

for (size_t i = 0; i < familiesLength; ++i) {
    //构建HttpListenCommand 对象 secure 默认为false 代表是否加密
    auto httpListenCommand = make_unique<HttpListenCommand>(e->newCUID(), e.get(), families[i], secure);
    //执行绑定,PREF_RPC_LISTEN_PORT 默认监听的端口为6800
    if (httpListenCommand->bindPort(op->getAsInt(PREF_RPC_LISTEN_PORT))) {
       e->addCommand(std::move(httpListenCommand));
       ok = true;
    }
}

bindPort 源码实现

//默认的端口为6800
bool HttpListenCommand::bindPort(uint16_t port)
{
  if (serverSocket_) {
    e_->deleteSocketForReadCheck(serverSocket_, this);
  }
  //构建一个SocketCore对象,默认的为tcp连接
  serverSocket_ = std::make_shared<SocketCore>();
  const int ipv = (family_ == AF_INET) ? 4 : 6;
  try {
    int flags = 0;
    if (e_->getOption()->getAsBool(PREF_RPC_LISTEN_ALL)) {
      flags = AI_PASSIVE;
    }
    //绑定端口
    serverSocket_->bind(nullptr, port, family_, flags);
    //开始监听
    serverSocket_->beginListen();
    LOGD(MSG_LISTENING_PORT, getCuid(), port);
    e_->addSocketForReadCheck(serverSocket_, this);
    LOGD(("IPv%d RPC: listening on TCP port %u"), ipv, port);
    return true;
  }
  catch (RecoverableException& e) {
    A2_LOG_ERROR_EX(fmt("IPv%d RPC: failed to bind TCP port %u", ipv, port), e);
    serverSocket_->closeConnection();
  }
  return false;
}

判断是否有收到内容

bool HttpListenCommand::execute()
{
  //当前引擎退出的时候，这个command才会退出,这个command会一直存在，用于监听tcp的连接请求
  if (e_->getRequestGroupMan()->downloadFinished() || e_->isHaltRequested()) {
      return true;
  }
  try {
     //如果这个 rpc监听的端口有读取到内容的化
    if (serverSocket_->isReadable(0)) {
      //利用tcp的Accept函数，返回当前连接的socketCore对象
      std::shared_ptr<SocketCore> socket(serverSocket_->acceptConnection());
      //设置不要延迟
      socket->setTcpNodelay(true);

      //获取对方的ip和端口号
      auto endpoint = socket->getPeerInfo();
      //标识建立了一个连接
      LOGD("RPC: Accepted the connection from %s:%u.", endpoint.addr.c_str(), endpoint.port);
      e_->setNoWait(true);
      //构建一个 HttpServerCommand 对象，用于后面的通信
      e_->addCommand(make_unique<HttpServerCommand>(e_->newCUID(), e_, socket, secure_));
    }
  }
  catch (RecoverableException& e) {
    A2_LOG_DEBUG_EX(fmt(MSG_ACCEPT_FAILURE, getCuid()), e);
  }
  e_->addCommand(std::unique_ptr<Command>(this));
  return false;
}
```

#### 客户端发起一个请求
假设此时使用postMan发送了一个rpc的请求，请求的内容为
```json
{
 "jsonrpc":"2.0",
 "id":"ttrtrtrtrtrt",
 "method":"aria2.addUri",
 "params":[["https://github.com/aria2/aria2/archive/release-1.33.0.zip"]]
}
```

#### 服务端响应请求
```C++
也即是会执行到这里
if (serverSocket_->isReadable(0)) {
    //利用tcp的Accept函数，返回当前连接的socketCore对象
    std::shared_ptr<SocketCore> socket(serverSocket_->acceptConnection());
    //设置不要延迟
    socket->setTcpNodelay(true);

    //获取对方的ip和端口号
    auto endpoint = socket->getPeerInfo();
    //标识建立了一个连接
    LOGD("RPC: Accepted the connection from %s:%u.", endpoint.addr.c_str(), endpoint.port);
    e_->setNoWait(true);
    //构建一个 HttpServerCommand 对象，用于后面的通信
    e_->addCommand(make_unique<HttpServerCommand>(e_->newCUID(), e_, socket, secure_));
}

acceptConnection函数实现，会通过tcp的accept函数，监听到连接的请求，并且对应的创建一个用于俩者通信的socket返回
//服务端判断是否有客户端连接上实现
std::shared_ptr<SocketCore> SocketCore::acceptConnection() const
{
  sockaddr_union sockaddr;
  socklen_t len = sizeof(sockaddr);

  sock_t fd;

  //服务端调用accept函数接受连接，如果服务器调用accept()时还没有客户端的连接请求，就阻塞等待直到有客户端连接上来，addr是一个传出参数，accept()返回时传出客户端
  //地址和端口号，accept函数   成功返回一个新的socket文件描述符，用于和客户端通信，失败返回-1,设置errno
  while ((fd = accept(sockfd_, &sockaddr.sa, &len)) == (sock_t)-1 && SOCKET_ERRNO == A2_EINTR);

  //这里的fd 就是 成功返回的一个新的socket文件描述符，用于和客户端通信的，为什么要重新的返回一个新的socekt，因为每条tcp传输连接只能有俩个端点，只能进行
  //点对点的数据传输，不支持多播和广播传输方式
  int errNum = SOCKET_ERRNO;
  //判断这个新创建的socekt是否合法
  if (fd == (sock_t)-1) {
    throw DL_ABORT_EX(fmt(EX_SOCKET_ACCEPT, errorMsg(errNum).c_str()));
  }

  //设置socket，接受buff的大小
  applySocketBufferSize(fd);

  //构建一个SocketCore 对象，调用对应的构造函数，完成初始化,这里的fd是客户端连接上之后，重新创建的socket 返回的描述符，sockType_为TCP的类型
  auto sock = std::make_shared<SocketCore>(fd, sockType_);
  //设置不阻塞
  sock->setNonBlockingMode();
  return sock;
}

接着构建一个HttpServerCommand，e_->addCommand(make_unique<HttpServerCommand>(e_->newCUID(), e_, socket, secure_)); 这个command使用的参数socket为accept 返回的socket

//command命令的执行,这个command主要用来处理头部的内容
bool HttpServerCommand::execute()
{
  //当前引擎退出的还是才会回收这个command对象
  if (e_->getRequestGroupMan()->downloadFinished() || e_->isHaltRequested()) {
     return true;
  }
  try {
    //socket->isReadable判断是否有接受到内容
    if (socket_->isReadable(0) || (writeCheck_ && socket_->isWritable(0)) || socket_->getRecvBufferedLength() || !httpServer_->getSocketRecvBuffer()->bufferEmpty()) {
      //赋值超时的开始时间,防止连接通了之后，对方没有发送任何的内容造成浪费，
      timeoutTimer_ = global::wallclock();
      ...	
	  //httpServer接收请求  返回false代表没有成功解析到
      if (!httpServer_->receiveRequest()) {
        updateWriteCheck();
        e_->addCommand(std::unique_ptr<Command>(this));
        return false;
      }
      ...
	
      //判断是否超过了默认的请求长度内容限制,默认的大小为2M，如果超过了这个大小，将不会处理，直接return true，释放当前的连接
      if (e_->getOption()->getAsInt(PREF_RPC_MAX_REQUEST_SIZE) < httpServer_->getContentLength()) {
        LOGD("Request too long. ContentLength=%" PRId64 "."
                          " See --rpc-max-request-size option to loose"
                          " this limitation.",
                          httpServer_->getContentLength());
        return true;
      }

      //构建HttpServerBodyCommand对象，用来响应传递过来的body内容，之后返回true，回收这个对象
      e_->addCommand(make_unique<HttpServerBodyCommand>(getCuid(), httpServer_, e_, socket_));
      e_->setNoWait(true);
      return true;
    }catch (RecoverableException& e) {
    //出现了异常，回收这个对象
    LOGD("CUID#%" PRId64" - Error occurred while reading HTTP request", getCuid());
    return true;
  }
}

首先看receiveRequest 函数的实现，这个是用来接收传递的内容
//是否有接受到内容
bool HttpServer::receiveRequest()
{
  //如果缓冲空间为空
  if (socketRecvBuffer_->bufferEmpty()) {
     //从socket中尝试的获取内容，如果获取为空，并且socket不能读也不能写 抛出异常
    if (socketRecvBuffer_->recv() == 0 && !socket_->wantRead() && !socket_->wantWrite()) {
      throw DL_ABORT_EX(EX_EOF_FROM_PEER);
    }
  }

  //如果到了这里，就说明从socket读取到了内容， 解析读取的内容
  if (headerProcessor_->parse(socketRecvBuffer_->getBuffer(), socketRecvBuffer_->getBufferLength())) {
    //获取到请求头结果,将原本的对象转移到了 lastRequestHeader_中
    lastRequestHeader_ = headerProcessor_->getResult();
    //打印解析到的请求头
    LOGD("HTTP Server received request\n%s", headerProcessor_->getHeaderString().c_str());
    //移除当前读取的内容
    socketRecvBuffer_->drain(headerProcessor_->getLastBytesProcessed());
    bodyConsumed_ = 0;
    if (setupResponseRecv() < 0) {
      LOGD("Request path is invalid. Ignore the request body.");
    }
    //从请求头中获取到content_length 代表当前请求包含的长度
    const std::string& contentLengthHdr = lastRequestHeader_->find(HttpHeader::CONTENT_LENGTH);
    //如果请求长度不为空，并且成功的解析到lastContentLength_ 中
    if (!contentLengthHdr.empty()) {
      if (!util::parseLLIntNoThrow(lastContentLength_, contentLengthHdr) || lastContentLength_ < 0) {
         throw DL_ABORT_EX(fmt("Invalid Content-Length=%s", contentLengthHdr.c_str()));
      }
    }
    else {
      //如果请求的长度为空，则标识当前请求内容的长度为0
      lastContentLength_ = 0;
    }
    //执行清理操作
    headerProcessor_->clear();

    //从请求结果中找到对应的accept_encoding
    std::vector<Scip> acceptEncodings;
    const std::string& acceptEnc = lastRequestHeader_->find(HttpHeader::ACCEPT_ENCODING);
    util::splitIter(acceptEnc.begin(), acceptEnc.end(), std::back_inserter(acceptEncodings), ',', true);
    //判断是否支持gzip
    acceptsGZip_ = false;
    for (std::vector<Scip>::const_iterator i = acceptEncodings.begin(), eoi = acceptEncodings.end(); i != eoi; ++i) {
      if (util::strieq((*i).first, (*i).second, "gzip")) {
        acceptsGZip_ = true;
        break;
      }
    }
    //最后成功解析，返回true
    return true;
  }
  else {
    //从接受的缓冲区中移除内容，返回false
    socketRecvBuffer_->drain(headerProcessor_->getLastBytesProcessed());
    return false;
  }
}
可以看出这个函数主要用来接受数据，并处理对应的请求头，当处理完之后继续往下执行
//构建HttpServerBodyCommand对象，用来响应传递过来的body内容，之后返回true，回收这个对象
e_->addCommand(make_unique<HttpServerBodyCommand>(getCuid(), httpServer_, e_, socket_));
e_->setNoWait(true);
return true;

这里会构建另一个command，用于响应请求的内容
bool HttpServerBodyCommand::execute()
{
  //如果引擎退出，这个command才会回收
  if (e_->getRequestGroupMan()->downloadFinished() || e_->isHaltRequested()) {
     return true;
  }
  ...
  switch (httpServer_->getRequestType()) { //获取到请求的类型,这里只看json格式的，当然也有xml格式的等
    case RPC_TYPE_JSON:
    case RPC_TYPE_JSONP: {//json方式
        ...
        //转成json的时候出现了错误
        if (error < 0) {
            LOGD("CUID#%" PRId64" - Failed to parse JSON-RPC request", getCuid());
            rpc::RpcResponse res(rpc::createJsonRpcErrorResponse(-32700, "Parse error.", Null::g()));
            sendJsonRpcResponse(res, callback);
            return true;
        }
        Dict* jsondict = downcast<Dict>(json);
        if (jsondict) {//但个任务
            //处理json的请求,处理对应的请求，并且得到对应的处理结果封装到 RpcResponse 中
            auto res = rpc::processJsonRpcRequest(jsondict, e_);
            //发生响应的内容
            sendJsonRpcResponse(res, callback);
        }
	...	
    }
}

首先要转成json格式，如果不是标准的json格式的化，会得到一个解析错误的回应,如果正常解析 执行processJsonRpcRequest 函数解析json的内容
//读取传递过来的json内容
RpcResponse processJsonRpcRequest(Dict* jsondict, DownloadEngine* e)
{
  //读取json中id的内容
  auto id = jsondict->popValue("id");
  if (!id) {
    // 如果id为空，创建一个错误的响应返回
    return createJsonRpcErrorResponse(-32600, "Invalid Request.", Null::g());
  }
  //读取json中method的内容
  const String* methodName = downcast<String>(jsondict->get("method"));
  //如果method为空，创建一个错误的响应返回
  if (!methodName) {
    return createJsonRpcErrorResponse(-32600, "Invalid Request.", std::move(id));
  }
  //获取到json中params的内容,可以传递多个，所以这里是一个list
  std::unique_ptr<List> params;
  auto tempParams = jsondict->popValue("params");
  if (downcast<List>(tempParams)) {
    params.reset(static_cast<List*>(tempParams.release()));
  }
  else if (!tempParams) {
    params = List::g();
  }
  else {
    // TODO No support for Named params
    return createJsonRpcErrorResponse(-32602, "Invalid params.", std::move(id));
  }

  LOGD("Executing RPC method %s", methodName->s().c_str());
  //构建一个  RpcRequest 请求，
  RpcRequest req = {methodName->s(), std::move(params), std::move(id), true};
  //根据rpc请求的方式，创建对应的 RpcMethod 执行对应的请求
  return getMethod(methodName->s())->execute(std::move(req), e);
}

根据上面的函数实现，就知道了为什么我们在发送的时候要参照这样的格式写
{
 "jsonrpc":"2.0",
 "id":"ttrtrtrtrtrt",
 "method":"aria2.addUri",
 "params":[["https://github.com/aria2/aria2/archive/release-1.33.0.zip"]]
}

当json格式解析完之后，创建一个RpcRequest请求  RpcRequest req = {methodName->s(), std::move(params), std::move(id), true}; 这里的methodName 就为 aria2.addUri
params 就为 https://github.com/aria2/aria2/archive/release-1.33.0.zip  id 就为 ttrtrtrtrtrt ，接着执行 getMethod(methodName->s())->execute(std::move(req), e)

首先执行getMethod函数

//根据rpc的method 得到对应的 RpcMethod 方法，首先会从缓存中查找，如果不存在，就构建对应的，然后存储到缓存中
RpcMethod* getMethod(const std::string& methodName)
{
  //从map中查找methodName对应的  RpcMethod
  auto itr = cache.find(methodName);
  //如果没有找到
  if (itr == std::end(cache)) {
    //创建对应的RpcMethod 对象
    auto m = createMethod(methodName);
    if (m) {
      //创建成功，插入到缓存中，然后返回这个RpcMethod
      auto rv = cache.insert(std::make_pair(methodName, std::move(m)));
      return (*rv.first).second.get();
    }

    if (!noSuchRpcMethod) {
       noSuchRpcMethod = make_unique<NoSuchMethodRpcMethod>();
    }

    return noSuchRpcMethod.get();
  }
  //从缓冲中找到了，直接返回
  return (*itr).second.get();
}
假设当前是第一次请求，所以缓存表中不存在任何的对象，所以会执行createMethod 函数，至于这里为什么要使用缓存表，估计是因为考虑到rpc请求比较频繁

//根据methodName创建对应的RpcMethod对象 ,这里支持的接口有很多，这里只列出三个
std::unique_ptr<RpcMethod> createMethod(const std::string& methodName)
{
  //如果当前请求的方法为 aria2.addUri 代表添加一个url
  if (methodName == AddUriRpcMethod::getMethodName()) {
    //构建一个AddUriRpcMethod 对象
    return make_unique<AddUriRpcMethod>();
  }

#ifdef ENABLE_BITTORRENT
  //如果当前请求的方法为 aria2.addTorrent 代表添加一个种子
  if (methodName == AddTorrentRpcMethod::getMethodName()) {
    return make_unique<AddTorrentRpcMethod>();
  }

  //如果当前请求的方法为 aria2.getPeers 获取到Peer节点
  if (methodName == GetPeersRpcMethod::getMethodName()) {
    return make_unique<GetPeersRpcMethod>();
  }
  ...
}
由于当前我们传递的是 aria2.addUri 所以会执行 make_unique<AddUriRpcMethod>() 返回，接着存储到缓存表中，接着执行 getMethod(methodName->s())->execute(std::move(req), e)

//执行RPC的入口
RpcResponse RpcMethod::execute(RpcRequest req, DownloadEngine* e)
{
  auto authorized = RpcResponse::NOTAUTHORIZED;
  try {
    authorize(req, e);
    authorized = RpcResponse::AUTHORIZED;
    //调用子类对应的实现,并且得到处理之后的结果
    auto r = process(req, e);
    //构建一个RpcResponse 对象，用于响应
    return RpcResponse(0, authorized, std::move(r), std::move(req.id));
  }
  catch (RecoverableException& ex) {
    A2_LOG_DEBUG_EX(EX_EXCEPTION_CAUGHT, ex);
    return RpcResponse(1, authorized, createErrorResponse(ex, req), std::move(req.id));
  }
}

由于我们当前的子类是 AddUriRpcMethod 所以 process 由这个子类来实现

//addUri rpc请求的实现类
std::unique_ptr<ValueBase> AddUriRpcMethod::process(const RpcRequest& req, DownloadEngine* e)
{
  const List* urisParam = checkRequiredParam<List>(req, 0);
  const Dict* optsParam = checkParam<Dict>(req, 1);
  const Integer* posParam = checkParam<Integer>(req, 2);

  //获取到要添加的uri。这里可以添加多个
  std::vector<std::string> uris;
  extractUris(std::back_inserter(uris), urisParam);
  if (uris.empty()) {
    LOGD("URI is not provided.");
    throw DL_ABORT_EX("URI is not provided.");
  }

  //构建对应的Option对象
  auto requestOption = std::make_shared<Option>(*e->getOption());
  //将传递过来的参数整合到 requestOption 中
  gatherRequestOption(requestOption.get(), optsParam);

  //判断是否有指定下载的位置，比如添加下载到第一位等
  bool posGiven = checkPosParam(posParam);
  size_t pos = posGiven ? posParam->i() : 0;

  std::vector<std::shared_ptr<RequestGroup>> result;
  //根据uri创建对应的RequestGroup
  createRequestGroupForUri(result, requestOption, uris,
                           /* ignoreForceSeq = */ true,
                           /* ignoreLocalPath = */ true);

  //如果创建的RequestGroup不为空，添加请求
  if (!result.empty()) {
    //添加请求
    return addRequestGroup(result.front(), e, posGiven, pos);
  }
  else {
    LOGD("No URI to download.");
    throw DL_ABORT_EX("No URI to download.");
  }
}

上面的函数通过解析传递过来的参数，然后创建对应的RequestGroup对象，然后执行 addRequestGroup 添加请求，返回gid，这两个函数之前已经分析过，也即是只要调用了这俩个方法，就会触发对应的下载了
process函数执行完，得到对应的gid，接着构建一个 RpcResponse(0, authorized, std::move(r), std::move(req.id))，这里有process处理之后的返回值，这个返回值也即是要返回给客户端的

//执行RPC的入口
RpcResponse RpcMethod::execute(RpcRequest req, DownloadEngine* e)
{
  auto authorized = RpcResponse::NOTAUTHORIZED;
  try {
    authorize(req, e);
    authorized = RpcResponse::AUTHORIZED;
    //调用子类对应的实现,并且得到处理之后的结果
    auto r = process(req, e);
    //构建一个RpcResponse 对象，用于响应
    return RpcResponse(0, authorized, std::move(r), std::move(req.id));
  }
  catch (RecoverableException& ex) {
    A2_LOG_DEBUG_EX(EX_EXCEPTION_CAUGHT, ex);
    return RpcResponse(1, authorized, createErrorResponse(ex, req), std::move(req.id));
  }
}
 
继续回到这里 HttpServerBodyCommand 往下执行 接着就是响应了 sendJsonRpcResponse 
//处理json的请求,处理对应的请求，并且得到对应的处理结果封装到 RpcResponse 中
auto res = rpc::processJsonRpcRequest(jsondict, e_);
//发生响应的内容
sendJsonRpcResponse(res, callback);

//将处理之后的结果，返回响应
void HttpServerBodyCommand::sendJsonRpcResponse(const rpc::RpcResponse& res, const std::string& callback)
{
  bool notauthorized = rpc::not_authorized(res);
  //当前是否支持gzip
  bool gzip = httpServer_->supportsGZip();
  //转成成json
  std::string responseData = rpc::toJson(res, callback, gzip);
  if (res.code == 0) {
    //填充返回的内容
    httpServer_->feedResponse(std::move(responseData), getJsonRpcContentType(!callback.empty()));
  }
  ..
  //填充完了之后，准备发送,notauthorized 代表是否需要延迟
  addHttpServerResponseCommand(notauthorized);
}

namespace {
std::string getJsonRpcContentType(bool script)
{
  return script ? "text/javascript" : "application/json-rpc";
}
} // namespace


//填充返回的内容，这里填充俩个部分一个是Http的Header部分，一个是text，传递的内容，填充到socketBuffer_ 中
void HttpServer::feedResponse(int status, const std::string& headers,
                              std::string text, const std::string& contentType)
{
  std::string httpDate = Time().toHTTPDate();
  std::string header =
      fmt("HTTP/1.1 %s\r\n"
          "Date: %s\r\n"
          "Content-Length: %lu\r\n"
          "Expires: %s\r\n"
          "Cache-Control: no-cache\r\n",
          getStatusString(status), httpDate.c_str(),
          static_cast<unsigned long>(text.size()), httpDate.c_str());
  if (!contentType.empty()) {
    header += "Content-Type: ";
    header += contentType;
    header += "\r\n";
  }
  if (!allowOrigin_.empty()) {
    header += "Access-Control-Allow-Origin: ";
    header += allowOrigin_;
    header += "\r\n";
  }
  if (supportsGZip()) {
    header += "Content-Encoding: gzip\r\n";
  }
  if (!supportsPersistentConnection()) {
    header += "Connection: close\r\n";
  }
  header += headers;
  header += "\r\n";
  LOGD("HTTP Server sends response:\n%s", header.c_str());
  //填充头部部分
  socketBuffer_.pushStr(std::move(header));
  //填充传的内容
  socketBuffer_.pushStr(std::move(text));
}

这个函数主要是拼接要返回的内容，请求头，请求内容 并且添加到了socketBuffer_ 中此时并没有执行发送，接着执行addHttpServerResponseCommand 执行发送
//发送响应的请求
void HttpServerBodyCommand::addHttpServerResponseCommand(bool delayed)
{
  //构建一个 HttpServerResponseCommand对象
  auto resp = make_unique<HttpServerResponseCommand>(getCuid(), httpServer_, e_, socket_);
  ...
  e_->addCommand(std::move(resp));
  e_->setNoWait(true);
}

HttpServerResponseCommand 的基类是AbstractHttpServerResponseCommand ，这里的发送逻辑是在父类实现的,现在看下execute函数的实现

//command命令的执行
bool AbstractHttpServerResponseCommand::execute()
{
  if (e_->getRequestGroupMan()->downloadFinished() || e_->isHaltRequested()) {
    return true;
  }
  try {
    //发送响应的内容
    ssize_t len = httpServer_->sendResponse();
    if (len > 0) {
       //设置超时的时间
       timeoutTimer_ = global::wallclock();
    }
  }
  catch (RecoverableException& e) {
    A2_LOG_INFO_EX(fmt("CUID#%" PRId64" - Error occurred while transmitting response body.", getCuid()), e);
    return true;
  }

  //当所有的内容都发送完毕了之后
  if (httpServer_->sendBufferIsEmpty()) {
    LOGD("CUID#%" PRId64 " - HttpServer: all response transmitted.", getCuid());
    //回调执行afterSend，之后执行回收操作
    afterSend(httpServer_, e_);
    return true;
  }
  else {
    //代表缓冲区中还有内容，也即是内容还没有发送完，检查是否超时，这里为30秒，也即是开始发送的时间到终止的时间间隔为30，到了就回收这个command
    if (timeoutTimer_.difference(global::wallclock()) >= 30_s) {
      LOGD("CUID#%" PRId64" - HttpServer: Timeout while trasmitting response.", getCuid());
      return true;
    }
    else {
      updateReadWriteCheck();
      e_->addCommand(std::unique_ptr<Command>(this));
      return false;
    }
  }
}
```

上面的逻辑也是比较简单的，这样数据就发送出去了，这样另一端就收到了内容，这里是postman收到的内容为
![结果显示](/uploads/Aria2Rpc实现/rpc收到的内容.png)

#### 长连接的支持
当数据发送完之后，会执行 afterSend(httpServer_, e_); 这个函数会由具体的子类来实现，比如当前是HttpServerResponseCommand，所以会执行对应的函数
```C++
void HttpServerResponseCommand::afterSend(const std::shared_ptr<HttpServer>& httpServer, DownloadEngine* e)
{
  //判断是否支持长连接,如果支持长连接，这里会构建一个 HttpServerCommand 用于后面的通信
  if (httpServer->supportsPersistentConnection()) {
    LOGD("CUID#%" PRId64 " - Persist connection.", getCuid());
    e->addCommand(make_unique<HttpServerCommand>(getCuid(), httpServer, e, httpServer->getSocket()));
  }
}
```
> 这里做的处理就是判断当前是否支持长连接，如果支持就构建一个HttpServerCommand 进行后续的操作,这个command前面有介绍，也即是当收到请求的时候，第一个的command这个command主要用来接收内容以及解析http header内容，处理完之后，会构建另一个command用于解析请求的内容，这样就少了一个步骤也即是最开始的监听端口请求的部分 accept函数，这样就不会每次都构建一个socket ，一直使用同一个socket,当然如果不支持长连接的化，就会关闭掉当前的socket，下次再次连接只能重新的通过accept函数重新的创建一个socket来响应后续的操作

