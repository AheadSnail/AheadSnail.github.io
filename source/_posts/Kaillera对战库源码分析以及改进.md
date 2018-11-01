---
layout: pager
title: Kaillera对战库源码分析以及改进
date: 2018-10-16 09:21:05
tags: [Android,NDK,Kaillera]
description: Kaillera对战库源码分析以及改进
---

### 概述

> Aria2 Kaillera对战库源码分析以及改进

<!--more-->


### Kaillera测试代码的编写

前面一篇文章大致的介绍了Fba流程，这篇会大致的讲解下Kaillera对战库的实现，以及改进的方式，最后怎么样移植到Android上面,首先是对战库的大致的实现，我们可以在前面调试这个对战库的
代码来看代码这样会简单很多，那么我们就可以根据上一篇中发生怎么样使用Kaillera 对战库，下面是对应的测试代码

```C
static int WINAPI gameCallback(char* game, int player, int numplayers)
{
	printf("select game == %s\n",game);
	printf("game support player == %d\n",player);
	printf("game support numplayers == %d\n",numplayers);

	isBeignGame = true;

	//模拟发送游戏数据
	unsigned char nControls[8 * (4 + 8)] = { 0 };
	kailleraModifyPlayValues(nControls, 1);

	return 0;
}

static void WINAPI kChatCallback (const char *nick, const char *text)
{
	printf("receive chat callBack nick == %s\n", nick);
	printf("receive chat callBack message == %s\n", text);
}

static void WINAPI kDropCallback (const char *nick, int playernb)
{
	printf("kDropCallback nick == %s\n", nick);
	printf("kDropCallback playernb == %d\n", playernb);
}


int main()
{
	char version[100] = { 0 };
	kailleraGetVersion(version);
	printf("version == %s\n",version);

	//获取支持对战的游戏集合
	char* gameList;
	getSupportNetGames(&gameList);

	//初始化
	kailleraInit();
	
	//设置参数信息
	info.appName = "KailDemo";
	info.gameList = gameList;
	info.gameCallback = &gameCallback;
	info.chatReceivedCallback = &kChatCallback;
	info.clientDroppedCallback = &kDropCallback;
	info.moreInfosCallback = NULL;

	//设置回调的参数信息
	kailleraSetInfos(&info);

	//打开对话框
	kailleraSelectServerDialog(NULL);
	....
}
```

测试代码相应的源码结构为：

![结果显示](/uploads/Fba对战/kaillera对战库源码结构.jpg)

### Kaillera对战库Android移植

移植也不算难，他本身就是支持linux，window 只是他还有界面，我们要做的就是看懂代码的基础上，去除对应的界面逻辑，然后将原有界面的逻辑，采用NDK的方式调用java层，达到类似的界面改变
下面是对应的移植代码的结果

进入服务器

![结果显示](/uploads/Fba对战/进入服务器.png)

相互聊天

![结果显示](/uploads/Fba对战/相互聊天已经创建游戏.png)

进入游戏房间

![结果显示](/uploads/Fba对战/进入游戏房间.png)


客户端CmakeList的写法

```cmake
cmake_minimum_required(VERSION 3.6.0)
add_library(kailleraDemo
    SHARED
    # Provides a relative path to your source file(s).
    ../../com_m4399_netplay_KailleraJni.cpp
    ../../common/BsdSocket.cpp
    ../../common/ConfigurationManager.cpp
    ../../common/Logger.cpp
    ../../common/PosixThread.cpp
    ../../common/SocketAddress.cpp
    ../../common/StringUtils.cpp
    ../../common/_common.cpp
    ../../common/common.cpp
    ../../common/trace.cpp
    ../../common/signals_GNU.cpp
    ../../interface/GamesList.cpp
    ../../interface/gameplay.cpp
    ../../interface/n02.cpp
    ../../interface/kailleraclient.cpp
    ../autorun.cpp
    ../module.cpp
    ../transport.cpp
    ../deferment/defermentInputManager.cpp
    ../deferment/fixed.cpp
    ../deferment/traditional.cpp
    kaillera_ClientCore.cpp
    modKaillera.cpp
    kaillera_ClientMessaging.cpp
    ../../kaiserver/k_socket.cpp
    ../../kaiserver/k_user.cpp
    ../../kaiserver/k_utils.cpp
    ../../kaiserver/settings.cpp
    ../../kaiserver/k_server.cpp
)
include_directories(../../common ../../interface ../  ../../)
find_library(log-lib log)
target_link_libraries(kailleraDemo ${log-lib})
```

服务端的CmakeList写法

```cmake
cmake_minimum_required(VERSION 3.6.0)
#file (GLOB COMMON_SRC *.cpp EXCLUDE k_server.cpp)
set (
    COMMON_SRC
    k_socket.cpp
    k_user.cpp
    k_utils.cpp
    settings.cpp
)
find_library(llog log)

add_library(kaisev SHARED k_server.cpp ${COMMON_SRC})
target_link_libraries(kaisev ${llog})

```

由于对战库内部是新建了一个线程，所以对应JNI的调用，要注意线程的问题，要采用这种方法

```C++
static void N02CCNV gameClosed ()
{
    TRACE();
    //回调java层，关闭加入游戏的弹窗
    JNIEnv *env;
    jVM->GetEnv((void**) &env, JNI_VERSION_1_4);
    int attached  = 0;
    if(env==NULL)
    {
        attached  = 1;
        jVM->AttachCurrentThread(&env, NULL);
    }
    //sendCommand(LISTCMD_HIDEGAME, 0);
    if(android_closeJoinGameWindow != NULL)
    {
        env->CallStaticVoidMethod(kailleraJni,android_closeJoinGameWindow);
    }
    if(attached)
    {
        jVM->DetachCurrentThread();
    }
    //标识窗口关闭
    gameWindowVisible = false;
    TRACE();
}
```

### Kaillera不足以及改进

代码移植完之后， 下面就来分析Kailler对战库的不足，所以这里先要了解帧锁定以及乐观帧锁定概念

#### 帧锁定
> 1．客户端定时（比如每五帧）上传控制信息。
2．服务器收到所有控制信息后广播给所有客户。
3．客户端用服务器发来的更新消息中的控制信息进行游戏。
4．如果客户端进行到下一个关键帧（5帧后）时没有收到服务器的更新消息则等待。
5．如果客户端进行到下一个关键帧时已经接收到了服务器的更新消息，则将上面的数据用于游戏，并采集当前鼠标键盘输入发送给服务器，同时继续进行下去。
6．服务端采集到所有数据后再次发送下一个关键帧更新消息。
这个等待关键帧更新数据的过程称为“帧锁定”


#### 乐观帧锁定

针对传统严格帧锁定算法中网速慢会卡到网速快的问题，实践中线上动作游戏通常用“定时不等待”的乐观方式再每次Interval时钟发生时固定将操作广播给所有用户，不依赖具体每个玩家是否有操作更新：

> 1. 单个用户当前键盘上下左右攻击跳跃是否按下用一个32位整数描述，服务端描述一局游戏中最多8玩家的键盘操作为：int player_keyboards[8];
2. 服务端每秒钟20-50次向所有客户端发送更新消息（包含所有客户端的操作和递增的帧号）：update=（FrameID，player_keyboards）
3. 客户端就像播放游戏录像一样不停的播放这些包含每帧所有玩家操作的 update消息。
4. 客户端如果没有update数据了，就必须等待，直到有新的数据到来。
5. 客户端如果一下子收到很多连续的update，则快进播放。
6. 客户端只有按键按下或者放开，就会发送消息给服务端（而不是到每帧开始才采集键盘），消息只包含一个整数。服务端收到以后，改写player_keyboards

#### 相同点：
> 俩种实现方式都要有一个服务端，服务端做的事情只是用来收集数据，转发数据，而且如果没有服务端的化，后面设计到网络对战的时候，还要解决打洞的问题，也即是p2p通信，但是当用户处于多重Nat设备下，或者对称型网络，是不能打洞的，这种情况下，最好就是将服务端放在外网的ip上，由服务端来转发数据，这样就不存在这样的问题

#### 不同点：
> 虽然网速慢的玩家网络一卡，可能就被网速快的玩家给秒了（其他游戏也差不多）。但是网速慢的玩家不会卡到快的玩家，只会感觉自己操作延迟而已。另一个侧面来说，土豪的网宿一般比较快，我们要照顾。所以相比于帧锁定，乐观帧锁定才是更好的选择，要不然采用帧锁定当多个玩家一起玩游戏，只要有一个玩家网速慢，大家都要等待这个玩家，这样会导致独家都卡顿，显然这是不合理的，

而Kaillera对战库采用的就是帧锁定下面是对应的服务端代码体现：

```C++
if(players.length > 0) {
			//遍历当前所有的玩家，
			for (int index_ = 0; index_ < players.length; index_++) {
				k_user * kusr = players.get(index_);
				//状态为2，代表在游戏中
				if(kusr->status == 2) {
					if (kusr->client_reply_frame <= frame_counter && kusr->untrimmed_used_data == 0) {
						if(kusr->has_data() == 0) {
							xrv = 0;
						} else {
							memset(tmp_big_big_buffer, 0, 1024);
							int framelen = -1;
							bool skip = false;
							bool pack_success = false;
							if (players.length> 0) {
								for (int i = 0; i < players.length; i++){
									k_user * ku = players.get(i);
									if(ku->status == 2) {
										if(ku->client_reply_frame <= frame_counter) {
											//判断当前玩家是否有数据到来，如果没有就退出这个循环
											if(ku->has_data() == 0) {
												xrv = false;
												skip = true;
												break;
											}
											//当前玩家有数据，获取到对应的数据
											int len = ku->peek_frame(tmp_buffer);
											if(len == 0) {
												//如果长度为0，也代表没有数据
												xrv = false;
												skip = true;
												break;
											} else {
												//填充内容
												if(framelen == -1) {
													framelen = len;
												}
												memcpy(&tmp_big_big_buffer[len * i], tmp_buffer, len);
												//标识可以发送数据
												pack_success = true;
											}
										}
									}
								}
								if (skip)
								{
									continue;
								}
								//标识是否可以发送数据
								if (pack_success) {
									//发送数据
									kusr->send_data(tmp_big_big_buffer, players.length * framelen);
									kusr->untrimmed_used_data = 1;
								}
							}
						}
					}
				}
		....		
	}
}
```

从上面的逻辑可以发现，当其中有一个玩家没有数据到来的时候，就退出了这个循环，尽管其他的玩家有数据，也要等待这个玩家的数据到来，显然这就是帧锁定的关键实现,

#### 大量发送冗余数据包

> 问题二，Kaillera 存在大量的发送冗余包的问题,而且会出现由于丢失了某个数据包，导致一直卡顿的问题

客户端发送冗余数据包体现

![结果显示](/uploads/Fba对战/客户端发送冗余包.png)

服务端发送冗余数据包体现

![结果显示](/uploads/Fba对战/服务端发送冗余数据.png)


客户端以及服务端是怎么样保证数据按照顺序到达，由于采用的是udp实现的，所以肯定会存在丢包的情况，相应的就应该有集合来保存双方发送的数据包，方便重发数据，还要有标识，标识当前数据包的序列号，以及当前已经接受的数据包的序列号，类比于tcp的化，就是seq以及ack

客户端同步代码的实现
```C++
// recv input data
int  N02CCNV recvSyncData(void * value, const int /*len*/)
{
    if (state==RUNNING) {
        if (++gameInfo.frame >= gameInfo.delay) {
	    //判断是否有接受到数据，如果没有就一直处于while循环，那客户端就阻塞了,也就达到了同步
            while (gameInfo.inBuffer.length() < gameInfo.totalInputLength && state == RUNNING) {
                stepRunning();
        }
		....
		memcpy(value, gameInfo.defaultInput, gameInfo.totalInputLength);
	}
	
	return -1;
}
```

前面说过客户端，以及服务端都会发送冗余的数据包，对应的找到正确的数据包的体现

```C++
void dataArrivalCallback()
{
   
    int bufLen = BIG_RECV_BUFFER_SIZE;

    //从lastAddress地址中接收数据，数据存储在bigRecvBuff中
    if (BsdSocket::recvFrom(BsdSocket::bigRecvBuffer, bufLen, lastAddress)) {
        //当在客户端接收到服务端发送的HELLOW000的时候，就将这个状态改为了INSTRMSGS
        if (state == INSTRMSGS) {
            TRACE();
            //LOGBUFFER("Received", BsdSocket::bigRecvBuffer, bufLen);
            if (bufLen > 1 + 4 + 2) {
                char instructionsCount = *bigRecvBuffer;
                //找到当前应该要解析的数据包索引号
                while (searchProcessInstruction(lastReceivedInstruction+1, reinterpret_cast<unsigned char*>(bigRecvBuffer+1), bufLen, instructionsCount))
                {
                    lastReceivedInstruction++;
                }
                } else {
                    require(bufLen > 1 + 4 + 2);
                }
            } else {
                TRACE();
                core::dataArrivalCallback(BsdSocket::bigRecvBuffer, bufLen);
            }
	} else {
        LOG(Socket Error occured %i, errno);
    }
    TRACE();
}

inline bool searchProcessInstruction(unsigned short forSerial, unsigned char * buffer, int bufferLen, int instrsCount) {
    while (instrsCount > 0) {
        unsigned short serial = *((unsigned short*)buffer); buffer+= 2;
        unsigned short length = *((unsigned short*)buffer); buffer+= 2;
        if (serial == forSerial) {//接受到的数据包是下一个要解析数据包的索引才会返回true，通知有数据
            TRACE();
            Instruction instr(buffer, length);
            core::instructionArrivalCallback(instr);
            TRACE();
            return true;
        } else {
            buffer += length;
            bufferLen -= (4 + length);
            instrsCount--;
        }
    }
    TRACE();
    return false;
}
```

当然这些问题相应的在服务端也是有的，下面提出对应的解决方案,对应发送冗余数据包的问题，我们可以采用第三库的来实现，由于是基于udp的，所以推荐的有俩个，一个kcp，一个utp，大体的实现
思路都是模拟tcp，像utp可能做的更好，他有做对应的tcp的拥塞控制，以及滑动窗口，但是使用都差不多，对于使用层来说，不用理会他的内部实现，他会自动的做到丢包冲发，保证序列号的问题
他们不会管理数据的发送，已经数据的接受，对应的这些操作要你来完成，当你收到数据的时候，要丢给他们，他们会解析这些数据，返回给你原始的数据，发送数据的时候，发送之前你要先丢给
他们，他们会做进一步的处理，大致就是往头里面添加一些内容，然后处理完之后，会通知你发送数据,具体的使用可以查看官网的demo

下面是服务端的关键代码实现

![结果显示](/uploads/Fba对战/服务端发送kcp数据包.png)
![结果显示](/uploads/Fba对战/服务端接受数据kcp.png)
![结果显示](/uploads/Fba对战/服务端接受数据kcp2.png)

下面是客户端的关键代码实现
![结果显示](/uploads/Fba对战/客户端关键代码kcp发送.png)
![结果显示](/uploads/Fba对战/客户度kcp发送数据包2.png)
![结果显示](/uploads/Fba对战/客户端kcp接受数据.png)

#### 修改这种帧锁定机制

> 首先是：
1. 单个用户当前键盘上下左右攻击跳跃是否按下用一个32位整数描述，服务端描述一局游戏中最多8玩家的键盘操作为：int player_keyboards[8];
6. 客户端只有按键按下或者放开，就会发送消息给服务端（而不是到每帧开始才采集键盘），消息只包含一个整数。服务端收到以后，改写player_keyboards

收集当前用户是否有操作

```C++
//先将第一个玩家的当前按键不为0的值，存储到nControls 里面
for (i = 0, j = 0; i < nPlayerInputs[0]; i++, j++) {
	BurnDrvGetInputInfo(&bii, i + nPlayerOffset[0]);
	//这里要注意 bii.pVal存储的为按键的地址，如果这个按键的值不为0, 会在GetInput(true);的时候完成赋值操作 	*(pgi->Input.pVal) = pgi->Input.nVal;
	if (*bii.pVal && bii.nType == BIT_DIGITAL) {
		isProcess = 1;//代表用户有操作
		//记录当前发送的按键信息
		//WRITE_LOG("BurnInputInfo nPlayerInputs name %s\n",bii.szName);
		nControls[j >> 3] |= (1 << (j & 7));
	}
}
	
//公用按钮的信息  比如Reset,Diagnostic,Service,Volume Up,Volume Down
//nCommonOffset = 16 存储的是当前游戏支持按键，这里是俩个玩家
for (i = 0; i < nCommonInputs; i++, j++) {
	//直接获取到公用按键的信息
	BurnDrvGetInputInfo(&bii, i + nCommonOffset);
	//如果公用按键的值不为0，就存储到nControls 里面
	if (*bii.pVal) {
		isProcess = 1;//代表用户有操作
		nControls[j >> 3] |= (1 << (j & 7));
	}
}

//发送玩家一当前的按键消息，也同时获取到其他玩家的按键情况，如果返回值为-1，代表失败,直接退出
//连接网络差的情况下，这个会在这边阻塞，在联网情况下，退出引擎没有反应就是这个地方
unsigned int startTime = GlobalTimer::getTime();
//第三个参数 客户端只有按键按下或者放开，就会发送消息给服务端（而不是到每帧开始才采集键盘），isProcess是表示当前的用户有没有操作的意思
if (kailleraModifyPlayValues(nControls, k,isProcess) == -1) {
	//如果出现了失败，标识联网队战失败
	printf("kailleraModifyPlayValues fail kNetGame == 0");
	kNetGame = 0;
	//返回1代表连接网络失败
	return 1;
}

//最终会调用到对战库的这个方法
void N02CCNV sendSyncData(const void * value, const int len,int isNeedSend)
{
    ...
    //如果是前面的60帧数据，默认的填充为 第一帧的游戏数据
    //如果当前没有操作，就没有必要发送数据给服务端，但是客户端界面的响应还是要等服务端的返回内容来填充
    //客户端只有按键按下或者放开，就会发送消息给服务端（而不是到每帧开始才采集键盘），
    if (gameInfo.frame + gameInfo.delay > MASK_INITIAL_FRAMES && isNeedSend) {
        //后面60帧的数据
        register int requiredBufferLen = len * userInfo.connectionSetting;
        // if not, send a full packet
        Instruction data(GAMEDATA);
        data.writeUnsignedInt16((unsigned short)requiredBufferLen);
        data.writeBytes(value, requiredBufferLen);
        send(data);
    } else {
        //前面的60帧数据不用发送给服务端，并且后面60帧之后的数据如果没有操作的化，也不用发送给服务端
        //LOGD("gameInfo.frame %d + gameInfo.delay %d > 60 && isNeedSend %d",gameInfo.frame,gameInfo.delay,isNeedSend);
        //当gameInfo.frame == gameInfo.delay 帧的时候，数据要传递给服务端，告知当前客户端每次传递游戏内容的长度，这样服务端才知道每次返回多长的长度
        if(gameInfo.frame == gameInfo.delay)
        {
            register int requiredBufferLen = len * userInfo.connectionSetting;
            Instruction data(GAMEDATA);
            data.writeUnsignedInt16((unsigned short)requiredBufferLen);
            data.writeBytes(value, requiredBufferLen);
            send(data);
        }
    }	
}
```

其次是 2. 服务端每秒钟20-50次向所有客户端发送更新消息（包含所有客户端的操作和递增的帧号）：update=（FrameID，player_keyboards）响应的代码体现是

```C++
//乐观帧的机制实现,乐观帧的实现原理即为，不管当前的玩家有没有数据到来，只要到了服务端发送数据的时间，就收集当前所有玩家距离上一次
//收集到的数据包，发送给每一个用户，这样块的用户就不用等待慢的用户。。。
bool k_user::k_game::game_data_time_to_send()
{
	....
	//每15毫秒发送一次数据包
	unsigned int currentTime = GlobalTimer::getTime();
	if (currentTime - last_send_time > 15)
	{
		MyLOG("time to collect all players game data \n");
		//收集所有的玩家数据包的信息
		collect_all_players_game_data();
		last_send_time = currentTime;
	}
	...
}
//收集所有的玩家数据包的信息
void k_user::k_game::collect_all_players_game_data()
{
	//用来存储当前时间点，应该要发送出去的全部数据
	char  big_send_buffer[1024];
	//用来存储一个数据包里面存储的玩家的信息
	char  package_tmp_buffer[256];
	//用来记录每一个玩家应该要传输的内容
	char  tmp_buffer[256];
	//重置传输的buff
	memset(big_send_buffer, 0, 1024);

	//将每一个玩家当前的数据收集起来,这中有可能玩家A收集到了5个数据包，玩家B收集到了1个数据包，这里的全部数据的发送，要取最大的，也即是自己也要封装5个数据包的内容
	//至于玩家B的数据包是在玩家A第二个数据包收集的时候，收到的，还是第三个数据包收集的，这里不关心。。。只要保证传递一次就好了，客户端解析玩之后，就会表现一样
	int player_receive_max_count = 0;
	if (players.length > 0) {
		//确定当前哪个玩家收集到了最多的数据包
		for (int i = 0; i < players.length; i++) {
			k_user * ku = players.get(i);
			if (ku->receive_package_count > player_receive_max_count)
			{
				player_receive_max_count = ku->receive_package_count;
			}
		}

		//如果到了服务端要收集发送数据包的时候，没有接受到数据包的到来，服务端也要发送一个数据包，保持帧步进
		//防止客户端没有收到update信息，而出现卡主的现象
		if (player_receive_max_count == 0)
		{
			player_receive_max_count = 1;
		}

		//用来标识总的要发送的长度,第一个字节存储为当前要发送数据包的总的个数
		int totalLength = 1;
		//防止首地址发生改变，用另一个指针变量指向他
		char * temp_big_buff = big_send_buffer;

		//用一个字节来标识有多少个数据包的存在
		*temp_big_buff++ = player_receive_max_count;
		//以接受到的数据包的数量为循环，每次循环，就将所有的玩家数据都收集起来，然后下一个数据包
		for (int j = 0; j < player_receive_max_count; j++)
		{
			//收集五个数据包
			int peer_package_data_len = 0;//标识当前数据包的总长度
			for (int i = 0; i < players.length; i++)
			{
				k_user * ku = players.get(i);
				//当前玩家的状态是处于游戏中的
				if (ku->status == 2) {
					//当前玩家有数据，拉取当前玩家在服务端时间间隔之内收集到的数据包,peel_frame不会移除收集到的内容
					int len = ku->get_data(tmp_buffer);
					MyLOG("username %s real receive len %d \n", ku->username, len);
					//当前玩家没有数据到来，这个玩家当前的数据用0来填充
					int finally_frame_size = 0;
					if (len == 0) {
						if (ku->frame_size == -1)
						{
							finally_frame_size = 9;
						}
						else {
							finally_frame_size = ku->frame_size;
						}
						memset(tmp_buffer, 0, finally_frame_size);
						//填充的长度就为每次数据的长度
						len = finally_frame_size;
					}
					//当前数据包的总和
					peer_package_data_len += len;
					//将当前玩家收集的数据，添加到package_tmp_buffer里面
					memcpy(&package_tmp_buffer[len * i], tmp_buffer, len);
				}
			}
			//将temp_big_buff转为 unsigned short * ，转为什么没有关系，数据类型只是设计到了怎么解析，怎么写入,存储当前数据包里面游戏数据的长度
			(*(unsigned short *)temp_big_buff) = peer_package_data_len;
			//俩个字节存储当前的这个数据包有多少个gameData字节
			temp_big_buff += 2;
			totalLength += 2;
			//再将package_tmp_buffer里面的内容，添加到temp_big_buff里面
			memcpy(temp_big_buff, package_tmp_buffer, peer_package_data_len);
			//指针位移
			temp_big_buff += peer_package_data_len;
			totalLength += peer_package_data_len;
		}

		//然后给每一个玩家都发送同样的数据
		for (int i = 0; i < players.length; i++) {
			k_user * ku = players.get(i);
			//发送数据
			ku->send_data(big_send_buffer, totalLength);
			//标识当前玩家有发送数据出去
			ku->untrimmed_used_data = 1;
		}

		//删除集合buff里面缓存的内容
		frame_counter = frame_counter + 1;
		if (players.length > 0) {
			for (int edi = 0; edi < players.length; edi++) {
				k_user * kux = players.get(edi);
				if (kux->untrimmed_used_data != 0) {
					//服务端接受的数据包重置为0
					kux->receive_package_count = 0;
					//移除服务端收集数据包的时间段内的数据包
					//kux->get_data(big_send_buffer);
					kux->client_reply_frame++;
					kux->untrimmed_used_data = 0;
				}
			}
		}
	}
}
```

最后是：
3. 客户端就像播放游戏录像一样不停的播放这些包含每帧所有玩家操作的 update消息。
4. 客户端如果没有update数据了，就必须等待，直到有新的数据到来。
5. 客户端如果一下子收到很多连续的update，则快进播放。

```C++
// recv input data 同步的接受数据
int  N02CCNV recvSyncData(void * value, const int len)
{
    if (state==RUNNING) {
       if (++gameInfo.frame >= gameInfo.delay) {
            //同步等待服务端的关键帧的到来
            while (!gameInfo.isProcessFinish && state == RUNNING) {
                stepRunning();
            }
            ...
            //保证在运行的状态下，才去获取数据
            if(state == RUNNING)
            {
                //遍历解析每一个关键帧里面的游戏数据
                for(int i = 0;i<gameInfo.currentReceiveGameCount;i++) {
                    //获取到关键帧
                    k_game_data_ptr kInstructionPtr = gameInfo.inCache.getItem(gameInfo.gameDataForArrayIndex + i);
                    //解析每一个关键帧里面包含的数据包
                    unsigned char *gameP = (unsigned char *)kInstructionPtr.body;
                    //解析游戏数据
                    //得到有多少个数据包的存在
                    int packageCount = *gameP;
                    //指针移动一个字节，指向后面要解析的内容
                    gameP += 1;

                    //客户端收到了服务端多个数据包，客户端要加速的播放处理,遍历处理每一个数据包
                    while (packageCount > 1) {
                        memset(nControls,0,INPUTSIZE);
                        //获取当前这个数据包的长度
                        unsigned short length = *((unsigned short *) gameP);
                        //后面的就是游戏的内容了
                        gameP += 2;
                        //通知引擎要加速播放，传递的参数为游戏的数据
                        memcpy(nControls,gameP,length);
                        //这里要注意不能单单的返回length长度的内容，游戏引擎还会解析对应玩家的按键信息，所以返回的buff的长度要正确
                        modHelper.speedGameRunInfo(nControls, len);

                        //下一次循环设置
                        gameP += length;
                        packageCount--;
                    }

                    //标识处理了最后一个关键帧,最后一个关键帧要执行return，要不然客户端会一直等待
                    if (i == gameInfo.currentReceiveGameCount - 1) {
                        //客户端正常处理，数据包为1
                        //获取当前这个数据包的长度
                        unsigned short length = *((unsigned short *) gameP);
                        //后面的就是游戏的内容了
                        gameP += 2;
                        //最后一个数据包的解析
                        memcpy(value, gameP, length);
					} else {
                        memset(nControls,0,INPUTSIZE);
                        //获取当前这个数据包的长度
                        unsigned short length = *((unsigned short *) gameP);
                        //后面的就是游戏的内容了
                        gameP += 2;
                        //通知引擎要加速播放，传递的参数为游戏的数据
                        //这里要注意不能单单的返回length长度的内容，游戏引擎还会解析对应玩家的按键信息，所以返回的buff的长度要正确
                        memcpy(nControls,gameP,length);
                        modHelper.speedGameRunInfo(nControls, len);   
                        }
                    }
                    return len;
                }
        } else {
            memcpy(value, gameInfo.defaultInput, gameInfo.totalInputLength);
            return 0;
        }
    }
    LOGD("Ending it here");
    modHelper.endGame();
    return -1;
}
```

相应的客户端加速也要在游戏引擎中设置对应的快进的回调，gameSpeedRunCallBack 函数指针

```C++
//设置游戏加速的回调
speedGameInfo.speedGameRunInfo = gameSpeedRunCallBack;
kailleraSetSpeddGameRunInfo(&speedGameInfo);

//游戏加速的回调
void gameSpeedRunCallBack(void * data, int len) {

	//前面是解析到传递过来的按键信息，然后设置到对应的游戏引擎中
	//也即是游戏的状态都是取服务端返回的数据的结果，nControls 代表服务端返回的游戏数据，下面解析数据，改变对应的值
	//到了这里的话，就代表玩家1的按键已经发送到了服务端，而且也已经获取到了其他玩家的按键情况,这里就要解析这些按键的值
	//playersInputs = {8,8,0,0}  , nPlayerOffset = {0,8,16,16};
	for (i = 0, j = 0; i < nPlayerInputs[0]; i++, j++) {
		//获取信息,将信息存储到 bii 中
		BurnDrvGetInputInfo(&bii, i + nPlayerOffset[0]);
		if (bii.nType == BIT_DIGITAL) {
			//*bii.pVal = 0x01; 修改按键的值，只要改为1就表示按了这个按键
			if (nControls[j >> 3] & (1 << (j & 7))) {
				//记录收到的操作
				//WRITE_LOG("BurnInputInfo receive nPlayerInputs name %s\n",bii.szName);
				*bii.pVal = 0x01;
			} else {
				*bii.pVal = 0x00;
			}
		}
	}
	....

	//游戏界面的渲染
	//都是定义在burn.h 里面的，使用  extern UINT32 nFramesRendered ，这样的方式
	nFramesRendered++;
	//正常渲染界面的方法
	if (VidFrame()) {					// Do one frame
		memset(nAudNextSound, 0, nAudSegLen << 2);//extern INT32 nAudSegLen;  在interface.h文件中
	}
	SndProcessFrame();
}
```
