---
layout: pager
title: Aria2 限速原理以及Utp限速处理
date: 2018-11-06 14:09:12
tags: [NDK,Aria2,utp]
description: Aria2 限速原理以及Utp限速处理
---
### 简介
绝大多数的Bt下载都含有限速的功能，包括限制上传的速度，以及下载的速度，更有甚者拿限速来要求用户充值会员，比如百度网盘下载，迅雷等，

为什么我们需要去限速
> 因为我们使用Aria2下载只会在用户的下载过程中去贡献带宽，至于为什么只在下载过程中贡献带宽,考虑到了俩点一点是目前手机在下载完安装包之后，都会提示用户删掉安装包， 其次 如果用户下载完了，还在后台一直的做种，或多或少的导致耗电量的增加等，所以只考虑在下载过程中贡献带宽，所以就有必要去调整对应的下载速度，以及上传的速度，使他在下载的时候长一点，进而 增加贡献带宽的概率还有一点就是如果我们不限制下载速度的化，那么会导致某些网速很快的用户直接把我们的带宽吃掉，导致其他的用户下载变慢，所以要做一个`平衡`做到大家的下载速度都不会相差太远

### Aria2限速的使用
首先打开Aria2  配置参数为 **--max-download-limit=20k** 此时就是限制下载的速度在20k
![结果显示](/uploads/Aria2Rpc实现/限速的结果.png)

至于我怎么知道具体的参数设置 可以查看官网 
> http://aria2.github.io/manual/en/html/aria2c.html

下载限速参数设置
![结果显示](/uploads/Aria2Rpc实现/Aria2限速支持.png)

相应的上传速度的参数设置
![结果显示](/uploads/Aria2Rpc实现/Aria2上传限速支持.png)

### 限速处理源码分析
#### 首先要开启限速的支持
```CPP
//设置最大的下载速度, 由于utp采用的是udp的原因，如果做限速处理的化，无法达到tcp限速的效果，所以下载不支持限速
gloableOptions.push_back(std::pair<std::string,std::string> ("max-download-limit","50k"));
//设置最大的上传速度
gloableOptions.push_back(std::pair<std::string,std::string> ("max-upload-limit","100k"));
```

#### 参数的解析使用
```CPP

首先会在创建对应的RequestGroup 中解析，然后设置maxDownloadSpeedLimit_ ，maxUploadSpeedLimit_ 保存值

//创建请求组的构造函数,传递的参数一个为当前请求gid，一个为当前请求组的选项
RequestGroup::RequestGroup(const std::shared_ptr<GroupId>& gid,
                           const std::shared_ptr<Option>& option)
    : belongsToGID_(0),//标识这个是属于一个请求组
      gid_(gid),//当前请求组的唯一的id
      option_(option),//将当前请求组对应的选项保存起来
      progressInfoFile_(std::make_shared<NullProgressInfoFile>()),//首先构造一个NullProgressInfoFile对象，调用对应的构造函数，并将引用赋值给progressInfoFile_
      uriSelector_(make_unique<InorderURISelector>()),//构造一个InorderURISelector对象，并将引用赋值给uriSelector_
      requestGroupMan_(nullptr),
#ifdef ENABLE_BITTORRENT
      btRuntime_(nullptr),
      peerStorage_(nullptr),
#endif // ENABLE_BITTORRENT
      followingGID_(0),
      lastModifiedTime_(Time::null()),//上次修改的时间
      timeout_(option->getAsInt(PREF_TIMEOUT)),//timeout 超时时间设置，默认为60秒，
      state_(STATE_WAITING),//当前的状态，这里置为等待中
      numConcurrentCommand_(option->getAsInt(PREF_SPLIT)),//split 默认值为5，这个值代表同时有多少个连接参与下载，即一个下载可以有多少个并发下载
      numStreamConnection_(0),
      numStreamCommand_(0),
      numCommand_(0),
      fileNotFoundCount_(0),//文件没有找到的个数
      maxDownloadSpeedLimit_(option->getAsInt(PREF_MAX_DOWNLOAD_LIMIT)),//max-download-limit 代表最大下载的限制，这个可以改变的。。默认值为没有限制
      maxUploadSpeedLimit_(option->getAsInt(PREF_MAX_UPLOAD_LIMIT)),//max-upload-limit 代表最大上传的限制  这个可以改变的。。默认值为没有限制
      ....
}

并且在当前类中定义了对应的方法

//判断当前的总下载是否超过了最大的下载速度
bool RequestGroup::doesDownloadSpeedExceed()
{
  int spd = downloadContext_->getNetStat().calculateDownloadSpeed();
  return maxDownloadSpeedLimit_ > 0 && maxDownloadSpeedLimit_ < spd;
}

//判断当前的总上传是否超过了最大的上传速度
bool RequestGroup::doesUploadSpeedExceed()
{
  int spd = downloadContext_->getNetStat().calculateUploadSpeed();
  return maxUploadSpeedLimit_ > 0 && maxUploadSpeedLimit_ < spd;
}
```

#### 下载速度限制

具体的使用doesDownloadSpeedExceed只有俩个地方，而且都只作用在Bt下载请求内容的时候,先来看第一处PeerInteractionCommand ,他的父类为 PeerAbstractCommand,主要完成超时的处理
```CPP
//command 命令的执行
bool PeerInteractionCommand::executeInternal()
{
    ...
    //判断当前的上传以及下载是否超过了最大的速度限制,这里会更新Peer的超时时间设置
    if (getDownloadEngine()->getRequestGroupMan()->doesOverallDownloadSpeedExceed() || requestGroup_->doesDownloadSpeedExceed()) {
        //如果超过了,设置读不可写
        disableReadCheckSocket();
        //设置不检查
        setNoCheck(true);
        LOGD("PeerInteractionCommand DownloadSpeedExceed checkPoint_ reset");
    }
    ...
}

//移除socektfd 读的检查 设置checkSocketIsReadable_ 标识为false
void PeerAbstractCommand::disableReadCheckSocket()
{
  if (checkSocketIsReadable_) {
    e_->deleteSocketForReadCheck(readCheckTarget_, this);
    checkSocketIsReadable_ = false;
    readCheckTarget_.reset();
  }
}

//设置不用检查超时
void PeerAbstractCommand::setNoCheck(bool check) { noCheck_ = check; }

这个超时的判断主要是用来更新超时的时间点，主要是通过上面的俩个方法，改变对应的标识,但是具体的使用，已经超时的机制使用是在父类的

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
   ...
}

由于执行了setNoCheck 函数，修改了noCheck_ 为true，所以上面的代码会执行 checkPoint_ = global::wallclock(); 修改了超时点的设置

第二处使用的地方为 DefaultBtInteractive 也即是真正的接受消息的地方

//接受消息
size_t DefaultBtInteractive::receiveMessages()
{
  //当前还有多少条消息没有发送
  size_t countOldOutstandingRequest = dispatcher_->countOutstandingRequest();
  //代表接受到了多少条消息,这里的while(1) 的意思就是说，会一直的读取socekt的内容，也即是如果一次传递的内容超过了缓冲区的内容，也没关系，会在这里将全部的内容读取到，所以对应我们的
  //utp来说，不用担心我们边处理数据，边收到内容
  size_t msgcount = 0;
  while (1) {
    //如果当前的下载速度，超过了超过了最大的速度限制，直接退出while循环，下次再来读取内容
    if (requestGroupMan_->doesOverallDownloadSpeedExceed() || downloadContext_->getOwnerRequestGroup()->doesDownloadSpeedExceed()) {
      LOGD("DefaultBtInteractive downloadSpeedExceed break");
      break;
    }
    //接受消息
    auto message = btMessageReceiver_->receiveMessage();
    if (!message) {
       break;
    }
    ...
}

```
限速的根本是
> 真正的接收消息的地方是在 btMessageReceiver_->receiveMessage(); 这里就不看具体的细节了， 可以看到上面的逻辑在接受消息之前会判断当前的是否超过了设置的下载速度，如果超过了就直接break掉所以当前本应该要接收的消息就没有立刻的接受了，而是等到了下次的接收，这里要注意这里并没有抛弃掉当前应该要接收的内容，而是根本就没有去接收，下次等到条件满足了再去接收数据，这样做是可以的主要是因为tcp的特性，tcp是一个`长连接`，所以可以做到这点


#### 上传速度限制

对应上传速度的使用，只有在DefaultBtMessageDispatcher 有做对应的处理

```CPP
void DefaultBtMessageDispatcher::sendMessagesInternal()
{
  auto tempQueue = std::vector<std::unique_ptr<BtMessage>>{};
  //从messsageQueue中移除先前添加进去的消息
  while (!messageQueue_.empty()) {
    //获取到messageQueue_ 中最前面的一个元素内容
    auto msg = std::move(messageQueue_.front());
    //移除集合messageQueue_中最前面的一个元素
    messageQueue_.pop_front();

    //如果当前是正在上传的消息
    if (msg->isUploading()) {
      if (requestGroupMan_->doesOverallUploadSpeedExceed() || downloadContext_->getOwnerRequestGroup()->doesUploadSpeedExceed()) {
         tempQueue.push_back(std::move(msg));
        continue;
      }
    }

    //这里只是创建对应的消息内容，然后填充到 socketBuffer_.pushBytes(std::move(data), std::move(progressUpdate))，也即是填充到缓冲区中
    msg->send();
  }

  if (!tempQueue.empty()) {
     messageQueue_.insert(std::begin(messageQueue_), std::make_move_iterator(std::begin(tempQueue)), std::make_move_iterator(std::end(tempQueue)));
  }
}

先来看看怎么样调用到这个方法  首先在PeerInteractionCommand,对应的case 为 WIRED 也即是到了最后一步互通资源状态

//command 命令的执行
bool PeerInteractionCommand::executeInternal()
{
    ...
    case WIRED:
      //互通对资源的意愿情况，包括interested、nointerested、choke、unchoke等4种  。 这边会一直的执行，下载，上传数据也是这里实现
      btInteractive_->doInteractionProcessing();

      //numReceivedMessage_     代表当前收到了多少条信息 ,就有必要保持连接 对于piece数据，如果只是收到了一小块的内容是不会发送这个消息的，只有完整的收到了一块数据
      //才会发送这个保持连接的消息,也即是ack
      if (btInteractive_->countReceivedMessageInIteration() > 0) {
          updateKeepAlive();
      }

      //判断当前的上传以及下载是否超过了最大的速度限制,这里会更新Peer的超时时间设置
      if (getDownloadEngine()->getRequestGroupMan()->doesOverallDownloadSpeedExceed() || requestGroup_->doesDownloadSpeedExceed()) {
         //如果超过了,设置读不可写
         disableReadCheckSocket();
         //设置不检查
         setNoCheck(true);
         LOGD("PeerInteractionCommand DownloadSpeedExceed checkPoint_ reset");
      }
      else {
        setReadCheckSocket(getSocket());
      }
      done = true;
      break;
    }
    ...
}

首先这个command会一直执行的，直到出现了错误，或者下载完成才会退出  btInteractive_->doInteractionProcessing(); 中有这样的实现
// 互通对资源的意愿情况，包括interested、nointerested、choke、unchoke等4种。
void DefaultBtInteractive::doInteractionProcessing()
{	
    ...
    //LOGD("DefaultBtInteractive doInteractionProcessing");
    //发送消息
    sendPendingMessage();
    ...
}

//发送等待中的消息 
void DefaultBtInteractive::sendPendingMessage() { dispatcher_->sendMessages(); }

而这个dispatcher就是 DefaultBtMessageDispatcher 所以会调用到对应的函数实现

//执行消息的发送
void DefaultBtMessageDispatcher::sendMessages()
{
   //A2_IOV_MAX 128 也即是如果内容还没有达到这个值，从消息队列中获取到消息，然后对应的创建对应的消息的内容，填充到缓冲区中，所以对应的peerConnection_->getBufferEntrySize() 就会加一
  //所以这边的逻辑可以达到一种效果，当缓冲区的内容当前达到了128 当前不会再往里面添加，下次再来添加，当前应该要先执行发送的逻辑
  if (peerConnection_->getBufferEntrySize() < A2_IOV_MAX) {
      sendMessagesInternal();
  }
  //这里执行发送缓冲区的内容
  peerConnection_->sendPendingData();
}

当数据包没有达到限制的时候，会调用sendMessagesInternal,而这个函数就是前面所说的，实际处理限制上传速度的地方,这里再次贴一下对应代码实现

void DefaultBtMessageDispatcher::sendMessagesInternal()
{
  auto tempQueue = std::vector<std::unique_ptr<BtMessage>>{};
  //从messsageQueue中移除先前添加进去的消息
  while (!messageQueue_.empty()) {
    //获取到messageQueue_ 中最前面的一个元素内容
    auto msg = std::move(messageQueue_.front());
    //移除集合messageQueue_中最前面的一个元素
    messageQueue_.pop_front();

    //如果当前是正在上传的消息
    if (msg->isUploading()) {
      if (requestGroupMan_->doesOverallUploadSpeedExceed() || downloadContext_->getOwnerRequestGroup()->doesUploadSpeedExceed()) {
         tempQueue.push_back(std::move(msg));
        continue;
      }
    }

    //这里只是创建对应的消息内容，然后填充到 socketBuffer_.pushBytes(std::move(data), std::move(progressUpdate))，也即是填充到缓冲区中
    msg->send();
  }

  if (!tempQueue.empty()) {
     messageQueue_.insert(std::begin(messageQueue_), std::make_move_iterator(std::begin(tempQueue)), std::make_move_iterator(std::end(tempQueue)));
  }
}

首先是messageQueue_ 这是一个消息的队列是一个未发送的消息队列，当要发送消息的时候，会先将消息添加到这个消息队列中，然后在这边执行消息的填充,这里看下msg->isUploading()代表的是什么意思

首先所有的Bt消息都是继承于这个AbstractBtMessage 基类 ，这个基类中定义了一个成员uploading_ 用来标识是否需要上传，默认的情况是false，下面看下什么时候会改成true
class AbstractBtMessage : public BtMessage {
private:
  bool invalidate_;
  //是否上传中，当我们给peer回应块的时候，这个值为true
  bool uploading_;
  ...  
}

首先看Bt消息接收创建的地方，具体的流程前面有分析过，这里直接分析 

//根据接受过来的数据， 创建BtMessage  这里是解析对方 互通对资源的意愿情况，包括interested、nointerested、choke、unchoke等4种。
std::unique_ptr<BtMessage>
DefaultBtMessageFactory::createBtMessage(const unsigned char* data, size_t dataLength)
{
    ...
    case BtRequestMessage::ID: {//创建request 类型的消息   也即是收到了对方的请求piece的消息
      auto m = BtRequestMessage::create(data, dataLength);
      if (!metadataGetMode_) {
          m->setBtMessageValidator(make_unique<RangeBtMessageValidator>(static_cast<BtRequestMessage*>(m.get()), downloadContext_->getNumPieces(), pieceStorage_->getPieceLength(m->getIndex())));
      }
      msg = std::move(m);
      break;
    }
    ...
}

上面的逻辑我们直接分析 request消息类型，也即是收到了对方的请求piece的消息

//解析传递过来的内容，然后构造对应的对象完成初始化
std::unique_ptr<BtRequestMessage>
BtRequestMessage::create(const unsigned char* data, size_t dataLength)
{
  return RangeBtMessage::create<BtRequestMessage>(data, dataLength);
}

在构建完这个消息对象之后，填充对应的参数 也即是会触发对应的doReceivedAction 函数

//接受传递过来的内容，给peer对应的成员赋值
void BtRequestMessage::doReceivedAction()
{
  //默认为fasle
  if (isMetadataGetMode()) {
    return;
  }
  //如果当前已经下载了这块的内容，并且本地不阻塞peer,
  if (getPieceStorage()->hasPiece(getIndex()) && (!getPeer()->amChoking() || getPeer()->isInAmAllowedIndexSet(getIndex()))) {
      //获取到接受到的内容，解析参数 然后 构建一个BtPieceMessage 对象，调用对应的构造函数完成初始化，然后添加到 messageQueue_ 队列中,等待发送
      getBtMessageDispatcher()->addMessageToQueue(getBtMessageFactory()->createPieceMessage(getIndex(), getBegin(), getLength()));
  }
  else {
    if (getPeer()->isFastExtensionEnabled()) {
        getBtMessageDispatcher()->addMessageToQueue(getBtMessageFactory()->createRejectMessage(getIndex(), getBegin(), getLength()));
    }
  }
}

也即是会执行 getBtMessageDispatcher()->addMessageToQueue(getBtMessageFactory()->createPieceMessage(getIndex(), getBegin(), getLength()));

//构建一个BtPieceMessage消息，这个消息是响应peer请求index所指定块的
//request消息包含index、begin、length三个字段。最后两个是字节偏移。length通常是2的指数除非是文件的最后一块。当前所有bt协议的实现版本中length的值是16kiB，关闭连接的request中length字段的值要大于16kiB
std::unique_ptr<BtPieceMessage>
DefaultBtMessageFactory::createPieceMessage(size_t index, int32_t begin,
                                            int32_t length)
{
  //构建一个BtPieceMessage 对象，调用对应的构造函数完成初始化
  auto msg = make_unique<BtPieceMessage>(index, begin, length);
  //设置当前种子的下载上下文对象
  msg->setDownloadContext(downloadContext_);
  //配置公有的属性
  setCommonProperty(msg.get());
  return msg;
}

//响应对方peer请求index 所指定的块的内容 构造函数的初始化
BtPieceMessage::BtPieceMessage(size_t index, int32_t begin, int32_t blockLength)
    : AbstractBtMessage(ID, NAME),
      index_(index),
      begin_(begin),
      blockLength_(blockLength),
      data_(nullptr),
      downloadContext_(nullptr),
      peerStorage_(nullptr)
{
  //设置上传中  ，这个可以用来区分当前是解析下载的内容，还是用来传递内容
  setUploading(true);
}

看到没有，这里就会将这个变量标识为true，标识为是一个需要上传的消息，继续回到DefaultBtMessageDispatcher
void DefaultBtMessageDispatcher::sendMessagesInternal()
{
  auto tempQueue = std::vector<std::unique_ptr<BtMessage>>{};
  //从messsageQueue中移除先前添加进去的消息
  while (!messageQueue_.empty()) {
    //获取到messageQueue_ 中最前面的一个元素内容
    auto msg = std::move(messageQueue_.front());
    //移除集合messageQueue_中最前面的一个元素
    messageQueue_.pop_front();

    //如果当前是正在上传的消息
    if (msg->isUploading()) {
	  //如果当前的消息是一个上传的消息，并且当前的上传速度已经超过了设置，就会将当前的消息添加到临时队列tempQueue中，跳过当前的消息
      if (requestGroupMan_->doesOverallUploadSpeedExceed() || downloadContext_->getOwnerRequestGroup()->doesUploadSpeedExceed()) {
         tempQueue.push_back(std::move(msg));
        continue;
      }
    }

    //这里只是创建对应的消息内容，然后填充到 socketBuffer_.pushBytes(std::move(data), std::move(progressUpdate))，也即是填充到缓冲区中
    msg->send();
  }

  //将临时消息队列tempQueue  中的内容，添加回messageQueue_ 中
  if (!tempQueue.empty()) {
     messageQueue_.insert(std::begin(messageQueue_), std::make_move_iterator(std::begin(tempQueue)), std::make_move_iterator(std::end(tempQueue)));
  }
}
```
上传限速的根本是
> 首先会判断当前要发送的消息类型，如果是一个上传的类型，比如是 对应的request消息，我们要给对方回应数据，这里就要去判断是否超过了上传的限制，其他的消息 不做任何的处理，针对当前要发送的上传消息，会先临时添加到tempQueue 队列中，并且跳过当前的消息发送，最终又会添加到messageQueue_ 总的消息队列中，等下次再次的触发

### Utp限速的处理
utp下载限速
> 前面已经分析了Tcp的限速处理，对于tcp的下载限速处理也是比较简单的，在接收Bt消息的时候，先判断当前是否已经超过了最大的下载速度，如果超过了，当前就不去读取，等到下次条件满足的时候再去读取对应Tcp来说，这是可以做到的，因为他是一个`长连接`，但是如果对应utp，本质也即是udp来说，他是一个`无连接`的过程，如果对方将数据发送过来了，当前你不去接收他，你想通过tcp的处理那样下次再来接受是不可能的，这样做的结果就是相当于把当前的数据丢掉了，下次还要对方发送数据，这样会增加对方的带宽，这是不合理的，当然有人可能会说将当前收到的内容存储起来，等下次条件满足了再去读取存储的内容虽然能做出`类似的限速效果`，但其实是没有用的，没有办法做到`平衡带宽`的作用，对应网速很快的用户，也是会很快把带宽吃掉，导致其他的用户下载变慢，所以对应udp 来减小带宽的化，估计是做不到的

utp的上传限速
>对应tcp上传限速的处理算是比较简单的，通过判断当前的消息类型是否是一个上传的类型也即是request类型，再进一步判断当前的上传速度是否超过了限制，如果超过了那当前的这条消息就不先发送，先存储到消息队列中，下次再来发送，所以对应utp来说，这个上传的限速也是支持的，所以既然我们不能在下载的时候做utp的限速处理，那么我们在上传的这一端做处理，这样就能控制每个用户的下载速度，也是一种`间接`的实现控制带宽


