---
layout: pager
title: Android中实现Aira2
date: 2018-05-18 11:01:40
tags: [AndroidStudio,Aria2,P2P]
description: Android 中利用Aria2 完成一个P2P的下载神器
---

Android 中利用Aria2 完成一个P2P的下载神器
<!--more-->

简介
```
上一篇文章中，介绍了怎么将Aria2库移植到了AndroidStudio中，而且使用了最新的编译方式CmakeList的方式接入，在接下来的一个月实现Android p2p下载的时候，发现使用CmakeList来编译真的
很重要，因为他可以使你断点调试代码。。如果不能调试的化。。那么这个工作的进程就会大打折扣，甚至有可能误解，现在请允许我来吹一下目前的实现成果

1.首先我们是用使用Aria2库实现的，所以对于他的优点我们肯定是有的，他的优点有aria2是一款用于下载文件的工具。 支持的协议是HTTP（S），FTP，SFTP，BitTorrent和Metalink。
aria2可以从多种来源/协议下载文件，并尝试利用您的最大下载带宽。它支持从HTTP（S）/ FTP / SFTP和BitTorrent同时下载文件，而从HTTP（S）/ FTP / SFTP下载的数据则上传到BitTorrent群。 
使用Metalink块校验和，aria2在下载文件时自动验证数据块。网络上关于他的各种优点也是有很多。。不太懂的可以自行百度

2.支持并发下载，支持断点下载
2.当下载磁力链接或者本地的种子文件，Metalink文件的时候，支持选择文件下载,做到想下什么就下什么文件
3.下载的过程中支持暂停，恢复，删除操作

```

支持并发下下载
![结果显示](/uploads/Arir2Android实现/Aria并发下载.jpg)

支持选择文件下载
![结果显示](/uploads/Arir2Android实现/Aria2支持选择文件.jpg)

支持断点下载，甚至可以做到当你出现异常的情况导致退出，下次进来也可以断点下载
![结果显示](/uploads/Arir2Android实现/Aria2支持断点下载.jpg)

JNI接口
```java

/**
 * Project Name: Aria2AndroidProject
 * File Name:    AriaApi.java
 * ClassName:    AriaApi
 *
 * Description: Aria2Api JNI接口 需知由于JNI层是开启一个线程维护一个下载队列在跑，所以对于添加的任务既有可能不会立刻的被执行，
 * 只是会将当前的下载任务，添加到一个队列里面，所以native函数的返回值都为void，对于添加下载我们需要获的一个唯一的标识，JNI层将会通过回调我们Java方法，将值传递过来
 * 为了确保能准确的找到，还要传递一个java层的唯一标识,这里简单的通过一个int类型来标识，这个只是用来匹配用
 *
 * @author Zhangyuhui
 * @date 2018年04月18日 10:21
 *
 * Copyright (c) 2018年, 4399 Network CO.ltd. All Rights Reserved.
 */
public class AriaApi
{
    private static IDownLoadModel mIDownLoadModelListener;
    private static IAddDownLoadModel mIAddDownloadModelListener;

    public static void setDownLoadModelListener(IDownLoadModel downLoadModelListener){
        mIDownLoadModelListener = downLoadModelListener;
    }

    public static void setAddDownLoadModelListener(IAddDownLoadModel addDownLoadModelListener){
        mIAddDownloadModelListener = addDownLoadModelListener;
    }

    public static void removeAddDownloadModelListener(){
        mIAddDownloadModelListener = null;
    }

    /**
     * Aria2 初始化函数，初始化library 以及创建子线程
     * @param aria2WorkSpacePath aria2工作目录
     * @param aria2SessionPath  保存aria2 下载等信息
     * @param aria2LogPath  保存下载的日志信息
     * @param aria2DhtPath Dht文件路径，用来保存种子的信息
     * @return 返回值如果为0 代表初始化正常，如果为1代表失败
     */
    public static native int Aria2Init(String aria2WorkSpacePath,String aria2SessionPath,String aria2LogPath,String aria2DhtPath);

    /**
     * 添加新的HTTP（S）/ FTP / BitTorrent Magnet URI (磁力链接)  对于BitTorrent Magnet URI，uris必须只有一个元素，它应该是BitTorrent Magnet URI
     * @param url 要下载的地址
     * @param id 标识java层当前下载任务的唯一标识
     * @return 返回值为每一个下载任务的唯一标识,返回的是一个字符串的形式表示 失败返回null
     */
    public static native void Aria2AddUrlDownLoad(String url,int id);

    /**
     * 添加Metalink下载。 Metalink文件的路径由metalinkFile指定。 成功返回时，如果gid不为NULL，则添加的下载的GID将附加到* gids
     *
     * Metalink的简介....
     * Metalink标准体现在一个扩展名是.metalink的XML文件，这个文件里记录着下载的URL信息。这个文件里记录着你想下载的文件的镜像服务器的地址。除了支持HTTP和FTP的镜像地址外，
     * Metalink还支持着包括BitTorrent,ed2k和magnet links在内的P2P下载源的信息。在OpenOffice发布的metalink文件中就包含了50多条HTTP和FTP镜像服务器地址和一个torrent文件地址。
     *
     * @param metalinkPath 指定Metalink文件的路径
     * @param id 标识java层当前下载任务的唯一标识
     * @return 因为Metalink里面一般会存在多个的下载链接，所以成功返回时 会创建返回一组 下载的唯一标识gid 失败返回null
     */
    public static native void Aria2AddMetalink(String metalinkPath,int id);


    /**
     * 添加BitTorrent下载。 成功返回时，如果gid不为NULL，则添加的下载的GID将分配给* gid。 “.torrent”文件的路径由torrentFile指定。
     * 注意，这个选项不支持 下载磁力链接 BitTorrent Magnet URI不能与此功能一起使用。 使用addUri（）代替。
     * @param torrentFilePath 种子文件的路径 文件的扩展名为 .torrent
     * @param id 标识java层当前下载任务的唯一标识
     * @return 返回值如果不为空，说明添加下载成功，返回下载的唯一标识gid ，如果失败返回null
     */
    public static native void Aria2AddTorrent(String torrentFilePath,int id);


    /**
     * 删除由gid表示的下载。 如果指定的下载正在进行，则首先停止。 已删除的下载状态变为DOWNLOAD_REMOVED。
     * 如果force是真实的，将会在没有任何需要时间的行动的情况下进行移除 如果成功，该函数返回0，否则返回负的错误代码。
     * @param gid  要删除任务的唯一标识
     * @param force 是不是强制的删除
     * @return 如果成功，该函数返回0，否则返回负的错误代码。
     */
    public static native void Aria2RemoveDownload(String gid,boolean force);

    /**
     * 暂停由gid表示的下载。 暂停下载的状态变为DOWNLOAD_PAUSED。 如果下载处于活动状态，则将下载放在等待队列的第一个位置。 只要状态为DOWNLOAD_PAUSED，下载将不会开始。
     * 要将状态更改为DOWNLOAD_WAITING，请使用{@link Aria2ResumeDownload}函数。 如果force为真，暂停将会发生，不需要任何需要时间的操作，例如联系BitTorrent跟踪器。
     * @param gid 要暂停任务的唯一标识
     * @param force 是否是强制的停止
     * @return  如果成功，该函数返回0，否则返回负的错误代码。 请注意，要暂停工作，应用程序必须将SessionConfig :: keepRunning设置为true。 否则，行为是不确定的。(这个已经确保)
     */
    public static native void Aria2PauseDownload(String gid,boolean force);

    /**
     * 将由gid表示的下载状态从DOWNLOAD_PAUSED更改为DOWNLOAD_WAITING。 这使得下载可以重新启动 但是不是立刻的，因为有一个等待队列的存在，如果当前有任务执行的化
     * @param gid 要恢复下载的gid唯一标识
     * @return  如果成功，该函数返回0，否则返回负的错误代码。
     */
    public static native void Aria2ResumeDownload(String gid);

    /**
     * 可以动态修改当前gid指示的下载的参数配置。比如 如果当前正处于下载中(DOWNLOAD_ACTIVE) 状态下可以更改以下选项：
     *
     * bt-max-peers 指定每个torrent的最大对等数量。 0意味着无限。 另见--bt-request-peer-speed-limit选项。 默认：55
     * bt-request-peer-speed-limit 如果每个torrent的整个下载速度都低于SPEED，则aria2会暂时增加对等点的数量以尝试提高下载速度。 在某些情况下，使用首选下载速度配置此选项可以提高下载速度。 您可以附加K或M（1K = 1024，1M = 1024K）。 默认：50K
     * bt-remove-unselected-file 在BitTorrent中完成下载后删除未选定的文件。 要选择文件，请使用--select-file选项。 如果未使用，则假定所有文件都被选中。 请小心使用此选项，因为它实际上会从磁盘中删除文件。 默认值：false
     * force-save    使用--save-session选项保存下载，即使下载完成或删除。 在这种情况下，该选项也可以保存控制文件
     * max-download-limit  下载限速
     * max-upload-limit    上传限速
     *
     * 对于DOWNLOAD_WAITING或DOWNLOAD_PAUSED状态的下载，除上述选项外，还可使用输入文件子部分中列出的选项，
     * 但以下选项除外： dry-run, metalink-base-uri, parameterized-uri, pause, piece-length and rpc-save-upload-metadata。
     *
     * @param gid 要对哪个下载进行操作
     * @param modifyOption 要修改的选项
     * @return  如果成功，该函数返回0，否则返回负的错误代码。
     */
    public static native void Aria2ChangeDownloadOption(String gid,String[] modifyOption);


    /**
     * 更改由gid表示的下载位置。 如果它处于DOWNLOAD_WAITING或DOWNLOAD_PAUSED状态。 如果 mode 为 OFFSET_MODE_SET ，它将下载移动到相对于队列开始的位置pos。
     * 如果mode 是 OFFSET_MODE_CUR，它会将下载移动到相对于当前位置的pos位置。 如果OFFSET_MODE_END如何，它将下载移动到相对于队列末尾的位置pos。
     * 如果目标位置小于0或超出队列末尾，它将分别将下载移动到队列的开始或结束位置。
     *
     例如，如果具有GID gid的下载位于位置3，则changePosition（gid，-1，OFFSET_MODE_CUR）将其位置更改为2. 附加调用changePosition（gid，0，OFFSET_MODE_SET）将其位置更改为0（开始 的队列）。
     * @param gid
     * @param pos
     * @param mode
     * @return 返回值是成功的postion 此函数返回此下载的最终目标位置或负面的错误代码。
     */
    public static native void Aria2ChangeDownloadPosition(String gid,int pos,int mode);

    /**
     * Aria2 销毁函数,执行一些释放
     * @return  如果成功，该函数返回0，否则返回负的错误代码。
     */
    public static native int Aria2Destroy();

    /**
     * 显示下载所包含的文件，这个只能针对于本地存在的种子文件，或者是本地存在的Metalink文件,对于磁力链接等url是不允许的
     * @param path
     */
    public static native void Aria2ShowFiles(String path);

    /**
     * Aria2针对于种子文件选择文件下载的支持，
     * @param selectStr 用户选择文件的索引数组
     * @param filePath  本地种子文件的路径，或者是本地Metalink文件的路径
     * @param id        java层对应的唯一标识，用于方便找到是哪个item触发的
     */
    public static native void Aria2SelectFilesForTorrentDownload(String selectStr,String filePath,int id);

    /**
     * Aria2针对于Metalink文件选择文件下载的支持，
     * @param selectStr 用户选择文件的索引数组
     * @param filePath  本地种子文件的路径，或者是本地Metalink文件的路径
     * @param id        java层对应的唯一标识，用于方便找到是哪个item触发的
     */
    public static native void Aria2SelectFilesForMetalinkDownload(String selectStr,String filePath,int id);


    /**
     * Aria2 针对于磁力链接的下载，这个方法会先将磁力链接的元数据下载到本地保存为一个种子文件
     * @param url 传递的为磁力链接的url
     */
    public static native void Aria2DownloadMagnetUrl(String url);

    /**
     * JNI层回调执行java的方法,显示一些信息
     * @param message
     */
    public static void showToast(byte[] message){
        if(mIDownLoadModelListener != null){
            mIDownLoadModelListener.onShowMessage(new String(message));
        }
    }

    /**
     * JNI层回调，下载失败，告知当前下载失败的原因
     * @param needSize
     * @param errorCode
     * @param gid
     */
    public static void downloadErrorCode(long needSize,int errorCode,byte[] gid){
        if(mIDownLoadModelListener != null){
            mIDownLoadModelListener.downloadErrorCode(needSize,errorCode,new String(gid));
        }
    }


    /**
     * JNI层回调，添加下载任务成功，这个是添加除了MetaLinkFile文件成功之外的回调
     * @param gid 当前下载的唯一标识，这个gid为Aria2库层生成
     * @param javaId java传递的标识
     */
    public static void addNormalDownLoadSuccess(byte[] gid,int javaId){
        if(mIDownLoadModelListener != null){
            mIDownLoadModelListener.onAddDownLoadSuccess(new String(gid),javaId);
        }
    }

    /**
     * JNI层回调，JNI层会将当前正在下载的对象，封装成一个集合，返回给android层，Androi层完成界面的展示
     * @param entities
     */
    public static void downloadInfoChange(ArrayList<DownloadEntity> entities){
        if(mIDownLoadModelListener != null){
            mIDownLoadModelListener.downloadInfoChange(entities);
        }
    }


    /**
     * JNI层回调，JNI层将会频缓的调用这个方法，返回一个DownloadGlobalEntity对象
     * @param globalEntity 当前下载的整体信息
     */
    public static void downLoadTotalInfo(DownloadGlobalEntity globalEntity){
        if(mIDownLoadModelListener != null){
            mIDownLoadModelListener.downLoadTotalInfo(globalEntity);
        }
    }

    /**
     *JNI层回调,添加MetalinkFile成功之后的回调
     * @param gids 一组gid的集合
     * @param javaId
     */
    public static void addMetalinkFfileDownloadSuccess(List<byte[]> gids, int javaId){

    }

    /**
     * JNI层回调，通知gid对应的下载已经暂停成功
     * @param gid
     */
    public static void downloadPauseSuccess(byte[] gid){
        if(mIDownLoadModelListener != null){
            mIDownLoadModelListener.downloadPauseSuccess(new String(gid));
        }
    }

    /**
     * JNI层回调，通知gid对应的下载已经恢复成功
     * @param gid
     */
    public static void downloadResumeSuccess(byte[] gid){
        if(mIDownLoadModelListener != null){
            mIDownLoadModelListener.downloadResumeSuccess(new String(gid));
        }
    }

    /**
     * JNI层回调，通知gid对应的下载已经删除成功
     * @param gid
     */
    public static void downloadRemoveSuccess(byte[] gid)
    {
        if (mIDownLoadModelListener != null)
        {
            mIDownLoadModelListener.downloadRemoveSuccess(new String(gid));
        }
    }

    /**
     * JNI层回调，通知gid对应的下载已经下载成功
     * @param gid 下载对应的唯一的gid
     */
    public static void downloadSuccess(byte[] gid)
    {
        if (mIDownLoadModelListener != null)
        {
            mIDownLoadModelListener.downloadSuccess(new String(gid));
        }
    }

    /**
     * JNI层回调，通过gid对应的下载开始制作种子
     * @param gid 下载对应的唯一的gid
     */
    public static void downloadSuccessStartSeed(byte[] gid)
    {
        if (mIDownLoadModelListener != null)
        {
            mIDownLoadModelListener.downloadSuccessStartSeed(new String(gid));
        }
    }

    /**
     * JNI层回调，通知获取文件成功，返回一个文件的集合
     * @param entities
     */
    public static void showFileSuccess(ArrayList<ShowFileEntity> entities,byte[] filePath)
    {
        if(mIAddDownloadModelListener != null)
        {
            mIAddDownloadModelListener.showFilesSuccess(entities,new String(filePath));
        }
    }

    /**
     * JNI层回调，通知将磁力链接的元数据下载保存成功，
     * @param savePath 保存成功的路径
     */
    public static void downloadManagetUrlSuccess(byte[] savePath)
    {
        if(mIAddDownloadModelListener != null)
        {
            mIAddDownloadModelListener.downloadManagetUrlSuccess(new String(savePath));
        }
    }
}

```

JNI函数的编写
```java
由于下载涉及到了一些耗时的操作，所以我们会在JNI中创建一个子线程来执行这些内容
    //2创建子线程,用于下载，检测等,创建成功之后，就会执行run的回调
    if (pthread_create(&pthread_tid, NULL, run, NULL)) {
        //创建失败
        pthread_detach(pthread_tid);
        rev = 1;
        return rev;
    }
    return rev;
	
//全局的配置选项
static aria2::KeyVals gloableOptions;
	
/**
 * 线程执行的方法回调
 * @param pVoid
 * @return
 */
void *run(void *pVoid) {
    //创建默认的config对象 构造函数填充所有成员的默认值。
    aria2::SessionConfig config;
    //添加下载的监听回调
    config.downloadEventCallback = downloadEventCallback;
    //如果keepRunning成员为true，则即使没有要执行的下载，运行（会话，RUN_ONCE）也会返回1 ，也即是下面的这个调用 int rv = aria2::run(session, aria2::RUN_ONCE);也会返回1
    config.keepRunning = true;
	
	//指定Aria2日志输出的文件,这个可选择，这里选择是为了方便开发得到信息
    gloableOptions.push_back(std::pair<std::string, std::string> ("log", c_Aria2LogPath));
    // 指定日志的级别 "--log-level=debug";
    gloableOptions.push_back(std::pair<std::string, std::string> ("log-level", "debug"));
	//gloableOptions中可以添加很多的参数选项，有关参数的详细，可以查看官网
	....
    //使用这些选项创建新的Session对象作为附加参数。 这些选项被视为在命令行中指定给aria2c传递的命令行。 如果成功，该函数返回指向创建的Session对象的指针，或返回NULL。
    //请注意，每个进程只能创建一个Session对象。
    globalSession = aria2::sessionNew(gloableOptions, config);
    if(globalSession == NULL){
        LOGD("Create session error direct exit");
        pthread_exit(NULL);
    }
    auto start = std::chrono::steady_clock::now();
    for(;;){
        //执行事件轮询和操作, 如果|模式|是 ：c：macro：`RUN_DEFAULT`，这个函数在没有下载时返回 留下来处理。 在这种情况下，这个函数返回0。
        //如果|模式| 是：c：macro：`RUN_ONCE`，这个函数返回之后  在当前的实现中，事件轮询1秒内超时。如果函数返回0则代表出现了异常.否则，返回1，表示该 调用者必须调用此函数一次或多次完成下载。
		//只有执行了aria2::shoutdow();函数这个返回值才会为1，所以会退出无限循环
        int rv = aria2::run(globalSession, aria2::RUN_ONCE);
        if (rv != 1) {
            LOGD("aria2 thread_run exit");
            break;
        }
        auto now = std::chrono::steady_clock::now();
        auto count = std::chrono::duration_cast<std::chrono::milliseconds>(now - start).count();
        //如果任务队列不为空，执行任务，所以这个任务可以是下载，暂停，恢复等,并发的
        while (!jobQueue.empty()) {
            std::unique_ptr<Job> job = jobQueue.pop();
            int ret = job->execute(globalSession);
            //得到错误的信息
            if(ret != 0){
                std::string errorInfo = job->getErrorInfo();
                ShowToastCallbackForAndroid(errorInfo);
            }else{
                //执行任务成功之后的
                job->executeSuccess();
            }
        }
        //每隔500毫秒更新一次界面
        if (count >= 500) {
            start = now;
            //获取到全局的下载统计
            aria2::GlobalStat gstat = aria2::getGlobalStat(globalSession);
            //回调通知总体的下载状况
            downLoadTotalInfoCallBack(gstat);
            //获取到全局的正在下载的数量
            std::vector<aria2::A2Gid> gids = aria2::getActiveDownload(globalSession);
            //用来存储所有的下载的状态集合
            std::vector<DownloadStatus> v;
            for (auto gid : gids) {
                //获取到每一个正在下载任务的详细对象，获取到里面的信息
                aria2::DownloadHandle* dh = aria2::getDownloadHandle(globalSession, gid);
                //任务不为空，获取到里面的信息
                if (dh) {
                    //封装一个下载的任务结构体对象
                    DownloadStatus st;
                    st.gid = gid;
                    //以字节为单位返回此下载的总长度。
                    st.totalLength = dh->getTotalLength();
                    //以字节为单位返回此下载的完整长度。
                    st.completedLength = dh->getCompletedLength();
                    //返回此下载的下载速度，单位为字节/秒。
                    st.downloadSpeed = dh->getDownloadSpeed();
                    //返回此下载的上传速度，单位为字节/秒。
                    st.uploadSpeed = dh->getUploadSpeed();
                    //赋值followGid
                    aria2::A2Gid followGid = dh->getFollowing();
                    if(!aria2::isNull(followGid)){
                        //LOGD("followGid %s",aria2::gidToHex(followGid).c_str());// 磁力链接下载会先下载元数据，再继续下载，所以存在一个followGid
                        st.followGid = followGid;
                    }
                    //获取到当前的连接数
                    st.numbersConnection = dh->getConnections();
                    //获取到当前可用的种子数
                    st.usefulSeedNumbers = aria2::getFindUsefulSeedNumbers(globalSession,gid);
                    //保存下载的目录保存的地方
                    st.dir = dh->getDir();
                    int numbers = dh->getNumFiles();
                    //获取用户选择的文件
                    int selectCount = 0;
                        for (int i = 1; i <= numbers; i++) {
                            aria2::FileData file = dh->getFile(i);
                            if (file.selected) {
                                selectCount++;
                            }
                    }
                    //当前下载所包含的文件数量
                    st.fileNumbsers = selectCount;
                    //返回文件数，及是当前下载的地址里面包含有多少个文件要下载
                    if (dh->getNumFiles() > 0) {
                        aria2::FileData file = dh->getFile(1);
                        st.filename = file.path;                        //文件名
                    }
                    //添加到下载的状态的集合里面
                    v.push_back(std::move(st));
                    //这边使用完之后，就要移除掉
                    aria2::deleteDownloadHandle(dh);
                }
            }
            //回调给Android，更新界面
            downloadInfoChangeCallBackForAndroid(v);
        }
    }
    //调用sessionFinal（）很重要，因为它执行下载后操作，包括保存会话和销毁会话对象。 所以没有调用这个函数会导致失去下载进度和内存泄漏。 sessionFinal（）返回在EXIT STATUS中定义的代码
    int rv = aria2::sessionFinal(globalSession);
    LOGD("aria2 sessionFinal after");
    return reinterpret_cast<void *>(rv);
}

可以看到我们在run方法中写了一个无限循环的逻辑，让他一直的执行任务
//如果任务队列不为空，执行任务，所以这个任务可以是下载，暂停，恢复等,并发的
        while (!jobQueue.empty()) {
            std::unique_ptr<Job> job = jobQueue.pop();
            int ret = job->execute(globalSession);
            //得到错误的信息
            if(ret != 0){
                std::string errorInfo = job->getErrorInfo();
                ShowToastCallbackForAndroid(errorInfo);
            }else{
                //执行任务成功之后的
                job->executeSuccess();
            }
        }

其中的JobQueue是一个全局的变量，这样当要执行什么任务的时候，都要添加到这个队列里面，然后来执行		
typedef SynchronizedQueue<Job> JobQueue;
//维护一个工作队列,
static JobQueue jobQueue;	

//Job抽象接口
struct Job {
    virtual ~Job() = default;
    virtual std::string getErrorInfo() = 0;
    virtual int execute(aria2::Session *session) = 0;
    //执行任务成功,不强制需要实现
    virtual void executeSuccess(){

    }
};

所以具体的子类可以实现要实现的方法，比如
//暂停的任务 int pauseDownload(Session* session, A2Gid gid, bool force)
struct PauseDownloadJob : public Job{
    //构造函数，完成赋初值
    PauseDownloadJob(aria2::A2Gid gid,bool force) : gid(gid),force(force) {}
    virtual std::string getErrorInfo()
    {
        return "gid" + aria2::gidToHex(gid) + "暂停任务失败";
    }
    //执行暂停的实现
    virtual int execute(aria2::Session* session)
    {
        int ret = aria2::pauseDownload(session,gid,force);
        LOGD("pauseDownloadJob execute ret %d",ret);
        return ret;
    }

    bool force;
    //要暂停任务的唯一标识gid
    aria2::A2Gid gid;
};

	

比如 添加下载的url,支持添加http 连接，或者磁力链接 ,
JNIEXPORT void JNICALL Java_com_example_com_aria2libandroidproject_AriaApi_Aria2AddUrlDownLoad
        (JNIEnv * env, jclass clz, jstring javaDownloadUrl,jint jDownLoadId)
{
    const char * downloadUri = env->GetStringUTFChars(javaDownloadUrl,NULL);
    //定义一个集合用来存储我们传递的url,这里目前只支持传递一个
    std::vector<std::string> uris = {downloadUri};
    //构建完成之后，将当前的任务 封装成一个AddUriJob 对象，里面有对应的下载uri 跟 下载的参数选项,然后添加到jobq_ 任务队列里面,这里下载的选项传递的是null，用全局的
    //std::move函数可以以非常简单的方式将左值引用转换为右值引用,move 将左值变成右值 std::move是将对象的状态或者所有权从一个对象转移到另一个对象，只是转移，没有内存的搬迁或者内存拷贝
    //左值引用的基本语法：type &引用名 = 左值表达式；  右值引用的基本语法type &&引用名 = 右值表达式；
    jobQueue.push(std::unique_ptr<Job>(new AddUriJob(std::move(uris), std::move(gloableOptions),jDownLoadId)));
}


/**
 * 销毁函数，在应用程序退出的时候，执行这个方法
 */
JNIEXPORT jint JNICALL Java_com_example_com_aria2libandroidproject_AriaApi_Aria2Destroy
        (JNIEnv * env, jclass clz)
{
    //添加一个关闭的任务，这个任务执行的化，就会终止无限循环，所以要添加这个任务
    jobQueue.push(std::unique_ptr<Job>(new ShutdownJob(true)));
    //代码中如果没有pthread_join主线程会很快结束从而使整个进程结束，从而使创建的线程没有机会开始执行就结束了。
    //加入pthread_join后，主线程会一直等待直到等待的线程结束自己才结束，使创建的线程有机会执行。
    pthread_join(pthread_tid,NULL);
    //如果到了这里说明终止的Job已经被执行完毕了，可以执行销毁了
    //释放全局数据。 如果成功，该函数返回0，否则返回负的错误代码。 在应用程序结束时只调用一次该函数。
    int rev = aria2::libraryDeinit();
    pthread_kill (pthread_tid,SIGUSR1);
    LOGD("aria2 destroy");
    return rev;
}

//关闭的任务
struct ShutdownJob : public Job {
    ShutdownJob(bool force) : force(force) {}
    virtual int execute(aria2::Session* session)
    {
	    //终止无限循环的关键代码,当执行了这段代码之后，int rv = aria2::run(globalSession, aria2::RUN_ONCE); 才会返回1，这样才会进入break，退出无限循环
        int ret = aria2::shutdown(session, force);
        return ret;
    }
    virtual std::string getErrorInfo()
    {
        return "shutDown任务失败";
    }
    bool force;
};


//System.loadLibrary的时候会回调这个函数,初始化一些资源,可以将一些经常要使用到的变量变成一个全局的变量，这样下次使用的时候就可以快速的获取到
int JNI_OnLoad(JavaVM* vm, void* reserved) {

    JNIEnv *env;
    if(vm ->GetEnv((void**) &env, JNI_VERSION_1_4) != JNI_OK)
    {
        LOGD("Failed to get the environment using GetEnv()");
        return -1;
    }
    //持有env的引用
    jVM = vm;
    //demo/kaillera/org/myapplication/KailleraJni com.example.com.aria2libandroidproject
    aria2ApiClass = env->FindClass ("com/example/com/aria2libandroidproject/AriaApi");
    if(aria2ApiClass == NULL)
    {
        LOGD("Failed to find class com.example.com.aria2libandroidproject.AriaApi");
        return -1;
    }
    aria2ApiClass = (jclass) env->NewGlobalRef(aria2ApiClass);
    //找到对应的method   public static void showToast(byte[] message)
    android_showToast = env->GetStaticMethodID(aria2ApiClass,"showToast","([B)V");
    if(android_showToast == NULL)
    {
        LOGD("Failed to find method showToast");
        return -1;
    }
	....
}

/**
 * 回调执行Android中的showToast方法
 * @param info
 */
void ShowToastCallbackForAndroid(std::string info){
    //回调java层，弹出toast 由于我们是在子线程中触发的，所以在子线程中要使用下面的方式来回调执行Android的代码
    JNIEnv *env;
    jVM->GetEnv((void**) &env, JNI_VERSION_1_4);
    int attached  = 0;
    if(env==NULL)
    {
        attached  = 1;
        jVM->AttachCurrentThread(&env, NULL);
    }
    //showToast(LISTCMD_HIDEGAME, 0);
    if(android_showToast != NULL)
    {
	    //直接使用env->NewStringUTF() 在部分的手机有问题，所以这里先转成jbyteArray数组，对应的android就为byte[]数组
        const char * c_info = info.c_str();
        int strLen = strlen(c_info);
        jbyteArray byteArray = env->NewByteArray(strLen);
        env->SetByteArrayRegion(byteArray, 0, strLen, reinterpret_cast<const jbyte*>(c_info));
        env->CallStaticVoidMethod(aria2ApiClass,android_showToast,byteArray);
		//释放局部引用
        env->DeleteLocalRef(byteArray);
    }
    if(attached)
    {
        jVM->DetachCurrentThread();
    }
}

```

整体来讲目前已经可以满足我一个做为种子用户的需求,我也是基于种子用户的角度来实现的
以上就是主要的实现，对于部分的细节，关于Android界面就没有必要详解了，最后贴下项目的工程结构
![结果显示](/uploads/Arir2Android实现/Aria2工厂目录.png)




