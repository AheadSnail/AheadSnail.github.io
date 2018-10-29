---
layout: pager
title: Aria2 JNI代码的优化
date: 2018-10-27 14:21:41
tags: [Android,NDK,Aria2]
description:  Aria2 JNI代码的优化
---

Aria2 JNI代码的优化
<!--more-->
****简介****
===
```java
前几天有一个需求是当Bt下载在一段时间之内当平均的下载速度小于某一个值的时候，要自动的切换成传统的Http下载，当然也要提供一个可以外部手动切换的功能，考虑到每一个下载任务都要做监听
就会去考虑性能的问题，而且之前老大发现这个程序挺耗电量的，所以就去考虑下性能的问题

由于下载的所有的内容，都是通过Aria2下载库提供的，然后通过JNI将这些信息封装成一个对象然后返回给java层代码，让java层代码刷新显示，先看看之前刷新代码的逻辑

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
    ....
    //每隔500毫秒更新一次界面
    if (count >= 500) {
        //回调给Android，更新界面
        downloadInfoChangeCallBackForAndroid(v);
    }
}

这个方法是跑在子线程中的，所以这里有一个无限的for循环，当然出现了错误的时候，会退出这个循环，如果在正常的情况下，会每隔500毫秒收集一次当前正在下载的内容，暂停，停止不会收集，然后将
收集的信息调用 downloadInfoChangeCallBackForAndroid(v); 来更新界面显示,对应的java代码是这样的
public void downloadInfoChange(final ArrayList<DownloadEntity> entities)
{
    ArrayList<DownloadEntity> adapterData = mAdapter.getDownLodels();
    for (DownloadEntity entity : entities){
        for(DownloadEntity datItem:adapterData){
        //判断gid是否是一样，如果是一样，替换掉model，通知适配器局部刷新
        if(!TextUtils.isEmpty(datItem.getGid()))
        {
        if (datItem.getGid().equals(entity.getGid()) || (!TextUtils.isEmpty(entity.getFollowGid()) && entity.getFollowGid().equals(datItem.getGid())))
        {
            datItem.setCompletedLength(entity.getCompletedLength());
            datItem.setTotalLength(entity.getTotalLength());
            datItem.setDownloadFileName(entity.getDownloadFileName());
            datItem.setDownloadSpeed(entity.getDownloadSpeed());
			...
            //通过cell的方式来改变界面的内容，而不使用  mAdapter.notifyItemChanged(); 后一种会发生界面的闪动
            RecyclerView.ViewHolder viewHolder = mRecycleView.findViewHolderForAdapterPosition(adapterData.indexOf(datItem));
            if (viewHolder != null && viewHolder instanceof DownLoadAdapter.DownLoadViewHolder)
            {
                ((DownLoadAdapter.DownLoadViewHolder) viewHolder).bindData(datItem);
                break;
            }
        }
    }
}

先看看downloadInfoChangeCallBackForAndroid 函数的实现

/**
 * 回调执行Android中的downloadInfoChange
 * @param vector 传递的是引用
 */
void downloadInfoChangeCallBackForAndroid(std::vector<DownloadStatus>& vector){
    JNIEnv *env;
    jVM->GetEnv((void**) &env, JNI_VERSION_1_4);
    int attached  = 0;
    if(env==NULL)
    {
        attached  = 1;
        jVM->AttachCurrentThread(&env, NULL);
    }
    ...
    //构造Java DownloadEntity的构造函数
    //public DownloadEntity(byte[] gid,long totalLength,long completedLength,int downloadSpeed,int uploadSpeed,byte[] downloadFileName,
    //byte[] downloadDirPath,boolean isDownloadMemory,int downloadFileNumbers,byte[] followGid,int numberConnection,int usefulBtNumbers){
    jobject downloadEntiry = env->NewObject(android_downloadEntityClass,android_downloadEntiryConstructId,gid_byteArray,
                                                    downloadStatusModel.totalLength,downloadStatusModel.completedLength,
                                                    downloadStatusModel.downloadSpeed,downloadStatusModel.uploadSpeed,cName_byteArray,
                                                    dir_byteArray,isDownloadMetadata,downloadStatusModel.fileNumbsers,follow_gid_byteArray,
                                                    downloadStatusModel.numbersConnection,downloadStatusModel.usefulSeedNumbers);
    ...
    if(attached)
    {
        jVM->DetachCurrentThread();
    }
}

也即是每500毫秒刷新一次界面，然后调用 public void downloadInfoChange(final ArrayList<DownloadEntity> entities)，所以我就得在JNI层构建这样的对象，这里就存在一个问题，这样会导致
中间生成了大量的对象，而这些对象只是用来传递一些内容，这样就会导致大量内存分配，会导致频繁的gc

问题2，JNI层没有一个对应的Java层对象的引用，导致每次状态的改变等都需要通过回调java层来改变对应的状态，这样导致状态的管理很麻烦

针对上面的俩个问题，其实只要做到JNI层也能够持有java层对应的引用，这样在JNI层能改变对应的状态，java层也能改变对应的状态，而且对应的另一边都能够获取改变之后的值，这样就解决了第二个问题
如果这个问题解决了，那对于第一个问题，那也简单了，我们可以在JNI中使用一个集合保存对应的java 实体引用，当要刷新界面的时候，我们通过修改实体的值，当修改完成之后，我们通过java层改变界面
当然这个的前提是JNI中持有了java同一个引用对象
```

**** 验证JNI对象跟java层对象是同一个 ****
===
```java
首先讲解下JNI中多线程对象共享的问题，由于我们想免去中间生成的对象，那我们就必须将java层的对象在JNI中持有，由于我们JNI中还有一个子线程，所以就涉及到了多线程的问题
//保存java层对应的下载实体类
static std::vector<jobject> downloadEntitys;
当执行添加下载的时候，简单的将这个对象保存
downloadEntitys.push_back(entity);

然后在子线程中获取到这个对象，进而更改他的状态,比如在下面代码中通过获取到这个对象的属性判断当前下载是哪一个
jobject entiry = NULL;
for(jobject objz : downloadEntitys)
{
    int objdownloadJavaId = env->CallIntMethod(objz,downloadEntiry_getDownLoadId_methodId);
    if(objdownloadJavaId == javaId)
    {
        entiry = objz;
        break;
    }
}

运行不出现意外的化会得到下面的这样错误信息
```
![结果显示](/uploads/Aria优化/多线程出现异常.png)
```java
NDK开发，一般出现的错误，都是很难定位的，如果单纯的是通过出现错误信息获取的化，但是我们可以在调试模式下，出现异常的时候，他会停在那里，会返回当前的堆栈信息
这样你就可以知道哪里出现了问题 这就非常的牛逼了，这个功能的支持前提是ndk的项目配置要让Android Studio所能识别，这样他才能开启JNI的调试
```
![结果显示](/uploads/Aria优化/JNI多线程错误定位.png)
```
通过上面的错误，我们可以找到对应的错误在哪一行，出现这个问题的原因是多线程的问题，在多线程环境下，我们的对象应该使用GlobalReference来创建，也即是全局引用的意思,下面是关于他的介绍
```
![结果显示](/uploads/Aria优化/全局引用的作用.png)
```java
所以之前添加的代码，只要改成这样就可以了，后面在子线程中获取到对应的值就不会出现问题了
jobject globalRefObj = env->NewGlobalRef(entity);
//首先保存java层的下载实体类
downloadEntitys.push_back(globalRefObj);

当然其他类似的要在多线程访问的变量也要做类似的处理，比如
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
	....
}
这个class的引用，如果不使用NewGlobalRef的化，在子线程中访问也会有类似的问题

但是如果使用了全局引用，一定记得当不用的时候，要调用 env->DeleteGlobalRef(xxx); 释放, 类似这样
//释放class全局引用
env->DeleteGlobalRef(downloadGlobalEntity);
env->DeleteGlobalRef(aria2ApiClass);
env->DeleteGlobalRef(android_arrayListClass);
env->DeleteGlobalRef(android_downloadEntityClass);
env->DeleteGlobalRef(android_downloadglobalClass);
env->DeleteGlobalRef(android_downloadShowFileEntityClass);
env->DeleteGlobalRef(android_downloadPeerEntityClass);

下面我们要验证java层对象跟JNI层的对象是同一个对象 我们通过简单的验证俩者的hashCode，比如我们在java层可以这样写
public void addDownLoad(DownloadEntity downloadEntity)
{
    Log.d(TAG,"javaHashCode " + downloadEntity.hashCode());
    AriaApi.Aria2AddDownloadEntity(downloadEntity);
}

JNI中可以这样写
jclass objClass = env->GetObjectClass(entity);
jmethodID hashCodeId = env->GetMethodID(objClass,"hashCode","()I");
jint hashCode = env->CallIntMethod(entity,hashCodeId);
LOGD("entity hashCode %d",hashCode);
```
![结果显示](/uploads/Aria优化/hashCode结果.png)
```java
可以看得出来，JNI中传递的对象跟java层是一样的， 使用了全局引用之后，我们可以再次的查看一下最后得到的对象是否是hashCode一样的
下面是验证的代码

jclass objClass = env->GetObjectClass(entity);
jmethodID hashCodeId = env->GetMethodID(objClass,"hashCode","()I");
jint hashCode = env->CallIntMethod(entity,hashCodeId);
LOGD("entity hashCode %d",hashCode);

jobject globalRefObj = env->NewGlobalRef(entity);
//首先保存java层的下载实体类
downloadEntitys.push_back(globalRefObj);

jclass globalObjClass = env->GetObjectClass(globalRefObj);
jmethodID globalHashCodeId = env->GetMethodID(globalObjClass,"hashCode","()I");
jint globalHashCode = env->CallIntMethod(globalRefObj,globalHashCodeId);
LOGD("global entity hashCode %d",globalHashCode);

```
![结果显示](/uploads/Aria优化/全局引用也是同一个对象.png)
```java
如果你还存在怀疑，最简单粗暴的直接在JNI子线程中改变全局对象的属性，然后在java层获取这个属性的值，这里就不验证了，我之前验证是一样的，那既然是同一个引用的化，那处理起来就非常简单了
我们可以在JNI中或者在java层都可以改变这个对象，而且另一边都可以正确的获取到改变之后的值
```

****代码的优化****
===
```java
前面已经验证了是同一个对象，那下面代码改起来就非常简单了
void downloadInfoChangeCallBackForAndroid(std::vector<DownloadStatusMode>& vector){
    JNIEnv *env;
    jVM->GetEnv((void**) &env, JNI_VERSION_1_4);
    int attached  = 0;
    if(env==NULL)
    {
        attached  = 1;
        jVM->AttachCurrentThread(&env, NULL);
    }
	...
	//遍历集合vector
        size_t len = vector.size();
        for (size_t i =0; i < len; i ++) {
            DownloadStatusMode downloadStatusModel = vector[i];
            std::string c_gid_str = aria2::gidToHex(downloadStatusModel.gid);
            jobject findEntiry = NULL;
            for(jobject objz : downloadEntitys)
            {
                //获取到gid
                jstring gidJStr = (jstring) env->CallObjectMethod(objz,downloadEntiry_getGid_methodId);
                const char * c_gidJStr = env->GetStringUTFChars(gidJStr,NULL);
                //找到gid一样的元素 ,注意这里使用strcmp 比较会偶现比较错误 if(strcmp(c_gid,c_gidJStr) == 0)//,要换成c++的比较主要是因为aria2::gidToHex的原因
                if(c_gid_str == c_gidJStr) //if(c_gid_str.compare(c_gidJStr) == 0)
                {
                    findEntiry = objz;
                    env->ReleaseStringUTFChars(gidJStr,c_gidJStr);
                    break;
                }
                env->ReleaseStringUTFChars(gidJStr,c_gidJStr);
            }
            //对于磁力链接，首先会先下载元数据，然后下载种子文件，会返回俩个gid,这个时候followGid跟之前实体的gid是一样的，是一样的
            if(findEntiry == NULL)
            {
                //如果还是为空，那就是出现了错误，直接退出
                if(findEntiry == NULL)
                {
                    if(attached)
                    {
                        jVM->DetachCurrentThread();
                    }
                    LOGD("downloadInfoChangeCallBackForAndroid findEntiry NULL");
                    return;
                }
            }
            //找到了对应的实体，更新对应的值
            //setTotalLength
            env->CallVoidMethod(findEntiry,downloadEntiry_setTotalLength_methodId,downloadStatusModel.totalLength);
            //setCompleteLength
            env->CallVoidMethod(findEntiry,downloadEntiry_setCompletedLength_methodId,downloadStatusModel.completedLength);
            //setDownloadSpeed
            env->CallVoidMethod(findEntiry,downloadEntiry_setDownloadSpeed_methodId,downloadStatusModel.downloadSpeed);
            //setUploadSpeed
            env->CallVoidMethod(findEntiry,downloadEntiry_setUploadSpeed_methodId,downloadStatusModel.uploadSpeed);
            //setDownloadFileName
            jstring  jfileName = env->NewStringUTF(downloadStatusModel.filename.c_str());
            env->CallVoidMethod(findEntiry,downloadEntiry_setDownloadFileName_methodId,jfileName);
            .... 
        }
        //最后调用downloadInfoChange 函数，public static void downloadInfoChange() 通知界面要发生改变了，只是通知
        env->CallStaticVoidMethod(aria2ApiClass,android_downloadInfoChange);
	}
	...
}

而我们java层的 downloadInfoChange函数也变成了这样
@Override
public void downloadInfoChange()
{
    ...
    ArrayList<DownloadEntity> adapterData = mAdapter.getDownLodels();
    for(DownloadEntity entity:adapterData)
    {
        //通过cell的方式来改变界面的内容，而不使用  mAdapter.notifyItemChanged(); 后一种会发生界面的闪动
        RecyclerView.ViewHolder viewHolder = mRecycleView.findViewHolderForAdapterPosition(adapterData.indexOf(entity));
        if (viewHolder != null && viewHolder instanceof DownLoadAdapter.DownLoadViewHolder)
        {
            ((DownLoadAdapter.DownLoadViewHolder) viewHolder).bindData(entity);
        }
    }
}

相比之前我们构建一堆的对象然后返回给java层，java层再来刷新，现在的简单多了，JNI修改对应的属性值，修改完成之后，通知java改变，不用传递任何的对象，也是基于这个优化，我才能完成后面的
自动切换的逻辑，代码写起来也不用这么恶心

```
