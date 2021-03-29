---
layout: pager
title: Matrix IO Canary源码解析
date: 2019-03-13 09:51:57
tags: [Android,Matrix]
description:  Matrix IO Canary源码解析
---
### 简介
> 前面分析了Matrix 中 Trace Canary 模块，了解了对应的检测原理实现，本文继续分析 IO Canary 模块,在分析这块内容的时候最好知道对于IO的性能主要检测的内容有哪些，这个可以参考极客时间的Android 开发高手 张绍文大佬 写的  11 | I/O优化（下）：如何监控线上I/O操作，里面说到检测的方式采用Native的hook，没有采用java的hook，是因为无法监控到native代码，性能极差，甚至由于采用java hook 要对应的做兼容处理，所以相比之下采用native hook,监测文件的操作函数，比如open,read,write,close通过hook这些函数我们可以拿到对应的信息，比如hook open函数，我们可以拿到操作的文件名，fd，文件的原始大小等，hook read,write可以得到对应的读写次数，读写总大小，使用buffer大小，读写总耗时等，而对于要检测的内容包括下面几种
1.主线程的Io操作
2.读写Buff过小
3.重复读
4.资源泄漏

### 读写Buff过小检测的原因
其他检测的点，估计大家都觉得理所当然，这里解释下为啥要检测Buff过小的原因,我们知道，对于文件系统是以block为单位读写，对于磁盘是以page 为单位读写，看起来即使我们在应用程序上面使用
很小的Buffer，在底层应该差别不大，那是不是这样呢？
![结果显示](/uploads/Matrix Trace分析/Io文件Buff使用.png)
虽然后面俩次系统调用的时间的确会少一些，但是也会有一定的耗时，如果我们的Buffer太小，会导致多次无用的系统调用和内存拷贝，导致read/write的次数增多，从而影响性能，那多大的Buff Size
合适呢，一般为4KB,在实际应用中，ObjectOutputStream 就是一个很好的例子，ObjectOutputStream使用的buff size非常小，如果我们使用BufferInputStream或者ByteArrayOutputStream后整体的性能,会有非常明显的提升,下面是检测的结果
![结果显示](/uploads/Matrix Trace分析/ObjectOutputSteam读写性能.png)
所以检测读写Buff过小是非常重要的

### 使用
在Io Canary页面中包含了下面的检测条目,也验证了我们前面说的要检测的条目
![结果显示](/uploads/Matrix Trace分析/IO检测的种类.png)


### 源码分析
```java
Plugin plugin = Matrix.with().getPluginByClass(IOCanaryPlugin.class);
    if (!plugin.isPluginStarted()) {
          MatrixLog.i(TAG, "plugin-io start");
     plugin.start();
}

public class IOCanaryPlugin extends Plugin {
    private static final String TAG = "Matrix.IOCanaryPlugin";

    private final IOConfig     mIOConfig;
    private IOCanaryCore mCore;
	
    ...
    @Override
    public void init(Application app, PluginListener listener) {
        super.init(app, listener);
        IOCanaryUtil.setPackageName(app);
        mCore = new IOCanaryCore(this);
    }

    @Override
    public void start() {
        super.start();
        mCore.start();
    }
    ...
}
在init的时候会创建一个 IOCanaryCore的对象,接着在start的时候会调用 IOCanaryCore 中对应的start 函数		
public class IOCanaryCore implements OnJniIssuePublishListener, IssuePublisher.OnIssueDetectListener {
    ...
    //标识是否启动
    private boolean           mIsStart;
    private CloseGuardHooker  mCloseGuardHooker;

	
    /**
     * 开始检测
     */
    public void start() {
        //初始化操作
        initDetectorsAndHookers(mIOConfig);
        //标识资源已经启动
        synchronized (this) {
            mIsStart = true;
        }
    }
	
    /**
     * 初始化操作
     * @param ioConfig
     */
    private void initDetectorsAndHookers(IOConfig ioConfig) {
        assert ioConfig != null;

        //是否检测 主线程的IO
        if (ioConfig.isDetectFileIOInMainThread()
            || ioConfig.isDetectFileIOBufferTooSmall()//是否检测Buff 过小
            || ioConfig.isDetectFileIORepeatReadSameFile()) {//是否检测重复读的情况

            //执行JNI 的hook
            IOCanaryJniBridge.install(ioConfig, this);
        }

        //if only detect io closeable leak use CloseGuardHooker is Better   是否检测 资源泄漏的情况 ,默认为true
        if (ioConfig.isDetectIOClosableLeak()) {
            //Hook CloseGuide 接受到 StrideMode 的结果
            mCloseGuardHooker = new CloseGuardHooker(this);
            mCloseGuardHooker.hook();
        }
    }
}	

private static final boolean DEFAULT_DETECT_MIAN_THREAD_FILE_IO     = true;
private static final boolean DEFAULT_DETECT_SMALL_BUFFER            = true;
private static final boolean DEFAULT_DETECT_REPEAT_READ_SAME_FILE   = true;
private static final boolean DEFAULT_DETECT_CLOSABLE_LEAK           = true;

public boolean isDetectFileIOInMainThread() {
   return mDynamicConfig.get(IDynamicConfig.ExptEnum.clicfg_matrix_io_file_io_main_thread_enable.name(), DEFAULT_DETECT_MIAN_THREAD_FILE_IO);
}

public boolean isDetectFileIORepeatReadSameFile() {
    return mDynamicConfig.get(IDynamicConfig.ExptEnum.clicfg_matrix_io_repeated_read_enable.name(), DEFAULT_DETECT_REPEAT_READ_SAME_FILE);
}

public boolean isDetectFileIOBufferTooSmall() {
    return mDynamicConfig.get(IDynamicConfig.ExptEnum.clicfg_matrix_io_small_buffer_enable.name(), DEFAULT_DETECT_SMALL_BUFFER);
}

public boolean isDetectIOClosableLeak() {
    return mDynamicConfig.get(IDynamicConfig.ExptEnum.clicfg_matrix_io_closeable_leak_enable.name(), DEFAULT_DETECT_CLOSABLE_LEAK);
}

所以会执行  IOCanaryJniBridge.install(ioConfig, this);
public class IOCanaryJniBridge {
    ...
    public static void install(IOConfig config, OnJniIssuePublishListener listener) {
        ...
        //load lib  加载so
        if (!loadJni()) {
            MatrixLog.e(TAG, "install loadJni failed");
            return;
        }

        //set listener  到了这里就说明加载so 成功，JNI 也初始化成功
        sOnIssuePublishListener = listener;

        try {
            //set config
            if (config != null) {
                //如果当前要检测 主线程的 IO操作
                if (config.isDetectFileIOInMainThread()) {
                    //告知JNI 层监测 主线程的Io情况
                    enableDetector(DetectorType.MAIN_THREAD_IO);

                    // ms to μs  也即是 主线程的Io 连续读写 不能低于500毫秒 ,防止将主线程的Io  都收集起来
                    setConfig(ConfigKey.MAIN_THREAD_THRESHOLD, config.getFileMainThreadTriggerThreshold() * 1000L);
                }

                //如果要检测 读写Buff过小的操作
                if (config.isDetectFileIOBufferTooSmall()) {
                    //告知 JNI 检测 Buff 过小的情况
                    enableDetector(DetectorType.SMALL_BUFFER);
                    //告知 检测 Buff 过小的限制，这里是限制为 4k
                    setConfig(ConfigKey.SMALL_BUFFER_THRESHOLD, config.getFileBufferSmallThreshold());
                }

                //如果要检测 重复读 的操作
                if (config.isDetectFileIORepeatReadSameFile()) {
                    //告知JNI 检测 重复读的情况
                    enableDetector(DetectorType.REPEAT_READ);
                    //告知检测 重复读的次数，这里是5次
                    setConfig(ConfigKey.REPEAT_READ_THRESHOLD, config.getFileRepeatReadThreshold());
                }
            }

            //hook  执行 hook
            doHook();

            //标识执行成功
            sIsTryInstall = true;
        } catch (Error e) {
            MatrixLog.printErrStackTrace(TAG, e, "call jni method error");
        }
    }
    ...
    /**
     * 加载 so
     * @return
     */
    private static boolean loadJni() {
        if (sIsLoadJniLib) {
            return true;
        }

        //加载so  ,就会调用 JNI_ONLOAD
        try {
            System.loadLibrary("io-canary");
        } catch (Exception e) {
            MatrixLog.e(TAG, "hook: e: %s", e.getLocalizedMessage());
            sIsLoadJniLib = false;
            return false;
        }

        //标识加载so 成功
        sIsLoadJniLib = true;
        return true;
    }
    ...
}		
首先执行loadJni()函数，就会执行 System.loadLibrary("io-canary"); 就会调用到 ，io_canary_jni.cc 中的  JNI_OnLoad()函数，这个会由 System.loadLibrary时候掉用
        /**
         * 加载 so的时候 触发的时机
         * @param vm
         * @param reserved
         * @return
         */
        JNIEXPORT jint JNICALL JNI_OnLoad(JavaVM *vm, void *reserved){
            __android_log_print(ANDROID_LOG_DEBUG, kTag, "JNI_OnLoad");
            kInitSuc = false;


            //初始化环境, 找到 Java 相关的类
            if (!InitJniEnv(vm)) {
                return -1;
            }

            //首先构建一个IOCanary对象，然后设置 OnIssuePublish  函数指针对象
            iocanary::IOCanary::Get().SetIssuedCallback(OnIssuePublish);

            //标识初始化成功
            kInitSuc = true;
            __android_log_print(ANDROID_LOG_DEBUG, kTag, "JNI_OnLoad done");
            return JNI_VERSION_1_6;
        }
		
        /*
         * 初始化环境,找到对应的java 类
         */
        static bool InitJniEnv(JavaVM *vm) {
            kJvm = vm;
            JNIEnv* env = NULL;
            if (kJvm->GetEnv((void**)&env, JNI_VERSION_1_6) != JNI_OK){
                __android_log_print(ANDROID_LOG_ERROR, kTag, "InitJniEnv GetEnv !JNI_OK");
                return false;
            }

            jclass temp_cls = env->FindClass("com/tencent/matrix/iocanary/core/IOCanaryJniBridge");
            if (temp_cls == NULL)  {
                __android_log_print(ANDROID_LOG_ERROR, kTag, "InitJniEnv kJavaBridgeClass NULL");
                return false;
            }
            //构建成全局变量，因为要多线程中使用
            kJavaBridgeClass = reinterpret_cast<jclass>(env->NewGlobalRef(temp_cls));

            ....
			
            //获取到ArrayList 的构造函数，含有add 函数
            jclass list_cls = env->FindClass("java/util/ArrayList");
            kListClass = reinterpret_cast<jclass>(env->NewGlobalRef(list_cls));
            kMethodIDListConstruct = env->GetMethodID(list_cls, "<init>", "()V");
            kMethodIDListAdd = env->GetMethodID(list_cls, "add", "(Ljava/lang/Object;)Z");
            return true;
        }
		
    接着调用 iocanary::IOCanary::Get().SetIssuedCallback(OnIssuePublish);
    首先看iocanary::IOCanary 的类声明		
	
    class IOCanary {
    public:
        //不允许构造函数的拷贝，以即是单例的形式
        IOCanary(const IOCanary&) = delete;
        IOCanary& operator=(IOCanary const&) = delete;

        //提供get 函数，获取具体的对象
        static IOCanary& Get();

        void RegisterDetector(DetectorType type);
        void SetConfig(IOCanaryConfigKey key, long val);
        void SetJavaMainThreadId(long main_thread_id);

        void SetIssuedCallback(OnPublishIssueCallback issued_callback);
        ...

    private:
        //单例
        IOCanary();
        ~IOCanary();
        ...
	}
	
	可以看出这个类是一个单例的形式，他的实现为
	IOCanary& IOCanary::Get() {
        //构建一个 IOCanary 对象 ，这里为 静态的，也即是第一次的时候才会执行初始化
        static IOCanary kInstance;
        return kInstance;
    }

    //默认的构造函数
    IOCanary::IOCanary() {
        //创建一个线程, 线程执行的时候会执行 IOCanary::Detect 回调函数
        std::thread detect_thread(&IOCanary::Detect, this);
        //detach是用来和线程对象分离的，这样线程可以独立地执行，
        detect_thread.detach();
    }
    在构建函数执行的时候，会创建一个线程，这样线程执行的时候就会执行 IOCanary::Detect 函数指针 	
    /**
     * 子线程中执行
     */
    void IOCanary::Detect() {
        std::vector<Issue> published_issues;
        std::shared_ptr<IOInfo> file_io_info;

        //无线循环
        while (true) {
            published_issues.clear();

            //返回值如果为 0 代表从 queue 队列中取出了第一个元素，取出的内容放到了 file_io_info 中
            int ret = TakeFileIOInfo(file_io_info);

            if (ret != 0) {
                break;
            }

            //执行检测，这里面包括 主线程的Io 检测， Buff 过小的检测，重复读的检测,将结果保存到  published_issues 中
            for (auto detector : detectors_) {
                detector->Detect(env_, *file_io_info, published_issues);
            }

            //将结果分发出去
            if (issued_callback_ && !published_issues.empty()) {
                issued_callback_(published_issues);
            }

            file_io_info = nullptr;
        }
    }
	先看 TakeFileIOInfo() 函数，这个函数就是判断queue_队列是否为空，如果不为空则取出里面的元素，如果为空，则处于阻塞的状态，这里由于是刚运行为空，处于阻塞的状态
	int IOCanary::TakeFileIOInfo(std::shared_ptr<IOInfo> &file_io_info) {
        //加锁   更加灵活的锁管理类模板，构造时是否加锁是可选的，在对象析构时如果持有锁会自动释放锁，所有权可以转移。对象生命期内允许手动加锁和释放锁。
        std::unique_lock<std::mutex> lock(queue_mutex_);

        //如果这个队列为空，则当前线程等待
        while (queue_.empty()) {
            queue_cv_.wait(lock);
            //如果当前要退出的状态，返回-1
            if (exit_) {
                return -1;
            }
        }

        //到了这里就说明队列不为空，取出第一个 返回
        file_io_info = queue_.front();
        queue_.pop_front();
        return 0;
    }	
	
    继续回到 iocanary::IOCanary::Get().SetIssuedCallback(OnIssuePublish);
    /**
     * 传递回调
     * @param issued_callback
     */
    void IOCanary::SetIssuedCallback(OnPublishIssueCallback issued_callback) {
        issued_callback_ = issued_callback;
    }
    这里是设置一个结果的函数指针，
         /**
         * 回调函数，用来接受IO 处理的结果，这边接受到之后，再将结果传回Java层
         * @param published_issues
         */
        void OnIssuePublish(const std::vector<Issue>& published_issues) {
            if (!kInitSuc) {
                __android_log_print(ANDROID_LOG_ERROR, kTag, "OnIssuePublish kInitSuc false");
                return;
            }

            //由于涉及到子线程，所以要先Attach 才能正确的获取到Env对象
            JNIEnv* env;
            bool attached;
            jint j_ret = kJvm->GetEnv((void**)&env, JNI_VERSION_1_6);
            if (j_ret == JNI_EDETACHED) {
                jint jAttachRet = kJvm->AttachCurrentThread(&env, nullptr);
                if (jAttachRet != JNI_OK) {
                    __android_log_print(ANDROID_LOG_ERROR, kTag, "onIssuePublish AttachCurrentThread !JNI_OK");
                    return;
                } else {
                    attached = true;
                }
            } else if (j_ret != JNI_OK || env == NULL) {
                return;
            }

            //检查是否有异常发生，如果有抛出异常
            jthrowable exp = env->ExceptionOccurred();
            if (exp != NULL) {
                __android_log_print(ANDROID_LOG_INFO, kTag, "checkCanCallbackToJava ExceptionOccurred, return false");
                env->ExceptionDescribe();
                return;
            }
            //下面就是返回结果了
            jobject j_issues = env->NewObject(kListClass, kMethodIDListConstruct);
            ...
            //调用Java层的 onIssurePublish 函数
            env->CallStaticVoidMethod(kJavaBridgeClass, kMethodIDOnIssuePublish, j_issues);
            env->DeleteLocalRef(j_issues);
            if (attached) {
                kJvm->DetachCurrentThread();
            }
        } 		
        也就是设置一个结果的中转函数指针，然后在这个函数指针中再将结果回调到java层,这样JNI_OnLoad函数执行完毕，继续回到java层 中的
        public static void install(IOConfig config, OnJniIssuePublishListener listener) {
                ...
                //如果当前要检测 主线程的 IO操作
                if (config.isDetectFileIOInMainThread()) {
                    //告知JNI 层监测 主线程的Io情况
                    enableDetector(DetectorType.MAIN_THREAD_IO);
                    // ms to μs  也即是 主线程的Io 连续读写 不能低于500毫秒 ,防止将主线程的Io  都收集起来
                    setConfig(ConfigKey.MAIN_THREAD_THRESHOLD, config.getFileMainThreadTriggerThreshold() * 1000L);
                }

                //如果要检测 读写Buff过小的操作
                if (config.isDetectFileIOBufferTooSmall()) {
                    //告知 JNI 检测 Buff 过小的情况
                    enableDetector(DetectorType.SMALL_BUFFER);
                    //告知 检测 Buff 过小的限制，这里是限制为 4k
                    setConfig(ConfigKey.SMALL_BUFFER_THRESHOLD, config.getFileBufferSmallThreshold());
                }

                //如果要检测 重复读 的操作
                if (config.isDetectFileIORepeatReadSameFile()) {
                    //告知JNI 检测 重复读的情况
                    enableDetector(DetectorType.REPEAT_READ);
                    //告知检测 重复读的次数，这里是5次
                    setConfig(ConfigKey.REPEAT_READ_THRESHOLD, config.getFileRepeatReadThreshold());
                }
                ...
        }
		
        根据当前配置要检测的内容，调用  enableDetector(DetectorType.MAIN_THREAD_IO); 告知JNI要检测的内容
        //Java 层调用 告知 JNI 层要检测的内容
        JNIEXPORT void JNICALL
        Java_com_tencent_matrix_iocanary_core_IOCanaryJniBridge_enableDetector(JNIEnv *env, jclass type, jint detector_type) {
            iocanary::IOCanary::Get().RegisterDetector(DetectorType(detector_type));
        }
		
    /**
     * 注册
     * @param type
     */
    void IOCanary::RegisterDetector(DetectorType type) {
        switch (type) {
            case DetectorType::kDetectorMainThreadIO:
                // 检测主线程的 Io 操作
                detectors_.push_back(new FileIOMainThreadDetector());
                break;
            case DetectorType::kDetectorSmallBuffer:
                //检测 Buff 过小的操作
                detectors_.push_back(new FileIOSmallBufferDetector());
                break;
            case DetectorType::kDetectorRepeatRead:
                //检测 重复读的 情况
                detectors_.push_back(new FileIORepeatReadDetector());
                break;
            default:
                break;
        }
    }
    这里的 detector 的定义是这样的 
    //集合，用来存储当前要检测的种类
    std::vector<FileIODetector*> detectors_; 也即是根据java层要检测的内容，响应的在JNI层中创建对应的对象，然后添加到 detectors_ 集合中
    接着java层调用
    // ms to μs  也即是 主线程的Io 连续读写 不能低于500毫秒 ,防止将主线程的Io  都收集起来
    setConfig(ConfigKey.MAIN_THREAD_THRESHOLD, config.getFileMainThreadTriggerThreshold() * 1000L);
    相应的调用到JNI层的实现为
    //Java 层调用 告知 JNI 层要检测的标准
    JNIEXPORT void JNICALL
    Java_com_tencent_matrix_iocanary_core_IOCanaryJniBridge_setConfig(JNIEnv *env, jclass type, jint key, jlong val) {
        iocanary::IOCanary::Get().SetConfig(IOCanaryConfigKey(key), val);
    }
    //传递要检测的标准,存储到env 中
    void IOCanary::SetConfig(IOCanaryConfigKey key, long val) {
        env_.SetConfig(key, val);
    }
    这里的env 为  IOCanary 中的一个成员
    //这里会构建一个 IOCanaryEnv 对象
    IOCanaryEnv env_;
	
    //默认构造函数的执行
    IOCanaryEnv::IOCanaryEnv() {
        //添加默认的检测标准
        configs_[IOCanaryConfigKey::kMainThreadThreshold] = kDefaultMainThreadTriggerThreshold;
        configs_[IOCanaryConfigKey::kSmallBufferThreshold] = kDefaultBufferSmallThreshold;
        configs_[IOCanaryConfigKey::kRepeatReadThreshold] = kDefaultRepeatReadThreshold;
    }
    其中configs_的定义为
    //声明一个数组，用来存储检测的标准
    long configs_[IOCanaryConfigKey::kConfigKeysLen];
    //保存动态设置的标准
    void IOCanaryEnv::SetConfig(IOCanaryConfigKey key, long val) {
        if (key >= IOCanaryConfigKey::kConfigKeysLen) {
            return;
        }

        configs_[key] = val;
    }
	也即是会将java层的检测标准保存到 这个数组中，这些检测标准中分别为 主线程的Io操作500毫秒，Buff过小的标准为4KB，重复读写的标准为5次
	接着java层调用
        //hook  执行 hook
        doHook();
        对应到JNI层的实现为
	//Java 层调用， Hook 对应的so
        JNIEXPORT jboolean JNICALL
        Java_com_tencent_matrix_iocanary_core_IOCanaryJniBridge_doHook(JNIEnv *env, jclass type) {
            __android_log_print(ANDROID_LOG_INFO, kTag, "doHook");

            //分别Hook     "libopenjdkjvm.so","libjavacore.so","libopenjdk.so"
            for (int i = 0; i < TARGET_MODULE_COUNT; ++i) {
                const char* so_name = TARGET_MODULES[i];
                __android_log_print(ANDROID_LOG_INFO, kTag, "try to hook function in %s.", so_name);

                //采用elf 打开这个so
                loaded_soinfo* soinfo = elfhook_open(so_name);
                if (!soinfo) {
                    __android_log_print(ANDROID_LOG_WARN, kTag, "Failure to open %s, try next.", so_name);
                    continue;
                }

                //然后hook 原本so 中 open, open64 函数，替换成 ProxyOpen  ,ProxyOpen64  ,同时将 原本要hook的函数指针接受过来，方便后面调用系统原本的方法
                elfhook_replace(soinfo, "open", (void*)ProxyOpen, (void**)&original_open);
                elfhook_replace(soinfo, "open64", (void*)ProxyOpen64, (void**)&original_open64);

                //判断当前so 是否为  libjavacore.so
                bool is_libjavacore = (strstr(so_name, "libjavacore.so") != nullptr);
                if (is_libjavacore) {
                    //如果当前so 为 libjavacore.so  hook read 函数
                    if (!elfhook_replace(soinfo, "read", (void*)ProxyRead, (void**)&original_read)) {
                        __android_log_print(ANDROID_LOG_WARN, kTag, "doHook hook read failed, try __read_chk");

                        //如果 hook  libjavacore.so 中的 read 函数失败之后， 尝试的hook  __read_chk 函数
                        //这是由于不同版本的 Android 系统实现有所不同，在Android 7.0之后，我们还需要替换下面这三个方法 open64,__read_chk,__write_chk
                        if (!elfhook_replace(soinfo, "__read_chk", (void*)ProxyRead, (void**)&original_read)) {
                            __android_log_print(ANDROID_LOG_WARN, kTag, "doHook hook failed: __read_chk");
                            return false;
                        }
                    }

                    //hook write 函数
                    if (!elfhook_replace(soinfo, "write", (void*)ProxyWrite, (void**)&original_write)) {
                        __android_log_print(ANDROID_LOG_WARN, kTag, "doHook hook write failed, try __write_chk");

                        //如果 hook  libjavacore.so 中的 write 函数失败之后，尝试的 hook __write_chk 函数
                        if (!elfhook_replace(soinfo, "__write_chk", (void*)ProxyWrite, (void**)&original_write)) {
                            __android_log_print(ANDROID_LOG_WARN, kTag, "doHook hook failed: __write_chk");
                            return false;
                        }
                    }
                }

                //hook so 中的close 函数
                elfhook_replace(soinfo, "close", (void*)ProxyClose, (void**)&original_close);

                //关闭soinfo
                elfhook_close(soinfo);
            }
            return true;
        }
	可以看出来 分别Hook     "libopenjdkjvm.so","libjavacore.so","libopenjdk.so" 然后 hook 对应的open，close，read，write，在Android 7.0之后，
	我们还需要替换下面这三个方法 open64,__read_chk,__write_chk ，同时我们将原本的函数指针保存下来，毕竟真正的逻辑实现还得交给系统的方法来实现,所以要将原本的函数纸指针保存下来	
	现在来总结下 当前所做的操作 java层加载了so函数，之后在JNI层执行了一系列的初始化，将当前java层要检测的选项，已经检查的标准传递到了JNI层，JNI层根据要检测的内容，创建对应的
	FileIODetector 类来对应的检测，同时 IOCanary 启动了一个线程在等待结果，当前没有结果处于阻塞的状态
```
现在来看看打开一个文件的时候会做什么操作
```java
	由于我们hook了系统的open函数，所以当调用open的时候会执行到这个方法
	/**
	*  Proxy for open: callback to the java layer   open 函数的代理方法
     */
     //todo astrozhou 解决非主线程打开，主线程操作问题
     int ProxyOpen(const char *pathname, int flags, mode_t mode) {
         //如果当前不是主线程，直接执行原本的系统的函数
         if(!IsMainThread()) {
                return original_open(pathname, flags, mode);
         }
         //执行系统原本的open函数,返回值为打开的文件描述符
         int ret = original_open(pathname, flags, mode);

         //如果返回的文件描述符为 -1 代表打开失败
         if (ret != -1) {
              DoProxyOpenLogic(pathname, flags, mode, ret);
         }
         return ret;
     }

	 可以看到还是调用系统的open函数来执行文件的打开操作，注意这个open函数得到是linux 的文件操作符，接着调用DoProxyOpenLogic()函数进行信息的收集操作
	 
		/**
         * 监控到了系统的open函数之后，收集信息  ,只有在open的时候才会
         * @param pathname   当前打开的文件路径
         * @param flags
         * @param mode  当前打开的文件模式
         * @param ret   当前打开的文件描述符
         */
        static void DoProxyOpenLogic(const char *pathname, int flags, mode_t mode, int ret) {
            //获取到env
            JNIEnv* env = NULL;
            kJvm->GetEnv((void**)&env, JNI_VERSION_1_6);
            //如果env 获取为空，或者还没有初始化成功
            if (env == NULL || !kInitSuc) {
                __android_log_print(ANDROID_LOG_ERROR, kTag, "ProxyOpen env null or kInitSuc:%d", kInitSuc);
            } else {

                //执行 IOCanaryJniBridge 中的 getJavaContext 函数 ,获取到 JavaContext 对象
                jobject java_context_obj = env->CallStaticObjectMethod(kJavaBridgeClass, kMethodIDGetJavaContext);
                if (NULL == java_context_obj) {
                    return;
                }

                //获取到JavaContext 中的 stack 字段，这个字段保存了当前的 堆栈信息
                jstring j_stack = (jstring) env->GetObjectField(java_context_obj, kFieldIDStack);
                //获取到 JavaContext 中的 threadName 字段，这个字段 标识了当前线程名
                jstring j_thread_name = (jstring) env->GetObjectField(java_context_obj, kFieldIDThreadName);

                //转成c的字符串
                char* thread_name = jstringToChars(env, j_thread_name);
                char* stack = jstringToChars(env, j_stack);

                //然后构建一个JavaContext 对象
                JavaContext java_context(GetCurrentThreadId(), thread_name == NULL ? "" : thread_name, stack == NULL ? "" : stack);
                free(stack);
                free(thread_name);

                //调用IOCanary 中的 onOpen函数 ,通知当前执行了open函数了 ret为打开的结果
                iocanary::IOCanary::Get().OnOpen(pathname, flags, mode, ret, java_context);

                //释放局部对象
                env->DeleteLocalRef(java_context_obj);
                env->DeleteLocalRef(j_stack);
                env->DeleteLocalRef(j_thread_name);
            }
        }
	在这个函数中利用JNI调用java层的 IOCanaryJniBridge 中的 getJavaContext 函数 ,获取到 JavaContext 对象，接着得到了这个对象宏的stack，threadName字段的值,现在看看这个对象
	/**
     * 声明为private，给c++部分调用！！！不要干掉！！！
     * @return
     */
    private static JavaContext getJavaContext() {
        try {
            return new JavaContext();
        } catch (Throwable th) {
            MatrixLog.printErrStackTrace(TAG, th, "get javacontext exception");
        }
        return null;
    }
	
	private static final class JavaContext {
        private final String stack;
        private final String threadName;

        private JavaContext() {
            //stack 为当前的堆栈信息,这里调用getThrowableStack 将这个堆栈信息变成一个字符串的形势，这里就不往下看了
            stack = IOCanaryUtil.getThrowableStack(new Throwable());
            //为当前的线程名
            threadName = Thread.currentThread().getName();
        }
    }
		
	接着在JNI层构建一个 JavaContext对象，保存当前java的堆栈信息，线程名
	class JavaContext {
    public:
        JavaContext(intmax_t thread_id , const std::string& thread_name, const std::string& stack)
                : thread_id_(thread_id), thread_name_(thread_name), stack_(stack) {
        }

        //当前的线程id
        const intmax_t thread_id_;
        //线程明
        const std::string thread_name_;
        //当前堆栈信息
        const std::string stack_;
    };	
	最后调用 iocanary::IOCanary::Get().OnOpen(pathname, flags, mode, ret, java_context);
    void IOCanary::OnOpen(const char *pathname, int flags, mode_t mode, int open_ret, const JavaContext& java_context) {
        collector_.OnOpen(pathname, flags, mode, open_ret, java_context);
    }
	这里的collector_对象为 IOCanary 中的一个成员
	//这里会构建一个 IOInfoCollector 对象，这个对象用来收集Io的信息，并且IOInfoCollector 中有一个Map集合用来存储当前的IO信息，当打开一个文件的时候就会添加到这个map中
    IOInfoCollector collector_;
	
	// A singleton to collect and generate operation info
    class IOInfoCollector {
    public:
        void OnOpen(const char *pathname, int flags, mode_t mode, int open_ret, const JavaContext& java_context);
        void OnRead(int fd, const void *buf, size_t size, ssize_t read_ret, long read_cost);
        void OnWrite(int fd, const void *buf, size_t size, ssize_t write_ret, long write_cost);
        std::shared_ptr<IOInfo> OnClose(int fd, int close_ret);

    private:
		...
        //存储了 Io 收集信息的集合 ,key 为打开的文件描述符
        std::unordered_map<int, std::shared_ptr<IOInfo>> info_map_;
    };
	
	当调用到 collector_.OnOpen()函数的时候就会执行到
	/**
     * 收集系统执行open 函数之后的内容
     * @param pathname
     * @param flags
     * @param mode
     * @param open_ret   打开文件之后返回的文件描述符
     * @param java_context
     */
    void IOInfoCollector::OnOpen(const char *pathname, int flags, mode_t mode
            , int open_ret, const JavaContext& java_context) {
        //__android_log_print(ANDROID_LOG_DEBUG, kTag, "OnOpen fd:%d; path:%s", open_ret, pathname);

        //首先当前的文件描述符要为合法
        if (open_ret == -1) {
            return;
        }

        //根据当前的文件描述符从map 中查找,如果 满足了 info_map_.find(open_ret) != info_map_.end() 那说明当前的文件描述符已经在这个map中了，直接返回
        if (info_map_.find(open_ret) != info_map_.end()) {
            //__android_log_print(ANDROID_LOG_WARN, kTag, "OnOpen fd:%d already in info_map_", open_ret);
            return;
        }

        //到这里，说明map中不存在当前的文件描述符,那么构建一个  IOInfo 对象,添加到 map 集合中
        std::shared_ptr<IOInfo> info = std::make_shared<IOInfo>(pathname, java_context);
        info_map_.insert(std::make_pair(open_ret, info));
    }
	也即是会根据当前的文件描述符在这个集合中查找是否存在，如果不存在就封装成一个IOInfo 对象，然后保存到这个集合中,接下来看看 IOInfo 对象，这是收集当前文件描述符对应的IO信息的重要类
	//rw short for read/write operation
    class IOInfo {
    public:
        IOInfo() = default;
        IOInfo(const std::string path, const JavaContext java_context)
                : start_time_μs_(GetSysTimeMicros()),op_type_(kInit)
                , path_(path), java_context_(java_context) {
        }
        //当前打开的文件路径
        const std::string path_;
        //当前对应的Java 堆栈信息
        const JavaContext java_context_;
        //当前的时间
        int64_t start_time_μs_;
        //类型，默认是初始化
        FileOpType op_type_ = kInit;
        //读写的次数
        int op_cnt_ = 0;
        //当前读写 buff 的最大值
        long buffer_size_ = 0;
        //读写的大小
        long op_size_ = 0;
        //读写发费的时间
        long rw_cost_μs_ = 0;
        //最大继续读的时间
        long max_continual_rw_cost_time_μs_ = 0;
        //统计读写 单次花费时间的最大值
        long max_once_rw_cost_time_μs_ = 0;
        //当前继续读的时间
        long current_continual_rw_time_μs_ = 0;
        //上一次读写的时间
        int64_t last_rw_time_μs_ = 0;
        //当前文件的大小
        long file_size_ = 0;
        //总共花费的时间，也即是打开到关闭的时间
        long total_cost_μs_ = 0;
    };
	从IOInfo构造函数中可以知道当执行了open()函数，添加到info_map_ 集合中的时候，就为 start_time_μs_ 赋值了当前的时间，标识IO操作的开始时间
	
	到这里总结下打开文件发生的事情，首先利用系统原本的open函数进行文件的打开操作，得到对应的文件描述符，当文件描述符正确的时候，利用JNI得到java层对应的JavaContext对象，这个对象
	包含了对应java层的堆栈信息，已经线程名，接着将这些信息保存到 info_map_集合中
```
现在来看看读,写文件的时候发生了什么
```java
ssize_t ProxyRead(int fd, void *buf, size_t size) {
     //如果当前不是主线程，直接执行原本的系统的函数
     if(!IsMainThread()) {
         return original_read(fd, buf, size);
     }

     //获取到当前的时间
     int64_t start = GetTickCountMicros();

     //调用系统原本的read 函数读取 ，ret 为读取的大小，也即是真实的读取大小,size 为buff的大小
     size_t ret = original_read(fd, buf, size);

     //得到系统读取发费了多少时间
     long read_cost_μs = GetTickCountMicros() - start;

     //__android_log_print(ANDROID_LOG_DEBUG, kTag, "ProxyRead fd:%d buf:%p size:%d ret:%d cost:%d", fd, buf, size, ret, read_cost_μs);

     //统计当前读取的情况
     iocanary::IOCanary::Get().OnRead(fd, buf, size, ret, read_cost_μs);
     return ret;
}
可以看到首先是调用系统原本的read函数进行文件读操作，在调用读函数之前记录下当前的时间，然后读取完毕之后，就能得到读取花费的时间,系统的read的返回值为系统真正读取的大小，接着调用
iocanary::IOCanary::Get().OnRead(fd, buf, size, ret, read_cost_μs); 
    /**
     * 通知 系统执行了 read 函数了，这里要将这些信息收集起来
     * @param fd                当前的文件描述符
     * @param buf
     * @param size              buff的大小
     * @param read_ret          真正读取的大小
     * @param read_cost         读取花费的时间
     */
    void IOCanary::OnRead(int fd, const void *buf, size_t size, ssize_t read_ret, long read_cost) {
        collector_.OnRead(fd, buf, size, read_ret, read_cost);
    }
	
	void IOInfoCollector::OnRead(int fd, const void *buf, size_t size, ssize_t read_ret, long read_cost) {
        //检查参数是否合法
        if (read_ret == -1 || read_cost < 0) {
            return;
        }

        //根据当前的文件文件描述符从map 中查找 ,如果 满足 info_map_.find(fd) == info_map_.end() 说明 当前文件描述符不在map集合中，说明是非法的
        if (info_map_.find(fd) == info_map_.end()) {
             //__android_log_print(ANDROID_LOG_DEBUG, kTag, "OnRead fd:%d not in info_map_", fd);
            return;
        }

        //收集当前读写的信息
        CountRWInfo(fd, FileOpType::kRead, size, read_cost);
    }
	在看CountRWInfo 函数之前，先看写的操作，因为这俩个函数最终都是调用这个函数完成信息的收集操作
	ssize_t ProxyWrite(int fd, const void *buf, size_t size) {
        //如果当前不是主线程，直接执行原本的系统的函数
        if(!IsMainThread()) {
            return original_write(fd, buf, size);
        }

        //获取到当前的时间
        int64_t start = GetTickCountMicros();

        //调用系统原本的write 函数写 ，ret 为真正的写的大小，也即是真实的读取大小,size 为buff的大小
        size_t ret = original_write(fd, buf, size);

        //得到系统写发费了多少时间
        long write_cost_μs = GetTickCountMicros() - start;

        //__android_log_print(ANDROID_LOG_DEBUG, kTag, "ProxyWrite fd:%d buf:%p size:%d ret:%d cost:%d", fd, buf, size, ret, write_cost_μs);

        //统计当前读取的情况
        iocanary::IOCanary::Get().OnWrite(fd, buf, size, ret, write_cost_μs);
        return ret;
   }
   逻辑跟读是差不多的，在真正调用系统的写函数之前记录下当前的时间，之后再调用系统的write函数，结束之后我们就能得到调用write函数消耗的时间,接着调用
   iocanary::IOCanary::Get().OnWrite(fd, buf, size, ret, write_cost_μs);
   void IOCanary::OnWrite(int fd, const void *buf, size_t size, ssize_t write_ret, long write_cost) {
       collector_.OnWrite(fd, buf, size, write_ret, write_cost);
   }
   
   void IOInfoCollector::OnWrite(int fd, const void *buf, size_t size, ssize_t write_ret, long write_cost) {
        //检查参数是否合法
        if (write_ret == -1 || write_cost < 0) {
            return;
        }

        //根据当前的文件文件描述符从map 中查找 ,如果 满足 info_map_.find(fd) == info_map_.end() 说明 当前文件描述符不在map集合中，说明是非法的
        if (info_map_.find(fd) == info_map_.end()) {
            //__android_log_print(ANDROID_LOG_DEBUG, kTag, "OnWrite fd:%d not in info_map_", fd);
            return;
        }

        //收集当前读写的信息
        CountRWInfo(fd, FileOpType::kWrite, size, write_cost);
    }
	
   看到这里读跟写的操作都是一样的，最终都会调用到 CountRWInfo(fd, FileOpType::kWrite, size, write_cost); 完成信息的收集操作
   /**
     * 收集当前读写的信息
     * @param fd
     * @param fileOpType
     * @param op_size
     * @param rw_cost
     */
    void IOInfoCollector::CountRWInfo(int fd, const FileOpType &fileOpType, long op_size, long rw_cost) {

        //确保当前打开的文件描述符在 map中
        if (info_map_.find(fd) == info_map_.end()) {
            return;
        }

        //当前的时间
        const int64_t now = GetSysTimeMicros();

        //次数加一
        info_map_[fd]->op_cnt_ ++;
        //读写的buff大小累加
        info_map_[fd]->op_size_ += op_size;
        //读写花费的时间累加
        info_map_[fd]->rw_cost_μs_ += rw_cost;

        //更新单次读写 时间的最大值
        if (rw_cost > info_map_[fd]->max_once_rw_cost_time_μs_) {
            info_map_[fd]->max_once_rw_cost_time_μs_ = rw_cost;
        }
        //判断当前的时间跟上一次读写的时间差，是否 大于 8毫秒 ,也即是 16毫秒的一半(一帧刷新的时间)
        if (info_map_[fd]->last_rw_time_μs_ > 0 && (now - info_map_[fd]->last_rw_time_μs_) < kContinualThreshold) {
            //如果小于，累加
            info_map_[fd]->current_continual_rw_time_μs_ += rw_cost;
        } else {
            //如果大于，替换掉
            info_map_[fd]->current_continual_rw_time_μs_ = rw_cost;
        }

        //赋值最大的读写时间
        if (info_map_[fd]->current_continual_rw_time_μs_ > info_map_[fd]->max_continual_rw_cost_time_μs_) {
            info_map_[fd]->max_continual_rw_cost_time_μs_ = info_map_[fd]->current_continual_rw_time_μs_;
        }

        //赋值上一次的读写的时间
        info_map_[fd]->last_rw_time_μs_ = now;

        //如果当前的buff_size 小于 当前读的大小，赋值操作
        if (info_map_[fd]->buffer_size_ < op_size) {
            info_map_[fd]->buffer_size_ = op_size;
        }

        //辅助当前的type
        if (info_map_[fd]->op_type_ == FileOpType::kInit) {
            info_map_[fd]->op_type_ = fileOpType;
        }
    }
	这里会统计很多的信息，比如读写的次数，读写的总时间，读写buff的总大小，单次读写时间的最大值，单次读写buff的最大值等
```
现在来看看读,关闭文件的时候发生了什么
```java
/**
*  Proxy for close: callback to the java layer  系统 close 函数的代理方法
*/
int ProxyClose(int fd) {
    //如果当前不是主线程，直接执行原本的系统的函数
    if(!IsMainThread()) {
        return original_close(fd);
    }

    //调用系统的close 函数，ret 代表关闭的结果
    int ret = original_close(fd);
	
    //__android_log_print(ANDROID_LOG_DEBUG, kTag, "ProxyClose fd:%d ret:%d", fd, ret);

    //收集当前关闭的信息
    iocanary::IOCanary::Get().OnClose(fd, ret);
    return ret;
}
调用系统原本的close函数执行真正的文件关闭操作，最后在关闭的时候，调用     iocanary::IOCanary::Get().OnClose(fd, ret); 
	/**
     * 通知 系统执行了 close 函数了，这里要将这些信息收集起来
     * @param fd                   当前要关闭的文件描述符
     * @param close_ret            关闭之后的结果
     */
    void IOCanary::OnClose(int fd, int close_ret) {
        //收集信息,更新打开文件的时间， 文件的大小，并从集合中移除出去
        std::shared_ptr<IOInfo> info = collector_.OnClose(fd, close_ret);
        if (info == nullptr) {
            return;
        }
        //将当前的io信息提交
        OfferFileIOInfo(info);
    }
	
   collector_.OnClose(fd, close_ret); 函数的实现
   /**
     * 收集系统执行close 函数之后的内容
     * @param fd
     * @param close_ret
     * @return
     */
    std::shared_ptr<IOInfo> IOInfoCollector::OnClose(int fd, int close_ret) {

        //根据当前的文件文件描述符从map 中查找 ,如果 满足 info_map_.find(fd) == info_map_.end() 说明 当前文件描述符不在map集合中，说明是非法的
        if (info_map_.find(fd) == info_map_.end()) {
            //__android_log_print(ANDROID_LOG_DEBUG, kTag, "OnClose fd:%d not in info_map_", fd);
            return nullptr;
        }

        //文件从打开到关闭的时间
        info_map_[fd]->total_cost_μs_ = GetSysTimeMicros() - info_map_[fd]->start_time_μs_;
        //当前文件的大小
        info_map_[fd]->file_size_ = GetFileSize(info_map_[fd]->path_.c_str());
        //从集合中找到对应的对象，然后从这个集合中移除出去，返回这个对象
        std::shared_ptr<IOInfo> info = info_map_[fd];
        info_map_.erase(fd);
        return info;
    }
    接着调用OfferFileIOInfo(info)函数提交了这次文件操作的结果
     /**
     * 将当前的io 信息添加到
     * @param file_io_info
     */
    void IOCanary::OfferFileIOInfo(std::shared_ptr<IOInfo> file_io_info) {
        //加锁
        std::unique_lock<std::mutex> lock(queue_mutex_);
        //添加到集合中
        queue_.push_back(file_io_info);

        //唤醒等待的函数
        queue_cv_.notify_one();
        //释放锁
        lock.unlock();
    }
    这里做的操作就是将当前的文件操作对应的信息从集合 info_map_中取出来，因为当前执行了关闭的操作，所以对于当前这个文件来说可以分析当前的IO信息了，所以从集合中移除，然后添加到
    queue_ 队列中,接着调用了  queue_cv_.notify_one(); 这样我们前面等待的线程就不会阻塞了,继续回到 IOCanary::Detect() 函数
	
    /**
     * 子线程中执行
     */
    void IOCanary::Detect() {
        std::vector<Issue> published_issues;
        std::shared_ptr<IOInfo> file_io_info;

        //无线循环
        while (true) {
            published_issues.clear();

            //返回值如果为 0 代表从 queue 队列中取出了第一个元素，取出的内容放到了 file_io_info 中
            int ret = TakeFileIOInfo(file_io_info);

            if (ret != 0) {
                break;
            }

            //执行检测，这里面包括 主线程的Io 检测， Buff 过小的检测，重复读的检测,将结果保存到  published_issues 中
            for (auto detector : detectors_) {
                detector->Detect(env_, *file_io_info, published_issues);
            }

            //将结果分发出去
            if (issued_callback_ && !published_issues.empty()) {
                issued_callback_(published_issues);
            }

            file_io_info = nullptr;
        }
    }
    继续分析 TakeFileIOInfo()函数，因为前面在文件关闭的时候，往这个队列中添加了内容，所以这里就不会为空，就能返回这个元素了
	
	int IOCanary::TakeFileIOInfo(std::shared_ptr<IOInfo> &file_io_info) {
        //加锁   更加灵活的锁管理类模板，构造时是否加锁是可选的，在对象析构时如果持有锁会自动释放锁，所有权可以转移。对象生命期内允许手动加锁和释放锁。
        std::unique_lock<std::mutex> lock(queue_mutex_);

        //如果这个队列为空，则当前线程等待
        while (queue_.empty()) {
            queue_cv_.wait(lock);
            //如果当前要退出的状态，返回-1
            if (exit_) {
                return -1;
            }
        }

        //到了这里就说明队列不为空，取出第一个 返回
        file_io_info = queue_.front();
        queue_.pop_front();
        return 0;
    }
	
    接着执行
    //执行检测，这里面包括 主线程的Io 检测， Buff 过小的检测，重复读的检测,将结果保存到  published_issues 中
    for (auto detector : detectors_) {
        detector->Detect(env_, *file_io_info, published_issues);
   }
```
主线程的Io 检测
```java
	/**
     * 主线程 Io的检测
     * @param env
     * @param file_io_info
     * @param issues
     */
    void FileIOMainThreadDetector::Detect(const IOCanaryEnv &env, const IOInfo &file_io_info,std::vector<Issue>& issues) {
        //判断当前是否为主线程
        if (GetMainThreadId() == file_io_info.java_context_.thread_id_) {
            int type = 0;

            //当前文件描述符中最大的连续读写时间 大于 13毫秒
            if (file_io_info.max_continual_rw_cost_time_μs_ > IOCanaryEnv::kPossibleNegativeThreshold) {
                type = 1;
            }

            //根据当前的 主线程的检测标准，读要大于500毫秒
            if(file_io_info.max_continual_rw_cost_time_μs_ > env.GetMainThreadThreshold()) {
                type |= 2;
            }

            if (type != 0) {
                //构建一个 Issue
                Issue issue(kType, file_io_info);
                issue.repeat_read_cnt_ = type;  //use repeat to record type
                //将issure 添加到 issues中
                PublishIssue(issue, issues);
            }
        }
    }
	可以看到首先判断 当前的线程是否为主线程，判断的依据为 GetMainThreadId() == file_io_info.java_context_.thread_id_，这个thread_id_ 辅助的时机为当执行open函数的时候，创建JavaContext
	JavaContext java_context(GetCurrentThreadId(), thread_name == NULL ? "" : thread_name, stack == NULL ? "" : stack);赋值的，也即是通过gettid()获取到当前的线程id
	 /**
     * 获取到当前线程的id
     * @return
     */
    intmax_t GetCurrentThreadId() {
        return gettid();
    }
	
	而对于GetMainThreadId() 我们可以直接通过getpid()获取到当前进程的id，所以如果当前进程的id跟线程id一样的化，那就说明为主线程
	 /**
     * 获取主线程的id
     * @return
     */
    intmax_t GetMainThreadId() {
        static intmax_t pid = getpid();
        return pid;
    }
	接着判断当前Io操作的时间是否大于500毫秒，如果都满足的化，构建一个Issue 对象，然后通过 PublishIssue(issue, issues);
	/**
     * 将target 添加到 issues 集合中
     * @param target
     * @param issues
     */
    void FileIODetector::PublishIssue(const Issue &target, std::vector<Issue>& issues) {
        //如果当前的key 在 published_issue_set_ 中，代表之前已经发送过了，直接返回
        if (IsIssuePublished(target.key_)) {
            return;
        }
        //添加到 issures 集合中
        issues.push_back(target);
        //将当前的key 添加到 published_issue_set_ 中，代表这个key 已经发送过
        MarkIssuePublished(target.key_);
    }
	
	//添加key 到 published_issue_set_ 代表当前的key 已经发送过
    void FileIODetector::MarkIssuePublished(const std::string &key) {
        published_issue_set_.insert(key);
    }

    //判断当前的key 是否在 published_issue_set_集合中能否找到，如果满足 published_issue_set_.find(key) != published_issue_set_.end() 说明在集合中，返回true
    bool FileIODetector::IsIssuePublished(const std::string &key) {
        return published_issue_set_.find(key) != published_issue_set_.end();
    }
    //用来存储已经发送过
    std::set<std::string> published_issue_set_;
	可以看到这里检测符合主线程Io检测的条件就通过构建一个Issue 对象，然后添加到 issues 中，这个集合为我们前面传递进去的
```
Buff过小的 检测
```java
	/**
     * 判断 是否 Buff 过小
     * @param env
     * @param file_io_info
     * @param issues
     */
    void FileIOSmallBufferDetector::Detect(const IOCanaryEnv &env, const IOInfo &file_io_info,std::vector<Issue>& issues) {
	
        //读写的次数大于 20次 , 平均 每次 Buff 的 大小小于 Buff过小的标识， 并且最大继续读的时间大于 14毫秒
        if (file_io_info.op_cnt_ > env.kSmallBufferOpTimesThreshold && (file_io_info.op_size_ / file_io_info.op_cnt_) < env.GetSmallBufferThreshold()
                && file_io_info.max_continual_rw_cost_time_μs_ >= env.kPossibleNegativeThreshold) {

            //发布Buff过小的结论
            PublishIssue(Issue(kType, file_io_info), issues);
        }
    }
	可以看到检测重复读的条件为 重写的次数要达到20次，平均每场Buff的大小不应该小于检测标准，当满足了之后，通过PublishIssue(Issue(kType, file_io_info), issues);保存到issues 集合中
```
重复读的 检测
```java
    void FileIORepeatReadDetector::Detect(const IOCanaryEnv &env,
                                          const IOInfo &file_io_info,
                                          std::vector<Issue>& issues) {

        //当前操作的文件路径
        const std::string& path = file_io_info.path_;

        //如果满足observing_map_.find(path) == observing_map_.end() 说明 这个path 没有在这个集合中,则添加到这个集合中
        if (observing_map_.find(path) == observing_map_.end()) {

            //如果最大连续读写的时间小于13毫秒 直接返回
            if (file_io_info.max_continual_rw_cost_time_μs_ < env.kPossibleNegativeThreshold) {
                return;
            }
            //添加到集合中
            observing_map_.insert(std::make_pair(path, std::vector<RepeatReadInfo>()));
        }

        //根据path 取出 对应的  RepeatReadInfo 集合
        std::vector<RepeatReadInfo>& repeat_infos = observing_map_[path];

        //如果当前的类型为写的 ，直接返回
        if (file_io_info.op_type_ == FileOpType::kWrite) {
            repeat_infos.clear();
            return;
        }

        //到了这里就说明当前为都读的情况 ,构建一个RepeatReadInfo 对象
        RepeatReadInfo repeat_read_info(file_io_info.path_, file_io_info.java_context_.stack_, file_io_info.java_context_.thread_id_,
                                      file_io_info.op_size_, file_io_info.file_size_);

        //如果当前repeat_infos 集合的大小小于0 ，将当前的 repeat_read_info 添加到集合中，直接返回
        if (repeat_infos.size() == 0) {
            repeat_infos.push_back(repeat_read_info);
            return;
        }

        //repeat_infos 集合大小不等于0,查看最后一个元素的时间跟当前的时间比 如果大于 17毫秒的化，那清空 repeat_infos 集合
        if((GetTickCount() - repeat_infos[repeat_infos.size() - 1].op_timems) > 17) {   //17ms todo astrozhou add to params
            repeat_infos.clear();
        }

        //最先添加元素的时间跟当前的时间差小于17毫秒

        //标识是否从repeat_infos 集合中找到 repeat_read_info
        bool found = false;
        int repeatCnt;
        for (auto& info : repeat_infos) {
            if (info == repeat_read_info) {
                //标识找到了
                found = true;
                //增加重复读的次数
                info.IncRepeatReadCount();
                //得到当前重复读的次数
                repeatCnt = info.GetRepeatReadCount();
                break;
            }
        }

        //如果没有找到,repeat_infos 集合中添加 repeat_read_info
        if (!found) {
            repeat_infos.push_back(repeat_read_info);
            return;
        }

        //检查重复读的次数是否达到了设置的重复读的次数
        if (repeatCnt >= env.GetRepeatReadThreshold()) {
            //如果达到了，构建一个Issure，添加到queue 中
            Issue issue(kType, file_io_info);
            issue.repeat_read_cnt_ = repeatCnt;
            issue.stack = repeat_read_info.GetStack();
            PublishIssue(issue, issues);
        }
    }
	检测的标准为重复读写的次数达到了5次，之后会通过构建一个Issue 对象，然后添加到issues 集合中
```
结果的分发
```java
    void IOCanary::Detect() {
        std::vector<Issue> published_issues;
        std::shared_ptr<IOInfo> file_io_info;

        //无线循环
        while (true) {
            published_issues.clear();

            //返回值如果为 0 代表从 queue 队列中取出了第一个元素，取出的内容放到了 file_io_info 中
            int ret = TakeFileIOInfo(file_io_info);

            if (ret != 0) {
                break;
            }

            //执行检测，这里面包括 主线程的Io 检测， Buff 过小的检测，重复读的检测,将结果保存到  published_issues 中
            for (auto detector : detectors_) {
                detector->Detect(env_, *file_io_info, published_issues);
            }

            //将结果分发出去
            if (issued_callback_ && !published_issues.empty()) {
                issued_callback_(published_issues);
            }

            file_io_info = nullptr;
        }
    }
	
	当检测完了之后，执行 issued_callback_(published_issues); 将结果传递出去，这个issued_callback_ 为我们在初始化的时候调用下面的函数传递进来的，所以会执行对应的函数
	//首先构建一个IOCanary对象，然后设置 OnIssuePublish  函数指针对象
    iocanary::IOCanary::Get().SetIssuedCallback(OnIssuePublish);
	/**
         * 回调函数，用来接受IO 处理的结果，这边接受到之后，再将结果传回Java层
         * @param published_issues
         */
        void OnIssuePublish(const std::vector<Issue>& published_issues) {
            ...
            //由于涉及到子线程，所以要先Attach 才能正确的获取到Env对象
            JNIEnv* env;
            bool attached;
            jint j_ret = kJvm->GetEnv((void**)&env, JNI_VERSION_1_6);
            if (j_ret == JNI_EDETACHED) {
                jint jAttachRet = kJvm->AttachCurrentThread(&env, nullptr);
                if (jAttachRet != JNI_OK) {
                    __android_log_print(ANDROID_LOG_ERROR, kTag, "onIssuePublish AttachCurrentThread !JNI_OK");
                    return;
                } else {
                    attached = true;
                }
            } else if (j_ret != JNI_OK || env == NULL) {
                return;
            }

            //检查是否有异常发生，如果有抛出异常
            jthrowable exp = env->ExceptionOccurred();
            if (exp != NULL) {
                __android_log_print(ANDROID_LOG_INFO, kTag, "checkCanCallbackToJava ExceptionOccurred, return false");
                env->ExceptionDescribe();
                return;
            }

            //下面就是返回结果了
            jobject j_issues = env->NewObject(kListClass, kMethodIDListConstruct);

            //遍历当前要分析 published_issues
            for (const auto& issue : published_issues) {
                //类型
                jint type = issue.type_;
                //文件名
                jstring path = env->NewStringUTF(issue.file_io_info_.path_.c_str());
                //文件的大小
                jlong file_size = issue.file_io_info_.file_size_;
                //读写的次数
                jint op_cnt = issue.file_io_info_.op_cnt_;
                //当前读写buff的最大值
                jlong buffer_size = issue.file_io_info_.buffer_size_;
                //花费的时间
                jlong op_cost_time = issue.file_io_info_.rw_cost_μs_/1000;
                //当前的类型
                jint op_type = issue.file_io_info_.op_type_;
                //总的buff 大小
                jlong op_size = issue.file_io_info_.op_size_;
                //线程名
                jstring thread_name = env->NewStringUTF(issue.file_io_info_.java_context_.thread_name_.c_str());
                //java的堆栈信息
                jstring stack = env->NewStringUTF(issue.stack.c_str());
                //重复的次数
                jint repeat_read_cnt = issue.repeat_read_cnt_;

                //构建一个Java 层的 Issure对象
                jobject issue_obj = env->NewObject(kIssueClass, kMethodIDIssueConstruct, type, path, file_size, op_cnt, buffer_size,
                                                   op_cost_time, op_type, op_size, thread_name, stack, repeat_read_cnt);

                //调用ArrayList 的add 函数
                env->CallBooleanMethod(j_issues, kMethodIDListAdd, issue_obj);

                env->DeleteLocalRef(issue_obj);
                env->DeleteLocalRef(stack);
                env->DeleteLocalRef(thread_name);
                env->DeleteLocalRef(path);
            }

            //调用Java层的 onIssurePublish 函数
            env->CallStaticVoidMethod(kJavaBridgeClass, kMethodIDOnIssuePublish, j_issues);

            env->DeleteLocalRef(j_issues);

            if (attached) {
                kJvm->DetachCurrentThread();
            }
        } 
	
	最终会调用到 IOCanaryJniBridge 中的 onIssuePublish 函数 ,这样就将结果转回到了java层,这里不往下看
	/**
     * JNI方法调用
     * @param issues
     */
    public static void onIssuePublish(ArrayList<IOIssue> issues) {
        if (sOnIssuePublishListener == null) {
            return;
        }
        sOnIssuePublishListener.onIssuePublish(issues);
    }
```
资源泄漏的检测
```java
这里先收下对应资源的检测，Android 本身是有严苛模式的。开启这个模式之后会帮我们检测，具体的类为StrictMode,关于使用 StrictMode 可以网上查看对应的内容

public class IOCanaryCore implements OnJniIssuePublishListener, IssuePublisher.OnIssueDetectListener {
    ...
    private void initDetectorsAndHookers(IOConfig ioConfig) {
        ...
        //if only detect io closeable leak use CloseGuardHooker is Better   是否检测 资源泄漏的情况 ,默认为true
        if (ioConfig.isDetectIOClosableLeak()) {
            //Hook CloseGuide 接受到 StrideMode 的结果
            mCloseGuardHooker = new CloseGuardHooker(this);
            mCloseGuardHooker.hook();
        }
    }
    ...
}
public final class CloseGuardHooker {
    ...
    public CloseGuardHooker(IssuePublisher.OnIssueDetectListener issueListener) {
        this.issueListener = issueListener;
    }

    /**
     * set to true when a certain thread try hook once; even failed.
     */
    public void hook() {
        MatrixLog.i(TAG, "hook sIsTryHook=%b", mIsTryHook);
        if (!mIsTryHook) {
            boolean hookRet = tryHook();
            MatrixLog.i(TAG, "hook hookRet=%b", hookRet);
            //标识hook成功
            mIsTryHook = true;
        }
    }
    ...
}
private boolean tryHook() {
        try {
            Class<?> closeGuardCls = Class.forName("dalvik.system.CloseGuard");
            Class<?> closeGuardReporterCls = Class.forName("dalvik.system.CloseGuard$Reporter");
            //得到字段 REPORTER
            Field fieldREPORTER = closeGuardCls.getDeclaredField("REPORTER");
            //得到字段 ENABLED
            Field fieldENABLED = closeGuardCls.getDeclaredField("ENABLED");

            fieldREPORTER.setAccessible(true);
            fieldENABLED.setAccessible(true);

            sOriginalReporter = fieldREPORTER.get(null);
            //将Enable 字段改为true
            fieldENABLED.set(null, true);

            // open matrix close guard also
            MatrixCloseGuard.setEnabled(true);

            ClassLoader classLoader = closeGuardReporterCls.getClassLoader();
            if (classLoader == null) {
                return false;
            }

            //将CloseGuard 字段中的 REPORTER 代理为我们的 IOCloseLeakDetector ,这样我们就能收到这个结果
            fieldREPORTER.set(null, Proxy.newProxyInstance(classLoader,
                new Class<?>[]{closeGuardReporterCls},
                new IOCloseLeakDetector(issueListener, sOriginalReporter)));

            fieldREPORTER.setAccessible(false);
            return true;
        } catch (Throwable e) {
            MatrixLog.e(TAG, "tryHook exp=%s", e);
        }

        return false;
    }
	
	具体的思路为利用反射,把CloseGuard 中的ENABLED 值设为true，利用动态代理，把REPORTER替换成我们定义的proxy,这里先大致讲下严苛模式的检查流程，下面是基本的使用
	StrictMode.setThreadPolicy(new StrictMode.ThreadPolicy.Builder()
                 .detectDiskReads()
                 .detectDiskWrites()
                 .detectNetwork()   // or .detectAll() for all detectable problems
                 .penaltyLog()
                 .build());
	可以看出我们通过构建了一个ThreadPolicy ,调用setThreadPolicy函数传递到了StrictMode 中
    public static void setThreadPolicy(final ThreadPolicy policy) {
        setThreadPolicyMask(policy.mask);
    }
    private static void setThreadPolicyMask(final int policyMask) {
        ...
        setBlockGuardPolicy(policyMask);
        Binder.setThreadStrictModePolicy(policyMask);
    }
    // Sets the policy in Dalvik/libcore (BlockGuard)
    private static void setBlockGuardPolicy(final int policyMask) {
        if (policyMask == 0) {
            BlockGuard.setThreadPolicy(BlockGuard.LAX_POLICY);
            return;
        }
        final BlockGuard.Policy policy = BlockGuard.getThreadPolicy();
        final AndroidBlockGuardPolicy androidPolicy;
        if (policy instanceof AndroidBlockGuardPolicy) {
            androidPolicy = (AndroidBlockGuardPolicy) policy;
        } else {
            androidPolicy = threadAndroidPolicy.get();
            BlockGuard.setThreadPolicy(androidPolicy);
        }
        androidPolicy.setPolicyMask(policyMask);
    }
	
    首先获取到 BlockGuard 中的 Policy 对应的函数实现为
    public static Policy getThreadPolicy() {
        return threadPolicy.get();
    }
	private static ThreadLocal<Policy> threadPolicy = new ThreadLocal<Policy>() {
        @Override protected Policy initialValue() {
            return LAX_POLICY;
        }
    };
	public static final Policy LAX_POLICY = new Policy() {
            public void onWriteToDisk() {}
            public void onReadFromDisk() {}
            public void onNetwork() {}
            public int getPolicyMask() {
                return 0;
            }
        };
	所以这里获取到的 LAX_POLICY 不是 AndroidBlockGuardPolicy的实现，所以会走androidPolicy = threadAndroidPolicy.get();,而threadAndroidPolicy 定义为
	private static final ThreadLocal<AndroidBlockGuardPolicy> threadAndroidPolicy = new ThreadLocal<AndroidBlockGuardPolicy>() {
        @Override
        protected AndroidBlockGuardPolicy initialValue() {
            return new AndroidBlockGuardPolicy(0);
        }
    };
	上面的逻辑就是会设置一个AndroidBlockGuardPolicy 到 BlockGuard.setThreadPolicy(androidPolicy);中，现在设置进来了那么看看系统是怎么使用的，比如我们的BlockGuardOs，
	这是文件操作的基本类,这里就看open函数
	@Override public FileDescriptor open(String path, int flags, int mode) throws ErrnoException {
        BlockGuard.getThreadPolicy().onReadFromDisk();
        if ((mode & O_ACCMODE) != O_RDONLY) {
            BlockGuard.getThreadPolicy().onWriteToDisk();
        }
        return os.open(path, flags, mode);
    }
	在open函数中有这样的调用  BlockGuard.getThreadPolicy().onReadFromDisk();,由于前面我们设置了policy 为 AndroidBlockGuardPolicy 所以会执行对应的函数
	public void onReadFromDisk() {
       if ((mPolicyMask & DETECT_DISK_READ) == 0) {
            return;
        }
        if (tooManyViolationsThisLoop()) {
            return;
        }
        BlockGuard.BlockGuardPolicyException e = new StrictModeDiskReadViolation(mPolicyMask);
        e.fillInStackTrace();
        startHandlingViolationException(e);
   }
   在startHandlingViolationException()进行分析，如果出现了问题，就会在这个函数的内部调用handleViolation(v),然后通过IPC的机制调用到ActivityManagerService中的
   handleApplicationStrictModeViolation（）函数，在这个方法的内部会通过handler将 警告的弹窗显示出来
   synchronized (this) {
       final long origId = Binder.clearCallingIdentity();
       Message msg = Message.obtain();
       msg.what = SHOW_STRICT_MODE_VIOLATION_UI_MSG;
       HashMap<String, Object> data = new HashMap<String, Object>();
       data.put("result", result);
       data.put("app", r);
       data.put("violationMask", violationMask);
       data.put("info", info);
       msg.obj = data;
       mUiHandler.sendMessage(msg);
       Binder.restoreCallingIdentity(origId);
  }
  那他是怎么样将结果告诉我们的呢 ，为什么要hook CloseGuard 的成员呢，发现CloseGuard 中有这样的函数,感觉像是回调结果的的样子，接下来我们在源码中全局搜索
  public void warnIfOpen() {
        if (allocationSite == null || !ENABLED) {
            return;
        }

        String message =
                ("A resource was acquired at attached stack trace but never released. "
                 + "See java.io.Closeable for information on avoiding resource leaks.");

        REPORTER.report(message, allocationSite);
    }
    实现的地方非常多，这里只看一个 比如 ActivityView 中有这样方法
    @Override
    protected void finalize() throws Throwable {
            if (DEBUG) Log.v(TAG, "ActivityContainerWrapper: finalize called");
            try {
                if (mGuard != null) {
                    mGuard.warnIfOpen();
                    release();
                }
            } finally {
                super.finalize();
            }
        }
		
mGuard.warnIfOpen(); 由于我们代理了这个REPORT 所以我们可以获取到结果，接下来看看我们hook之后做的处理
	
public class IOCloseLeakDetector extends IssuePublisher implements InvocationHandler {

    private final Object originalReporter;

    public IOCloseLeakDetector(OnIssueDetectListener issueListener, Object originalReporter) {
        super(issueListener);
        this.originalReporter = originalReporter;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        MatrixLog.i(TAG, "invoke method: %s", method.getName());
        if (method.getName().equals("report")) {
            if (args.length != 2) {
                MatrixLog.e(TAG, "closeGuard report should has 2 params, current: %d", args.length);
                return null;
            }
            if (!(args[1] instanceof Throwable)) {
                MatrixLog.e(TAG, "closeGuard report args 1 should be throwable, current: %s", args[1]);
                return null;
            }
            Throwable throwable = (Throwable) args[1];

            //获取到当前堆栈信息
            String stackKey = IOCanaryUtil.getThrowableStack(throwable);
            if (isPublished(stackKey)) {
                MatrixLog.d(TAG, "close leak issue already published; key:%s", stackKey);
            } else {
				//构建一个Issue
                Issue ioIssue = new Issue(SharePluginInfo.IssueType.ISSUE_IO_CLOSABLE_LEAK);
                ioIssue.setKey(stackKey);
                JSONObject content = new JSONObject();
                try {
                    content.put(SharePluginInfo.ISSUE_FILE_STACK, stackKey);
                } catch (JSONException e) {
//                e.printStackTrace();
                    MatrixLog.e(TAG, "json content error: %s", e);
                }
                //执行结果的公布,这样就能获取到泄漏的内容了,还加上了当前堆栈信息
                ioIssue.setContent(content);
                publishIssue(ioIssue);
                MatrixLog.d(TAG, "close leak issue publish, key:%s", stackKey);
                markPublished(stackKey);
            }
            return null;
        }
        return method.invoke(originalReporter, args);
    }
}
```
### 总结
> 通过native hook的方式，hook了系统中 libopenjdkjvm.so,libjavacore.so,libopenjdk.so 中对应的open，open64，read，write，__read_chk，__write_chk，close函数，通过hook这些函数
我们得到IO操作的信息，比如文件的大小，路径，buff的大小，真正读写内容的大小，单次读写的耗时等
主线程Io的检测，通过在回调执行open函数的时候，获取到java层的堆栈信息，以及当前的线程名，之后在文件关闭的时候，会判断当前的进程id，如果进程的id跟记录下来的线程id一样，说明为主线程
操作Io，然后判断是否满足了主线程操作Io的阈值，这里设置为最小不能小于500毫秒
Buff过小的检测，每次系统回调执行read,write的时候，我们可以知道单次读写的耗时，buff的大小，真正读写的大小，将这些信息收集起来，当文件关闭的时候执行了close函数的时候，将这些结果进行分析
这里的检测标准为 Buff的平均读写大小不应该小于4KB
重复读的检测，这里的标准为 不能小于5次
资源泄漏的检测，通过hook CloseGuard 的REPORT 成员，并且将ENABLED 设置为true，这样我们就能收到严苛模式检测下的结果，我们就能收集当前的java堆栈信息


### 参考资料
1. [Android代码检测优化之StricMode](https://blog.csdn.net/qq_25804863/article/details/48566925)
2. [11 | I/O优化（下）：如何监控线上I/O操作？](https://time.geekbang.org/column/article/0?cid=142)
3. [Matrix 介绍](https://github.com/Tencent/matrix#matrix_cn)







