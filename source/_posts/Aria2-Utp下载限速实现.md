---
layout: pager
title: Aria2 Utp下载限速实现
date: 2018-11-12 09:22:42
tags: [NDK,Aria2,utp]
description: Aria2,transmission 速度统计原理以及Utp下载限速实现
---

### 概述

> Aria2,transmission 速度统计原理以及Utp下载限速实现

<!--more-->

### 简介
前面一篇中介绍了Aria2 tcp下载，上传限速的原理，并且对utp的的限速也做了大概的分析，对应上传限速来说，本身是没有任何的区别，但对于utp的下载限速来说前面分析到按理来说是做不到真正意义上的限速，但是我观察发现transmission是可以实现utp的下载限速的，所以这篇文章会先去分析下transmission中utp限速的原理，以及Aria2中速度的计算，之后再将transmission中utp的限速原理，相应的修改到Aria2，实现真正意义上的utp限速

### transmission限速演示
首先要开启对应的限速配置，对应的就是
![结果显示](/uploads/Utp下载限速/transmission限速配置.png)
上面是设置为允许限速，并且限速的大小为50k，结果显示为
![结果显示](/uploads/Utp下载限速/transmission限速的结果.png)
可以看到transmission是可以做到下载限速的，而且是采用utp库来实现的，前面有说过，utp是模拟实现tcp，对于utp库使用者来说不用去管理类似的udp丢包，包的顺序等问题，还既有tcp的滑动窗口，拥塞控制等而实现utp的下载限速就是要使用到滑动窗口这个特性，这个特性是告知对方我当前所能接收的容量，你不应该发送大于这个值的内容，想想也是，要想做到真正意义上的下载限速，只有控制发送方才能做到

要想检测分析utp发送的内容，可以通过Wireshake检测到,至于要想看到utp的兼容的信息，可以这样配置 下载一个 utp.lua 放到wireshark/plugins/2.6/目录下把bt-utp的端口改回去，比方说0 然后就可以看了
![结果显示](/uploads/Utp下载限速/检测utp滑动窗口.png)


### transmission源码分析
首先要开启对应的限速配置，对应的就是
![结果显示](/uploads/Utp下载限速/transmission限速配置.png)
之后解析这个配置
![结果显示](/uploads/Utp下载限速/解析限速配置.png)
tr_sessionLimitSpeed函数实现
![结果显示](/uploads/Utp下载限速/limitSpeed函数实现.png)
session中含有这俩个成员变量
![结果显示](/uploads/Utp下载限速/session下载配置.png)
updateBandwidth函数实现
![结果显示](/uploads/Utp下载限速/全局下载速度控制.png)
tr_bandwidth 结构体定义为
![结果显示](/uploads/Utp下载限速/tr_bandwidth结构体定义.png)
要注意这里的bandwidth为session中的bandwidth成员变量,也即是这是一个全局的速度监控器对象，对于Bt下载来说，有三个速度监控器对象，一个是全局的速度监控器对象，也即是监测所有种子下载速度的所以相应的也有全局的下载限速，还有对于当个任务的速度监控对象，也即是对一个种子的速度监控，相应的也有当个任务的下载限速，最后一个是针对当前peer的速度监控，相应的也有对应的限速设置

tr_bandwidthSetDesiredSpeed_Bps函数实现，这里保存了期望的限速值
![结果显示](/uploads/Utp下载限速/保存期望的下载速度值.png)
相应的 tr_sessionLimitSpeed函数实现，保存允许设置下载限速
![结果显示](/uploads/Utp下载限速/tr_sessionLimitSpeed函数实现.png)
种子任务速度监控器对象初始化，这是针对当前任务的，这里要注意第三个参数为session中的bandwidth对象，也即是全局的速度监控器对象
![结果显示](/uploads/Utp下载限速/当前任务速度监控器对象初始化.png)
将当前任务的速度监控器对象添加到全局的速度监控器对象中，也即是可以通过父类找到下面的孩子,这样当前任务的速度监控器也初始化完成了
![结果显示](/uploads/Utp下载限速/将当前速度监控器设置父类.png)
接着定义当前对应的定时器，这里其中有一个为bandwidthPulse定时器，这个定时器是设置为500毫秒触发
![结果显示](/uploads/Utp下载限速/定义当前种子对应的定时器.png)
bandwidthPulse函数实现
![结果显示](/uploads/Utp下载限速/bandwidthPulse函数.png)
tr_bandwidthAllocate函数实现,这里要注意函数的第一个参数为session中的brandWidth对象，也即是全局的速度监控器对象
![结果显示](/uploads/Utp下载限速/tr_bandwidthAllocate函数实现.png)
allocateBandwidth函数实现 
![结果显示](/uploads/Utp下载限速/allocateBandwidth函数实现.png)

到此，这个定时器主要做的事情就要，每隔500毫秒 就触发一次重置byteLeft值，从全局的速度监控器对象开始，包括他下面的孩子监控器对象，比如种子的任务监控器对象，都会触发对应的重置操作，赋值byteLeft值

utp获取滑动窗口大小实现
![结果显示](/uploads/Utp下载限速/utp获取滑动窗口大小.png)
这里的get_rb_size 为 transmission设置utp滑动窗口回调函数
![结果显示](/uploads/Utp下载限速/transmission设置utp滑动窗口函数.png)
utp_get_rb_size 函数实现
![结果显示](/uploads/Utp下载限速/transmission获取滑动窗口函数实现.png)
tr_bandwidthClap函数实现，这里注意第一个参数io->bandwidth代表当前任务的速度监控器对象
![结果显示](/uploads/Utp下载限速/bandwidthClamp上.png)
![结果显示](/uploads/Utp下载限速/tr_bandwidth下.png)

从函数中可以知道，这里当前滑动窗口的大小跟 byteLeft大小有关，这个代表最大的滑动的窗口大小，如果当前下载速度已经超过了限速的值，那么就直接byteCount返回为0，相应的对应父类的速度监控器对象也要更新获取当前速度部分等下再来看，先来看下载的时候做了什么处理

utp读取数据回调
![结果显示](/uploads/Utp下载限速/utp读取数据回调.png)
这个函数会触发canReadWrapper函数,相应的函数实现为
![结果显示](/uploads/Utp下载限速/canReadWrapper.png)
这里有一个逻辑为判断是否为pieceData，先看看canRead函数的实现原型
![结果显示](/uploads/Utp下载限速/utpCanRead函数回调设置.png)
相应的canRead函数实现为
![结果显示](/uploads/Utp下载限速/utpCandRead函数实现.png)
通过函数可知，只有读取piece数据的时候，才会给这个pieceLength赋值，所以外面才可以根据这个值来判断是否为pieceData,所以当判断是否为 pieceData 之后，调用 tr_bandWidthUsed  函数,下面是这个函数主要的部分
![结果显示](/uploads/Utp下载限速/tr_bandWidthUsed函数实现.png)
从函数实现中可以知道，这里会获取到当前任务的速度监控器对象，如果当前有设置限速，并且为pieceData 就会更新当前速度监控器对象 bytesLeft值，这边更新了这个值，那么utp获取滑动窗口就会变化当然如果变为0的时候会怎么办，这个不用担心，前面有分析过 专门有一个定时器 用来重置这个值的，会每隔500毫秒重置为最开始设置限速的值，之后再慢慢变小，接着调用了bytesUsed函数bytesUsed函数这个是用来统计下载速度的 

bratecontrol结构体定义为
![结果显示](/uploads/Utp下载限速/bratecontrol结构体.png)
![结果显示](/uploads/Utp下载限速/bytesUsed函数实现.png)
从函数实现可知，这里是用来统计2000毫秒之内的值的， 当前速度监控器对象更新了，相应的父的速度监控器对象也要执行更新，也即是全局的速度监控器对象
这里再次回到获取滑动窗口那部分，获取当前速度值的函数，也即是tr_bandwidthGetRawSpeed_Bps函数

tr_bandwidthGetRawSpeedBps函数实现
![结果显示](/uploads/Utp下载限速/tr_bandwidthGetRawSpeedBps函数实现.png)
getSpeed_Bps 函数实现为
![结果显示](/uploads/Utp下载限速/getSpeed_Bps函数实现.png)
通过函数可知，这里是获取到统计的速度情况，算一个平均值，注意这里有一个这样的逻辑，如果当前统计的时间还没有达到2000毫秒就返回上一次统计的值
获取到期望值函数实现
![结果显示](/uploads/Utp下载限速/获取到期望值.png)
判断是否超速下载
![结果显示](/uploads/Utp下载限速/判断是否超速下载.png)

也即是如果速度超过了，将滑动窗口设置为0，这就是utp限速的关键

### transmission限速总结
> 首先有一个定时器会每隔500毫秒重置bytesLeft的值，代表当前滑动窗口的最大值，值为设置的下载限速的值，当下载到piece 数据的时候，会让当前的bytesLeft值变小，而且统计下载的情况，当utp触发获取到滑动窗口的时候，就会根据当前的下载速度是否超过了下载的速度，如果超过了就将滑动窗口设置为0，否则设置为剩余的bytesLeft的大小

### Aria2 限速演示
首先打开Aria2  配置参数为 **--max-download-limit=20k** 此时就是限制下载的速度在20k
![结果显示](/uploads/Utp下载限速/Aria2开启速度监控.png)

至于我怎么知道具体的参数设置 可以查看官网 
> http://aria2.github.io/manual/en/html/aria2c.html


### Aria2 下载速度统计源码分析
```C++

首先是开启对应的速度监控器
//设置最大的下载速度, 由于utp采用的是udp的原因，如果做限速处理的化，无法达到tcp限速的效果，所以下载不支持限速
gloableOptions.push_back(std::pair<std::string,std::string> ("max-download-limit","50k"));
//这是针对总的下载速度
gloableOptions.push_back(std::pair<std::string,std::string> ("max-overall-download-limit","1M"));


Aria2 速度监控也是有分三个对象的，一个是全局的速度监控器对象，对应的也即是
class RequestGroupMan {
private:
  //全局的速度监控器对象
  NetStat netStat_;
}

class NetStat {
public:
  enum STATUS {
    IDLE,//空闲状态
    ACTIVE,//活动状态
  };

  NetStat();
  ~NetStat();
...
private:
  //用于计算下载的速度 ,默认是一个午参的构造函数
  SpeedCalc downloadSpeed_;
  //用户计算上传的速度
  SpeedCalc uploadSpeed_;
  //下载开始的时间
  Timer downloadStartTime_;
  //状态
  STATUS status_;
  //平均的下载速度
  int avgDownloadSpeed_;
  //平均的上传速度
  int avgUploadSpeed_;
  //session下载的长度
  int64_t sessionDownloadLength_;
  //session上传的长度
  int64_t sessionUploadLength_;
  ...
}
而NetStat内部含有俩个成员变量一个是针对下载的统计，一个是针对上传的统计
class SpeedCalc {
private:
  //用来存储10秒之内的下载内容
  std::deque<std::pair<Timer, size_t>> timeSlots_;
  //下载开始的时间
  Timer start_;
  //用来统计总的下载大小
  int64_t accumulatedLength_;
  //用来统计10秒之内的下载大小
  int64_t bytesWindow_;
  //最大的下载速度
  int maxSpeed_;
  ...
}

任务的速度监控器对象，也即是针对每一个下载任务都会有一个DownloadContext的存在
class DownloadContext {
private:
  ...
  NetStat netStat_;
  ...
}

peer速度监控器对象，也即是点对点的速度监控器对象，也即是最小的速度监控器对象了
class Peer {
private:
  //代表当前peer 资源的情况，比如是否阻塞，感兴趣，等状态的封装
  std::unique_ptr<PeerSessionResource> res_;
  ...
}

而PeerSessionResource 成员含有netStat_速度监控器对象
class PeerSessionResource {
private:
  ...
  NetStat netStat_;
}

其实Aria2做法更加简单的，他限速的处理只针对piece数据才会统计分析，对于其他的消息，不管,下面来分析下Aria2 接收到Piece Data的逻辑


//接受传递过来的内容，这里既是 接受到了peer传递回来的我们需要的pieceIndex 所对应的块的内容
void BtPieceMessage::doReceivedAction()
{
  //默认为false
  if (isMetadataGetMode()) {
      return;
  }
  
  auto slot = getBtMessageDispatcher()->getOutstandingRequest(index_, begin_, blockLength_);
  //更新下载的速度,这是针对当前peer的
  getPeer()->updateDownload(blockLength_);
  //更新下载的速度，这是针对当前这一个种子的速度
  downloadContext_->updateDownload(blockLength_);
  ...  
}

首先更新Peer的速度监控器对象

//更新下载的速度，针对当前的peer
void Peer::updateDownload(int32_t bytes)
{
  assert(res_);
  res_->updateDownload(bytes);
}

接着更新当前任务的速度监控器对象，以及全局的速度监控器对象

//更新下载的速度，这是针对当前一个种子任务总的下载速度
void DownloadContext::updateDownload(size_t bytes)
{
  //LOGD("DownloadContext updateDownload bytes %u \n",bytes);
  //这是当前这个任务的下载速度
  netStat_.updateDownload(bytes);

  //这是全局的下载速度
  RequestGroupMan* rgman = ownerRequestGroup_->getRequestGroupMan();
  if (rgman) {
     rgman->getNetStat().updateDownload(bytes);
  }
}

下面来分析下updateDownload 的函数实现
void NetStat::updateDownload(size_t bytes)
{
  downloadSpeed_.update(bytes);
  sessionDownloadLength_ += bytes;
}

会调用到downloadSpeed_.update(bytes); 

//更新下载的总和
void SpeedCalc::update(size_t bytes)
{
  const auto& now = global::wallclock();
  //保证队列的内容是10秒钟之内的值
  removeStaleTimeSlot(now);
  //如果当前队列为空 或者 当前队列的最后一个元素添加的时间跟当前的时间间隔为一秒的化，就直接往队列里面添加一个元素对pair
  if (timeSlots_.empty() || std::chrono::duration_cast<std::chrono::seconds>(timeSlots_.back().first.difference(now)) >= 1_s) {
     timeSlots_.push_back(std::make_pair(now, bytes));
  }
  else {
    //这里就是最后一个元素时间跟当前时间还没有一秒，那就直接累加，这样做就是也即是统计一秒的下载量了
    timeSlots_.back().second += bytes;
  }
  //更新下载的总和
  bytesWindow_ += bytes;
  accumulatedLength_ += bytes;
}

removeStaleTimeSlot函数实现

//下面可以做到只统计10秒的内容
void SpeedCalc::removeStaleTimeSlot(const Timer& now)
{
  //当前统计的队列不为空，可以做到只统计10秒之内的内容
  while (!timeSlots_.empty()) {
    //constexpr auto WINDOW_TIME = 10_s;  也即是如果第一次添加的时间跟当前的时间间隔小于10秒的化，直接退出,也即是统计时间最少也要有10秒钟的间隔
    if (timeSlots_[0].first.difference(now) <= WINDOW_TIME) {
       break;
    }
    //大于10秒, bytesWindow_ 减去要弹出的元素的值 ,这里防止队列太多,也即是只统计10秒的内容
    bytesWindow_ -= timeSlots_[0].second;
    //弹出第一个
    timeSlots_.pop_front();
  }
}

从函数的实现可以知道，Aria2最多统计最近10秒之内的下载情况，接下来分析下怎么判断是否超过了限制
//判断当前的上传以及下载是否超过了最大的速度限制,这里会更新Peer的超时时间设置
if (getDownloadEngine()->getRequestGroupMan()->doesOverallDownloadSpeedExceed() || requestGroup_->doesDownloadSpeedExceed()) {
    ...
}

对应的函数实现为

//判断全局的下载速度是否超过了最大的设置值
bool RequestGroupMan::doesOverallDownloadSpeedExceed()
{
  return maxOverallDownloadSpeedLimit_ > 0 && maxOverallDownloadSpeedLimit_ < netStat_.calculateDownloadSpeed();
}


//判断当前的任务zong1下载是否超过了最大的下载速度
bool RequestGroup::doesDownloadSpeedExceed()
{
  int spd = downloadContext_->getNetStat().calculateDownloadSpeed();
  return maxDownloadSpeedLimit_ > 0 && maxDownloadSpeedLimit_ < spd;
}

//RequestGroupMan 构造函数的定义 ,第一个参数传递为定义的请求组集合
RequestGroupMan::RequestGroupMan(
    std::vector<std::shared_ptr<RequestGroup>> requestGroups,
    int maxConcurrentDownloads, const Option* option)
    : maxConcurrentDownloads_(maxConcurrentDownloads),//标识最多支持多少个并发的下载
    ...
    maxOverallDownloadSpeedLimit_(option->getAsInt(PREF_MAX_OVERALL_DOWNLOAD_LIMIT)),//最大的下载速度限制 ,可以通过配置选项改变
    maxOverallUploadSpeedLimit_(option->getAsInt(PREF_MAX_OVERALL_UPLOAD_LIMIT)),//最大的上传速度限制，可以通过配置选项改变
    ...
}

也即是我们设置的 这是针对总的下载速度
gloableOptions.push_back(std::pair<std::string,std::string> ("max-overall-download-limit","1M"));

//创建请求组的构造函数,传递的参数一个为当前请求gid，一个为当前请求组的选项
RequestGroup::RequestGroup(const std::shared_ptr<GroupId>& gid,
                           const std::shared_ptr<Option>& option)
    : belongsToGID_(0),//标识这个是属于一个请求组
    ...
    maxDownloadSpeedLimit_(option->getAsInt(PREF_MAX_DOWNLOAD_LIMIT)),//max-download-limit 代表最大下载的限制，这个可以改变的。。默认值为没有限制
    maxUploadSpeedLimit_(option->getAsInt(PREF_MAX_UPLOAD_LIMIT)),//max-upload-limit 代表最大上传的限制  这个可以改变的。。默认值为没有限制
    ...
}

也即是我们设置的  设置当前任务的最大的下载速度
gloableOptions.push_back(std::pair<std::string,std::string> ("max-download-limit","50k"));

所以判断是否超过了速度也是非常简单的，判断是否有设置对应的选项配置 比如 max-download-limit，或者 max-overall-download-limit,如果没有，那么默认为0，那么doesOverallDownloadSpeedExceed，或者 doesDownloadSpeedExceed默认返回false
如果有配置的化，那么就要根据当前的下载统计情况跟这个值做比较，当前速度获取是通过 calculateDownloadSpeed 实现

/**
 * Returns current download speed in byte per sec.
 * 计算下载的进度
 */
int NetStat::calculateDownloadSpeed()
{
   return downloadSpeed_.calculateSpeed();
}

//计算下载的速度
int SpeedCalc::calculateSpeed()
{
  //获取到当前的时间
  const auto& now = global::wallclock();
  //保证队列的内容是10秒钟之内的值
  removeStaleTimeSlot(now);
  //统计的队列不为空
  if (timeSlots_.empty()) {
      return 0;
  }

  //统计队列的大小
  auto elapsed = std::chrono::duration_cast<std::chrono::milliseconds>(timeSlots_[0].first.difference(now)).count();
  if (elapsed <= 0) {
    elapsed = 1;
  }

  //计算速度,由于timeSlots_ 队列里面的元素都是一秒添加的，所以timeSlots_队列的大小也即是还有多少秒
  int speed = bytesWindow_ * 1000 / elapsed;
  //更新最大的速度值
  maxSpeed_ = std::max(speed, maxSpeed_);
  return speed;
}

也即是求平均值的意思,这就是Aria2 判断是否超过了限制的实现，也即是Aria2 统计速度实现
```

### Aria2 Utp限速实现
```C++
我们可以参考transmission的做法，实现Utp的限速处理

由于我们的限速针对是整个任务，不是单个peer，也不是全局的速度监控，为什么是整个任务，不是单个peer呢，如果限制了单个peer的化，如果此时有好些人连接上这个用户，也会把用户的带宽吃掉，如果设置到
整个任务的化，就不会存在这个问题，所有连接上这个用户共享最大的限速值，这样才能保障带宽，所以我们将 bytesLeft 放到 RequestGroup ,并有三个对应的方法操作这个变量

class RequestGroup {
privae:
  enum HaltReason { NONE, SHUTDOWN_SIGNAL, USER_REQUEST };
  //代表当前RequestGroup 可以允许接收的大小
  int bytesLeft;
  ... 
  
public:  
  //重置BytesLeft 也即是重置滑动窗口的大小
  void resetBytesLeft();

  //更新滑动窗口的大小
  void updateBytesLeft(int pieceDataLength);

  //获取到当前允许的滑动窗口的大小
  int getBytesLeft();
  ...
}

对应的函数实现为

//重置 bytesLeft，参照transmission 每500毫秒重置这个值为默认的限速的值
void RequestGroup::resetBytesLeft()
{
    bytesLeft = maxDownloadSpeedLimit_;
}

//更新滑动窗口的大小
void RequestGroup::updateBytesLeft(int pieceDataLength)
{
    //获取到最小值
    int minValue = bytesLeft > pieceDataLength ? pieceDataLength : bytesLeft;
    bytesLeft -= minValue;
}

//获取到当前允许的滑动窗口的大小
int RequestGroup::getBytesLeft()
{
    return bytesLeft;
}

接下来就是重置这个byteLeft触发，transmission是每隔500毫秒触发一次更新,由于是针对当前整个下载任务的，我们可以将这部分代码写到 TrackerWatcherCommand
class TrackerWatcherCommand : public Command {
private:
  ...	
  //超时重置 RequestGroup 的bytesLeft
  Timer checkPoint_;
  //每隔多少毫秒重置一次
  std::chrono::milliseconds refreshInterval_;
  ...
}

TrackerWatcherCommand::TrackerWatcherCommand(cuid_t cuid, RequestGroup* requestGroup, DownloadEngine* e)
    : Command(cuid),
      requestGroup_(requestGroup),
      e_(e),
      udpTrackerClient_(e_->getBtRegistry()->getUDPTrackerClient()),
      checkPoint_(global::wallclock()), //检查的时间点
      refreshInterval_(std::chrono::milliseconds(1000)) // 重置滑动窗口的设置
{
    ...
}

//Command命令的执行
bool TrackerWatcherCommand::execute()
{
  //如果当前的请求是 强制暂停。这里要注意这里是针对当前的请求,如果当前requestGroup不是退出的状态是不会进入里面的
  if (requestGroup_->isForceHaltRequested()) {
    //如果目前的trackerRequest对象为空，则直接返回true，那么这个对象就要被销毁掉
    if (!trackerRequest_) {
      return true;
    }
    else if (trackerRequest_->stopped() || trackerRequest_->success()) {//当前是暂停的状态
      return true;
    }
    else {
      trackerRequest_->stop(e_);
      e_->setRefreshInterval(std::chrono::milliseconds(0));
      e_->addCommand(std::unique_ptr<Command>(this));
      return false;
    }
  }

  //只有当前的requestGroup 退出的时候，才会为true
  if (btAnnounce_->noMoreAnnounce()) {
    A2_LOG_DEBUG("no more announce");
    return true;
  }

  //每500毫秒更新超时设置,utp 超时检查，内部会将那些没有使用的utpSocket对象释放掉 TODO tcp不用
  if(checkPoint_.difference(global::wallclock()) >= refreshInterval_)
  {
      //重置刷新的时间
      checkPoint_ = global::wallclock();
      //参照transmission 每隔500毫秒重置一次bytesLeft的值，这个值代表utp滑动窗口的大小
      requestGroup_->resetBytesLeft();
  }
  ...	
}

添加utp 获取滑动窗口回调函数
utp_set_callback(utpContext,UTP_GET_READ_BUFFER_SIZE,&callback_get_rb_size);//utp 限速回调函数

//utp 控制滑动窗口回调，用来实现限速处理
uint64 callback_get_rb_size(utp_callback_arguments *a)
{
    utp_socket * utpSocket = a->socket;

    void* utp_socket_userData = utp_get_userdata(utpSocket);
    //我们在超时的时候，手动的置为了NULL
    if(utp_socket_userData)
    {
        PeerAbstractCommand * peerAbstractCommand = static_cast<PeerAbstractCommand *>(utp_socket_userData);
        if(peerAbstractCommand)
        {
            //对应的子类实现 分发消息
            return peerAbstractCommand->utpGetRbSize();
        }
    }
    return 0;
}

由于Aria2 限速的原理是针对piece Data才会做限速，所以我们也做类似的处理
class PeerAbstractCommand : public Command {
public:
   ...
   //新增utp 控制滑动窗口函数，默认为0，则走默认值，由于限速只针对下载piece数据有用，所以默认为0
   virtual int utpGetRbSize(){ return 0;};
   ...
}

我们需要在PeerInteractionCommand 中重写这个方法，这个command就是获取到piece了

class PeerInteractionCommand : public PeerAbstractCommand {
public:
   //重新父类的实现，事实的控制滑动窗口的大小，从而实现utp限速的处理
   virtual int utpGetRbSize() CXX11_OVERRIDE; 
   ...
}

//当前command是下载种子数据的，所以这里要重写这个函数，返回的值决定滑动窗口的大小
int PeerInteractionCommand::utpGetRbSize()
{
    //当前只针对Bt消息要处理
    if(sequence_ == WIRED)
    {
        utp_context * utpContext = getDownloadEngine()->getBtRegistry()->getUtpContext();
        if(utpContext == NULL)
        {
            LOGD("PeerInteractionCommand::utpGetRbSize utpContext == NULL error");
            return 0;
        }
        //utp 默认的滑动窗口的大小
        int opt_rcvbuf = utp_context_get_option(utpContext,UTP_RCVBUF);
        //取的最小值
        int minValue = opt_rcvbuf > requestGroup_->getBytesLeft() ? requestGroup_->getBytesLeft() : opt_rcvbuf;
        int byteCount = minValue;
        if(byteCount > 0)
        {
            //判断utp下载是否超过了下载的限制
            if (getDownloadEngine()->getRequestGroupMan()->doesOverallDownloadSpeedExceed() || requestGroup_->doesDownloadSpeedExceed()) {
                LOGD("PeerInteractionCommand::utpGetRbSize minValue %d doesDownloadUtpExceed",minValue);
                //超过了将滑动窗口设置为0
                byteCount = 0;
            }
        }
        LOGD("CUID#%" PRId64 " PeerInteractionCommand utpGetRbSize byteCount %d",getCuid(),byteCount);
        //这是utp的函数实现， return opt_rcvbuf > numbuf ? opt_rcvbuf - numbuf : 0; numbuf 为我们当前函数的返回值,所以限速的处理就直接将滑动窗口为0，所以numbuf = opt_rcvbuf
        return opt_rcvbuf - byteCount;
    }
    //其他的情况下，返回0，则会执行默认的滑动窗口设置
    return 0;
}

最后在更新这个bytesLeft值，可以放到DownloadContext 中，因为这个是针对当前任务

//更新下载的速度，这是针对当前一个种子任务总的下载速度
void DownloadContext::updateDownload(size_t bytes)
{
  //LOGD("DownloadContext updateDownload bytes %u \n",bytes);
  //这是当前这个任务的下载速度
  netStat_.updateDownload(bytes);

  //更新RequestGroup中的bytesLeft 也即是滑动窗口
  ownerRequestGroup_->updateBytesLeft(bytes);

  //这是全局的下载速度
  RequestGroupMan* rgman = ownerRequestGroup_->getRequestGroupMan();
  if (rgman) {
     rgman->getNetStat().updateDownload(bytes);
  }
}
```
注意这里如果不参照transmission的做法设置一个bytesLef,直接判断如果当前速度超过了，就直接设置滑动窗口为0，如果没有超过，就设置为默认值，这样会导致速度的波动，还会导致对方一直的发送数据等错误

这里还有一个细节就是transmission中有这样的处理，也即是当尝试的获取数据的时候，获取不到，就主动调用这个方法
![结果显示](/uploads/Utp下载限速/utp_PBDrained函数调用.png)

utp_PBDrained函数实现,内部会调用send_ack
![结果显示](/uploads/Utp下载限速/utp_PBDrained函数实现.png)

utp_sendAck函数实现,这内部会获取到当前滑动窗口的大小，如果不这样处理的化，会导致滑动窗口的设置太慢，会出现问题
![结果显示](/uploads/Utp下载限速/utp_sendAck函数实现.png)

相应的我们可以在
```C++
bool PeerConnection::receiveMessage(unsigned char* data, size_t& dataLength)
{
    ...
	//缓冲区中有内容才要拷贝
    if(btResBufLength_ != 0)
    {
        //一次性拷贝会有问题，会偶现溢出缓冲空间，这里控制在64k之内，所以这里不能一次性拷贝
        int finallyLen = btResBufLength_ > nread ? nread : btResBufLength_;
        memcpy(resbuf_.get()+resbufLength_,btResbuf_.get(),finallyLen);
        resbufLength_ += finallyLen;

        //拷贝完之后要判断是否还剩下有数据，如果有要将剩余的数据，移到bt缓冲区的前面，下次继续再后面添加
        if(finallyLen != btResBufLength_)
        {
            //将bt缓冲区的剩下的内容移动到前面
            memmove(btResbuf_.get(), btResbuf_.get() + finallyLen, btResBufLength_ - finallyLen);
        }
        //标识当前bt缓冲区中还剩余的长度
        //int leftLength = btResBufLength_ - finallyLen;
        LOGD("utp receiveMessage before btResBufLength_ %d after leftLength %d resbufLength_ %d",btResBufLength_,(btResBufLength_ - finallyLen),resbufLength_);
        btResBufLength_ -= finallyLen;
    } else{
        if(getSocketBuffer()->getSocketCore()->getUtpSocket())
        {
            //参照transmission 的做法，这个可以更改utp的滑动窗口
            utp_read_drained(getSocketBuffer()->getSocketCore()->getUtpSocket());
            //缓冲区没有内容，中断循环，下次继续执行
            LOGD("btResBufLength_ == 0 break   utp_read_drained");
        }
        break;
    }
    ...
}
```
![结果显示](/uploads/Utp下载限速/Aria2 Utp限速结果.png)


