---
layout: pager
title: Aria2 Bt 辅助 Http下载实现
date: 2019-03-11 16:41:21
tags: [NDK,Aria2]
description:  Aria2 Bt 辅助 Http下载实现
---
### 简介
> 上一年12月份的时候将修改后的Aria2 移植到Window 上面，做了一个简单的下载器，看看下载的情况，大体测试了一个月左右，虽然 Bt 下载表现的不错，每天都能省下一半多的流量，但是通过下载的成功率用户的留存，发现都普遍低于传统的Http下载，大体的原因有，Bt 获取到Peer的过程存在不确定性，比如 Tracker服务器挂掉，虽然有Dht实现，但是dht 寻找可用的节点太慢，再次就是就算获取到了Peer也无法确定哪些Peer是有效的，也即是有哪些Peer是支持打洞的，哪些是不支持的，而且目前对于要连接的Peer数量还有限制，再次就是Bt的下载速度是跟同一时刻用户量有关的，如果同一时刻用户量大的化那么Bt的下载速度就会上升，相反Bt下载速度就会极低，这就会影响用户的使用，影响用户的留存等，针对这种情况，我去分析了解下 英雄联盟(LOL) 下载器，发现他是通过Bt 辅助 Http来实现的

LOL 下载器效果
![结果显示](/uploads/Aria2Bt辅助Http下载/LOL下载器.png)
通过观察可以发现蓝色的为Http 下载的速度，这里估计实现了限速，因为从不会超过400kb/s，后面红色的为Bt下载的速度，由于实现了Bt 辅助Http下载，对用户的体验就会更好，用户不会因为没有速度或者下载速度太低导致的不想下载，这个可以通过Http下载来解决，对应Bt 下载算一种额外的收益，所以基于这种Bt 辅助 Http下载才是最终的实现

下面是我基于Aria2实现的效果
![结果显示](/uploads/Aria2Bt辅助Http下载/Aria2Bt辅助Http下载实现.png)


### Bt辅助Http下载实现难点
> 1.由于采用的是Aria2，原本是不支持Bt 跟Http 想结合下载的，他们俩个是完全不相关的
2.原本的Http下载由于一开始不知道要下载的内容的长度所以第一个请求是获取要下载内容的长度的，知道之后才会进行文件的预分配，而对于Bt下载来说由于存在Bt种子，一开始就知道了要下载内容的长度所以直接就进行文件的分配，所以当我们结合实现的化，可以省掉Http的第一个请求下载长度的request，对于文件分配，也只需要一方来执行文件的分配,这里由Bt来分配，Http不分配文件，直接跳过
3.资源片的问题，怎么动态的调整Bt和Http下载块的索引
4.文件描述符的问题，如果有多个地方文件描述法访问同一个文件，轮流的往这个文件读写内容，可能会导致读写的错误，
5.Bt跟Http通信的问题，比如Http 下载完应该告诉Bt 当前块下载完了，BT 此时应该发送have 消息等
6.相应的进度的保存，也即是当前下载的长度，下次进来应该继续下载
7.多文件的下载实现


### Bt辅助Http 难点解决方案
1.由于采用的是Aria2，原本是不支持Bt 跟Http 想结合下载的，他们俩个是完全不相关的
```cpp
我们可以在RequestGroup 中添加另一个RequestGroup
//当前RequestGroup 绑定的另一个RequestGroup 对象
std::shared_ptr<RequestGroup> otherRequest_;

//并提供对应的get，set 方法,设置其他下载的请求
void RequestGroup::setOtherRequestGroup(std::shared_ptr<RequestGroup>& otherRequest)
{
    otherRequest_ = otherRequest;
}

std::shared_ptr<RequestGroup>& RequestGroup::getOtherRequestGroup()
{
    return otherRequest_;
}

//注意对于Http的下载请求，这里就不需要添加到RequestGroupMan中了 ,而是这里要完成Bt任务跟Http任务的绑定,这边由于采用的是智能指针，而且是局部变量所以不能使用引用
//addRequestGroup(result.front(), e.get(), 0);
btGroup->setOtherRequestGroup(result.front());
//注意这里是互相持有的状态
result.front()->setOtherRequestGroup(btGroup);
```

2.在Http 不请求对应的下载内容的长度的时候，直接知道 要下载的总长度，这个我们可以通过Bt下载获取到对应的信息
```cpp
//在创建 Bt辅助 Http 下载任务的时候, 将Bt的FileEntry设置到Http中
result.front()->getDownloadContext()->setFileEntries(btContext->getFileEntries().begin(), btContext->getFileEntries().end());
//将Http 要下载的 url 保存 , 对于Bt 来说 FileEntry 中的uri 为空，而 Http下载来说 FileEntry 中的 uri 集合为存储要下载的地址 这里只是下载单个文件，多个文件要额外的处理
//result.front()->getDownloadContext()->getFirstFileEntry()->setUris(uris);
//将Bt 所拥有的Hash 字段 也一起保存到 Http 下载中，这样当Http下载完成的时候，也可以用于校验,也即是 Http 下载也拥有了Bt 的校验功能
result.front()->getDownloadContext()->setPieceHashes(btContext->getPieceHashType(),btContext->getPieceHashes().begin(),btContext->getPieceHashes().end());

由于我们Http下载任务是绑定在Bt上面，所以我们需要手动的执行这个Http RequestGroup,而且我们知道一个Bt 任务的初始化，是在Bt.setUp函数中执行的，所以我们可以在这个方法中添加下面的逻辑

//这里要判断是否是Bt辅助Http下载 TODO  执行Http的初始化操作等
auto& httpRequestGroup = requestGroup->getOtherRequestGroup();
if(httpRequestGroup)
{
  //初始化PieceStorage等成员变量,这里传递Bt对应的 BitfieldMan对象，这样对于Http，和Bt下载来说，就可以共用这个对象, Diskadaptor 为写文件的对象，这里也要共用
  httpRequestGroup->initPieceStorage(pieceStorage->getBitfieldMan(),pieceStorage->getDiskAdaptor());
  LOGD("BtSetup httpRequestGroup  processCheckIntegrityEntry begin ");

  //检查是否本地的下载进度文件，加载进度文件，决定当前要下载的长度，或者本身就下载完成了
  auto checkEntry = httpRequestGroup->createCheckIntegrityEntry(false);
  if (checkEntry) {
     //第三个参数表示， 对于 Http下载 没有必要再次的分配文件的大小
     httpRequestGroup->processCheckIntegrityEntry(commands, std::move(checkEntry), e,false);
  }
  LOGD("BtSetup httpRequestGroup processCheckIntegrityEntry init finish");
}
```
针对 问题3,4 我们可以简单的让Http RequestGroup 共享 Bt 对应的对象，因为通过源码分析他们在这方面是相同的处理
```cpp
httpRequestGroup->initPieceStorage(pieceStorage->getBitfieldMan(),pieceStorage->getDiskAdaptor());

//初始化PieceStorage
void RequestGroup::initPieceStorage(const std::shared_ptr<BitfieldMan>& bitfieldMan,const std::shared_ptr<DiskAdaptor>& diskAdaptor)
{
    ...
    //构建一个DefaultPieceStorage 对象，调用对应的构造函数完成初始化,内部会创建对应的 BitfieldMan 对象，以及 DiskWriterFactory 引用对象等 TODO
    auto ps = std::make_shared<DefaultPieceStorage>(downloadContext_, option_.get());
    //如果bitfieldMan不为空
    if(bitfieldMan)
    {
       ps->setSharedBitfiledMan(bitfieldMan);
    }
    ....
    //TODO 如果 diskAdaptor 不为空，说明是Bt 辅助 Http 下载 TODO 保存Bt 传递过来的 DiskAdaptor对象 ，这个对象负责写文件的操作 ,这样就没有必要执行  initStorage 操作
    if(diskAdaptor)
    {
       tempPieceStorage->setDiskAdaptor(diskAdaptor);
    } else{
       //执行初始化操作 ,默认 tempPieceStorage 对象为  DefaultPieceStorage TODO Bt对象传递到Http 对象
       tempPieceStorage->initStorage();
    }
    ....
}
```

针对怎么样避开Http RequestGroup 不执行文件的分配
```cpp
//检查是否本地的下载进度文件，加载进度文件，决定当前要下载的长度，或者本身就下载完成了
auto checkEntry = httpRequestGroup->createCheckIntegrityEntry(false);
if (checkEntry) {
    //第三个参数表示， 对于 Http下载 没有必要再次的分配文件的大小
    httpRequestGroup->processCheckIntegrityEntry(commands, std::move(checkEntry), e,false);
}

//添加对应的 StreamFileAllocationEntry 对象，用于检查是否需要分配文件
void StreamCheckIntegrityEntry::onDownloadIncomplete(std::vector<std::unique_ptr<Command>>& commands, DownloadEngine* e,bool isNeedCheckFileSize)
{
  //为 DefaultStorage 对象
  auto& ps = getRequestGroup()->getPieceStorage();
  ps->onDownloadIncomplete();

  //hash-check-only 选项 默认为 false
  if (getRequestGroup()->getOption()->getAsBool(PREF_HASH_CHECK_ONLY)) {
     return;
  }

  //处理文件的提前分配
  proceedFileAllocation(commands, make_unique<StreamFileAllocationEntry>(getRequestGroup(), popNextCommand()), e,isNeedCheckFileSize);
}


//处理文件的提前分配
void CheckIntegrityEntry::proceedFileAllocation(std::vector<std::unique_ptr<Command>>& commands, std::unique_ptr<FileAllocationEntry> entry, DownloadEngine* e,bool isNeedCheckFileSize)
{
  //isNeedCheckFileSize 默认为true，只有在 Bt 辅助 Http 下载的时候，这里会设置为false，此时不用再次的检查
  if(!isNeedCheckFileSize)
  {
      //文件已经分配完成，直接跳过分配文件的操作
      LOGD("CheckIntegrityEntry proceedFileAllocation !isNeedCheckFileSize no Need FileAllocation prepareForNextAction");
      entry->prepareForNextAction(commands, e);
      return;
  }

  //判断是否需要文件分配,比如是否分配完成了等
  if (getRequestGroup()->needsFileAllocation()) {
    //如果需要，就将entry添加到fileAllocationMan_ ,  FileAllocationMan 这个为全局唯一的 负责管理文件的分配 typedef SequentialPicker<FileAllocationEntry> FileAllocationMan;
    // 这里的entry 实体为 StreamFileAllocationEntry 类型  ,如果这里需要分配文件，这里会将entry 转移到  entries_ 集合中
    e->getFileAllocationMan()->pushEntry(std::move(entry));
  }
  else {
    //文件已经分配完成，直接跳过分配文件的操作
    LOGD("CheckIntegrityEntry proceedFileAllocation no Need FileAllocation prepareForNextAction");
    entry->prepareForNextAction(commands, e);
  }
}

原本的逻辑是会执行 getRequestGroup()->needsFileAllocation() 函数，动态的根据目标文件的实际长度来决定是否需要文件
//判断是否需要文件分配
bool RequestGroup::needsFileAllocation() const
{
  //pieceStorage_->getDiskAdaptor()->fileAllocationIterator() 这里每次调用都会构建一个新的对象
  //PREF_NO_FILE_ALLOCATION_LIMIT 对大小小于SIZE的文件不进行文件分配。 您可以附加K或M（1K = 1024，1M = 1024K）。 默认：5M
  return isFileAllocationEnabled() && option_->getAsLLInt(PREF_NO_FILE_ALLOCATION_LIMIT) <= getTotalLength() && !pieceStorage_->getDiskAdaptor()->fileAllocationIterator()->finished();
}

pieceStorage_->getDiskAdaptor()->fileAllocationIterator()函数实现

//每次调用这个方法，都会重新的构建一个 FileAllocationIterator 对象 返回回去，而且此时还要获取到当前目标文件的大小 通过 size() 函数
std::unique_ptr<FileAllocationIterator>
AbstractSingleDiskAdaptor::fileAllocationIterator()
{
  switch (getFileAllocationMethod()) {
#ifdef HAVE_SOME_FALLOCATE
  case (DiskAdaptor::FILE_ALLOC_FALLOC):
    return make_unique<FallocFileAllocationIterator>(diskWriter_.get(), size(),
                                                     totalLength_);
#endif // HAVE_SOME_FALLOCATE
  case (DiskAdaptor::FILE_ALLOC_TRUNC):
    return make_unique<TruncFileAllocationIterator>(diskWriter_.get(), size(), totalLength_);
  default:
    //默认会构建一个 AdaptiveFileAllocationIterator 对象  size() 获取到当前目标文件的实际长度，也即是在原本的长度上进行分配,totalLength_ 总的目标长度
    return make_unique<AdaptiveFileAllocationIterator>(diskWriter_.get(), size(), totalLength_);
  }
}

而第二个参数获取到当前目标文件的实际大小是通过
//获取到当前目标文件的实际大小
int64_t AbstractSingleDiskAdaptor::size()
{
    int64_t fileSize = File(getFilePath()).size();
    //LOGD("AbstractSingleDiskAdaptor getFile Size path %s fileSize == %lld",getFilePath().c_str(),fileSize);
    return fileSize;
}

int64_t File::size()
{
#ifdef __MINGW32__
  // _wstat cannot be used for symlink.  It always returns 0.  Quoted
  // from https://msdn.microsoft.com/en-us/library/14h5k7ff.aspx:
  //
  //   _wstat does not work with Windows Vista symbolic links. In
  //   these cases, _wstat will always report a file size of 0. _stat
  //   does work correctly with symbolic links.
  auto hn = openFile(name_);
  if (hn == INVALID_HANDLE_VALUE) {
    return 0;
  }
  LARGE_INTEGER fileSize;
  const auto rv = GetFileSizeEx(hn, &fileSize);
  CloseHandle(hn);
  return rv ? fileSize.QuadPart : 0;
#else  // !__MINGW32__
  a2_struct_stat fstat;
  if (fillStat(fstat) < 0) {
    return 0;
  }
  return fstat.st_size;
#endif // !__MINGW32__
}

但是按照我的观察，在确保目标文件路径是正确的情况下，这里获取到的长度确有时候获取不到，有时候又可以获取到，为了避免这种麻烦，所以我增加了一个参数 isNeedCheckFileSize，直接跳过
```
针对 6.相应的进度的保存，也即是当前下载的长度，下次进来应该继续下载
```cpp
原本Http跟Bt 下载都是有进度保存的，而且会每隔一段时间触发一次进度的保存，而且使用的都是 DefaultBtProgressInfoFile ，所以我们完全没有必要俩边都保存，我的做法不调用原本要调用的
//保存进度对象的引用
setProgressInfoFile(progressInfoFile);

这样对于Http下载内部就会持有一个空的进度保存对象
progressInfoFile_(std::make_shared<NullProgressInfoFile>()),//首先构造一个NullProgressInfoFile对象，调用对应的构造函数，并将引用赋值给progressInfoFile_

这个对象是不会进行任何的操作的
class NullProgressInfoFile : public BtProgressInfoFile {
public:
  virtual ~NullProgressInfoFile() = default;

  virtual std::string getFilename() CXX11_OVERRIDE { return A2STR::NIL; }

  virtual bool exists() CXX11_OVERRIDE { return false; }

  virtual void save() CXX11_OVERRIDE {}

  virtual void load() CXX11_OVERRIDE {}

  virtual void removeFile() CXX11_OVERRIDE {}

  virtual void updateFilename() CXX11_OVERRIDE {}
};
```
针对 5.Bt跟Http通信的问题，比如Http 下载完应该告诉Bt 当前块下载完了，BT 此时应该发送have 消息等,我们可以在Http 下载完一块的时候，手动的通知Bt
```cpp
//标识当前  segment 已经下载完成了， 要告知 PieceStorage 当前这块piece 下载完成了,然后从  usedSegmentEntries_ 集合中移除这个元素
bool SegmentMan::completeSegment(cuid_t cuid, const std::shared_ptr<Segment>& segment,std::shared_ptr<PieceStorage> btPieceStorage)
{
  //告知PieceStorage 当前这块piece 下载完成了
  pieceStorage_->completePiece(segment->getPiece());
  //将当前下载完的piece添加到 已经下载完的集合中
  pieceStorage_->advertisePiece(cuid, segment->getPiece()->getIndex(), global::wallclock());

  //TODO 由于BT跟Http下载不使用同一个DefaultPieceStorage对象，所以当Http下载完，要手动的通知Bt下载完成,Bt才能走原本的逻辑发送他当前拥有的Piece
  if(btPieceStorage)
  {
      btPieceStorage->advertisePiece(cuid, segment->getPiece()->getIndex(), global::wallclock());
      LOGD("SegmentMan completeSegment tell bt piece %d is DownloadSuccess",segment->getPiece()->getIndex());
  }
  
  //然后从  usedSegmentEntries_ 集合中 移除 当前下载成功的 Segment
  auto itr = std::find_if(usedSegmentEntries_.begin(), usedSegmentEntries_.end(), FindSegmentEntry(segment));
  //没有找到，返回false
  if (itr == usedSegmentEntries_.end()) {
    return false;
  }
  else {
    //找到了移除这个 元素
    usedSegmentEntries_.erase(itr);
    return true;
  }
}

在DownloadCommand中有这样的逻辑
//针对Http，ftp，sftp下载的Comand基类的执行
bool DownloadCommand::executeInternal()
{
  ...
  //如果一块完成了,对应的也即是以piece
  if (segmentPartComplete) {
    //判断是否这个piece 块下载完成了
    if (segment->complete() || segment->getLength() == 0) {
      // If segment->getLength() == 0, the server doesn't provide
      // content length, but the client detected that download
      // completed.
      //打印The download for one segment completed successfully 下载一块完整的信息
      LOGD(MSG_SEGMENT_DOWNLOAD_COMPLETED, getCuid());
      {
        //TODO 获取到期望的Hash值,这个只有Bt才有 pieceHash 值，也即是只有种子才要对比是否hash值相等,当然对应Bt辅助Http下载来说，我们在前面就将Bt的Hash字段保存到了 Http下载实体里面
        //TODO 这样Http也可以既有 校验的功能，前提是 对应Bt下载的piece 长度一定要跟Http下载一样，默认Http下载为1M
        const std::string& expectedPieceHash = getDownloadContext()->getPieceHash(segment->getIndex());

        // pieceHashValidationEnabled_ 默认为 true ?
        if (pieceHashValidationEnabled_ && !expectedPieceHash.empty()) {
          if (
#ifdef ENABLE_BITTORRENT
              // TODO Is this necessary?   确保不为种子下载， 貌似种子下载不通过 DownloadCommand来触发
              (!getPieceStorage()->isEndGame() || !getDownloadContext()->hasAttribute(CTX_ATTR_BT)) && //如果当前piece 已经计算过hash 值了
#endif // ENABLE_BITTORRENT
            segment->isHashCalculated()) {
            //LOGD("DownloadCommand Hash is available! index=%lu", static_cast<unsigned long>(segment->getIndex()));
            //检验hash值,segment->getDigest() 为当前piece 所对应的hash值 ,检验不通过会发生异常
            validatePieceHash(segment, expectedPieceHash, segment->getDigest());
          }
          else {
            //当前piece 还没有校验过 hash 值 当前完成计算，然后比较hash值
            try {
              std::string actualHash = segment->getPiece()->getDigestWithWrCache(segment->getSegmentLength(), diskAdaptor);
              validatePieceHash(segment, expectedPieceHash, actualHash);
            }
            catch (RecoverableException& e) {
              //比较出现了异常，清除处理
              segment->clear(getPieceStorage()->getWrDiskCache());
              getSegmentMan()->cancelSegment(getCuid());
              throw;
            }
          }
        }
        else {
            //对应Http下载来说，不用检验
            LOGD("DownloadCommand expectedPieceHash empty done validatePieceHash segmentPartComplete");
            completeSegment(getCuid(), segment);
        }
      }
    }
    ...
}

原本的Http下载来说由于不具备种子文件，所以没有办法校验Piece值，但是我们通过Bt 辅助 Http 下载来说我们是有种子文件的，所以我们完全可以让Http下载也走Piece的校验
//TODO 将Bt 所拥有的Hash 字段 也一起保存到 Http 下载中，这样当Http下载完成的时候，也可以用于校验,也即是 Http 下载也拥有了Bt 的校验功能
result.front()->getDownloadContext()->setPieceHashes(btContext->getPieceHashType(),btContext->getPieceHashes().begin(),btContext->getPieceHashes().end());
```
针对 Bt辅助 Http下载 暂停，恢复，移除也要做特殊的处理，特殊在我们需要控制对应的Http RequestGroup的行为
```cpp
//forcePause 代表是否强制暂停,默认的为false
bool pauseRequestGroup(const std::shared_ptr<RequestGroup>& group, bool reserved, bool forcePause)
{
  if ((reserved && !group->isPauseRequested()) || (!reserved && !group->isForceHaltRequested() && ((forcePause && group->isHaltRequested() && group->isPauseRequested()) || (!group->isHaltRequested() && !group->isPauseRequested())))) {
    if (!reserved) {
      // Call setHaltRequested before setPauseRequested because  setHaltRequested calls setPauseRequested(false) internally.
      //如果是强制暂停
      if (forcePause) {
          //如果当前任务还有额外的任务，说明是Bt 辅助 Http 下载，暂停的时候，要手动的暂停Http 下载
          if(group->getOtherRequestGroup())
          {
              group->getOtherRequestGroup()->setForceHaltRequested(true, RequestGroup::NONE);
          }
          //设置当前的请求强制停止
          group->setForceHaltRequested(true, RequestGroup::NONE);
      }
      else {
          //如果当前任务还有额外的任务，说明是Bt 辅助 Http 下载，暂停的时候，要手动的暂停Http 下载
          if(group->getOtherRequestGroup())
          {
              group->getOtherRequestGroup()->setHaltRequested(true, RequestGroup::NONE);
          }
          //设置当前请求停止
          group->setHaltRequested(true, RequestGroup::NONE);
      }
    }

    //如果当前任务还有额外的任务，说明是Bt 辅助 Http 下载，暂停的时候，要手动的暂停Http 下载
    if(group->getOtherRequestGroup())
    {
        //设置为暂停的状态
       group->getOtherRequestGroup()->setPauseRequested(true);
    }
    //设置当前的group的状态为暂停状态
    group->setPauseRequested(true);
    return true;
  }
  else {
    return false;
  }
}

可以看到这里只是通过设置一个状态，那么这个状态是哪里出发的呢

//移除停止的请求组 requetGroup
void RequestGroupMan::removeStoppedGroup(DownloadEngine* e)
{
  size_t numPrev = requestGroups_.size();//获取到当前的请求组的数量
  //检查当前正在请求的 任务组集合 requestGroups_ 将处于暂停的任务，添加到  reservedGroups_ 中
  requestGroups_.remove_if(ProcessStoppedRequestGroup(e, reservedGroups_));
  //打印当前停止的数量
  size_t numRemoved = numPrev - requestGroups_.size();
  if (numRemoved > 0) {
    LOGD("%lu RequestGroup(s) deleted.", static_cast<unsigned long>(numRemoved));
  }
}

是通过RequestGroupMan 执行对应的 removeStoppedGroup 函数执行相应的操作,而至于哪里调用 FillRequestGroupCommand command执行的时候触发，这个command只有在引擎退出的时候才会销毁
所以会一直触发这个函数检查对应的状态

//返回值代表当前RequestGroup是否停止成功，如果为false代表当前 RequestGroup 没有停止成功，比如当前RequestGroup还有Command没有销毁，必须要等待都销毁才返回true
//所以当返回为false的时候，并不会从 当前正在请求的集合里面销毁，会下次再次触发检查
bool operator()(const RequestGroupList::value_type& group)
{
   if(group->getOtherRequestGroup())
   {
      //TODO 多次处理是否会有问题
      bool checkResult = checkStopProcess(group->getOtherRequestGroup(),true);
      LOGD("ProcessStoppedRequestGroup extraRequestGroup checkStopProcess finish");
      if(checkResult)
      {
         return checkStopProcess(group,false);
      }
      return checkResult;
   }else{
      //普通的下载
      return checkStopProcess(group,false);
   }
}

bool checkStopProcess(const RequestGroupList::value_type &group,bool isExtraGroup) {
            //如果当前任务组的 Command数量为0 ,代表内部的Command都销毁了,才能执行后面的暂停等操作
            if (group->getNumCommand() == 0) {
                collectStat(group);
                const std::shared_ptr<DownloadContext> &dctx = group->getDownloadContext();

                //TODO  对于 Bt 辅助 Http 下载 ，不应该调用 decreaseNumActive 减少当前的任务数量，因为对于额外的任务，是我们启动的，并没有走原本的逻辑
                if(!isExtraGroup)
                {
                    if (!group->isSeedOnlyEnabled()) {
                        e_->getRequestGroupMan()->decreaseNumActive();
                    }
                }

                // DownloadContext::resetDownloadStopTime() is only called when
                // download completed. If
                // DownloadContext::getDownloadStopTime().isZero() is true, then
                // there is a possibility that the download is error or
                // in-progress and resetDownloadStopTime() is not called. So
                // call it here.
                //首先重置下载停止的时间
                if (dctx->getDownloadStopTime().isZero()) {
                    dctx->resetDownloadStopTime();
                }

                try {
                    //TODO Bt 辅助 Http 下载，进度的保存交给 Bt的RequestGroup来处理
                    if(!isExtraGroup)
                    {
                        //关闭当前任务组的文件
                        group->closeFile();
                        //如果当前任务组为暂停的状态
                        if (group->isPauseRequested()) {
                            if (!group->isRestartRequested()) {
                                A2_LOG_NOTICE(fmt(_("Download GID#%s paused"), GroupId::toHex(group->getGID()).c_str()));
                            }
                            //保存进度文件
                            group->saveControlFile();
                        } else if (group->downloadFinished() && !group->getDownloadContext()->isChecksumVerificationNeeded()) {//如果到了这里就说明当前任务不为暂停的状态,并且下载已经完成了,也即是因为完成了导致停止
                            group->applyLastModifiedTimeToLocalFiles();
                            group->reportDownloadFinished();
                            //如果当前任务组 下载完成了，移除进度文件，
                            if (group->allDownloadFinished() &&
                                !group->getOption()->getAsBool(PREF_FORCE_SAVE)) {
                                group->removeControlFile();
                                saveSignature(group);
                            } else {
                                //当前任务组还没有全部下载完，保存进度
                                group->saveControlFile();
                            }

                            std::vector<std::shared_ptr<RequestGroup>> nextGroups;
                            group->postDownloadProcessing(nextGroups);
                            if (!nextGroups.empty()) {
                                //A2_LOG_DEBUG(fmt("Adding %lu RequestGroups as a result of"" PostDownloadHandler.", static_cast<unsigned long>(nextGroups.size())));
                                e_->getRequestGroupMan()->insertReservedGroup(0, nextGroups);
                            }
#ifdef ENABLE_BITTORRENT
                            // For in-memory download (e.g., Magnet URI), the
                            // FileEntry::getPath() does not return actual file path, so
                            // we don't remove it.
                            if (group->getOption()->getAsBool(PREF_BT_REMOVE_UNSELECTED_FILE) &&
                                !group->inMemoryDownload() && dctx->hasAttribute(CTX_ATTR_BT)) {
                                A2_LOG_INFO(fmt(MSG_REMOVING_UNSELECTED_FILE, GroupId::toHex(group->getGID()).c_str()));
                                const std::vector<std::shared_ptr<FileEntry>> &files = dctx->getFileEntries();
                                for (auto &file : files) {
                                    if (!file->isRequested()) {
                                        if (File(file->getPath()).remove()) {
                                            A2_LOG_INFO(fmt(MSG_FILE_REMOVED, file->getPath().c_str()));
                                        } else {
                                            A2_LOG_INFO(fmt(MSG_FILE_COULD_NOT_REMOVED, file->getPath().c_str()));
                                        }
                                    }
                                }
                            }
#endif // ENABLE_BITTORRENT
                        } else {
                            //保存进度文件
                            LOGD(_("Download GID#%s not complete: %s"), GroupId::toHex(group->getGID()).c_str(), group->getDownloadContext()->getBasePath().c_str());
                            group->saveControlFile();
                        }
                    }
                }
                catch (RecoverableException &ex) {
                    A2_LOG_ERROR_EX(EX_EXCEPTION_CAUGHT, ex);
                }

                //如果当前任务组为暂停状态
                if (group->isPauseRequested()) {
                    //重新的设置当前任务组的状态为   默认的状态 即为 STATE_WAITING
                    group->setState(RequestGroup::STATE_WAITING);
                    //TODO 然后将当前暂停的任务，再次的添加到了 reservedGroups_ 集合中,下次 FillRequestGroupCommand的时候会再次的检查时候有必要执行任务,Bt辅助 Http下载 ，额外的 RequestGroup不应该添加到这个集合中
                    if(!isExtraGroup)
                    {
                        reservedGroups_.push_front(group->getGID(), group);
                    }

                    //释放当前任务组的 运行时资源
                    group->releaseRuntimeResource(e_);
                    //设置不为强制停止
                    group->setForceHaltRequested(false);

                    //更改参数配置
                    auto pendingOption = group->getPendingOption();
                    if (pendingOption) {
                        changeOption(group, *pendingOption, e_);
                    }

                    if (group->isRestartRequested()) {
                        group->setPauseRequested(false);
                    } else {
                        //TODO 通知暂停的事件, Bt 辅助 Http下载 额外的 RequestGroup 不应该触发回调
                        if(!isExtraGroup)
                        {
                            util::executeHookByOptName(group, e_->getOption(), PREF_ON_DOWNLOAD_PAUSE);
                            notifyDownloadEvent(EVENT_ON_DOWNLOAD_PAUSE, group);
                        }
                    }
                    // TODO Should we have to prepend spend uris to remaining uris
                    // in case PREF_REUSE_URI is disabled?
                } else {
                    // TODO 如果到了这里就为退出的事件，比如当前正在点击移除的操作， 这里我们应该要将 彼此双方的RequestGroup引用置为0 TODO 要不然对应的RequestGroup 不会执行销毁,造成内存泄漏
                    //将额外的RequestGroup 引用置为null,这里只有在真正退出的时候，才能重置掉他们之间的引用
                    if(group->isHaltRequested())
                    {
                        group->resetOtherRequestGroup();
                        LOGD("RequestGroupMan checkStopProcess !isPauseRequested resetOtherRequestGroup");
                    }
                    // 通知暂停的事件, Bt 辅助 Http下载 额外的 RequestGroup 不应该触发回调，
                    if(!isExtraGroup)
                    {
                        //当前RequestGroup 要停止了，根据当前任务组的信息，封装成一个DownloadResult对象
                        std::shared_ptr<DownloadResult> dr = group->createDownloadResult();
                        //添加到下载结果集合中
                        e_->getRequestGroupMan()->addDownloadResult(dr);
                        executeStopHook(group, e_->getOption(), dr->result);
                    }
                    //释放当前任务组的 运行时资源
                    group->releaseRuntimeResource(e_);
                    LOGD("RequestGroupMan checkStopProcess !isPauseRequested releaseRuntimeResource");
                }

                group->setRestartRequested(false);
                group->setPendingOption(nullptr);
                //返回true，当前请求 停止成功
                return true;
            }
            return false;
}

对于删除任务来说是类似的操作
//移除指定gid 对应的RequestGroup，force 代表是否强制执行
int removeDownload(Session* session, A2Gid gid, bool force)
{
  auto& e = session->context->reqinfo->getDownloadEngine();
  //根据gid 从 当前正在请求的集合，等待的集合中找到对应的RequestGroup
  std::shared_ptr<RequestGroup> group = e->getRequestGroupMan()->findGroup(gid);
  if (group) {
    //如果当前任务正在下载
    if (group->getState() == RequestGroup::STATE_ACTIVE) {
      if (force) {
        //如果当前任务还有额外的任务，说明是Bt 辅助 Http 下载，移除的时候，要设置额外的任务为强制退出的状态
        if(group->getOtherRequestGroup())
        {
           group->getOtherRequestGroup()->setForceHaltRequested(true, RequestGroup::USER_REQUEST);
        }
        group->setForceHaltRequested(true, RequestGroup::USER_REQUEST);
      }
      else {
        //如果当前任务还有额外的任务，说明是Bt 辅助 Http 下载，移除的时候，要设置额外的任务为退出的状态
        if(group->getOtherRequestGroup())
        {
            group->getOtherRequestGroup()->setHaltRequested(true, RequestGroup::USER_REQUEST);
        }
        group->setHaltRequested(true, RequestGroup::USER_REQUEST);
      }
      e->setRefreshInterval(std::chrono::milliseconds(0));
    }
    else {
      //如果当前任务不处于下载中，比如暂停中，那么直接简单的 在 RequestGroupMan 中移除当前gid 对应的 RequestGroup
      if (group->isDependencyResolved()) {
        e->getRequestGroupMan()->removeReservedGroup(gid);
      }
      else {
        return -1;
      }
    }
  }
  else {
    return -1;
  }
  return 0;
}
```
在 HttpRequestCommand 中有这样的逻辑
```cpp
//command执行
bool HttpRequestCommand::executeInternal()
{
    ...
    LOGD("CUID#%" PRId64 " !getSegments().empty(), getSegments.size() %d",getCuid(),getSegments().size());
    //如果当前请求的 segments_ 集合不为空, 这种情况针对的是上一次请求下载的片段的内容下载完成了，之后重新的获取到下一次要请求piece对应的Segment(跟cuid 绑定),同时添加到了 usedSegmentEntries_ 集合中
    //父类AbstractCommand 会调用  sm->getInFlightSegment(segments_, getCuid()); 将当前cuid 对应的segment 获取到,这里就开始请求这个piece 对应的内容
      for (auto& segment : getSegments()) {
        //确保当前连接 outstandingHttpRequests_ 正在请求的集合中，没有跟当前Segment 同样的元素，防止重复请求
        if (!httpConnection_->isIssued(segment)) {
          //找到了没有下载过的segment
          int64_t endOffset = 0;
          // FTP via HTTP proxy does not support end byte marker 不是ftp下载
          if (getRequest()->getProtocol() != "ftp" && getRequestGroup()->getTotalLength() > 0 && getPieceStorage()) {
            //TODO 获取到当前要下载的piece 索引,也即是返回当前索引对应的下一个终止的索引 这里会尽力的获取到连续没有下载的Piece 比如 1到5中，五已经下载过，那么这里返回的就是 5，也即是要下载 2,3,4的内容
            //TODO 由于前面有连接池的使用，所以可以改成只是下载多片,或者只是单纯的下载一片,由于DownloadCommand prepareForNextSegment 的逻辑，这里不能过多的请求下载的内容，防止造成浪费，浪费用户的带宽
            //TODO 所以对于 Bt 辅助 Http 下载来说，这里最好就是请求一个Piece数据的长度

            //判断是否为Bt 辅助 Http 下载
            if(getRequestGroup()->getOtherRequestGroup())
            {
                //获取到对应的终止位置,对于Bt辅助Http 下载这里每次请求只是请求一个Piece的长度
                endOffset = std::min(getFileEntry()->getLength(), getFileEntry()->gtoloff(static_cast<int64_t>(segment->getSegmentLength()) * (segment->getIndex() + 1)));
                LOGD("CUID#%" PRId64 " Bt Segment getIdex %d nextIdex %d endOffset %lld ",getCuid(),segment->getIndex(),(segment->getIndex() + 1),endOffset);
            } else{
                //原本 Http 下载的逻辑
                size_t nextIndex = getPieceStorage()->getNextUsedIndex(segment->getIndex());
                endOffset = std::min(getFileEntry()->getLength(), getFileEntry()->gtoloff(static_cast<int64_t>(segment->getSegmentLength()) * nextIndex));
                LOGD("CUID#%" PRId64 "Http Segment getIdex %d nextIdex %d endOffset %lld ",getCuid(),segment->getIndex(),nextIndex,endOffset);
            }
          }
          //由于当前的 segments_ 不为空，所以这里传递的Segment不为空
          httpConnection_->sendRequest(createHttpRequest(getRequest(), getFileEntry(), segment, getOption(), getRequestGroup(), getDownloadEngine(), proxyRequest_, endOffset));
    }
    ...
}

原本的逻辑是尽可能的下载长的内容，这样估计是可以减少重新请求，但是对于我们Bt 辅助 Http下载来说，我们不应该一次性请求那么多的内容，因为完全有可能客户端请求的长度，此时Bt 已经拥有了
这块的内容，那么就会导致Http 相冲突的内容直接抛去掉,具体体现在
bool DownloadCommand::prepareForNextSegment()
{
      ....
      //TODO 如果到了这里就是说当前请求返回的内容，还有内容没有解析完,所以尝试的获取到下一个piece 用来存储剩下的内容,在调用getSegmentWithIndex 的时候，就已经将当前的Segment添加到了 usedSegmentEntries_ 集合
      // 接下来 会在 sm->getInFlightSegment(segments_, getCuid()); 根据当前的cuid 就可以获取到当前的nextSegmentm,这里会检查 tempSegment->getIndex() + 1 的内容是否已经下载完，或者正在使用,如果已经下载
      //完成，或者正在使用，这里会返回一个null的
      std::shared_ptr<Segment> nextSegment = getSegmentMan()->getSegmentWithIndex(getCuid(), tempSegment->getIndex() + 1);

      //如果获取为空，尝试从下面获取,这是尝试的从正在下载的 Segment 集合 usedSegmentEntries_ 中 寻找 状态为 空闲的 piece 索引,然后尝试的获取到对应的Segment
      if (!nextSegment) {
          nextSegment = getSegmentMan()->getCleanSegmentIfOwnerIsIdle(getCuid(), tempSegment->getIndex() + 1);
          //LOGD("DownloadCommand::prepareForNextSegment getSegmentWithIndex nextSegmentIndex %d is Use getCleanSegmentIfOwnerIsIdle index %d",tempSegment->getIndex() + 1,nextSegment->getIndex());
      }

      //TODO 如果 nextSegment 为空，或者nextSegment 当前已经写了一部分的内容 ,注意如果当前Http请求的内容还有多，但是此时这一片的内容刚好Bt在使用，那么Http 会直接丢掉此时接收的内容剩余长度,所以对于Bt辅助Http下载
      //TODO 来说Http下载请求的范围不应该过长，防止造成浪费,
      if (!nextSegment || nextSegment->getWrittenLength() > 0) {
        // If nextSegment->getWrittenLength() > 0, current socket must
        // be closed because writing incoming data at
        // nextSegment->getWrittenLength() corrupts file.
        LOGD("CUID#%" PRId64 " DownloadCommand prepareForNextSegment !nextSegment %d hadUse prepareForRetry",getCuid(),tempSegment->getIndex()+1);
        return prepareForRetry(0);
      }
      else {
        //如果获取到 下一个piece 不为空，再次的触发,注意当前的请求并没有销毁掉，
        LOGD("DownloadCommand getSegmentWithIndex next %d piece data not Empty continue procee Data",tempSegment->getIndex()+1);
        checkSocketRecvBuffer();
        addCommandSelf();
        return false;
      }	
      ....
}
```
针对Bt 辅助 Http 多文件下载
```cpp
//TODO 定义一个集合用来存储我们传递的url,注意这里支持 多文件的下载，所以这里的对应的Http下载集合的顺序一定要跟Bt 种子文件的顺序保持一致，单文件就直接改为一个url 既可
std::vector<std::string> uris = {"http://112.90.50.248/dlied1.qq.com/lol/dltools/LOL_V4.1.2.2_FULL_0_tgod_signed.exe?mkey=5c6bbfd8d20df528&f=580c&cip=210.13.211.221&proto=http",
"http://dl.360safe.com/setup_11.5.0.2001m.exe"};
jobQueue.push(std::unique_ptr<Job>(new BtAuxiliaryHttpDownloadJob(std::move(uris),std::move(torrentUri), std::move(gloableOptions),downloadJavaId)));


//将当前Bt 下载的FileEntry 相应的设置上 Http 下载的uri
const std::vector<std::shared_ptr<FileEntry>>& fileEntries = btContext->getFileEntries();
int index = 0;
for (auto& entry : fileEntries) {
      std::vector<std::string> itemUri = {uris.at(index)};
      entry->setUris(itemUri);
      index++;
}

要注意对于Http下载来说，是通过 uris 来指定要请求的地址集合,也即是FileEntry 中的 uris_ 字段

//请求地址 队列.这个代表 当前 还没有被 请求的 的uri 地址的集合
std::deque<std::string> uris_;
而对于Bt 下载来说，是没有设置值，所以对于 Bt 辅助 Http下载来说，我们需要手动的设置对应的下载uris


//TODO 确保Bt的分片长度，确保要为1M ,由于Http的下载 PieceLength 默认为1M，所以这边要统一处理
if(btContext->getPieceLength() != requestOption->getAsInt(PREF_PIECE_LENGTH))
{
    LOGD("addBtHttpDownload Bt PieceLength no equal http PieceLength return");
    return -1;
}

//TODO 要确保Http 的uri 下载的 地址 跟 Bt 包含的下载文件大小一样
if(uris.size() != btContext->getFileEntries().size())
{
   LOGD("uris.size() != btContext->getFileEntries().size() return");
   return -1;
}

```
还有就是在当Http下载完最后一块内容的时候，这里要手动触发处于做种的状态，也即是告诉Bt 当前可以做种了
```cpp
//TODO 通知完成了当前块的下载,这里要注意，对于Bt 辅助 Http下载来说， 由于 Bt 跟 Http下载 分别拥有自己的 DefaultPieceStorage对象，俩者共用的只是 BitfiledMan对象，所以如果最后一个Piece的数据给
//TODO Http下载完成，那么就不会触发下载完成的回调，所以这边要手动的处理下
void DefaultPieceStorage::completePiece(const std::shared_ptr<Piece>& piece)
{
	...
    //当前下载为Bt 下载
    if (downloadContext_->hasAttribute(CTX_ATTR_BT)) {
      if (!bittorrent::getTorrentAttrs(downloadContext_)->metadata.empty()) {
#ifdef __MINGW32__
        // On Windows, if aria2 opens files with GENERIC_WRITE access
        // right, some programs cannot open them aria2 is seeding. To
        // avoid this situation, re-open the files with read-only
        // enabled.
        A2_LOG_INFO("Closing files and re-open them with read-only mode"
                    " enabled.");
        diskAdaptor_->closeFile();
        diskAdaptor_->enableReadOnly();
        diskAdaptor_->openFile();
#endif // __MINGW32__
        //获取到当前种子的RequestGroup对象
        auto group = downloadContext_->getOwnerRequestGroup();
        util::executeHookByOptName(group, option_, PREF_ON_BT_DOWNLOAD_COMPLETE);
        //回调通知下载完成
        SingletonHolder<Notifier>::instance()->notifyDownloadEvent(EVENT_ON_BT_DOWNLOAD_COMPLETE, group);
        LOGD("DefaultPieceStorage::completePiece EVENT_ON_BT_DOWNLOAD_COMPLETE");
        //做种的流程
        group->enableSeedOnly();
      }
    } else{
        //当前下载为非Bt 下载
        auto group = downloadContext_->getOwnerRequestGroup();
        //如果当前为Bt 辅助 Http下载,我们手动的触发 Bt的应有的回调
        if(group->getOtherRequestGroup())
        {
            util::executeHookByOptName(group->getOtherRequestGroup(), option_, PREF_ON_BT_DOWNLOAD_COMPLETE);
            //回调通知下载完成
            SingletonHolder<Notifier>::instance()->notifyDownloadEvent(EVENT_ON_BT_DOWNLOAD_COMPLETE, group->getOtherRequestGroup());
            LOGD("DefaultPieceStorage::completePiece group->getOtherRequestGroup EVENT_ON_BT_DOWNLOAD_COMPLETE");
            //做种的流程
            group->getOtherRequestGroup()->enableSeedOnly();
        }
    }
    ...
}
```












