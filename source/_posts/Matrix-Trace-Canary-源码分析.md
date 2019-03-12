---
layout: pager
title: Matrix Trace Canary 源码分析
date: 2019-03-12 13:58:27
tags: [Android,Matrix]
description:  Matrix Trace Canary 源码分析
---

### 概述

> Matrix Trace Canary 源码分析

<!--more-->

### 简介
> 最近微信开源了一个性能检测工具 Matrix 简称 APM，可以通过各种性能监控方案，对性能监控项的异常数据进行采集和分析，输出相应的问题分析、定位与优化建议，从而帮助开发者开发出更高质量的应用。做为一个Android开发低手的我，订阅了Android 开发高手，这门课确实很牛逼，含金量很高，在Android 开发高手这门课中一部门的内容是关于Matrix 检测的原理，做为一个Android 开发也不应该只局限于使用的地步，我们更加应该去阅读源码，所以接下来的文章会依次的解析Matrix的内容，这篇文章会先解析 Trace Canary


### 源码分析
Matrix github的地址为 
> https://github.com/Tencent/matrix#matrix_cn  

里面还有源码，也有demo，我们就以demo为分析的工程，首先demo一启动就会看到这样的界面
![结果显示](/uploads/Matrix Trace分析/Application 启动分析.png)
这是统计Application 启动的时长，以及第一个界面Splash界面显示的时长等 ,接下来按下返回键可以看到Matrix 包含的四个检测
![结果显示](/uploads/Matrix Trace分析/Matrix 检测分类.png)
第一个条目TRACE_CANARY 就是Trace 包含的内容
![结果显示](/uploads/Matrix Trace分析/Trace 检测的内容.png)
这四个选项包含检测的内容分别为FPS， Activity 进入的时间 ，ANR ，以及耗时的方法,由于 Trace 包含的源码众多，逻辑也是比较复杂，这里只介绍几个重要的类,首先是 ApplicationLifeObserver
```java
public class ApplicationLifeObserver implements Application.ActivityLifecycleCallbacks {
    ...
    private static final long CHECK_DELAY = 600;
    private static ApplicationLifeObserver mInstance;
    private final LinkedList<IObserver> mObservers;
    private final Handler mMainHandler;
    private boolean mIsPaused, mIsForeground;
    ...
    private ApplicationLifeObserver(@NonNull Application application) {
        if (null != application) {
            //注册全局的生命监听
            application.unregisterActivityLifecycleCallbacks(this);
            application.registerActivityLifecycleCallbacks(this);
        }
        mObservers = new LinkedList<>();
        mMainHandler = new Handler(Looper.getMainLooper());
    }

    //初始化
    public static void init(Application application) {
        if (null == mInstance) {
            mInstance = new ApplicationLifeObserver(application);
        }
    }
	
    //添加监听
    public void register(IObserver observer) {
        if (null != mObservers) {
            mObservers.add(observer);
        }
    }

    //注销监听
    public void unregister(IObserver observer) {
        if (null != mObservers) {
            mObservers.remove(observer);
        }
    }
	
    @Override
    public void onActivityCreated(final Activity activity, Bundle savedInstanceState) {
        for (IObserver listener : mObservers) {
            listener.onActivityCreated(activity);
        }
    }

    @Override
    public void onActivityStarted(Activity activity) {
        for (IObserver listener : mObservers) {
            listener.onActivityStarted(activity);
        }
    }
	
    @Override
    public void onActivityResumed(final Activity activity) {
        ...
        //置为false
        mIsPaused = false;
        //判断为前台
        mIsForeground = true;
        ...
    }

    @Override
    public void onActivityPaused(final Activity activity) {
        ...
        //标识当前为暂停状态
        mIsPaused = true;
        ...
        final WeakReference<Activity> mActivityWeakReference = new WeakReference<>(activity);
        //如果 执行了onActivityPaused 600 毫秒之内没有执行 onActivityResumed 那么判定为后台
        mMainHandler.postDelayed(mCheckRunnable = new Runnable() {
            @Override
            public void run() {
                if (mIsForeground && mIsPaused) {
                    mIsForeground = false;
                    Activity ac = mActivityWeakReference.get();
                    if (null == ac) {
                        MatrixLog.w(TAG, "onBackground ac is null!");
                        return;
                    }
                    for (IObserver listener : mObservers) {
                        listener.onBackground(ac);
                    }
                }
            }
        }, CHECK_DELAY);
    }
	
    //判断当前应用是否在前台
    public boolean isForeground() {
        return mIsForeground;
    }
    ...
}

可以看到 ApplicationLifeObserver 是一个单例，首次构建的时候就调用 application.registerActivityLifecycleCallbacks(this) 这样当前应用Activity的生命周期都可以监控到,对于其他地方
需要监听到生命周期的只要注册一个监听就可以了，所以提供了register,unregister 函数， 当收到了生命周期的时候，就相应的执行分发，比如onActivityCreated 函数实现,其实ApplicationLifeObserver
还有一个非常重要的方法就是可以事实的检测到当前应用是否在前台，具体是怎么做到的呢，就是在 onActivityResumed，onActivityPaused中做的处理,比如当回调了onActivityResumed那么当前应用肯定
在前台，当时当回调执行了 onActivityPaused 的时候并没有立刻的将 mIsForeground 改为false，而是通过 Handler 延迟执行 600毫秒检测 onActivityResumed 是否执行了，因为当回调了就会将mIsPaused
置为false，那么延迟handler执行的时候也不会满足条件 if (mIsForeground && mIsPaused) 也就不会将 mIsForeground = false; 这个可能会有点误判，比如当前Activity在启动的时候做了耗时的操作，但是
整体来说并没有多大的影响
```

接着 是函数的统计耗时 MethodBeat
```java
public class MethodBeat implements IMethodBeat, ApplicationLifeObserver.IObserver {
	...
	static {
        Hacker.hackSysHandlerCallback();
        sCurrentDiffTime = sLastDiffTime = System.nanoTime() / Constants.TIME_MILLIS_TO_NANO;
        sReleaseBufferHandler.sendEmptyMessageDelayed(RELEASE_BUFFER_MSG_ID, Constants.DEFAULT_RELEASE_BUFFER_DELAY);
    }
	
	/**
     * hook method when it's called in.  会由  MethodTracer 注入执行 会在方法执行之前执行
     *
     * @param methodId
     */
    public static void i(int methodId) {
        //处于后台的不管
        if (isBackground) {
            return;
        }

        //isRealTrace 默认为false，这里代表为第一次方法的调用
        if (!isRealTrace) {
            updateDiffTime();
            sTimeUpdateHandler.sendEmptyMessage(UPDATE_TIME_MSG_ID);
            sBuffer = new long[Constants.BUFFER_TMP_SIZE];
        }

        isRealTrace = true;

        //如果当前进程为主进程并且初始化完成
        if (isCreated && Thread.currentThread() == sMainThread) {
            if (sIsIn) {
                android.util.Log.e(TAG, "ERROR!!! MethodBeat.i Recursive calls!!!");
                return;
            }

            sIsIn = true;

            //如果当前 方法的索引 超过了 100 * 10000; // 7.6M ，要将map集合中的内容清空 ,通知处理buff的内容
            if (sIndex >= Constants.BUFFER_SIZE) {
                for (IMethodBeatListener listener : sListeners) {
                    listener.pushFullBuffer(0, Constants.BUFFER_SIZE - 1, sBuffer);
                }
                //重置
                sIndex = 0;
            } else {
                //如果还没有达到 100 * 10000; // 7.6M ，更新当前方法的耗时时间
                mergeData(methodId, sIndex, true);
            }
            ++sIndex;
            sIsIn = false;
        } else if (!isCreated && Thread.currentThread() == sMainThread && sBuffer != null) {//当前进程为主进程但是还没有初始化完成
            if (sIsIn) {
                android.util.Log.e(TAG, "ERROR!!! MethodBeat.i Recursive calls!!!");
                return;
            }

            sIsIn = true;


            if (sIndex < Constants.BUFFER_TMP_SIZE) {
                mergeData(methodId, sIndex, true);
                ++sIndex;
            }

            sIsIn = false;
        }


    }

    /**
     * hook method when it's called out.  会由  MethodTracer 注入执行 会在方法执行之后执行
     *
     * @param methodId
     */
    public static void o(int methodId) {
        if (isBackground || null == sBuffer) {
            return;
        }
        if (isCreated && Thread.currentThread() == sMainThread) {
            if (sIndex < Constants.BUFFER_SIZE) {
                mergeData(methodId, sIndex, false);
            } else {
                //已满
                for (IMethodBeatListener listener : sListeners) {
                    listener.pushFullBuffer(0, Constants.BUFFER_SIZE - 1, sBuffer);
                }
                sIndex = 0;
            }
            ++sIndex;
        } else if (!isCreated && Thread.currentThread() == sMainThread) {
            if (sIndex < Constants.BUFFER_TMP_SIZE) {
                mergeData(methodId, sIndex, false);
                ++sIndex;
            }
        }
    }

    /**
     * when the special method calls,it's will be called.  traceWindowFocusChangeMethod 中会触发这个方法
     *
     * @param activity now at which activity
     * @param isFocus  this window if has focus
     */
    public static void at(Activity activity, boolean isFocus) {
        MatrixLog.i(TAG, "[AT] activity: %s, isCreated: %b sListener size: %d，isFocus: %b", activity.getClass().getSimpleName(), isCreated, sListeners.size(), isFocus);
        if (isCreated && Thread.currentThread() == sMainThread) {
            for (IMethodBeatListener listener : sListeners) {
                listener.onActivityEntered(activity, isFocus, sIndex - 1, sBuffer);
            }
        }
    }
    ...
}

首先在静态函数块里面执行  hackSysHandlerCallback() 执行 ActivityThread 的 Handler Callback的动态代理
public static void hackSysHandlerCallback() {
    try {
         //注意这里 当执行 ActivityThread的时候，就标记为 Application创建的开始时间
         sApplicationCreateBeginTime = System.currentTimeMillis();
         sApplicationCreateBeginMethodIndex = MethodBeat.getCurIndex();
		 
         Class<?> forName = Class.forName("android.app.ActivityThread");
         Field field = forName.getDeclaredField("sCurrentActivityThread");
         field.setAccessible(true);
         Object activityThreadValue = field.get(forName);
         Field mH = forName.getDeclaredField("mH");
         mH.setAccessible(true);
         Object handler = mH.get(activityThreadValue);
         Class<?> handlerClass = handler.getClass().getSuperclass();
         Field callbackField = handlerClass.getDeclaredField("mCallback");
         callbackField.setAccessible(true);
         //HackCallback 来替换ActivityThread类里的mCallback，达到偷梁换柱的效果
         Handler.Callback originalCallback = (Handler.Callback) callbackField.get(handler);
         HackCallback callback = new HackCallback(originalCallback);
         callbackField.set(handler, callback);

         MatrixLog.i(TAG, "hook system handler completed. start:%s", sApplicationCreateBeginTime);
     } catch (Exception e) {
         MatrixLog.e(TAG, "hook system handler err! %s", e.getCause().toString());
     }
}

public class HackCallback implements Handler.Callback {
    private static final String TAG = "Matrix.HackCallback";
    private static final int LAUNCH_ACTIVITY = 100;
    private static final int ENTER_ANIMATION_COMPLETE = 149;
    private static final int CREATE_SERVICE = 114;
    private static final int RECEIVER = 113;

    //用于标识Application 是否已经创建, 因为在第一次收到 LAUNCH_ACTIVITY ,CREATE_SERVICE ,RECEIVER 的时候，将这个标识置位true，也就相当于是Application初始化了
    private static boolean isCreated = false;

    private final Handler.Callback mOriginalCallback;

    public HackCallback(Handler.Callback callback) {
        this.mOriginalCallback = callback;
    }

    @Override
    public boolean handleMessage(Message msg) {
//        MatrixLog.i(TAG, "[handleMessage] msg.what:%s begin:%s", msg.what, System.currentTimeMillis());

        //拦截到了LAUNCH_ACTIVITY和ENTER_ANIMATION_COMPLETE消息，这样就知道当前的activity创建到完成的时机。
        if (msg.what == LAUNCH_ACTIVITY) {
            //标识当前Activity 进入动画是否已经完成
            Hacker.isEnterAnimationComplete = false;
        } else if (msg.what == ENTER_ANIMATION_COMPLETE) {
            //标识当前Activity 进入动画已经完成
            Hacker.isEnterAnimationComplete = true;
        }

        //如果当前进程还没有初始化
        if (!isCreated) {
            //如果当前的消息是 创建Activity，创建Service 或者收到了Receiver
            if (msg.what == LAUNCH_ACTIVITY || msg.what == CREATE_SERVICE || msg.what == RECEIVER) {
                //当前进程Application 创建的终止时间点
                Hacker.sApplicationCreateEndTime = System.currentTimeMillis();
                //记录Application 创建成功的时候 对应的在MethodBeat 中对应的索引
                Hacker.sApplicationCreateEndMethodIndex = MethodBeat.getCurIndex();
                Hacker.sApplicationCreateScene = msg.what;
                isCreated = true;
            }
        }
        if (null == mOriginalCallback) {
            return false;
        }
        return mOriginalCallback.handleMessage(msg);
    }
}
所以通过hook Handler的 Callback函数，我们可以知道进程创建的终止时间这里的判断是根据当前应用程序 创建的第一个 Activity,Service,Receiver的时机点做为当前Application创建的终止时间点
并且通过接受 ENTER_ANIMATION_COMPLETE 函数，可以知道当前Activity 是否创建完成，对于类中的 i(),0(),at()函数是哪里调用的呢,这是通过自定义插件的方式来调用的，具体在matrix-gradle-plugin
工程中 通过Transform提供的api可以遍历所有文件，但是要实现Transform的遍历操作，得通过Gradle插件来实现，关于Gradle插件的知识可以看相关博客

class MatrixPlugin implements Plugin<Project> {
    private static final String TAG = "Matrix.MatrixPlugin"

    @Override
    void apply(Project project) {
        //就是build.gradle中 创建一个 matrix {} 配置,支持的配置选项 为 SystraceExtension 中字段
        project.extensions.create("matrix", MatrixExtension)
        //继续在 matrix{} 节点之下配置 trace{} ,支持的配置选项 为 MatrixTraceExtension 中的字段
        project.matrix.extensions.create("trace", MatrixTraceExtension)
        //继续在 matrix{} 节点之下配置 removeUnusedResources{} ，支持的配置选项 为 MatrixDelUnusedResConfiguration
        project.matrix.extensions.create("removeUnusedResources", MatrixDelUnusedResConfiguration)

        if (!project.plugins.hasPlugin('com.android.application')) {
            throw new GradleException('Matrix Plugin, Android Application plugin required')
        }

        project.afterEvaluate {
            //也即是找到  build.gradle中android{...}这一块
            def android = project.extensions.android
            //获取到 matrix {} 配置
            def configuration = project.matrix
            android.applicationVariants.all { variant ->

                //检查matrix中 trace 选项的内容 enable的值设置
                Log.i(TAG, "Trace enable is %s", configuration.trace.enable)
                //如果设置为允许
                if (configuration.trace.enable) {
                    //获取到 matrix 中 output 是否有值设置
                    String output = configuration.output
                    //如果没有，配置一个默认的输出路径
                    if (Util.isNullOrNil(output)) {
                        configuration.output = project.getBuildDir().getAbsolutePath() + File.separator + "matrix_output"
                        Log.i(TAG, "set matrix output file to " + configuration.output)
                    }
                    //执行注入
                    com.tencent.matrix.plugin.transform.MatrixTraceTransform.inject(project, variant)
                }
                ...
            }
        }
    }
}

class MatrixExtension {
    String clientVersion
    String uuid
    String output

    MatrixExtension() {
        clientVersion = ""
        uuid = ""
        output = ""
    }
    ...
}
class MatrixTraceExtension {
    String customDexTransformName
    //是否允许
    boolean enable
    //基本要处理的文件路径
    String baseMethodMapFile
    //黑名单配置
    String blackListFile

    MatrixTraceExtension() {
        customDexTransformName = ""
        enable = true
        baseMethodMapFile = ""
        blackListFile = ""
    }
    ...
}

而且demo 的配置是这样的
apply plugin: 'com.tencent.matrix-plugin'
matrix {
    trace {
        enable = true
        baseMethodMapFile = "${project.buildDir}/matrix_output/Debug.methodmap"
        blackListFile = "${project.projectDir}/matrixTrace/blackMethodList.txt"
    }
}
```
在demo中输出路径是这样的
![结果显示](/uploads/Matrix Trace分析/动态插件配置输出路径.png)
接着看Transform 类 ，其中最重要的就是transform 函数
```java
public class SystemTraceTransform extends BaseProxyTransform {

    Transform origTransform
    Project project
    def variant
    ...
 //transform 中一定要将那些修改过后的文件，输出到指定的位置，要不然会出现异常
    @Override
    public void transform(TransformInvocation transformInvocation) throws TransformException, InterruptedException, IOException {
        long start = System.currentTimeMillis()
        final boolean isIncremental = transformInvocation.isIncremental() && this.isIncremental()
        //设置输出的根目录
        final File rootOutput = new File(project.systrace.output, "classes/${getName()}/")
        if (!rootOutput.exists()) {
            rootOutput.mkdirs()
        }

        //根据build.gradle 中的 systrace 选项 构建一个 TraceBuildConfig 对象
        final TraceBuildConfig traceConfig = initConfig()
        Log.i("Systrace." + getName(), "[transform] isIncremental:%s rootOutput:%s", isIncremental, rootOutput.getAbsolutePath())

        final MappingCollector mappingCollector = new MappingCollector()
        //读取 Debug.methodmap 文件的内容
        File mappingFile = new File(traceConfig.getMappingPath());
        if (mappingFile.exists() && mappingFile.isFile()) {
            //完成解析，将解析之后的内容 存储到 MappingCollector 中
            MappingReader mappingReader = new MappingReader(mappingFile);
            mappingReader.read(mappingCollector)
        }


        // transformInvocation.inputs 可以代表俩种类型，一个是目录文件，一个是jar目录
        Map<File, File> jarInputMap = new HashMap<>()
        Map<File, File> scrInputMap = new HashMap<>()

        transformInvocation.inputs.each { TransformInput input ->
            input.directoryInputs.each { DirectoryInput dirInput ->
                collectAndIdentifyDir(scrInputMap, dirInput, rootOutput, isIncremental)
            }

            //如果是jar目录 ，对于jar只是单纯的写出去
            input.jarInputs.each { JarInput jarInput ->
                if (jarInput.getStatus() != Status.REMOVED) {
                    collectAndIdentifyJar(jarInputMap, scrInputMap, jarInput, rootOutput, isIncremental)
                }
            }
        }

        //构建一个MethodCollector 对象,将当前得到的要编译的集合跟当前我们配置的集合，做过滤处理
        MethodCollector methodCollector = new MethodCollector(traceConfig, mappingCollector)
        HashMap<String, TraceMethod> collectedMethodMap = methodCollector.collect(scrInputMap.keySet().toList(), jarInputMap.keySet().toList())

        //执行class的字节码的注入
        MethodTracer methodTracer = new MethodTracer(traceConfig, collectedMethodMap, methodCollector.getCollectedClassExtendMap())
        methodTracer.trace(scrInputMap, jarInputMap)

        origTransform.transform(transformInvocation)
        Log.i("Systrace." + getName(), "[transform] cost time: %dms", System.currentTimeMillis() - start)
    }
    ...
}
Debug.methodmap 文件的内容长的类似这样
-1,1,sample.tencent.matrix.resource.TestLeakActivity onWindowFocusChanged (Z)V
-1,1,sample.tencent.matrix.trace.TestEnterActivity onWindowFocusChanged (Z)V
-1,1,com.tencent.sqlitelint.behaviour.alert.CheckResultActivity onWindowFocusChanged (Z)V
-1,1,sample.tencent.matrix.MainActivity onWindowFocusChanged (Z)V
-1,1,sample.tencent.matrix.io.TestIOActivity onWindowFocusChanged (Z)V
-1,1,sample.tencent.matrix.issue.IssuesListActivity onWindowFocusChanged (Z)V
-1,1,com.tencent.sqlitelint.behaviour.alert.SQLiteLintBaseActivity onWindowFocusChanged (Z)V
-1,1,sample.tencent.matrix.trace.TestTraceMainActivity onWindowFocusChanged (Z)V
-1,1,sample.tencent.matrix.sqlitelint.TestSQLiteLintActivity onWindowFocusChanged (Z)V
...

将这些内容解析完之后，接着 构建一个 MethodCollector 对象，调用collect 函数过滤我们添加的黑名单设置
//做过滤处理
public HashMap collect(List<File> srcFolderList, List<File> dependencyJarList) {
   // 解析我们的黑名单文件，读取里面的内容到 mBlackPackageMap 集合中
   mTraceConfig.parseBlackFile(mMappingCollector);

   //读取methodmap文件
   File originMethodMapFile = new File(mTraceConfig.getBaseMethodMap());
   getMethodFromBaseMethod(originMethodMapFile);

   Log.i(TAG, "[collect] %s method from %s", mCollectedMethodMap.size(), mTraceConfig.getBaseMethodMap());
   retraceMethodMap(mMappingCollector, mCollectedMethodMap);

   collectMethodFromSrc(srcFolderList, true);
   collectMethodFromJar(dependencyJarList, true);
   collectMethodFromSrc(srcFolderList, false);
   collectMethodFromJar(dependencyJarList, false);
   Log.i(TAG, "[collect] incrementCount:%s ignoreMethodCount:%s", mIncrementCount, mIgnoreCount);

   //将当前的收集的Method集合，输出到指定的位置
   saveCollectedMethod(mMappingCollector);
   saveIgnoreCollectedMethod(mMappingCollector);
   return mCollectedMethodMap;
}

之后执行字节码的注入，这里字节码的注入采用的是ASM，ASM 可以直接产生二进制的class 文件，也可以在增强既有类的功能。Java class 被存储在严格格式定义的 .class文件里，
这些类文件拥有足够的元数据来解析类中的所有元素：类名称、方法、属性以及 Java 字节码（指令）。

具体使用ASM
ASM框架中的核心类有以下几个：
ClassReader：该类用来解析编译过的class字节码文件。
ClassWriter：该类用来重新构建编译后的类，比如说修改类名、属性以及方法，甚至可以生成新的类的字节码文件。
ClassVisitor：主要负责 “拜访” 类成员信息。其中包括标记在类上的注解，类的构造方法，类的字段，类的方法，静态代码块。
AdviceAdapter：实现了MethodVisitor接口，主要负责 “拜访” 方法的信息，用来进行具体的方法字节码操作。

调用 methodTracer.trace(scrInputMap, jarInputMap) 函数
public class MethodTracer {
	...
	public void trace(Map<File, File> srcFolderList, Map<File, File> dependencyJarList) {
		traceMethodFromSrc(srcFolderList);
		traceMethodFromJar(dependencyJarList);
	}
	private void traceMethodFromSrc(Map<File, File> srcMap) {
		if (null != srcMap) {
			for (Map.Entry<File, File> entry : srcMap.entrySet()) {
				innerTraceMethodFromSrc(entry.getKey(), entry.getValue());
			}
		}
	}

    /**
     * ASM框架中的核心类有以下几个：
     *
     * ClassReader：该类用来解析编译过的class字节码文件。
     * ClassWriter：该类用来重新构建编译后的类，比如说修改类名、属性以及方法，甚至可以生成新的类的字节码文件。
     * ClassVisitor：主要负责 “拜访” 类成员信息。其中包括标记在类上的注解，类的构造方法，类的字段，类的方法，静态代码块。
     * AdviceAdapter：实现了MethodVisitor接口，主要负责 “拜访” 方法的信息，用来进行具体的方法字节码操作。
     */
    private void innerTraceMethodFromSrc(File input, File output) {

        ArrayList<File> classFileList = new ArrayList<>();
        if (input.isDirectory()) {
            listClassFiles(classFileList, input);
        } else {
            classFileList.add(input);
        }

        for (File classFile : classFileList) {
            InputStream is = null;
            FileOutputStream os = null;
            try {
                final String changedFileInputFullPath = classFile.getAbsolutePath();
                final File changedFileOutput = new File(changedFileInputFullPath.replace(input.getAbsolutePath(), output.getAbsolutePath()));
                if (!changedFileOutput.exists()) {
                    changedFileOutput.getParentFile().mkdirs();
                }
                changedFileOutput.createNewFile();

                if (mTraceConfig.isNeedTraceClass(classFile.getName())) {
                    is = new FileInputStream(classFile);
                    //该类用来解析编译过的class 字节码文件
                    ClassReader classReader = new ClassReader(is);
                    //该类用来重新构建编译后的类
                    ClassWriter classWriter = new ClassWriter(ClassWriter.COMPUTE_MAXS);
                    //主要负责 拜访 类成员信息
                    ClassVisitor classVisitor = new TraceClassAdapter(Opcodes.ASM5, classWriter);
                    classReader.accept(classVisitor, ClassReader.EXPAND_FRAMES);
                    is.close();

                    if (output.isDirectory()) {
                        os = new FileOutputStream(changedFileOutput);
                    } else {
                        os = new FileOutputStream(output);
                    }
                    os.write(classWriter.toByteArray());
                    os.close();
                } else {
                    FileUtil.copyFileUsingStream(classFile, changedFileOutput);
                }
            } catch (Exception e) {
                e.printStackTrace();
            } finally {
                try {
                    is.close();
                    os.close();
                } catch (Exception e) {
                    // ignore
                }
            }
        }
    }
    ...
}

private class TraceClassAdapter extends ClassVisitor {
        //当前访问的类名
        private String className;
        //标识当前访问的方法 是否为抽象的方法或者接口
        private boolean isABSClass = false;
        //标识当前访问的类是否为 MethodBeat 类
        private boolean isMethodBeatClass = false;

        private boolean hasWindowFocusMethod = false;

        TraceClassAdapter(int i, ClassVisitor classVisitor) {
            super(i, classVisitor);
        }

        @Override
        public void visit(int version, int access, String name, String signature, String superName, String[] interfaces) {
            super.visit(version, access, name, signature, superName, interfaces);
            this.className = name;

            //过滤掉当前访问的是抽象的方法，或者是接口
            if ((access & Opcodes.ACC_ABSTRACT) > 0 || (access & Opcodes.ACC_INTERFACE) > 0) {
                this.isABSClass = true;
            }

            //判断当前访问的 class 是否为 MethodBeat 类
            if (mTraceConfig.isMethodBeatClass(className, mCollectedClassExtendMap)) {
                isMethodBeatClass = true;
            }
        }

        @Override
        public MethodVisitor visitMethod(int access, String name, String desc, String signature, String[] exceptions) {
            //如果是抽象方法，或者接口不作处理
            if (isABSClass) {
                return super.visitMethod(access, name, desc, signature, exceptions);
            } else {

                //判断当前访问的类是否为 Activity 中的 onWindowFocusChange 方法
                if (!hasWindowFocusMethod) {
                    hasWindowFocusMethod = mTraceConfig.isWindowFocusChangeMethod(name, desc);
                }

                // 拿到需要修改的方法，执行修改操作
                MethodVisitor methodVisitor = cv.visitMethod(access, name, desc, signature, exceptions);
                return new TraceMethodAdapter(api, methodVisitor, access, name, desc, this.className, hasWindowFocusMethod, isMethodBeatClass);
            }
        }


        @Override
        public void visitEnd() {
            //public final static String MATRIX_TRACE_ON_WINDOW_FOCUS_METHOD = "onWindowFocusChanged";   MATRIX_TRACE_ON_WINDOW_FOCUS_METHOD_ARGS = "(Z)V";
            //public void onWindowFocusChanged(boolean hasFocus) 这里也即是有创建一个 Activity 中的 onWindowFocusChanged 方法，手动的调用他们
            TraceMethod traceMethod = TraceMethod.create(-1, Opcodes.ACC_PUBLIC, className, TraceBuildConstants.MATRIX_TRACE_ON_WINDOW_FOCUS_METHOD, TraceBuildConstants.MATRIX_TRACE_ON_WINDOW_FOCUS_METHOD_ARGS);
           
            //如果当前访问的类是Activity 的类型,并且这个方法 在mCollectedMethodMap 中可以找到，系统也还没有调用 onWindowFocusChange 方法
            if (!hasWindowFocusMethod && mTraceConfig.isActivityOrSubClass(className, mCollectedClassExtendMap) && mCollectedMethodMap.containsKey(traceMethod.getMethodName())) {
                //往 WindowFocusChange 函数中，插入一个 MethodBeat 的at 方法，
                insertWindowFocusChangeMethod(cv);
            }
            super.visitEnd();
        }
}

开始访问的时候，判断当前的类是否为 MethodBeat 类,如果是使用  isMethodBeatClass 标记，接着访问函数的时候，抽象函数过滤掉,接着通过 mTraceConfig.isWindowFocusChangeMethod(name, desc)
来判断当前访问的是否为 Activity 中的 onWindowFocusChanged 函数，如果是将 hasWindowFocusMethod 标记为 true,接着将这些参数 传递到 TraceMethodAdapter 中,进一步的访问方法

private class TraceMethodAdapter extends AdviceAdapter {

        //当前的方法名
        private final String methodName;
        private final String name;
        //当前方法对应的类名
        private final String className;
        //标识当前是否已经执行过了 onWindowFocusChange 方法
        private final boolean hasWindowFocusMethod;
        //标识当前是否为 MethodBeat 类
        private final boolean isMethodBeatClass;

        protected TraceMethodAdapter(int api, MethodVisitor mv, int access, String name, String desc, String className, boolean hasWindowFocusMethod, boolean isMethodBeatClass) {
            super(api, mv, access, name, desc);
            TraceMethod traceMethod = TraceMethod.create(0, access, className, name, desc);
            this.methodName = traceMethod.getMethodName();
            this.isMethodBeatClass = isMethodBeatClass;
            this.hasWindowFocusMethod = hasWindowFocusMethod;
            this.className = className;
            this.name = name;
        }

        @Override
        protected void onMethodEnter() {
            //进入方法时可以插入字节码
            TraceMethod traceMethod = mCollectedMethodMap.get(methodName);
            if (traceMethod != null) {
                traceMethodCount.incrementAndGet();
                mv.visitLdcInsn(traceMethod.id);
                //访问 MethodBeat 中的 i方法，统计方法的执行开始
                mv.visitMethodInsn(INVOKESTATIC, TraceBuildConstants.MATRIX_TRACE_CLASS, "i", "(I)V", false);
            }
        }

        @Override
        protected void onMethodExit(int opcode) {
            //退出方法前可以插入字节码
            TraceMethod traceMethod = mCollectedMethodMap.get(methodName);
            if (traceMethod != null) {
                //
                if (hasWindowFocusMethod && mTraceConfig.isActivityOrSubClass(className, mCollectedClassExtendMap) && mCollectedMethodMap.containsKey(traceMethod.getMethodName())) {
                    //构建一个 onWindowFocusChange 的Method
                    TraceMethod windowFocusChangeMethod = TraceMethod.create(-1, Opcodes.ACC_PUBLIC, className, TraceBuildConstants.MATRIX_TRACE_ON_WINDOW_FOCUS_METHOD, TraceBuildConstants.MATRIX_TRACE_ON_WINDOW_FOCUS_METHOD_ARGS);
                    if (windowFocusChangeMethod.equals(traceMethod)) {
                        traceWindowFocusChangeMethod(mv);
                    }
                }
                traceMethodCount.incrementAndGet();
                mv.visitLdcInsn(traceMethod.id);
                //这里也即是调用 MethodBeat 中的 o 方法，统计这个方法的结束
                mv.visitMethodInsn(INVOKESTATIC, TraceBuildConstants.MATRIX_TRACE_CLASS, "o", "(I)V", false);
            }
        }
}

在函数 onMethodEnter 中有这样的代码 
mv.visitMethodInsn(INVOKESTATIC, TraceBuildConstants.MATRIX_TRACE_CLASS, "i", "(I)V", false);,其中MATRIX_TRACE_CLASS 为 
public final static String MATRIX_TRACE_CLASS = "com/tencent/matrix/trace/core/MethodBeat";
所以当函数执行的时候，就会先去执行 MethodBeat中的 i()函数

同理当函数退出的时候 就会执行 mv.visitMethodInsn(INVOKESTATIC, TraceBuildConstants.MATRIX_TRACE_CLASS, "o", "(I)V", false); 对应的在MethodBeat中的o()函数
这里还有一个用来统计onWindowFocusChange 的函数，这里退出的时候，hasWindowFocusMethod 代表当前函数为 onWindowFocusChange,所以这里会调用 traceWindowFocusChangeMethod()
private void traceWindowFocusChangeMethod(MethodVisitor mv) {
   mv.visitVarInsn(Opcodes.ALOAD, 0);
   mv.visitVarInsn(Opcodes.ILOAD, 1);
   mv.visitMethodInsn(Opcodes.INVOKESTATIC, TraceBuildConstants.MATRIX_TRACE_CLASS, "at", "(Landroid/app/Activity;Z)V", false);
}
也即是如果当前为onWindowFocusChange 函数，这个函数执行完之后，会调用到MethodBeat 中的at 函数,也就能知道Activity启动的时间了

继续回到MethodBeat 中的o()函数
 public static void o(int methodId) {
        if (isBackground || null == sBuffer) {
            return;
        }
        if (isCreated && Thread.currentThread() == sMainThread) {
            if (sIndex < Constants.BUFFER_SIZE) {
                mergeData(methodId, sIndex, false);
            } else {
                //已满,通知处理buff的内容
                for (IMethodBeatListener listener : sListeners) {
                    listener.pushFullBuffer(0, Constants.BUFFER_SIZE - 1, sBuffer);
                }
                sIndex = 0;
            }
            ++sIndex;
        } else if (!isCreated && Thread.currentThread() == sMainThread) {
			//buff还没有满，还可以继续的存储
            if (sIndex < Constants.BUFFER_TMP_SIZE) {
                mergeData(methodId, sIndex, false);
                ++sIndex;
            }
        }
}
	
 private static void mergeData(int methodId, int index, boolean isIn) {
        long trueId = 0L;
        if (isIn) {
            trueId |= 1L << 63;
        }
        trueId |= (long) methodId << 43;
        trueId |= sCurrentDiffTime & 0x7FFFFFFFFFFL;
        sBuffer[index] = trueId;
}

buff中存储的为 方法执行的耗时时间统计，统计是通过 mergeData函数来执行的，同时如果当前buff已经满的时候，会通过 listener.pushFullBuffer()通知处理buff的内容
```
接下来分析 FrameBeat
```java
public final class FrameBeat implements IFrameBeat, Choreographer.FrameCallback, ApplicationLifeObserver.IObserver {
    private static FrameBeat mInstance;
    private final LinkedList<IFrameBeatListener> mFrameListeners;
    private Choreographer mChoreographer;
    //标识是否创建完成
    private boolean isCreated;
    //标识是否暂停
    private volatile boolean isPause = true;
    //上一次收到 界面更新信号的时间
    private long mLastFrameNanos;

    private FrameBeat() {
        mFrameListeners = new LinkedList<>();
    }

    public static FrameBeat getInstance() {
        if (null == mInstance) {
            mInstance = new FrameBeat();
        }
        return mInstance;
    }

    //当前是否处于暂停的状态
    public boolean isPause() {
        return isPause;
    }

    //暂停的时候，移除frameCallback 监听
    public void pause() {
        if (!isCreated) {
            return;
        }

        isPause = true;
        if (null != mChoreographer) {
            mChoreographer.removeFrameCallback(this);
            mLastFrameNanos = 0;
            for (IFrameBeatListener listener : mFrameListeners) {
                listener.cancelFrame();
            }
        }
    }

    //设置
    public void resume() {
        if (!isCreated) {
            return;
        }
        isPause = false;
        //注册 frameCallback监听
        if (null != mChoreographer) {
            mChoreographer.removeFrameCallback(this);
            mChoreographer.postFrameCallback(this);
            mLastFrameNanos = 0;
        }
    }

    @Override
    public void onCreate() {
        if (!MatrixUtil.isInMainThread(Thread.currentThread().getId())) {
            MatrixLog.e(TAG, "[onCreate] FrameBeat must create on main thread");
            return;
        }

        MatrixLog.i(TAG, "[onCreate] FrameBeat real onCreate!");
        if (!isCreated) {
            isCreated = true;
            //注册生命周期监听
            ApplicationLifeObserver.getInstance().register(this);
            mChoreographer = Choreographer.getInstance();
            //判断当前应用程序是否为前台
            if (ApplicationLifeObserver.getInstance().isForeground()) {
                //如果处于前台，设置 frameCallback
                resume();
            }
        } else {
            MatrixLog.w(TAG, "[onCreate] FrameBeat is created!");
        }
    }


    @Override
    public void onDestroy() {
        //销毁的操作
        if (isCreated) {
            isCreated = false;
            if (null != mChoreographer) {
                mChoreographer.removeFrameCallback(this);
                //销毁的时候，执行cancelFrame的方法
                for (IFrameBeatListener listener : mFrameListeners) {
                    listener.cancelFrame();
                }
            }
            mChoreographer = null;
            if (null != mFrameListeners) {
                mFrameListeners.clear();
            }
            ApplicationLifeObserver.getInstance().unregister(this);
        } else {
            MatrixLog.w(TAG, "[onDestroy] FrameBeat is not created!");
        }

    }

    //添加listener的监听，用于分发 doFrame
    @Override
    public void addListener(IFrameBeatListener listener) {
        if (null != mFrameListeners && !mFrameListeners.contains(listener)) {
            mFrameListeners.add(listener);
            if (isPause()) {
                resume();
            }
        }
    }

    //移除监听
    @Override
    public void removeListener(IFrameBeatListener listener) {
        if (null != mFrameListeners) {
            mFrameListeners.remove(listener);
            if (mFrameListeners.isEmpty()) {
                pause();
            }
        }
    }

    /**
     * when the device's Vsync is coming,it will be called.   当收到了 界面跟新信号的时候，会触发这个函数
     *
     * @param frameTimeNanos The time in nanoseconds when the frame started being rendered.
     */
    @Override
    public void doFrame(long frameTimeNanos) {
        //如果当前处于暂停的状态，直接返回
        if (isPause) {
            return;
        }

        //当前 界面更新的时间 小于上次界面更新的时间,不用分析，因为 界面卡顿
        //doFrame方法里可以收到回调的当前时间，正常绘制两次doFrame的时间差应该是1000/60=16.6666毫秒（每秒60帧），但是遇到卡顿或过度重绘等会导致时间拉长。
        if (frameTimeNanos < mLastFrameNanos || mLastFrameNanos <= 0) {
            //更新上一次的值
            mLastFrameNanos = frameTimeNanos;
            //继续添加回调
            if (null != mChoreographer) {
                mChoreographer.postFrameCallback(this);
            }
            //直接返回
            return;
        }


        //如果到了这里说明当前执行 doFrame的时间 大于上一次界面刷新的时间，这里要判断是否卡顿
        if (null != mFrameListeners) {

            //将结果转出去
            for (IFrameBeatListener listener : mFrameListeners) {
                listener.doFrame(mLastFrameNanos, frameTimeNanos);
            }

            //继续设置回调
            if (null != mChoreographer) {
                mChoreographer.postFrameCallback(this);
            }
            mLastFrameNanos = frameTimeNanos;
        }

    }


    @Override
    public void onFront(Activity activity) {
        MatrixLog.i(TAG, "[onFront] isCreated:%s postFrameCallback", isCreated);
        //当处于前台的时候，添加frameCallback 监听
        resume();
    }

    @Override
    public void onBackground(Activity activity) {
        MatrixLog.i(TAG, "[onBackground] isCreated:%s removeFrameCallback", isCreated);
        //当处于后台的时候，移除 frameCallback 监听
        pause();
    }
    ...
}

可以看出这个类是单例的形势，本身通过  ApplicationLifeObserver.getInstance().register(this); 监听了生命周期的回调，这样当应用处于前台的时候调用 resume();函数
执行了mChoreographer.postFrameCallback(this); 这样我们就可以接受到界面刷新的回调，默认是16毫秒就要执行一次，所以我们通过监听这个函数，我们可以知道是否卡顿,可以知道丢帧率
同时提供了addListener,removeListener函数，这样其他的地方需要这个回调的是就能轻松获取到 ，并且在 doFrame 函数中，判断当前收到帧的时间如果大于上一帧的时候，将这个结果转出去,之后
继续设置这个监听 mChoreographer.postFrameCallback(this);， 当应用在后台的时候，就会调用 pause(); 函数，移除 这个监听  mChoreographer.removeFrameCallback(this);
```
差不多关键的类已经分析完了，下面分析下对应的函数检测
```java

首先查看耗时函数的检测,前面分析到 当收集的耗时函数在buff中满的时候，会通过 listener.pushFullBuffer()通知处理buff的内容,这个函数的具体实现在 EvilMethodTracer 中

/**
* when the buffer is full,this method will be called.  buff 已经存储满了
*
* @param start
* @param end
* @param buffer
*/
@Override
public void pushFullBuffer(int start, int end, long[] buffer) {
    long now = System.nanoTime() / Constants.TIME_MILLIS_TO_NANO - getMethodBeat().getLastDiffTime();
    MatrixLog.w(TAG, "[pushFullBuffer] now:%s diffTime:%s", now, getMethodBeat().getLastDiffTime());
    setIgnoreFrame(true);
    getMethodBeat().lockBuffer(false);

    //分析buff 中存储的内容
    handleBuffer(Type.FULL, start, end, buffer, now - (buffer[0] & 0x7FFFFFFFFFFL));
    mLazyScheduler.cancel();
}

同时 EvilMethodTracer 可以用来检测ANR,在构造函数的时候会构造一个 LazyScheduler 对象
public EvilMethodTracer(TracePlugin plugin, TraceConfig config) {
     super(plugin);
     this.mTraceConfig = config;

     //创建一个定时器，这个定时器用来检查ANR的情况  , public static final int DEFAULT_ANR = 5 * 1000; 默认ANR为 5秒
     mLazyScheduler = new LazyScheduler(MatrixHandlerThread.getDefaultHandlerThread(), Constants.DEFAULT_ANR);
     mActivityCreatedInfoMap = new HashMap<>();
}

注意这里的 getDefaultHandlerThread() 获取的是HandlerThread的Looper
public static HandlerThread getDefaultHandlerThread() {
    synchronized (MatrixHandlerThread.class) {
        //如果为空，创建一个 HandlerThread ,并且对应的创建一个 defaultHandler
        if (null == defaultHandlerThread) {
             defaultHandlerThread = new HandlerThread(MATRIX_THREAD_NAME);
             defaultHandlerThread.start();
             defaultHandler = new Handler(defaultHandlerThread.getLooper());
             MatrixLog.w(TAG, "create default handler thread, we should use these thread normal");
        }
        return defaultHandlerThread;
    }
}
public LazyScheduler(HandlerThread handlerThread, long delay) {
     this.delay = delay;
     mHandler = new Handler(handlerThread.getLooper());
}
所以LazyScheduler 拥有的Looper对象为子线程中的Looper对象，这样就可以用来执行耗时的操作，而不会阻塞主线程，由于EvilMethodTracer 在父类的OnCreate函数中，执行了
FrameBeat.getInstance().addListener(this);，所以可以接受 doFrame() 的回调

//也即是当前的帧刷新的时间大于上次帧的时间，这里要分析
@Override
public void doFrame(long lastFrameNanos, long frameNanos) {
    if (isIgnoreFrame) {
          mActivityCreatedInfoMap.clear();
          setIgnoreFrame(false);
          getMethodBeat().resetIndex();
          return;
     }
     int index = getMethodBeat().getCurIndex();

     //doFrame时间差超过1秒会执行handleBuffer，时间差超过5秒会执行mLazyScheduler里的onTimeExpire。先来看onTimeExpire方法：
     if (hasEntered && frameNanos - lastFrameNanos > mTraceConfig.getEvilThresholdNano()) {
          MatrixLog.e(TAG, "[doFrame] dropped frame too much! lastIndex:%s index:%s", 0, index);
          handleBuffer(Type.NORMAL, 0, index - 1, getMethodBeat().getBuffer(), (frameNanos - lastFrameNanos) / Constants.TIME_MILLIS_TO_NANO);
     }

     getMethodBeat().resetIndex();
     //当前收到了 那说明当前不可能产生ANR，那就取消，再次的设置ANR的响应
     mLazyScheduler.cancel();
	 //启动这个检测的任务
     mLazyScheduler.setUp(this, false);
}

这里会调用  mLazyScheduler.setUp(this, false);，将这个任务检测ANR的任务启动起来
public void setUp(final ILazyTask task, boolean cycle) {
    if (null != mHandler) {
         this.isSetUp = true;
         mHandler.removeCallbacksAndMessages(null);
         RetryRunnable retryRunnable = new RetryRunnable(mHandler, delay, task, cycle);
         mHandler.postDelayed(retryRunnable, delay);
   }
}

这里会通过 mHandler.postDelayed(retryRunnable, delay);的方式执行RetryRunnable ，注意这里的 delay设置为5秒
static class RetryRunnable implements Runnable {
        private final Handler handler;
        private final long delay;
        private final ILazyTask lazyTask;
        private final boolean cycle;

        RetryRunnable(Handler handler, long delay, ILazyTask lazyTask, boolean cycle) {
            this.handler = handler;
            this.delay = delay;
            this.lazyTask = lazyTask;
            this.cycle = cycle;
        }

        @Override
        public void run() {
            //提示告知时间到了
            lazyTask.onTimeExpire();
            //再次发出消息，定时器的作用
            if (cycle) {
                handler.postDelayed(this, delay);
            }
        }
}

所以根据上面doFrame 的逻辑，如果下一帧的界面刷新5秒钟之内到达的化，就会调用  mLazyScheduler.cancel(); 取消这个任务，如果没有取消，那么就说明了主线程阻塞了五秒发生了ANR
这里发生ANR的时候，会执行 lazyTask.onTimeExpire(); 而 EvilMethodTracer 实现了这个接口

 /**
 * when the ANR happened,this method will be called.  收到了ANR的情况,默认是五秒
 */
@Override
public void onTimeExpire() {
   // maybe ANR
   if (isBackground()) {
        MatrixLog.w(TAG, "[onTimeExpire] pass this time, on Background!");
        return;
   }
   long happenedAnrTime = getMethodBeat().getCurrentDiffTime();
   MatrixLog.w(TAG, "[onTimeExpire] maybe ANR!");
   //过滤掉一帧的数据
   setIgnoreFrame(true);
   getMethodBeat().lockBuffer(false);

   //分析当前ANR的数据
   handleBuffer(Type.ANR, 0, getMethodBeat().getCurIndex() - 1, getMethodBeat().getBuffer(), null, Constants.DEFAULT_ANR, happenedAnrTime, -1);
}

同时 EvilMethodTracer 还可以用来分析 启动Activity的耗时，EvilMethodTracer 通过 父类 执行 ApplicationLifeObserver.getInstance().register(this);从而可以接受到生命周期的回调
所以可以在 onActivityCreated的时候，将当前Activity 创建的信息，时间保存起来存储到 mActivityCreatedInfoMap 中

@Override
public void onActivityCreated(Activity activity) {
   MatrixLog.i(TAG, "[onActivityCreated] activity:%s hashCode:%s", activity.getClass().getSimpleName(), activity.hashCode());
   super.onActivityCreated(activity);
   getMethodBeat().lockBuffer(true);
   hasEntered = false;
   //构建一个 ActivityInfo 对象，添加到
   mActivityCreatedInfoMap.put(activity.hashCode(), new ActivityCreatedInfo(System.currentTimeMillis(), Math.max(0, getMethodBeat().getCurIndex() - 1)));
}

前面我们分析自定义插件的时候知道，当检测的函数为 onWindowFocusChange 的时候，会在调用结束的时候，调用 MethodBeat 中的at 函数，即下面的函数
/**
* when the special method calls,it's will be called.  traceWindowFocusChangeMethod 中会触发这个方法
*
* @param activity now at which activity
* @param isFocus  this window if has focus
*/
public static void at(Activity activity, boolean isFocus) {
    MatrixLog.i(TAG, "[AT] activity: %s, isCreated: %b sListener size: %d，isFocus: %b", activity.getClass().getSimpleName(), isCreated, sListeners.size(), isFocus);
    if (isCreated && Thread.currentThread() == sMainThread) {
        for (IMethodBeatListener listener : sListeners) {
             listener.onActivityEntered(activity, isFocus, sIndex - 1, sBuffer);
        }
    }
}

而我们的EvilMethodTracer 通过重写了 下面的函数，从而通过父类执行的耗时函数的监听, 所以可以接受到 onActivityEntered 的回调
//复写父类的方法，告知要监听耗时的函数
@Override
protected boolean isEnableMethodBeat() {
    return true;
}

/**
 * when the activity's onWindowFocusChanged method exec,it will be called.  当Activity的 onWindowFocusChanged 执行的时候，就会执行这个方法
 *
 * @param activity now activity
 * @param isFocus  whether this activity has focus
 * @param nowIndex the index of buffer when this method was called
 * @param buffer   trace buffer
 */
@Override
public void onActivityEntered(Activity activity, boolean isFocus, int nowIndex, long[] buffer) {
    MatrixLog.i(TAG, "[onActivityEntered] activity:%s hashCode:%s isFocus:%s nowIndex:%s", activity.getClass().getSimpleName(), activity.hashCode(), isFocus, nowIndex);
    if (isFocus && mActivityCreatedInfoMap.containsKey(activity.hashCode())) {
         long now = System.currentTimeMillis();
         ActivityCreatedInfo createdInfo = mActivityCreatedInfoMap.get(activity.hashCode());
         //判断当前Activity 消耗的时间
         long cost = now - createdInfo.startTimestamp;
         MatrixLog.i(TAG, "[activity load time] activity name:%s cost:%sms", activity.getClass(), cost);

         if (cost >= mTraceConfig.getLoadActivityThresholdMs()) {
             //收集当前Activity 显示的View的深度，个数等
             ViewUtil.ViewInfo viewInfo = ViewUtil.dumpViewInfo(activity.getWindow().getDecorView());
             viewInfo.mActivityName = activity.getClass().getSimpleName();
             MatrixLog.w(TAG, "[onActivityEntered] type:%s cost:%sms index:[%s-%s] viewInfo:%s", viewInfo.mActivityName, cost, createdInfo.index, nowIndex, viewInfo.toString());
             //分析当前Activty 进入的情况
             handleBuffer(Type.ENTER, createdInfo.index, nowIndex, buffer, cost, viewInfo);
         }
         //标识Activit 已经进入
         hasEntered = true;
         getMethodBeat().lockBuffer(false);
    }
    //移除当前Activity
    mActivityCreatedInfoMap.remove(activity.hashCode());
}
所以我们可以通过这个函数执行的时候，进而的分析得到当前Activity 进入的耗时
```
现在分析Application 启动耗时分析，这个耗时是通过  StartUpTracer 来实现的
```java
public class StartUpTracer extends BaseTracer implements IMethodBeatListener {
    private static final String TAG = "Matrix.StartUpTracer";
    private final TraceConfig mTraceConfig;
    //标识第一个Activity 是否创建成功
    private boolean isFirstActivityCreate = true;
    //第一个Activity的类名
    private String mFirstActivityName = "";
    //第一个Activity 在 MethodBeat 中的索引下标
    private static int mFirstActivityIndex;
    //用来存储第一个Activity的Map,key 为类名，value 为当前Activity 创建的时间
    private final HashMap<String, Long> mFirstActivityMap = new HashMap<>();
    //用来存储所有执行了onWinwdowFocusChange 函数的 类名以及 当前的时间
    private final HashMap<String, Long> mActivityEnteredMap = new HashMap<>();
    private final Handler mHandler;

    public StartUpTracer(TracePlugin plugin, TraceConfig config) {
        super(plugin);
        this.mTraceConfig = config;
        //子线程的Looper，所以可以这个handler可以用来执行耗时的任务
        this.mHandler = new Handler(MatrixHandlerThread.getDefaultHandlerThread().getLooper());
    }

    //重写父类的 isEnableMethodBeat，所以可以统计耗时的函数执行
    @Override
    protected boolean isEnableMethodBeat() {
        return true;
    }
	
    @Override
    public void onActivityCreated(Activity activity) {
        super.onActivityCreated(activity);
        //赋值第一个进入的Activity 以及存储到 集合 mFirstActivityMap 集合中
        if (isFirstActivityCreate && mFirstActivityMap.isEmpty()) {
            String activityName = activity.getComponentName().getClassName();
            //赋值第一个Activity的 索引下标
            mFirstActivityIndex = getMethodBeat().getCurIndex();
            //赋值第一个Activity 的类名
            mFirstActivityName = activityName;
            //将第一个Activity存储到 mFirstActivityMap 中
            mFirstActivityMap.put(activityName, System.currentTimeMillis());
            MatrixLog.i(TAG, "[onActivityCreated] first activity:%s index:%s", mFirstActivityName, mFirstActivityIndex);
            getMethodBeat().lockBuffer(true);
        }
    }

    @Override
    public void onBackground(Activity activity) {
        super.onBackground(activity);
        isFirstActivityCreate = true;
    }

    @Override
    public void onActivityEntered(Activity activity, boolean isFocus, int nowIndex, long[] buffer) {
        //如果第一个Activity 还没有赋值
        if (!isFirstActivityCreate || mFirstActivityName == null) {
            isFirstActivityCreate = false;
            getMethodBeat().lockBuffer(false);
            return;
        }

        //将当前的Activity 添加到 mActivityEnteredMap 集合中 ，保存进入的时间
        String activityName = activity.getComponentName().getClassName();
        if (!mActivityEnteredMap.containsKey(activityName) || isFocus) {
            mActivityEnteredMap.put(activityName, System.currentTimeMillis());
        }

        if (!isFocus) {
            MatrixLog.i(TAG, "[onActivityEntered] isFocus false,activityName:%s", activityName);
            return;
        }


        //检查当前进入的Activity 是否为 默认的启动页面 ,如果不是 直接返回
        if (mTraceConfig.isHasSplashActivityName() && activityName.equals(mTraceConfig.getSplashActivityName())) {
            MatrixLog.i(TAG, "[onActivityEntered] has splash activity! %s", mTraceConfig.getSplashActivityName());
            return;
        }

        //到了这里就说明当前是splash页面
        getMethodBeat().lockBuffer(false);

        //获取到Activity 终止的时间
        long activityEndTime = getValueFromMap(mActivityEnteredMap, activityName);
        //获取到Activity 开始的时间
        long firstActivityStart = getValueFromMap(mFirstActivityMap, mFirstActivityName);
        //判断是否合法
        if (activityEndTime <= 0 || firstActivityStart <= 0) {
            MatrixLog.w(TAG, "[onActivityEntered] error activityCost! [%s:%s]", activityEndTime, firstActivityStart);
            mFirstActivityMap.clear();
            mActivityEnteredMap.clear();
            return;
        }

        //判断是否为温启动
        boolean isWarnStartUp = isWarmStartUp(firstActivityStart);
        //启动splash界面花费的时间
        long activityCost = activityEndTime - firstActivityStart;
        //当前Application 创建的时间
        long appCreateTime = Hacker.sApplicationCreateEndTime - Hacker.sApplicationCreateBeginTime;
        //Application 创建完之后 到第一个 界面 花费的时间
        long betweenCost = firstActivityStart - Hacker.sApplicationCreateEndTime;
        //总的花费时间，也即是应用程序启动Splash界面花费的时间
        long allCost = activityEndTime - Hacker.sApplicationCreateBeginTime;

        if (isWarnStartUp) {
            betweenCost = 0;
            allCost = activityCost;
        }

        //获取到splash界面发费的时间
        long splashCost = 0;
        if (mTraceConfig.isHasSplashActivityName()) {
            long tmp = getValueFromMap(mActivityEnteredMap, mTraceConfig.getSplashActivityName());
            splashCost = tmp == 0 ? 0 : getValueFromMap(mActivityEnteredMap, activityName) - tmp;
        }

        //如果Application创建的时间小于 0，代表出现了错误
        if (appCreateTime <= 0) {
            MatrixLog.e(TAG, "[onActivityEntered] appCreateTime is wrong! appCreateTime:%s", appCreateTime);
            mFirstActivityMap.clear();
            mActivityEnteredMap.clear();
            return;
        }

        //如果配置了 Splash界面，但是Splash启动的时间小于0 ，代表出现了错误,返回
        if (mTraceConfig.isHasSplashActivityName() && splashCost < 0) {
            MatrixLog.e(TAG, "splashCost < 0! splashCost:%s", splashCost);
            return;
        }

        //根据 EvilMethodTracer 对象,这个对象里面含有全部方法的执行时间消耗
        EvilMethodTracer tracer = getTracer(EvilMethodTracer.class);
        if (null != tracer) {
            //获得 阈值， 对于温启动 为 6秒 否则为 8秒
            long thresholdMs = isWarnStartUp ? mTraceConfig.getWarmStartUpThresholdMs() : mTraceConfig.getStartUpThresholdMs();
            //如果为温启动 index 取 splash 界面的索引， 如果为冷启动 取 application 创建的索引
            int startIndex = isWarnStartUp ? mFirstActivityIndex : Hacker.sApplicationCreateBeginMethodIndex;
            int curIndex = getMethodBeat().getCurIndex();
            //当前用的总时间大于 阈值的时间
            if (allCost > thresholdMs) {
                MatrixLog.i(TAG, "appCreateTime[%s] is over threshold![%s], dump stack! index[%s:%s]", appCreateTime, thresholdMs, startIndex, curIndex);
                //分析启动的时间
                EvilMethodTracer evilMethodTracer = getTracer(EvilMethodTracer.class);
                if (null != evilMethodTracer) {
                    evilMethodTracer.handleBuffer(EvilMethodTracer.Type.STARTUP, startIndex, curIndex, MethodBeat.getBuffer(), appCreateTime, Constants.SUBTYPE_STARTUP_APPLICATION);
                }
            }

        }
        MatrixLog.i(TAG, "[onActivityEntered] firstActivity:%s appCreateTime:%dms betweenCost:%dms activityCreate:%dms splashCost:%dms allCost:%sms isWarnStartUp:%b ApplicationCreateScene:%s",
                mFirstActivityName, appCreateTime, betweenCost, activityCost, splashCost, allCost, isWarnStartUp, Hacker.sApplicationCreateScene);
        //启动一个分析的任务 执行
        mHandler.post(new StartUpReportTask(activityName, appCreateTime, activityCost, betweenCost, splashCost, allCost, isWarnStartUp, Hacker.sApplicationCreateScene));
        //分析完之后释放
        mFirstActivityMap.clear();
        mActivityEnteredMap.clear();
        isFirstActivityCreate = false;
        mFirstActivityName = null;
        onDestroy();
    }
    ...
}

//这里统计 当前Application 创建的时间
long appCreateTime = Hacker.sApplicationCreateEndTime - Hacker.sApplicationCreateBeginTime;
而开始的时间是在执行 hackSysHandlerCallback 辅助的，而终止时间是在当ActivityThread 收到创建第一个Actvity，service ，receiver的时候，辅助的
public static void hackSysHandlerCallback() {
    try {
         //注意这里 当执行 ActivityThread的时候，就标记为 Application创建的开始时间
         sApplicationCreateBeginTime = System.currentTimeMillis();
         sApplicationCreateBeginMethodIndex = MethodBeat.getCurIndex();
         ...
}

而第一个启动界面 Splash界面的统计耗时，类似跟统计Activity 进入耗时一样，在onActivityCreate生命周期回调的时候，标记为开始的时间，执行了onActivityEntered的时候为终止的时间
//启动splash界面花费的时间
long activityCost = activityEndTime - firstActivityStart;
//当前Application 创建的时间
long appCreateTime = Hacker.sApplicationCreateEndTime - Hacker.sApplicationCreateBeginTime;
//Application 创建完之后 到第一个 界面 花费的时间
long betweenCost = firstActivityStart - Hacker.sApplicationCreateEndTime;
//总的花费时间，也即是应用程序启动Splash界面花费的时间
long allCost = activityEndTime - Hacker.sApplicationCreateBeginTime;
```
接下来分析最后一个FPS检测 ,这个检测是通过 FPSTracer 来实现的 
```java
public class FPSTracer extends BaseTracer implements LazyScheduler.ILazyTask, ViewTreeObserver.OnDrawListener {
	...
    @Override
    public void onCreate() {
        super.onCreate();
        this.mFrameDataList = new LinkedList<>();
        this.mSceneToSceneIdMap = new HashMap<>();
        this.mSceneIdToSceneMap = new SparseArray<>();
        this.mPendingReportSet = new SparseArray<>();

        //  public static final int DEFAULT_REPORT = 2 * 60 * 1000;  俩分钟  这里是构建一个定时器，每个2分钟触发一次 onTimeExpire 报告当前fps的情况
        this.mLazyScheduler = new LazyScheduler(MatrixHandlerThread.getDefaultHandlerThread(), mTraceConfig.getFPSReportInterval());

        //如果当前进程处于前台进程
        if (ApplicationLifeObserver.getInstance().isForeground()) {
            onFront(null);
        }
    }
	
	 /**
     * When the application comes to the foreground,it will be called.  当收到了当前应用处于前台的时候，启动这个 mLazyScheduler 定时器
     *
     * @param activity
     */
    @Override
    public void onFront(Activity activity) {
        super.onFront(activity);
        if (null != mLazyScheduler) {
            mLazyScheduler.cancel();
            this.mLazyScheduler.setUp(this, true);
        }
    }

    /**
     * When the application comes to the background,it will be called.  当处于后台的时候，移除这个定时器
     *
     * @param activity
     */
    @Override
    public void onBackground(Activity activity) {
        super.onBackground(activity);
        if (null != mLazyScheduler) {
            mLazyScheduler.cancel();
            this.mLazyScheduler.setUp(this, false);
        }
    }
	
    /**
     * when the device's Vsync is coming,it will be called.
     * use {@link com.tencent.matrix.trace.core.FrameBeat}
     *
     * @param lastFrameNanos
     * @param frameNanos
     */
    @Override
    public void doFrame(long lastFrameNanos, long frameNanos) {
        //当前要处于绘制的状态，才能收集 fps的信息
        if (!isInvalid && isDrawing && isEnterAnimationComplete() && mTraceConfig.isTargetScene(getScene())) {
            handleDoFrame(lastFrameNanos, frameNanos, getScene());
        }
        //执行完一次之后，将isDrawing 置为false
        isDrawing = false;
    }

    @Override
    public void onActivityResume(final Activity activity) {
        super.onActivityResume(activity);
        this.isInvalid = false;

        //添加onDrawListener
        addDrawListener(activity);
    }

    @Override
    public void onActivityPause(Activity activity) {
        super.onActivityPause(activity);
        //暂停的时候，移除 onDrawListener
        removeDrawListener(activity);

        //设置为非法
        this.isInvalid = true;
    }		
			
	 private void addDrawListener(final Activity activity) {
        activity.getWindow().getDecorView().post(new Runnable() {
            @Override
            public void run() {
                //移除
                activity.getWindow().getDecorView().getViewTreeObserver().removeOnDrawListener(FPSTracer.this);
                //再次的添加
                activity.getWindow().getDecorView().getViewTreeObserver().addOnDrawListener(FPSTracer.this);
            }
        });
		
     /**
     * it's time to report,it will be called.  收集fps的信息，生成报告
     */
    @Override
    public void onTimeExpire() {
        doReport();
    }
}
一开始就创建一个延迟执行的 mLazyScheduler，这里指定的延迟时间为 2分钟,也即是最少要2分钟才会执行 onTimeExpire,由于通过父类BaseTrace 执行了 
//注册生命周期监听
ApplicationLifeObserver.getInstance().register(this);
FrameBeat.getInstance().addListener(this);
所以FPSTracer 可以接受到 生命周期的回调，也可以收到 doFrame的回调，接着在onActivityReume方法中，执行了 
activity.getWindow().getDecorView().getViewTreeObserver().addOnDrawListener(FPSTracer.this);，这样当界面刷新的时候就能收到onDraw回调,相应的在onActivityPause中移除了这个监听
在onDraw()回调中，只是简单的将isDrawing 设置为true，标识当前有发生界面的绘制操作,接着在 doFrame 的时候执行收集的操作,执行完之后又会将 isDrawing = false,因为这一帧界面显示完毕

public void doFrame(long lastFrameNanos, long frameNanos) {
    //当前要处于绘制的状态，才能收集 fps的信息
    if (!isInvalid && isDrawing && isEnterAnimationComplete() && mTraceConfig.isTargetScene(getScene())) {
        handleDoFrame(lastFrameNanos, frameNanos, getScene());
    }
    //执行完一次之后，将isDrawing 置为false,标识当前一帧绘制结束所以要重置
    isDrawing = false;
}

//isEnterAnimationComplete()执行 是否进入的动画执行完毕
protected boolean isEnterAnimationComplete() {
    return Hacker.isEnterAnimationComplete;
}
由前面我们知道我们通过hack ActivityThread 中的Handler Callback的时候有监听 ENTER_ANIMATION_COMPLETE 事件，所以这里能知道这个结果，当一切都满足的时候，就会执行handleDoFrame()
收集当前的界面信息，然后当俩分钟到的时候，就会执行 onTimeExpire()函数，继而 执行doReport() 生成 fps的报告
```

### 总结
> 耗时方法的检测，通过自定义插件的方式，在 MatrixTraceTransform 实现了 Transform,重写了transform函数，继而可以在编译的时候拿到编译的内容，然后通过ASM来访问这些字节码，在每个方法的执行前面插入 MethodBeat 的i()函数，在每个方法执行后面插入 MethodBeat的o()函数，当执行o()函数的时候，就可以知道这个函数执行的耗时了
Activity 进入的耗时检测，也是通过自定义插件的方式，在检测函数的时候，判断当前的函数是否为Activity的onWindowFocusChanged,如果是的化，会在这个方法执行结束之后，利用ASM插入MethodBeat的at()函数，告知当前Activity创建完成，对于Activity创建的开始时间，可以通过监听 application.registerActivityLifecycleCallbacks(this); 在onActivityCreate回调执行的时候做为开始的时间
ANR的检测，可以通过Choreographer.getInstance().postFrameCallback()的方式监听到界面刷新的时机点，如果当前没有发生卡顿那么就会每16毫秒回调执行onFrame()函数，在这个回调函数中，我们可以先取消一个定时的任务，这个定时任务为5秒钟，然后再开启这个定时任务，这样如果当发生ANR的时候就会因为没有执行onFrame()函数，进而没有取消到这个定时器，这样定时器触发的时候，也即是主线程阻塞了5秒钟了，就可以判定为主线程发生了ANR了
FPS 的检测，也是通过 Choreographer.getInstance().postFrameCallback(),事实的监听 onFrame()函数，由于监听的是FPS，所以一定要有界面的绘制操作，所以又通过getViewTreeObserver().addOnDrawListener这样当有界面绘制的时候就会回调执行onDraw函数，标识isDrawing 为true，标识当前发生了界面的绘制，这样在 onFrame回调的时候就可以执行收集数据了，这里收集的时长为2分钟
Application 的启动耗时检测，首先通过静态代码块，来执行ActivityThread Handler 中Callback 的时候就可以做为Application 的启动时间了，也即是静态代码块执行的时候就可以做为Application的启动时间，对于Application的终止时间，是通过动态代理第一次接收到 msg.what == LAUNCH_ACTIVITY || msg.what == CREATE_SERVICE || msg.what == RECEIVER的时候就可以做为Application的终止时间统计第一个界面的耗时SplashActivity跟统计Activity耗时是一样原理，这样long betweenCost = firstActivityStart - Hacker.sApplicationCreateEndTime; 就可以算出Application启动之后到第一个界面启动的中间时间了,当然总的花费时间可以通过 long allCost = activityEndTime - Hacker.sApplicationCreateBeginTime;


