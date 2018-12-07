---
layout: pager
title: Aria2 改造Tracker逻辑
date: 2018-12-06 09:22:43
tags: [NDK,Aria2]
description: Aria2 改造Tracker逻辑
---

### 概述

> Aria2 改造Tracker逻辑

<!--more-->

### 简介
> 最近在使用Aria2的时候，发现了一个问题，Aira2中的Tracker 地址请求是独立的，也即不是并发的请求的，对于一个种子有多个Tracker 的时候，当第一个主机跟第一个Tracker 地址连接上之后，对于其他的Tracker 地址，Aria2不会进行连接了，如果第一个Tracker地址请求失败的时候，才会去连接第二个Tracker地址，以此类推那么就会出现这样的情况，比如：用户A连接上TrackerA的地址，用户B一开始连接不上Tracker A的地址，那么用户B连接Tracker B的地址，而且连接上了，此时用户A就无法知道用户B的存在，按理来说，对于同一个种子，应该要彼此知道对方的存在，才能更好的实现连接的效果，也即是Tracker 之间应该存在沟通,而相比Utorrent 就做的很好，下面是他的效果

Utorrent Trakcer 实现的效果
![结果显示](/uploads/Aria2Tracker改造/UtorrentTracker.png)
可以看到对于Utorrent 中的Tracker 是并发执行的，每一个Trakcer 都有自己的下一次刷新的时间，对于各过Tracker之间也是存在通信的，可以按照上面的做法去验证，幸运的是对于开源的Bt下载，还有linux 下面的Transmission，我们可以看看他们是怎么实现的
### Transmission Trakcer分析
```C
//这个函数的执行是在libEvent线程执行的
static void tr_sessionInitImpl(void * vdata) {
	tr_variant settings;
	...	
	//初始化Announce
	tr_announcerInit(session);		
	...
}

//初始话Announcer 对象
void tr_announcerInit(tr_session * session) {
	tr_announcer * a;

	//确保session为Session对象
	assert(tr_isSession(session));

	//构建一块内存空间tr_announcer
	a = tr_new0(tr_announcer, 1);
	//赋默认值
	a->stops = TR_PTR_ARRAY_INIT;
	//随机一个关键值做为key
	a->key = tr_rand_int(INT_MAX);
	a->session = session;
	//赋值最多可以并发的任务数量，默认为48，也即是所有种子Tracker并发数量不应该超过这个数值
	a->slotsAvailable = MAX_CONCURRENT_TASKS;

	//创建一个定时器对象,用来更新Anouncer,当定时器执行的时候，会执行onUpkeepTimer 函数指针,a做为这个函数执行的时候传递的参数
	a->upkeepTimer = evtimer_new(session->event_base, onUpkeepTimer, a);
	//激活定时器
	tr_timerAdd(a->upkeepTimer, UPKEEP_INTERVAL_SECS, 0);

	//将当前的annoucer对象保存到sesssion对象中
	session->announcer = a;
}
这个函数主要做的就是，初始化了一个tr_announcer 对象，这个对象做为全局的Tracker 的管理，比如定义了定时器，最多的Tracker 并发数量,然后将这个对象保存到session中,
接着分析下单个种子的Tracker的初始化过程

//种子对象的初始话
static void torrentInit(tr_torrent * tor, const tr_ctor * ctor) {
	...
	//构建一个tr_torrent_tiers ,完成tor 中tiers成员的赋值,onTrackerResponse 为tracker 服务器响应的回调函数，NULL为回调函数执行的时候传递的参数
	tor->tiers = tr_announcerAddTorrent(tor, onTrackerResponse, NULL);
	
	//添加开始事件
	tr_announcerTorrentStarted(tor);
	...
	
}
//tr_announcerAddTorrent 函数实现
tr_torrent_tiers * tr_announcerAddTorrent(tr_torrent * tor, tr_tracker_callback callback,
		void * callbackData) {
	tr_torrent_tiers * tiers;
	//校验参数
	assert(tr_isTorrent(tor));

	//分配一块大小为tr_torrent_tiers 内存空间
	tiers = tiersNew();
	//tracker 请求的回调函数
	tiers->callback = callback;
	//trakcer 回调函数中传递的参数
	tiers->callbackData = callbackData;

	//为tr_torrent_tiers 中的trackers赋值，并且填充内容
	addTorrentToTier(tiers, tor);

	return tiers;
}

//addTorrentToTier 函数实现 赋值 tr_torrent_tiers 数组，从tor中获取到信息
static void addTorrentToTier(tr_torrent_tiers * tt, tr_torrent * tor) {
	int i, n;
	int tier_count;
	tr_tier * tier;

	//获取到当前种子合法的tracker url 数组,过滤掉那些不合法的
	tr_tracker_info * infos = filter_trackers(tor->info.trackers,tor->info.trackerCount, &n);

	/* build the array of trackers */
	//分配一块内存空间 数量为 n 类型 为 tr_tracker
	tt->trackers = tr_new0(tr_tracker, n);
	//标识当前种子含有多少个tracker
	tt->tracker_count = n;

	//为tt 中的成员变量 trackers数组填充内容
	for (i = 0; i < n; ++i)
	{
		//将info的对象的内容填充到trackers 数组成员中
		trackerConstruct(&tt->trackers[i], &infos[i]);
	}

	/* count how many tiers there are */
	//计算tier的数量,依据是根据infos成员中相邻的两个成员是否有一样的tier，如果不一样tier_count就加一处理,也即是过滤掉重复的tracker
	tier_count = 0;
	for (i = 0; i < n; ++i)
	{
		///如果infos相邻成员tier是否不一样,如果不一样，就加一处理,因为前面已经执行过了排序，所以这边比较下俩个相邻的是否一样就可以了，主要是用来过滤掉重复的tracker
		if (!i || (infos[i].tier != infos[i - 1].tier))
		{
			++tier_count;
		}
	}

	/* build the array of tiers */
	//构建tiers 数组
	tier = NULL;

	//根据tier_count的大小，分配一个数组，返回这块内存空间的首地址,然后赋值给tiers 成员
	tt->tiers = tr_new0(tr_tier, tier_count);
	tt->tier_count = 0;

	for (i = 0; i < n; ++i) {
		//如果infos相邻的元素 tier值一样，那么tracker_count 加1处理
		if (i && (infos[i].tier == infos[i - 1].tier))
		{
			//存在一样的tracker ，那么tier 所代表的tracker  ，tracker_count 表示这个tracker 存在 tracker_count 个
			++tier->tracker_count;
		}
		else {
			//如果不相同，则为tt数组中的每一个元素执行赋值操作
			tier = &tt->tiers[tt->tier_count++];

			//将tor中的内容，赋值给tier
			tierConstruct(tier, tor);
			
			//赋值当前索引对应的tracker数组,后面可以根据trakcer_currentIndex  得到对应的 currentTracker
			tier->trackers = &tt->trackers[i];

			//标识当前的tier只有一个tracker,默认是只有一个，表示代表多少个tracker,销毁的时候可以根据这个值来准确的销毁
			tier->tracker_count = 1;
			//给当前 tier 成员 赋值currentTracker 以及currentIndex 重置默认值
			tierIncrementTracker(tier);
		}
	}
	/* cleanup */
	tr_free(infos);
}

//而其中 tr_torrent_tiers 结构体为
typedef struct tr_torrent_tiers {
	//用来存储过滤之后的tracker数组, 这个是不可能包含重复的tracker
	tr_tier * tiers;
	//标识tier数组的长度
	int tier_count;

	//用来存储当前的tracker 数组,这个是可能包含重复的tracker
	tr_tracker * trackers;

	//用来标识当前种子的 含有多少 tracker ,这个是包含重复的tracker
	int tracker_count;

	//tracker请求结果的回调处理函数
	tr_tracker_callback callback;

	//tracker请求的回调处理函数中传递的参数
	void * callbackData;
} tr_torrent_tiers;

所以上面所做的操作就是 将当前种子解析出来的Trakcer 信息，填充到tr_torrent_tiers 结构体中，这个结构体代表一个种子所包含的Tracker信息，而其中tr_tracker 跟 tr_tier
的区别在以是否允许重复,接着执行 tr_announcerTorrentStarted(tor); 

//tracker请求开始
void tr_announcerTorrentStarted(tr_torrent * tor) {
	//标识当前的种子 Tracker请求开始 ,事件为 TR_ANNOUNCE_EVENT_STARTED
	printf("tracker tr_announcerTorrentStarted execute \n");
	torrentAddAnnounce(tor, TR_ANNOUNCE_EVENT_STARTED, tr_time());
}

//给当前种子所有的tracker 添加一个事件,以及这个 事件触发的时间，这个时间可以用来判断是否需要执行
static void torrentAddAnnounce(tr_torrent * tor, tr_announce_event e,
		time_t announceAt) {
	int i;
	//获取到之前存储在tro中的 tiers tracker集合
	struct tr_torrent_tiers * tt = tor->tiers;
	/* walk through each tier and tell them to announce */
	//遍历每一个tt数组，给每一个 tr_tier 添加事件 e
	for (i = 0; i < tt->tier_count; ++i)
	{
		tier_announce_event_push(&tt->tiers[i], e, announceAt);
	}
}

//添加事件  tier为需要为哪个trakcer添加事件， e为要添加的事件类型 announceAt为这个事件需要触发执行的时间
static void tier_announce_event_push(tr_tier * tier, tr_announce_event e,time_t announceAt) {
	int i;

	//参数的校验
	assert(tier != NULL);

	dbgmsg_tier_announce_queue(tier);
	dbgmsg(tier, "queued \"%s\"", tr_announce_event_get_string(e));

	//如果当前Announce 事件数量不为0
	if (tier->announce_event_count > 0) {
		/* special case #1: if we're adding a "stopped" event,
		 * dump everything leading up to it except "completed" */
		//如果当前是停止的事件
		if (e == TR_ANNOUNCE_EVENT_STOPPED) {
			//如果当前是停止的事件，我们要判断当前事件列表中是否有完成的事件
			bool has_completed = false;
			//判断当前tier 的事件列表中是否有完成的事件
			const tr_announce_event c = TR_ANNOUNCE_EVENT_COMPLETED;
			for (i = 0; !has_completed && i < tier->announce_event_count; ++i)
			{
				has_completed = c == tier->announce_events[i];
			}
			//因为要退出了，将事件数量改为0
			tier->announce_event_count = 0;
			//如果已经完成了，添加一个完成的事件
			if (has_completed)
			{
				tier->announce_events[tier->announce_event_count++] = c;
			}
		}
		/* special case #2: dump all empty strings leading up to this event */
		//移除TR_ANNOUNCE_EVENT_NONE 的事件
		tier_announce_remove_trailing(tier, TR_ANNOUNCE_EVENT_NONE);
		/* special case #3: no consecutive duplicates */
		//移除重复的事件
		tier_announce_remove_trailing(tier, e);
	}

	/* make room in the array for another event */
	//确保含有足够的内存空间,刚开始的时候为0
	if (tier->announce_event_alloc <= tier->announce_event_count) {
		//不够，扩大内存
		tier->announce_event_alloc += 4;
		//扩大内存空间，赋值首地址announce_events
		tier->announce_events = tr_renew(tr_announce_event,tier->announce_events, tier->announce_event_alloc);
	}

	/* add it */
	//将当前的事件添加到这个事件的集合中
	tier->announce_events[tier->announce_event_count++] = e;
	//赋值当前AnnounceAt的时间
	tier->announceAt = announceAt;
	...
}
tr_tier 结构体为，这个结构体代表种子里面一个Tracker 的状态信息
typedef struct tr_tier {
	...
	//tracker Annouce的时间
	time_t announceAt;

	//代表当前 Announce 事件的数组
	tr_announce_event * announce_events;

	//announce  表示当前 Announce 事件集合中含有多少个事件，标识当前事件的个数
	int announce_event_count;

	//announce evnet当前已经分配的
	int announce_event_alloc;
	//默认的Annouce 刷新的间隔时间,这个值的作用是在当tracker请求成功之后，用来安排当前tracker下一次触发连接的时间间隔
	int announceIntervalSec;

	//最小的Announce 刷新间隔时间
	int announceMinIntervalSec;

	//标识上一次trancer请求返回的peer数量
	int lastAnnouncePeerCount;

	bool isRunning;
	//标识当前的tier是否正在执行Annoue
	bool isAnnouncing;
	...
}

所以 tr_announcerTorrentStarted(tor); 所做的事情就是给当前种子中的每一个Trakcer 的封装对象 tr_tier 添加一个  TR_ANNOUNCE_EVENT_STARTED 事件，此时每一个种子的事件队列都不为空，
数量数量也不为0,announceAt 为当前事件触发的时间，也被赋值为当前的时间


单个种子已经初始化完成，接着定时器开始执行，触发 onUpkeepTimer 函数执行
//announcer 定时器执行的回调函数,这个定时器是要处理当前所有正在运行的种子的
static void onUpkeepTimer(evutil_socket_t foo UNUSED, short bar UNUSED,
		void * vannouncer) {
	...
	//如果当前不退出，就要去执行当前运行种子的tracker
	if (!is_closing) {
		announceMore(announcer);
	}
	...
}

//Announce的初始化操作
static void announceMore(tr_announcer * announcer) {
	int i;
	int n;
	tr_torrent * tor;

	tr_ptrArray announceMe = TR_PTR_ARRAY_INIT;

	//获取当前的时间
	const time_t now = tr_time();

	//如果当前允许的tracker并发任务数量小于1代表不能创建tracker 任务了
	if (announcer->slotsAvailable < 1)
	{
		return;
	}

	//建立需要公布的层级列表
	/* build a list of tiers that need to be announced */
	tor = NULL;

	//遍历所有的种子,注意这里是所有的种子
	while ((tor = tr_torrentNext(announcer->session, tor))) {

		//获取到当前种子的 tr_torrent_tiers ,   也即是tracker服务器的列表,  这个在前面的操作 tr_announcerAddTorrent已经初始话了
		struct tr_torrent_tiers * tt = tor->tiers;

		//遍历存储的tt集合的元素
		for (i = 0; tt && i < tt->tier_count; ++i) {
			tr_tier * tier = &tt->tiers[i];

			//当前的tier是否需要添加到集合中,其中有这样的判断 tier->announceAt != 0) && (tier->announceAt <= now),也即是判断当前事件的触发时间是否到了，如果没有达到不会触发添加操作
			if (tierNeedsToAnnounce(tier, now))
			{
				  //如果需要将tier添加到集合announceMe中，并完成了内存空间的分配
				  tr_ptrArrayAppend(&announceMe, tier);
			}
			...
		}
	}
	
	/* if there are more tiers than slots available, prioritize */
	//获取到announceMe集合中元素的个数
	n = tr_ptrArraySize(&announceMe);

	//最多可以允许的任务数量 默认为 48
	if (n > announcer->slotsAvailable)
	{
		//如果大于48要执行排序
		qsort(tr_ptrArrayBase(&announceMe), n, sizeof(tr_tier*), compareTiers);
	}

	/* announce some */
	//最多可以允许的任务数量 默认为 48 ,所以一般这里n的值就为announceMe的大小
	n = MIN(tr_ptrArraySize(&announceMe), announcer->slotsAvailable);

	//printf("announceMore tracker numbers %d \n",n);
	//同时执行每一个tracker
	for (i = 0; i < n; ++i) {
		//获取到announceMe 中i索引对应的值
		tr_tier * tier = tr_ptrArrayNth(&announceMe, i);
		tr_logAddTorDbg(tier->tor, "%s", "Announcing to tracker");
		printf( "announcing tier %d of %d \n", i, n);
		//将tier添加到Announcer中
		tierAnnounce(announcer, tier);
	}
	...
}

//当前的tier是否需要添加到Announce中
static bool tierNeedsToAnnounce(const tr_tier * tier, const time_t now) {
	//在刚开始添加的时候，就执行了tier_announce_event_push 函数，中赋值 announceAt 时间为 tr_time()。也即是当前的时间，并且 tier->announce_event_count++ ，添加了一个开始的事件
	return !tier->isAnnouncing && !tier->isScraping && (tier->announceAt != 0) && (tier->announceAt <= now) && (tier->announce_event_count > 0);
}

在初始化种子的时候，给当前种子的每一个Tracker 的封装对象 tr_tier 添加了一个开始的事件，所以这些Tracker 都会满足这个条件 isAnnouncing 代表是否正在执行请求，默认为false, 
isScraping为ipv6所有的，这里可以忽略，默认也为false, 将这些满足条件的tr_tier 添加到集合announceMe，然后根据当前最大的trakcer并发数量，决定还能触发多少个连接，最后执行 tierAnnounce()

//将tier的内容，构建一个tracker的请求，执行发送
static void tierAnnounce(tr_announcer * announcer, tr_tier * tier) {
	tr_announce_event announce_event;
	tr_announce_request * req;
	struct announce_data * data;

	//获取到种子对象
	tr_torrent * tor = tier->tor;
	//标识当前的时间点,用于tracker的超时管理
	const time_t now = tr_time();

	//从 tier 对象 中 获取第一个tr_announce_event, 同时announce_events 集合中移除
	announce_event = tier_announce_event_pull(tier);
	printf("tierAnnounce  announce_event %d \n",announce_event);

	//构建一个tr_announce_request 对象 ，封装了tracker请求的内容
	req = announce_request_new(announcer, tor, tier, announce_event);

	//构建一个announce_data
	data = tr_new0(struct announce_data, 1);

	//赋值session引用
	data->session = announcer->session;
	//赋值id为 当前tier的key
	data->tierId = tier->key;
	//赋值tier是否正在运行
	data->isRunningOnSuccess = tor->isRunning;
	//标识发送的时间，这里为当前
	data->timeSent = now;
	//赋值event为当前的事件,这个后面会用到，当请求失败的时候，会重发这个事件，会取这个事件
	data->event = announce_event;

	//标识当前的tier 是正在执行,这个也是挺重要的，tierNeedsToAnnounce（）函数中，根据这个值来判断，也即是实现的效果是前面一个请求执行完成了后面的才能执行
	tier->isAnnouncing = true;
	//标识上一次Announce的起始时间
	tier->lastAnnounceStartTime = now;

	//slotsAvailable 减一处理，也即是全局 还剩下多少个tracker 任务同时执行
	--announcer->slotsAvailable;

	//设置请求的代理  on_announce_done 为最终处理完后的回调函数，比如是http 请求的时候，他会自己有一个回调处理函数，但是最终还要转到这个回调函数   data为这个回调函数执行的参数
	announce_request_delegate(announcer, req, on_announce_done, data);
}

//从tier成员变量 announce_events 集合中，移出最前面的一个tr_announce_event，同时announce_event_count--
static tr_announce_event tier_announce_event_pull(tr_tier * tier) {
	//取出第一个事件，并且返回， 同时将这个事件从当前的事件集合中移除出去
	const tr_announce_event e = tier->announce_events[0];
	//将当前的成员从tier 的 announce_events 集合中移除，并且announce_event_count --
	tr_removeElementFromArray(tier->announce_events, 0,	sizeof(tr_announce_event), tier->announce_event_count--);
	//返回这个被移除的内容
	return e;
}

static tr_announce_request *announce_request_new(const tr_announcer * announcer, tr_torrent * tor,const tr_tier * tier, tr_announce_event event) {

	//首先构建一个tr_announce_request对象
	tr_announce_request * req = tr_new0(tr_announce_request, 1);
	//设置端口,这里的端口为public_peer_port
	req->port = tr_sessionGetPublicPeerPort(announcer->session);
	//当前tracker请求的url
	req->url = tr_strdup(tier->currentTracker->announce);
	//当前的trackerId
	req->tracker_id_str = tr_strdup(tier->currentTracker->tracker_id_str);
	//赋值种子的唯一hash
	memcpy(req->info_hash, tor->info.hash, SHA_DIGEST_LENGTH);
	//赋值种子的 peer_id
	memcpy(req->peer_id, tr_torrentGetPeerId(tor), PEER_ID_LEN);
	//当前上传的长度
	req->up = tier->byteCounts[TR_ANN_UP];
	//当前已经下载的长度
	req->down = tier->byteCounts[TR_ANN_DOWN];
	//告知当前下载了多少错误的长度
	req->corrupt = tier->byteCounts[TR_ANN_CORRUPT];
	//还剩下多少没有完成
	req->leftUntilComplete = tr_torrentHasMetadata(tor) ? tor->info.totalSize - tr_torrentHaveTotal(tor) :~(uint64_t) 0;
	//设置事件
	req->event = event;
	//设置我们需要同行的回复大小，如果当前的事件不为TR_ANNOUNCE_EVENT_STOPPED ，则默认为80个
	req->numwant = event == TR_ANNOUNCE_EVENT_STOPPED ? 0 : NUMWANT;
	req->key = announcer->key;
	//赋值是否局部做种
	req->partial_seed = tr_torrentGetCompleteness(tor) == TR_PARTIAL_SEED;
	//日志打印
	tier_build_log_name(tier, req->log_name, sizeof(req->log_name));
	//printf("tracker url %s create \n",req->url );
	return req;
}

tierAnnounce 函数所做的就是将 当前tracker的封装对象tr_tier 中取出第一个事件，并构建tr_announce_request，announce_data 赋值相应的值，比如 data->event = announce_event;
用户请求失败的时候，这个事件再次的触发,tier->isAnnouncing = true; 标识当前tier是否正在执行请求 ,接着执行 announce_request_delegate(announcer, req, on_announce_done, data);

//给announce设置代理请求
static void announce_request_delegate(tr_announcer * announcer,
		tr_announce_request * request, tr_announce_response_func callback,
		void * callback_data) {
	//获取到session 引用
	tr_session * session = announcer->session;
	//根据url的类型，构建不同的请求对象
	if (!memcmp(request->url, "http", 4))
	{
		//如果为http,构建http 的请求
		tr_tracker_http_announce(session, request, callback, callback_data);
	}
	else if (!memcmp(request->url, "udp://", 6))
	{
		//如果为udp,构建 udp的tracker,这里会将 tr_announce_request 的内容 添加到
		tr_tracker_udp_announce(session, request, callback, callback_data);
	}
	else
	{
		//不支持的类型，打印错误
		tr_logAddError("Unsupported url: %s", request->url);
	}

	//释放这个request, tr_announce_request 是在这里完成释放的
	announce_request_free(request);
}
announce_request_delegate 会根据当前trakcer的类型，执行不同的逻辑，先分析对应udp的类型,on_announce_done 为最终trakcer处理完成之后的回调函数，data为参数，为 announce_data 类型
由于这里主要分析trakcer之间的沟通，对于tracker请求的发送以及接收这里就不细看，处理之后的最终回调函数为 on_announce_done

//回调函数，当announce执行完毕之后的回调函数,当tracker响应成功之后，并且解析到返回的peer数组，就会触发这个方法,回调函数的参数为announce_data
//这个回调函数，不管是请求成功，还是请求失败都会触发这个最终的结果
static void on_announce_done(const tr_announce_response * response,
		void * vdata) {

	struct announce_data * data = vdata;
	tr_announcer * announcer = data->session->announcer;

	//根据info_hash，以及 tierId 获取到对应的tr_tier对象
	tr_tier * tier = getTier(announcer, response->info_hash, data->tierId);
	//获取到当前的时间
	const time_t now = tr_time();

	//获取到当前的事件,也即是当前tracker 执行的对应的event
	const tr_announce_event event = data->event;

	if (announcer)
	{
		//当前trakcer请求成功了，可以让并发的任务标识加一了
		++announcer->slotsAvailable;
	}

	if (tier != NULL) {
		tr_tracker * tracker;

		//重置变量
		//上一次Announce的时间
		tier->lastAnnounceTime = now;
		//赋值上一次请求tracker是否超时
		tier->lastAnnounceTimedOut = response->did_timeout;
		tier->lastAnnounceSucceeded = false;
		//当前请求结束了，所以置为false
		tier->isAnnouncing = false;
		tier->manualAnnounceAllowedAt = now + tier->announceMinIntervalSec;

		if (!response->did_connect) {
			//连接不上tracker
			on_announce_error(tier, _("Could not connect to tracker"), event);
		} else if (response->did_timeout) {
			//tracker 没有响应,设置当前tracker的时间，也即是下次多久再次触发他的连接
			on_announce_error(tier, _("Tracker did not respond"), event);
		} else if (response->errmsg) {
			//这个是tracker可以连接成功，也没有超时，但是还有错误信息
		
			if (tier->tor->info.trackerCount < 2)
			{
				publishError(tier, response->errmsg);
			}
			on_announce_error(tier, response->errmsg, event);
		} else {
			//tracker 请求成功处理
			
			//返回的最小刷新的时间
			if ((i = response->min_interval))
			{
				tier->announceMinIntervalSec = i;
			}
			...
			//如果当前不是停止事件，并且 当前 事件数量为 0,代表当前没有事件
			if (!isStopped && !tier->announce_event_count) {
				/* the queue is empty, so enqueue a perodic update */

				//获取到announce 刷新的时间间隔
				i = tier->announceIntervalSec;
				dbgmsg(tier, "Sending periodic reannounce in %d seconds", i);

				//添加一个事件 TR_ANNOUNCE_EVENT_NONE ，也即是当前tracker 下一次刷新的时间间隔
				tier_announce_event_push(tier, TR_ANNOUNCE_EVENT_NONE, now + i);
				printf("get Tracker success  put nextTrackerEvent  time %d \n",i);
			}
		}
	}
	tr_free(data);
}

首先请求完成，让并发tracker数量加一，如果当前tracker是请求失败，会调用on_announce_error()同时传递了当前tracker对应的event
//当前的tracker 出现了错误
static void on_announce_error(tr_tier * tier, const char * err , tr_announce_event e) {
	int interval;

	/* increment the error count */
	if (tier->currentTracker != NULL)
	{
		//标识当前tracker 失败的次数加一
	    ++tier->currentTracker->consecutiveFailures;
	}
	...
	/* schedule a reannounce */
	//根据当前tracker的失败情况，决定他下次触发tracker的连接时间
	interval = getRetryInterval(tier->currentTracker);

	//给当前的tier 内部的 Event事件结婚，添加一个Event, 触发时间为 tr_time() + interval
	tier_announce_event_push(tier, e, tr_time() + interval);
}

//获取到tracker失败重试的刷新时间,这个会跟 当前tracker失败的次数有关，失败次数雨多，时间越长
static int getRetryInterval(const tr_tracker * t) {
	switch (t->consecutiveFailures) {
	case 0:
		//如果一次都没有失败过,为0
		return 0;
	case 1:
		//失败过一次为 20
		return 20;
	case 2:
		return tr_rand_int_weak(60) + (60 * 5);
	case 3:
		return tr_rand_int_weak(60) + (60 * 15);
	case 4:
		return tr_rand_int_weak(60) + (60 * 30);
	case 5:
		return tr_rand_int_weak(60) + (60 * 60);
	default:
		return tr_rand_int_weak(60) + (60 * 120);
	}
}
on_announce_error 函数所做的就是，让当前tier的失败次数加一处理，然后根据当前的失败次数，决定下次trakcer触发的事件,通过tier_announce_event_push 再次添加当前的事件,只是改了时间
对应tracker请求成功 当前trakcer的刷新时间会取tracker服务器返回的刷新时间,然后判断当前tier事件队列中是否还有事件，如果没有就添加一个TR_ANNOUNCE_EVENT_NONE ,触发的时间就为tracker服务器返回
的刷新时间


对应当一个peer下载完成的时候，会触发
//给当前种子对应的所有tracker发送完成的事件
void tr_announcerTorrentCompleted(tr_torrent * tor) {
	torrentAddAnnounce(tor, TR_ANNOUNCE_EVENT_COMPLETED, tr_time());
}

//给当前种子所有的tracker 添加一个事件,以及这个 事件触发的时间，这个时间可以用来判断是否需要执行
static void torrentAddAnnounce(tr_torrent * tor, tr_announce_event e,
		time_t announceAt) {
	int i;
	//获取到之前存储在tro中的 tiers tracker集合
	struct tr_torrent_tiers * tt = tor->tiers;
	/* walk through each tier and tell them to announce */
	//遍历每一个tt数组，给每一个 tr_tier 添加事件 e
	for (i = 0; i < tt->tier_count; ++i)
	{
		  tier_announce_event_push(&tt->tiers[i], e, announceAt);
	}
}

也即是会给当前种子的所有trakcer 的封装对象 tier 添加一个 TR_ANNOUNCE_EVENT_COMPLETED 事件，同时事件的触发时间点为当前时间，对于其他的trakcer当前如果没有请求执行的时候，就会立刻发起一个
完成的请求事件告知trakcer 服务器，经管当前我下载的内容不是来自你这个服务器，但是我也告诉了你，这样其他的用户就知道从我这里下载，这就是tracker之间的通信
```
### Transmission Trakcer总结
> 首先是允许的Tracker并发数量最大为48，这是针对所有的种子的设置，如果当前正在运行的数量达到了这个最大值设置，后面要执行tracker请求的，要等待，并且有是一个定时器，当定时器触发的时候，会判断所有种子的各自对应的tracker是否应该执行，而判断是否应该执行，是通过当前Tracker中的事件队列是否不为空，当前Tracker 没有正在执行Tracker请求，事件触发的时间点要小于当前的时间，这些条件都满足的化，当前的这个tracker就允许执行，在种子初始化的时候，就会为当前种子的所有Tracker 都添加一个START的事件，所以，对于新的种子，第一次执行Tracker连接的时候，默认都会执行Tracker的连接这是并发执行的，Tracker执行连接的时候，会设置标识当前正在执行请求，对于请求失败的Tracker，比如超时，或者出现了异常，错误等，会再次的添加这个事件到事件队列中，只是这次事件的触发时间是未来的一个时间这个会根据当前Tracker的失败次数来决定的，失败次数越多，这个时间会越久,对于成功的请求，也会添加一个事件到事件队列中，这个时候Tracker触发的时间点取的是Tracker服务器返回的时间，当一个种子下载完成的时候每一个Tracker都会添加一个COMPLETE的事件，并且事件触发的时间点为当前的事件，如果当前tracker正好处于空闲的状态，那么这个COMPLETE的事件会立刻执行请求，如果当前tracker还处于请求中，那么要等待当前的事件处理完毕，才会发送这个COMPLETE事件，对于Transmission中的 Tracker 沟通，个人理解就是通过开始的时候，并发请求每一个Tracker，当种子下载完成的时候，又告知每一个Tracker，当前我已经下载完成了，这样所有的Tracker服务器都会知道我这个节点下载完成了，这样就可以做到 不同的Trakcer 服务器的用户也能知道哪里可以下载

### Aria2 Trakcer分析
```C++

Aria2 Tracker的逻辑都是由  TrackerWatcherCommand 来触发的,相比于 transmission，Aria2 的做法是每一个种子文件就一个对应的TrackerWatcherCommand对象，再分析这个command对象之前，
先看一下DefaultBtAnnounce 这个对象，这个对象用于管理Tracker的请求，做到任何时刻都会只有一个请求正在执行，下面是他的初始化函数

DefaultBtAnnounce::DefaultBtAnnounce(DownloadContext* downloadContext,
                                     const Option* option)
    : downloadContext_{downloadContext},
      trackers_(0), //标识当前正在连接的tracker数量
      prevAnnounceTimer_(Timer::zero()),   //用来做定时器，
      interval_(DEFAULT_ANNOUNCE_INTERVAL),//constexpr static auto DEFAULT_ANNOUNCE_INTERVAL = 2_min;         默认的tracker 的刷新时间
      minInterval_(DEFAULT_ANNOUNCE_INTERVAL),//constexpr static auto DEFAULT_ANNOUNCE_INTERVAL = 2_min;	  最小的tracker 的刷新时间		
      userDefinedInterval_(0_s),			  //用户定义的 tracker刷新时间，
      complete_(0),
      incomplete_(0),
      announceList_(bittorrent::getTorrentAttrs(downloadContext)->announceList),    //根据当前种子文件包含的 Tracker 集合 初始化AnnounceList
      option_(option),
      randomizer_(SimpleRandomizer::getInstance().get()),
      tcpPort_(0)
{

}

//AnnounceList 构造函数的执行
AnnounceList::AnnounceList(const std::vector<std::vector<std::string>>& announceList) : currentTrackerInitialized_(false)
{
  reconfigure(announceList);
}

//将每一个Trakcer 中的url
void AnnounceList::reconfigure(const std::vector<std::vector<std::string>>& announceList)
{
  for (const auto& vec : announceList) {
    if (vec.empty()) {
      continue;
    }
	//注意这里是集合，对于一个Tracker url 下面还有集合的目前还没有见过
    std::deque<std::string> uris(std::begin(vec), std::end(vec));
	//根据当前Tracker 下面的url集合，构建一个AnnounceTier 对象,然后添加到tiers_ 集合中
    auto tier = std::make_shared<AnnounceTier>(std::move(uris));
    tiers_.push_back(std::move(tier));
  }
  resetIterator();
}

//resetIterator 函数，赋值currentTier_ 为  tiers_ 的第一个元素,currentTrackerInitialized_ 赋值为 集合urls中的第一个url
void AnnounceList::resetIterator()
{ 
  currentTier_ = std::begin(tiers_);
  if (currentTier_ != std::end(tiers_) && (*currentTier_)->urls.size()) {
    currentTracker_ = std::begin((*currentTier_)->urls);
    currentTrackerInitialized_ = true;
  }
  else {
    currentTrackerInitialized_ = false;
  }
}

//接下来分析 TrackerWatcherCommand execute 命令的执行
bool TrackerWatcherCommand::execute()
{
  ....
  if (!trackerRequest_) {
    trackerRequest_ = createAnnounce(e_);
    if (trackerRequest_) {
      trackerRequest_->issue(e_);
      LOGD("tracker request created");
    }
  }
  else if (trackerRequest_->stopped()) {
    // We really want to make sure that tracker request has finished
    // by checking getNumCommand() == 0. Because we reset
    // trackerRequestGroup_, if it is still used in other Command, we
    // will get Segmentation fault.
    if (trackerRequest_->success()) {
      if (trackerRequest_->processResponse(btAnnounce_)) {
        btAnnounce_->announceSuccess();
        btAnnounce_->resetAnnounce();
        addConnection();
      }
      else {
        btAnnounce_->announceFailure();
        if (btAnnounce_->isAllAnnounceFailed()) {
          btAnnounce_->resetAnnounce();
        }
      }
      trackerRequest_.reset();
    }
    else {
      // handle errors here
      btAnnounce_->announceFailure(); // inside it, trackers = 0.
      trackerRequest_.reset();
      if (btAnnounce_->isAllAnnounceFailed()) {
        btAnnounce_->resetAnnounce();
      }
    }
  }
  ...
}

首先trackerRequest_ 为空，则执行createAnnounce(e_)
std::unique_ptr<AnnRequest>
TrackerWatcherCommand::createAnnounce(DownloadEngine* e)
{
  //如果当前允许构建Tracker
  while (!btAnnounce_->isAllAnnounceFailed() && btAnnounce_->isAnnounceReady()) {
    std::string uri = btAnnounce_->getAnnounceUrl();
    LOGD("TrackerWatcherCommand uri %s ",uri.c_str());
    uri_split_result res;
    memset(&res, 0, sizeof(res));
    if (uri_split(&res, uri.c_str()) == 0) {
      std::unique_ptr<AnnRequest> treq;
      if (udpTrackerClient_ &&
          uri::getFieldString(res, USR_SCHEME, uri.c_str()) == "udp") {
        uint16_t localPort;
        localPort = e->getBtRegistry()->getTcpPort();
        treq = createUDPAnnRequest(uri::getFieldString(res, USR_HOST, uri.c_str()), res.port, localPort);
      }
      else {
        treq = createHTTPAnnRequest(uri);
      }
      btAnnounce_->announceStart(); // inside it, trackers++.
      return treq;
    }
    else {
      btAnnounce_->announceFailure();
    }
  }
  if (btAnnounce_->isAllAnnounceFailed()) {
    btAnnounce_->resetAnnounce();
  }
  return nullptr;
}

//其中 btAnnounce_->isAnnounceReady() 会调用下面的函数,其中最重要的是判断当前的tracker触发时间是否到达,而默认执行的时候是满足的
bool DefaultBtAnnounce::isDefaultAnnounceReady()
{
  return (trackers_ == 0 && prevAnnounceTimer_.difference(global::wallclock()) >= (userDefinedInterval_.count() == 0 ? minInterval_ : userDefinedInterval_) 
  && !announceList_.allTiersFailed());
}

//btAnnounce_->getAnnounceUrl() 内部会调用 下面的函数，也即是获取到 currentTracker_所指向的url集合的首元素
std::string AnnounceList::getAnnounce() const
{
  if (currentTrackerInitialized_) {
    return *currentTracker_;
  }
  else {
    return A2STR::NIL;
  }
}

对于Tracker的请求成功，会调用最外部的 btAnnounce_->announceSuccess();,最终会调用下面的函数
void AnnounceList::announceSuccess()
{
  if (currentTrackerInitialized_) {
	 //事件为START，转为COMPLETE
    (*currentTier_)->nextEvent();
	//将当前执行成功的Tracker url ，提取出来，再次添加到头部
    auto url = *currentTracker_;
    (*currentTier_)->urls.erase(currentTracker_);
    (*currentTier_)->urls.push_front(std::move(url));
	
	//currentTier_  ，currentTracker_ 又重新的指向头部，导致接下来的tracker都没有机会执行
    currentTier_ = std::begin(tiers_);
    currentTracker_ = std::begin((*currentTier_)->urls);
  }
}

对于Tracker 的请求失败，会调用最外部的 btAnnounce_->announceFailure(); 最终会调用到下面的函数
void AnnounceList::announceFailure()
{
  if (currentTrackerInitialized_) {
    //指向下一个url
    ++currentTracker_;
	//如果指向到了结尾，指向下一个Tracker
    if (currentTracker_ == std::end((*currentTier_)->urls)) {
      // force next event
      (*currentTier_)->nextEventIfAfterStarted();
      ++currentTier_;
      if (currentTier_ == std::end(tiers_)) {
        currentTrackerInitialized_ = false;
      }
      else {
        currentTracker_ = std::begin((*currentTier_)->urls);
      }
    }
  }
}

同时对于失败的情况，会调用 下面的函数，判断当前的tracker是否都失败了，如果都失败了，要重置，调用 btAnnounce_->resetAnnounce();
if (btAnnounce_->isAllAnnounceFailed()) {
   btAnnounce_->resetAnnounce();
}
最终会调用到下面的函数
void AnnounceList::resetIterator()
{
  currentTier_ = std::begin(tiers_);
  if (currentTier_ != std::end(tiers_) && (*currentTier_)->urls.size()) {
    currentTracker_ = std::begin((*currentTier_)->urls);
    currentTrackerInitialized_ = true;
  }
  else {
    currentTrackerInitialized_ = false;
  }
}
```
### Aria2 Trakcer总结
> 根据上面的分析，大概可以知道Aria2 ，Tracker的逻辑，首先是同一时刻只能有一个Tracker请求正在执行，而且当一个Tracker请求成功之后，会重置currentTier_，currentTracker_ 重新的指向头部，导致下次又触发Tracker执行的时候，还是请求他，导致后面的Tracker根本没有机会执行，对于当前请求失败的Trakcer，才会有机会执行下一个Tracker

### Aria2 Trakcer 改造
```C++
//首先是  AnnounceList 初始化,参数为当前种子下载 announce-list, list, 可选, 可选的tracker服务器地址
AnnounceList::AnnounceList(const std::vector<std::vector<std::string>>& announceList)
{
  //配置种子的tracker 列表
  reconfigure(announceList);
}

//配置当前种子的tracker 集合,对应的给每一个tracker url 封装一个AnnounceTier 对象，然后添加到tiers_ 集合中
void AnnounceList::reconfigure(const std::vector<std::vector<std::string>>& announceList)
{
  for (const auto& vec : announceList) {
    if (vec.empty()) {
        continue;
    }
    std::deque<std::string> uris(std::begin(vec), std::end(vec));
    //这里是构建一个共享 AnnounceTier 对象，如果当前tracker 下面还有一个集合，只取第一个uri (这种情况很少见)
    auto tier = std::make_shared<AnnounceTier>(uris.front());
    //将  AnnounceTier 对象添加到集合tiers_ 中
    tiers_.push_back(std::move(tier));
  }
}

//每一个Trakcer的封装，类似于Transmission中的 tier结构体
class AnnounceTier {
    private:
        /**
         * 获取重试的间隔
         */
        int getRetryInterval();

        constexpr static auto DEFAULT_ANNOUNCE_INTERVAL = 2_min;

        //用来判断是否下次应该触发连接
        Timer announceAt;
        //当前trakcer 失败的次数
        int consecutiveFailures;
        //这是当前的时间跟上一次时间的间隔
        std::chrono::seconds nextExecuteTime_;
        //代表当个前tracker 里面包含的url 地址集合，允许有多个
        std::string url_;
        //下次再连接Tracker服务器的时间间隔，每隔多久就要跟Tracker服务器再连接一次
        std::chrono::seconds interval_;
        //最小跟Tracker服务器连接的最小时间间隔
        std::chrono::seconds minInterval_;
        //用户定义的时间
        std::chrono::seconds userDefinedInterval_;
        //peer 下载成功的数量
        int complete_;
        //peer 下载未完成的数量
        int incomplete_;
        //随机数对象引用
        Randomizer *randomizer_;

        //标识当前是否正在执行trakcer请求
        bool isAnnouncing;

    public:

        //标识当前tracker事件的状态，比如开始，暂停，下载完成等
        UDPTrackerEvent event;

        //用来存储当前tracker 还有多少个事件没有处理
        std::deque<UDPTrackerEvent> annouceEventQueue_;

        AnnounceTier(std::string& url);
        ~AnnounceTier();

        //代表的tracker 请求成功
        void trackerSuccess();

        //代表的tracker 请求失败
        void trackerFailer(UDPTrackerEvent trackerEvent);

        /**
         * 返回当前代表的tracker Url
        */
        std::string getUrl() {
            return url_;
        }

        /**
         * 判断当前是否有必要执行tracke请求
        */
        bool isNeedExecute();

        /**
        * 传递url 对应的 tracker 请求的结果
         */
        void processRequestResult(std::chrono::seconds interval,std::chrono::seconds minInterval,int complete,int incomplete);

        /**
        * 设置用户定义的时间间隔
        */
        void setUserDefinedInterval(std::chrono::seconds interval);

        /**
        * 返回当前tracker 对应的事件 比如 Start, Stopped completed etc.
        */
        const char* getEventString() const;

        /**
         * 设置当前正在执行tracker 请求
         */
        void setAnnounceExute();

        /**
         * 添加Announce事件
         */
        void addAnnounceEvent(UDPTrackerEvent announceEvent,std::chrono::seconds nextExuteTime);

        /**
         * 从事件队列中取出头部的事件
         */
        UDPTrackerEvent announceEventPull();

        /**
         * 添加一个完成的Announce事件
         */
        void addCompleteEvent();
};

//构造函数的执行 初始化默认的状态  比如 event 为 STARTED 赋值 当前tracker 包含的url
AnnounceTier::AnnounceTier(std::string& url): event(UDPT_EVT_NONE),
                                                 url_(url),
                                                 announceAt(global::wallclock()), //更新当前的检查时间点
                                                 nextExecuteTime_(std::chrono::seconds(0)), //下一次触发检查的时间点
                                                 consecutiveFailures(0),
                                                 randomizer_(SimpleRandomizer::getInstance().get()),//随机数对象
                                                 interval_(DEFAULT_ANNOUNCE_INTERVAL),
                                                 minInterval_(DEFAULT_ANNOUNCE_INTERVAL),
                                                 userDefinedInterval_(0_s),
                                                 complete_(0),
                                                 incomplete_(0),
                                                 isAnnouncing(false)
{
  //首先要先添加一个START事件,事件触发的时间为当前时间
  addAnnounceEvent(UDPT_EVT_STARTED,std::chrono::seconds(0));
  LOGD("AnnounceTier url %s",url.c_str());
}

/**
* 判断当前是否有必要执行tracke请求
*/
bool AnnounceTier::isNeedExecute()
{
   //当前不处于trakcer请求中，并检查当前的时间是否大于需要执行tracker的时间，并且事件队列中含有事件没有处理
   return !isAnnouncing && (announceAt.difference(global::wallclock()) >= nextExecuteTime_) && annouceEventQueue_.size() > 0;
}

//成功，失败次数重置为0
void AnnounceTier::trackerSuccess()
{
   isAnnouncing = false;
   consecutiveFailures = 0;

   //如果当前没有事件了,添加一个下次触发tracker的连接事件,成功的时候，连接时间为tracker返回的时间
   if(!annouceEventQueue_.size())
   {
      addAnnounceEvent(UDPT_EVT_NONE,nextExecuteTime_);
      LOGD("trackerSuccess addAnnounceEvent nextExecuteTime_ %lld",nextExecuteTime_.count());
   }
}

//失败，失败次数增加一
void AnnounceTier::trackerFailer(UDPTrackerEvent trackerEvent)
{
   isAnnouncing = false;
   //失败次数加一
   consecutiveFailures++;
   //失败的时候，设置下一次tracker 触发连接的时间
   addAnnounceEvent(trackerEvent,std::chrono::seconds(getRetryInterval()));
   LOGD("trackerFailer addAnnounceEvent consecutiveFailures %d nextExecuteTime_ %lld",consecutiveFailures,nextExecuteTime_.count());
}

//添加事件到事件队列中
void AnnounceTier::addAnnounceEvent(UDPTrackerEvent announceEvent,std::chrono::seconds nextExuteTime)
{
    if(annouceEventQueue_.size() > 0)
    {
        //如果当前添加的事件是停止的事件
        if(announceEvent == UDPT_EVT_STOPPED)
        {
             //从事件队列中查找是否有完成的事件
             const UDPTrackerEvent  c = UDPT_EVT_COMPLETED;
             auto result = std::find(std::begin(annouceEventQueue_), std::end(annouceEventQueue_), c);
             if (result != std::end(annouceEventQueue_)) {
                 //先清除事件的队列，再添加一个已经完成的事件到队列中
                 annouceEventQueue_.clear();
                 annouceEventQueue_.push_back(UDPT_EVT_COMPLETED);
             }
        }

        //将事件队列中移除NONE 对应的事件
        annouceEventQueue_.erase(std::remove_if(std::begin(annouceEventQueue_), std::end(annouceEventQueue_),
                                                    [&](const UDPTrackerEvent &ent) {
                                                        return UDPT_EVT_NONE == ent;
                                                    }),
                                     std::end(annouceEventQueue_));


        //防止事件队列重复,添加之前 要先移除跟当前要添加一样的事件
        annouceEventQueue_.erase(std::remove_if(std::begin(annouceEventQueue_), std::end(annouceEventQueue_),
                                                    [&](const UDPTrackerEvent &ent) {
                                                        return announceEvent == ent;
                                                    }),
                                     std::end(annouceEventQueue_));
   }
    //添加事件
    annouceEventQueue_.push_back(announceEvent);
    //重置定时器为当前的时间
    announceAt = global::wallclock();
    //下次触发tracker的时间更新
    nextExecuteTime_ = nextExuteTime;
}

而对于 TrackerWatcherCommand ，逻辑改为
//Command命令的执行
bool TrackerWatcherCommand::execute()
{
  ....
  //当前还允许tracker 并发执行的数量没有达到最大的限度
  if(btAnnounce_->getMaxConcurrentSize() > 0)
  {
      //获取到当前可以执行的 tracker 请求的 AnnounceTier 集合
      std::deque<std::shared_ptr<AnnounceTier>> needExuteAnnounceList = btAnnounce_->getCanExeuteUrlList();
      //当前最多可以执行的tracker 请求的并发数不应该大于 当前种子最多的tracker 并发数
      int minSize = btAnnounce_->getMaxConcurrentSize() > needExuteAnnounceList.size() ? needExuteAnnounceList.size() : btAnnounce_->getMaxConcurrentSize();
      //遍历将每一个AnnounceTier，变成对应的AnnRequest
      for(int i = 0; i< minSize;i++)
      {
          std::shared_ptr<AnnounceTier> tierItem = needExuteAnnounceList.front();
          auto request = checkAndExuteTracke(e_,tierItem);
          //开始执行请求
          request->issue(e_);
          //设置并发数量减一处理
          btAnnounce_->setConcurrentSize((btAnnounce_->getMaxConcurrentSize() - 1));
          //设置为请求开始了
          tierItem->setAnnounceExute();
          //将当前的请求添加到发送队列中
          trakerRequestList_.push_back(std::move(request));
          //弹出
          needExuteAnnounceList.pop_front();
          LOGD("tracker request created trakerRequestList_ size %d",trakerRequestList_.size());
      }
  }

  //请求中，判断是否获取到了结果
  for (auto &requestItem : trakerRequestList_) {
     if (requestItem->stopped())//如果当前请求已经停止
     {
        LOGD("trakcr request stop url %s",requestItem->getAnnounceTier()->getUrl().c_str());
        if (requestItem->success()) {//当前请求成功
             if (requestItem->processResponse(btAnnounce_))
             {
                 requestItem->getAnnounceTier()->trackerSuccess();
                 //根据请求traker 返回的peer 执行连接
                 addConnection();
             } else {
                 //解析失败
                 requestItem->getAnnounceTier()->trackerFailer(requestItem->getCurrentTrackerEvent());
             }
        } else {
             //由BtAnnounce 告知哪个tracker 请求失败
            requestItem->getAnnounceTier()->trackerFailer(requestItem->getCurrentTrackerEvent());
        }
     }
  }

  //整体移除掉那些已经停止的请求
  trakerRequestList_.erase(std::remove_if(std::begin(trakerRequestList_), std::end(trakerRequestList_),
                                            [&](const std::unique_ptr<AnnRequest> &ent) {
                                                if(ent->stopped())
                                                {
                                                    //移除的时候，让并发数量加一
                                                    btAnnounce_->setConcurrentSize((btAnnounce_->getMaxConcurrentSize() + 1));
                                                }
                                                return ent->stopped();
                                            }),
                             std::end(trakerRequestList_));
	
   ...
}

当然还有很多细节的问题，这里就不详细说明了
```