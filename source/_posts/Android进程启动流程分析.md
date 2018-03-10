---
layout: pager
title: Android进程启动流程分析
date: 2018-03-10 11:35:33
tags: [Android,IBinder,bindServer]
description: Android进程启动流程分析
---

Android进程启动流程分析
<!--more-->

```
Android应用程序框架层创建的应用程序进程的入口函数是ActivityThread.main比较好理解，即进程创建完成之后，Android应用程序框架层就会在这个进程中将ActivityThread类加载进来，
然后执行它的main函数，这个main函数就是进程执行消息循环的地方了。Android应用程序框架层创建的应用程序进程天然支持Binder进程间通信机制这个特点应该怎么样理解呢？
前面我们在学习Android系统的Binder进程间通信机制时说到，它具有四个组件，分别是驱动程序、守护进程、Client以及Server，其中Server组件在初始化时必须进入一个循环中
不断地与Binder驱动程序进行到交互，以便获得Client组件发送的请求，但是，当我们在Android应用程序中实现Server组件的时候，我们并没有让进程进入一个循环中去等待Client组件的请求，
然而，当Client组件得到这个Server组件的远程接口时，却可以顺利地和Server组件进行进程间通信，这就是因为Android应用程序进程在创建的时候就已经启动了一个线程池来支持
Server组件和Binder驱动程序之间的交互了，这样，极大地方便了在Android应用程序中创建Server组件。

在Android应用程序框架层中，是由ActivityManagerService组件负责为Android应用程序创建新的进程的，它本来也是运行在一个独立的进程之中，不过这个进程是在系统启动的过程中创建的。
ActivityManagerService组件一般会在什么情况下会为应用程序创建一个新的进程呢？当系统决定要在一个新的进程中启动一个Activity或者Service时，它就会创建一个新的进程了，
然后在这个新的进程中启动这个Activity或者Service
```


```java
public final class ActivityManagerService extends ActivityManagerNative
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {
	.....
	private final void startProcessLocked(ProcessRecord app, String hostingType,
            String hostingNameStr, String abiOverride, String entryPoint, String[] entryPointArgs) {
			
		.....
		
		 app.gids = gids;
        app.requiredAbi = requiredAbi;
        app.instructionSet = instructionSet;

        boolean isActivityProcess = (entryPoint == null);
        if (entryPoint == null) entryPoint = "android.app.ActivityThread";
        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "Start proc: " +
                    app.processName);
        checkTime(startTime, "startProcess: asking zygote to start proc");
        Process.ProcessStartResult startResult = Process.start(entryPoint,//这里的entryPoint 为 entryPoint = "android.app.ActivityThread";
				app.processName, uid, uid, gids, debugFlags, mountExternal,
                app.info.targetSdkVersion, app.info.seinfo, requiredAbi, instructionSet,
                app.info.dataDir, entryPointArgs);
        checkTime(startTime, "startProcess: returned from zygote!");
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
		.....
	}
	......
}

它调用了Process.start函数开始为应用程序创建新的进程，注意，它传入一个第一个参数为"android.app.ActivityThread"，这就是进程初始化时要加载的Java类了，
把这个类加载到进程之后，就会把它里面的静态成员函数main作为进程的入口点，后面我们会看到。
路径为 /Android7.1/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
public class Process {
	.....
	public static final ProcessStartResult start(final String processClass,
                                  final String niceName,
                                  int uid, int gid, int[] gids,
                                  int debugFlags, int mountExternal,
                                  int targetSdkVersion,
                                  String seInfo,
                                  String abi,
                                  String instructionSet,
                                  String appDataDir,
                                  String[] zygoteArgs) {
        try {
            return startViaZygote(processClass, niceName, uid, gid, gids,
                    debugFlags, mountExternal, targetSdkVersion, seInfo,
                    abi, instructionSet, appDataDir, zygoteArgs);
        } catch (ZygoteStartFailedEx ex) {
            Log.e(LOG_TAG,
                    "Starting VM process through Zygote failed");
            throw new RuntimeException(
                    "Starting VM process through Zygote failed", ex);
        }
    }
	....
}

继续调用startViazygote()函数，这个函数就在同一个文件里面
private static ProcessStartResult startViaZygote(final String processClass,
                                  final String niceName,
                                  final int uid, final int gid,
                                  final int[] gids,
                                  int debugFlags, int mountExternal,
                                  int targetSdkVersion,
                                  String seInfo,
                                  String abi,
                                  String instructionSet,
                                  String appDataDir,
                                  String[] extraArgs)
                                  throws ZygoteStartFailedEx {
        synchronized(Process.class) {
            ArrayList<String> argsForZygote = new ArrayList<String>();

            // --runtime-args, --setuid=, --setgid=,
            // and --setgroups= must go first
            argsForZygote.add("--runtime-args");
            argsForZygote.add("--setuid=" + uid);
            argsForZygote.add("--setgid=" + gid);
            if ((debugFlags & Zygote.DEBUG_ENABLE_JNI_LOGGING) != 0) {
                argsForZygote.add("--enable-jni-logging");
            }
            if ((debugFlags & Zygote.DEBUG_ENABLE_SAFEMODE) != 0) {
                argsForZygote.add("--enable-safemode");
            }
            if ((debugFlags & Zygote.DEBUG_ENABLE_DEBUGGER) != 0) {
                argsForZygote.add("--enable-debugger");
            }
            if ((debugFlags & Zygote.DEBUG_ENABLE_CHECKJNI) != 0) {
                argsForZygote.add("--enable-checkjni");
            }
            if ((debugFlags & Zygote.DEBUG_GENERATE_DEBUG_INFO) != 0) {
                argsForZygote.add("--generate-debug-info");
            }
            if ((debugFlags & Zygote.DEBUG_ALWAYS_JIT) != 0) {
                argsForZygote.add("--always-jit");
            }
            if ((debugFlags & Zygote.DEBUG_NATIVE_DEBUGGABLE) != 0) {
                argsForZygote.add("--native-debuggable");
            }
            if ((debugFlags & Zygote.DEBUG_ENABLE_ASSERT) != 0) {
                argsForZygote.add("--enable-assert");
            }
            if (mountExternal == Zygote.MOUNT_EXTERNAL_DEFAULT) {
                argsForZygote.add("--mount-external-default");
            } else if (mountExternal == Zygote.MOUNT_EXTERNAL_READ) {
                argsForZygote.add("--mount-external-read");
            } else if (mountExternal == Zygote.MOUNT_EXTERNAL_WRITE) {
                argsForZygote.add("--mount-external-write");
            }
            argsForZygote.add("--target-sdk-version=" + targetSdkVersion);

            //TODO optionally enable debuger
            //argsForZygote.add("--enable-debugger");

            // --setgroups is a comma-separated list
            if (gids != null && gids.length > 0) {
                StringBuilder sb = new StringBuilder();
                sb.append("--setgroups=");

                int sz = gids.length;
                for (int i = 0; i < sz; i++) {
                    if (i != 0) {
                        sb.append(',');
                    }
                    sb.append(gids[i]);
                }

                argsForZygote.add(sb.toString());
            }

            if (niceName != null) {
                argsForZygote.add("--nice-name=" + niceName);
            }

            if (seInfo != null) {
                argsForZygote.add("--seinfo=" + seInfo);
            }

            if (instructionSet != null) {
                argsForZygote.add("--instruction-set=" + instructionSet);
            }

            if (appDataDir != null) {
                argsForZygote.add("--app-data-dir=" + appDataDir);
            }

            argsForZygote.add(processClass);

            if (extraArgs != null) {
                for (String arg : extraArgs) {
                    argsForZygote.add(arg);
                }
            }

            return zygoteSendArgsAndGetResult(openZygoteSocketIfNeeded(abi), argsForZygote);
        }
}

这个函数将创建进程的参数放到argsForZygote列表中去，如参数"--runtime-args"表示要为新创建的进程初始化运行时库，然后调用zygoteSendArgsAndGetResult函数进一步操作。这个函数也是在同一个文件

private static ProcessStartResult zygoteSendArgsAndGetResult(
            ZygoteState zygoteState, ArrayList<String> args)
            throws ZygoteStartFailedEx {
        try {
            // Throw early if any of the arguments are malformed. This means we can
            // avoid writing a partial response to the zygote.
            int sz = args.size();
            for (int i = 0; i < sz; i++) {
                if (args.get(i).indexOf('\n') >= 0) {
                    throw new ZygoteStartFailedEx("embedded newlines not allowed");
                }
            }

            /**
             * See com.android.internal.os.ZygoteInit.readArgumentList()
             * Presently the wire format to the zygote process is:
             * a) a count of arguments (argc, in essence)
             * b) a number of newline-separated argument strings equal to count
             *
             * After the zygote process reads these it will write the pid of
             * the child or -1 on failure, followed by boolean to
             * indicate whether a wrapper process was used.
             */
            final BufferedWriter writer = zygoteState.writer;
            final DataInputStream inputStream = zygoteState.inputStream;

            writer.write(Integer.toString(args.size()));
            writer.newLine();

            for (int i = 0; i < sz; i++) {
                String arg = args.get(i);
                writer.write(arg);
                writer.newLine();
            }

            writer.flush();

            // Should there be a timeout on this?
            ProcessStartResult result = new ProcessStartResult();

            // Always read the entire result from the input stream to avoid leaving
            // bytes in the stream for future process starts to accidentally stumble
            // upon.
            result.pid = inputStream.readInt();
            result.usingWrapper = inputStream.readBoolean();

            if (result.pid < 0) {
                throw new ZygoteStartFailedEx("fork() failed");
            }
            return result;
        } catch (IOException ex) {
            zygoteState.close();
            throw new ZygoteStartFailedEx(ex);
        }
}

这里的sZygoteWriter是一个Socket写入流，是由openZygoteSocketIfNeeded函数打开的：
private static ZygoteState openZygoteSocketIfNeeded(String abi) throws ZygoteStartFailedEx {
        if (primaryZygoteState == null || primaryZygoteState.isClosed()) {
            try {
                primaryZygoteState = ZygoteState.connect(ZYGOTE_SOCKET);
            } catch (IOException ioe) {
                throw new ZygoteStartFailedEx("Error connecting to primary zygote", ioe);
            }
        }

        if (primaryZygoteState.matches(abi)) {
            return primaryZygoteState;
        }

        // The primary zygote didn't match. Try the secondary.
        if (secondaryZygoteState == null || secondaryZygoteState.isClosed()) {
            try {
				secondaryZygoteState = ZygoteState.connect(SECONDARY_ZYGOTE_SOCKET);
            } catch (IOException ioe) {
                throw new ZygoteStartFailedEx("Error connecting to secondary zygote", ioe);
            }
        }

        if (secondaryZygoteState.matches(abi)) {
            return secondaryZygoteState;
        }

        throw new ZygoteStartFailedEx("Unsupported zygote ABI: " + abi);
}

其中有这样的方法 secondaryZygoteState = ZygoteState.connect(SECONDARY_ZYGOTE_SOCKET);
public static ZygoteState connect(String socketAddress) throws IOException {
            DataInputStream zygoteInputStream = null;
            BufferedWriter zygoteWriter = null;
            final LocalSocket zygoteSocket = new LocalSocket();

            try {
                zygoteSocket.connect(new LocalSocketAddress(socketAddress,
                        LocalSocketAddress.Namespace.RESERVED));

                zygoteInputStream = new DataInputStream(zygoteSocket.getInputStream());

                zygoteWriter = new BufferedWriter(new OutputStreamWriter(
                        zygoteSocket.getOutputStream()), 256);
            } catch (IOException ex) {
                try {
                    zygoteSocket.close();
                } catch (IOException ignore) {
                }

                throw ex;
            }

            String abiListString = getAbiList(zygoteWriter, zygoteInputStream);
            Log.i("Zygote", "Process: zygote socket opened, supported ABIS: " + abiListString);

            return new ZygoteState(zygoteSocket, zygoteInputStream, zygoteWriter,
                    Arrays.asList(abiListString.split(",")));
}
在connetct的时候就会给对应的变量赋值，而且调用了zygoteSocket.connect的方法连接

这个Socket由frameworks/base/core/java/com/android/internal/os/ZygoteInit.java文件中的ZygoteInit类在runSelectLoopMode函数侦听的。
ZygoteInit.runSelectLoopMode
这个函数定义在frameworks/base/core/java/com/android/internal/os/ZygoteInit.java文件中：
public class ZygoteInit {  
    ......  
  
    /** 
    * Runs the zygote process's select loop. Accepts new connections as 
    * they happen, and reads commands from connections one spawn-request's 
    * worth at a time. 
    * 
    * @throws MethodAndArgsCaller in a child process when a main() should 
    * be executed. 
    */  
    private static void runSelectLoop(String abiList) throws MethodAndArgsCaller {
        ArrayList<FileDescriptor> fds = new ArrayList<FileDescriptor>();
        ArrayList<ZygoteConnection> peers = new ArrayList<ZygoteConnection>();

        fds.add(sServerSocket.getFileDescriptor());
        peers.add(null);

        while (true) {
            StructPollfd[] pollFds = new StructPollfd[fds.size()];
            for (int i = 0; i < pollFds.length; ++i) {
                pollFds[i] = new StructPollfd();
                pollFds[i].fd = fds.get(i);
                pollFds[i].events = (short) POLLIN;
            }
            try {
                Os.poll(pollFds, -1);
            } catch (ErrnoException ex) {
                throw new RuntimeException("poll failed", ex);
            }
            for (int i = pollFds.length - 1; i >= 0; --i) {
                if ((pollFds[i].revents & POLLIN) == 0) {
                    continue;
                }
                if (i == 0) {
                    ZygoteConnection newPeer = acceptCommandPeer(abiList);
                    peers.add(newPeer);
                    fds.add(newPeer.getFileDesciptor());
                } else {
                    boolean done = peers.get(i).runOnce();
                    if (done) {
                        peers.remove(i);
                        fds.remove(i);
                    }
                }
            }
        }
    }
    ......  
}  
当将数据通过Socket接口发送出去后，就会下面这个语句：
boolean done = peers.get(i).runOnce();
这里从peers.get(index)得到的是一个ZygoteConnection对象，表示一个Socket连接，因此，接下来就是调用ZygoteConnection.runOnce函数进一步处理了。

这个函数定义在/Android7.1/frameworks/base/core/java/com/android/internal/os/ZygoteConnection.java
boolean runOnce() throws ZygoteInit.MethodAndArgsCaller {

        String args[];
        Arguments parsedArgs = null;
        FileDescriptor[] descriptors;

        try {
            args = readArgumentList();
            descriptors = mSocket.getAncillaryFileDescriptors();
        } catch (IOException ex) {
            Log.w(TAG, "IOException on command socket " + ex.getMessage());
            closeSocket();
            return true;
        }

        if (args == null) {
            // EOF reached.
            closeSocket();
            return true;
        }

        /** the stderr of the most recent request, if avail */
        PrintStream newStderr = null;

        if (descriptors != null && descriptors.length >= 3) {
            newStderr = new PrintStream(
                    new FileOutputStream(descriptors[2]));
        }

        int pid = -1;
        FileDescriptor childPipeFd = null;
        FileDescriptor serverPipeFd = null;

        try {
            parsedArgs = new Arguments(args);

            if (parsedArgs.abiListQuery) {
                return handleAbiListQuery();
            }

            if (parsedArgs.permittedCapabilities != 0 || parsedArgs.effectiveCapabilities != 0) {
                throw new ZygoteSecurityException("Client may not specify capabilities: " +
                        "permitted=0x" + Long.toHexString(parsedArgs.permittedCapabilities) +
                        ", effective=0x" + Long.toHexString(parsedArgs.effectiveCapabilities));
            }

            applyUidSecurityPolicy(parsedArgs, peer);
            applyInvokeWithSecurityPolicy(parsedArgs, peer);

            applyDebuggerSystemProperty(parsedArgs);
            applyInvokeWithSystemProperty(parsedArgs);

            int[][] rlimits = null;

            if (parsedArgs.rlimits != null) {
                rlimits = parsedArgs.rlimits.toArray(intArray2d);
            }

            if (parsedArgs.invokeWith != null) {
                FileDescriptor[] pipeFds = Os.pipe2(O_CLOEXEC);
                childPipeFd = pipeFds[1];
                serverPipeFd = pipeFds[0];
                Os.fcntlInt(childPipeFd, F_SETFD, 0);
            }

            /**
             * In order to avoid leaking descriptors to the Zygote child,
             * the native code must close the two Zygote socket descriptors
             * in the child process before it switches from Zygote-root to
             * the UID and privileges of the application being launched.
             *
             * In order to avoid "bad file descriptor" errors when the
             * two LocalSocket objects are closed, the Posix file
             * descriptors are released via a dup2() call which closes
             * the socket and substitutes an open descriptor to /dev/null.
             */

            int [] fdsToClose = { -1, -1 };

            FileDescriptor fd = mSocket.getFileDescriptor();

            if (fd != null) {
                fdsToClose[0] = fd.getInt$();
            }

            fd = ZygoteInit.getServerSocketFileDescriptor();

            if (fd != null) {
                fdsToClose[1] = fd.getInt$();
            }

            fd = null;

            pid = Zygote.forkAndSpecialize(parsedArgs.uid, parsedArgs.gid, parsedArgs.gids,
                    parsedArgs.debugFlags, rlimits, parsedArgs.mountExternal, parsedArgs.seInfo,
                    parsedArgs.niceName, fdsToClose, parsedArgs.instructionSet,
                    parsedArgs.appDataDir);
        } catch (ErrnoException ex) {
            logAndPrintError(newStderr, "Exception creating pipe", ex);
        } catch (IllegalArgumentException ex) {
            logAndPrintError(newStderr, "Invalid zygote arguments", ex);
        } catch (ZygoteSecurityException ex) {
            logAndPrintError(newStderr,
                    "Zygote security policy prevents request: ", ex);
        }

        try {
            if (pid == 0) {
                // in child
                IoUtils.closeQuietly(serverPipeFd);
                serverPipeFd = null;
                handleChildProc(parsedArgs, descriptors, childPipeFd, newStderr);

                // should never get here, the child is expected to either
                // throw ZygoteInit.MethodAndArgsCaller or exec().
                return true;
            } else {
                // in parent...pid of < 0 means failure
                IoUtils.closeQuietly(childPipeFd);
                childPipeFd = null;
                return handleParentProc(pid, descriptors, serverPipeFd, parsedArgs);
            }
        } finally {
            IoUtils.closeQuietly(childPipeFd);
            IoUtils.closeQuietly(serverPipeFd);
        }
}

有Linux开发经验的读者很容易看懂这个函数调用，这个函数会创建一个进程，而且有两个返回值，一个是在当前进程中返回的，一个是在新创建的进程中返回，即在当前进程的子进程中返回，
在当前进程中的返回值就是新创建的子进程的pid值，、而在子进程中的返回值是0。因为我们只关心创建的新进程的情况，因此，我们沿着子进程的执行路径继续看下去：
if (pid == 0) {  
	// in child  
	handleChildProc(parsedArgs, descriptors, newStderr);  
	//should never happen  
	return true;  
   } else { /* pid != 0 */  
	......  
}  

private void handleChildProc(Arguments parsedArgs,  
            FileDescriptor[] descriptors, PrintStream newStderr)  
            throws ZygoteInit.MethodAndArgsCaller {  
    ......  
  
    if (parsedArgs.runtimeInit) {  
         RuntimeInit.zygoteInit(parsedArgs.remainingArgs);  
    } else {  
        ......  
    }  
}  

由于在前面的Step 3中，指定了"--runtime-init"参数，表示要为新创建的进程初始化运行时库，因此，这里的parseArgs.runtimeInit值为true，于是就继续执行RuntimeInit.zygoteInit进一步处理了。
 public static final void zygoteInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
    throws ZygoteInit.MethodAndArgsCaller {
    if (DEBUG) Slog.d(TAG, "RuntimeInit: Starting application from zygote");

    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "RuntimeInit");
    redirectLogStreams();

    commonInit();
    nativeZygoteInit();
    applicationInit(targetSdkVersion, argv, classLoader);
}

这里有两个关键的函数调用，一个是zygoteInitNative函数调用，一个是applicationInit函数调用，前者就是执行Binder驱动程序初始化的相关工作了，正是由于执行了这个工作，
才使得进程中的Binder对象能够顺利地进行Binder进程间通信，而后一个函数调用，就是执行进程的入口函数，这里就是执行startClass类的main函数了，
而这个startClass即是我们前面传进来的"android.app.ActivityThread"值，表示要执行android.app.ActivityThread类的main函数

我们先来看一下zygoteInitNative函数的调用过程，然后再回到RuntimeInit.zygoteInit函数中来，看看它是如何调用android.app.ActivityThread类的main函数的。
函数对应的文件在/Android7.1/frameworks/base/core/jni/AndroidRuntime.cpp

static void com_android_internal_os_RuntimeInit_nativeZygoteInit(JNIEnv* env, jobject clazz)
{
    gCurRuntime->onZygoteInit();
}

这里它调用了全局变量gCurRuntime的onZygoteInit函数，这个全局变量的定义在frameworks/base/core/jni/AndroidRuntime.cpp文件开头的地方：
static AndroidRuntime* gCurRuntime = NULL;  

这里可以看出，它的类型为AndroidRuntime，它是在AndroidRuntime类的构造函数中初始化的，AndroidRuntime类的构造函数也是定义在frameworks/base/core/jni/AndroidRuntime.cpp文件中：
AndroidRuntime::AndroidRuntime(char* argBlockStart, const size_t argBlockLength) :
        mExitWithoutCleanup(false),
        mArgBlockStart(argBlockStart),
        mArgBlockLength(argBlockLength)
{
	......
    assert(gCurRuntime == NULL);        // one per process
    gCurRuntime = this;                	//给这个成语变量赋值  
}

那么这个AndroidRuntime类的构造函数又是什么时候被调用的呢？AndroidRuntime类的声明在frameworks/base/include/android_runtime/AndroidRuntime.h文件中，
如果我们打开这个文件会看到，它是一个虚拟类，也就是我们不能直接创建一个AndroidRuntime对象，只能用一个AndroidRuntime类的指针来指向它的某一个子类，这个子类就是AppRuntime了，
它定义在frameworks/base/cmds/app_process/app_main.cpp文件中：

int main(int argc, const char* const argv[])  
{  
    ......  
  
    AppRuntime runtime;  
      
    ......  
}  

而AppRuntime类继续了AndroidRuntime类，它也是定义在frameworks/base/cmds/app_process/app_main.cpp文件中：
class AppRuntime : public AndroidRuntime  
{  
    ......  
  
};  

因此，在前面的com_android_internal_os_RuntimeInit_zygoteInit函数，实际是执行了AppRuntime类的onZygoteInit函数。
class AppRuntime : public AndroidRuntime  
{  
    ......  
  
    virtual void onZygoteInit()  
    {  
        sp<ProcessState> proc = ProcessState::self();  
        if (proc->supportsProcesses()) {  
            LOGV("App process: starting thread pool.\n");  
            proc->startThreadPool();  
        }  
    }  
  
    ......  
};  

这里它就是调用ProcessState::startThreadPool启动线程池了，这个线程池中的线程就是用来和Binder驱动程序进行交互的了。
这个函数定义在frameworks/base/libs/binder/ProcessState.cpp文件中：
void ProcessState::startThreadPool()  
{  
    AutoMutex _l(mLock);  
    if (!mThreadPoolStarted) {  
        mThreadPoolStarted = true;  
        spawnPooledThread(true);  
    }  
}  
ProcessState类是Binder进程间通信机制的一个基础组件
ProcessState.spawnPooledThread
这个函数定义在frameworks/base/libs/binder/ProcessState.cpp文件中
这里它会创建一个PoolThread线程类，然后执行它的run函数，最终就会执行PoolThread类的threadLoop函数了

PoolThread.threadLoop
这个函数定义在frameworks/base/libs/binder/ProcessState.cpp文件中 
class PoolThread : public Thread  
{  
public:  
    PoolThread(bool isMain)  
        : mIsMain(isMain)  
    {  
    }  
  
protected:  
    virtual bool threadLoop()  
    {  
        IPCThreadState::self()->joinThreadPool(mIsMain);  
        return false;  
    }  
  
    const bool mIsMain;  
};  
这里它执行了IPCThreadState::joinThreadPool函数进一步处理。IPCThreadState也是Binder进程间通信机制的一个基础组件
IPCThreadState.joinThreadPool
这个函数定义在frameworks/base/libs/binder/IPCThreadState.cpp文件中：
void IPCThreadState::joinThreadPool(bool isMain)  
{  
    ......  
  
    mOut.writeInt32(isMain ? BC_ENTER_LOOPER : BC_REGISTER_LOOPER);  
  
    ......  
  
    status_t result;  
    do {  
        int32_t cmd;  
  
        ......  
  
        // now get the next command to be processed, waiting if necessary  
        result = talkWithDriver();  
        if (result >= NO_ERROR) {  
            size_t IN = mIn.dataAvail();  
            if (IN < sizeof(int32_t)) continue;  
            cmd = mIn.readInt32();  
            ......  
  
            result = executeCommand(cmd);  
        }  
  
        ......  
    } while (result != -ECONNREFUSED && result != -EBADF);  
  
    ......  
      
    mOut.writeInt32(BC_EXIT_LOOPER);  
    talkWithDriver(false);  
}  

这个函数首先告诉Binder驱动程序，这条线程要进入循环了
mOut.writeInt32(isMain ? BC_ENTER_LOOPER : BC_REGISTER_LOOPER);  
然后在中间的while循环中通过talkWithDriver不断与Binder驱动程序进行交互，以便获得Client端的进程间调用：
result = talkWithDriver();  
result = executeCommand(cmd);  
最后，线程退出时，也会告诉Binder驱动程序，它退出了，这样Binder驱动程序就不会再在Client端的进程间调用分发给它了：
mOut.writeInt32(BC_EXIT_LOOPER);  
talkWithDriver(false);  
		
继续回到到RuntimeInit.zygoteInit函数中，在初始化完成Binder进程间通信机制的基础设施后，它接着就要进入进程的入口函数了。	
 private static void applicationInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
            throws ZygoteInit.MethodAndArgsCaller {
    ......
    // Remaining arguments are passed to the start class's static main
    invokeStaticMain(args.startClass, args.startArgs, classLoader);
}	

private static void invokeStaticMain(String className, String[] argv, ClassLoader classLoader)
            throws ZygoteInit.MethodAndArgsCaller {
        Class<?> cl;

        try {
            cl = Class.forName(className, true, classLoader);
        } catch (ClassNotFoundException ex) {
            throw new RuntimeException(
                    "Missing class when invoking static main " + className,
                    ex);
        }

        Method m;
        try {
            m = cl.getMethod("main", new Class[] { String[].class });
        } catch (NoSuchMethodException ex) {
            throw new RuntimeException(
                    "Missing static main on " + className, ex);
        } catch (SecurityException ex) {
            throw new RuntimeException(
                    "Problem getting static main on " + className, ex);
        }

        int modifiers = m.getModifiers();
        if (! (Modifier.isStatic(modifiers) && Modifier.isPublic(modifiers))) {
            throw new RuntimeException(
                    "Main method is not public and static on " + className);
        }

        /*
         * This throw gets caught in ZygoteInit.main(), which responds
         * by invoking the exception's run() method. This arrangement
         * clears up all the stack frames that were required in setting
         * up the process.
         */
        throw new ZygoteInit.MethodAndArgsCaller(m, argv);
}
	
前面我们说过，这里传进来的参数className字符串值为"android.app.ActivityThread"，这里就通ClassLoader.loadClass函数将它加载到进程中：
cl = loader.loadClass(className);  
然后获得它的静态成员函数main：
m = cl.getMethod("main", new Class[] { String[].class });  
函数最后并没有直接调用这个静态成员函数main，而是通过抛出一个异常ZygoteInit.MethodAndArgsCaller，然后让ZygoteInit.main函数在捕获这个异常的时候再调用
android.app.ActivityThread类的main函数。为什么要这样做呢？注释里面已经讲得很清楚了，它是为了清理堆栈的，这样就会让android.app.ActivityThread类的main函数觉得自己是进程的入口函数，
而事实上，在执行android.app.ActivityThread类的main函数之前，已经做了大量的工作了。
	
public class ZygoteInit {  
    ......  
  
    public static void main(String argv[]) {  
        try {  
            ......  
        } catch (MethodAndArgsCaller caller) {  
            caller.run();  
        } catch (RuntimeException ex) {  
            ......  
        }  
    }  
  
    ......  
}  

它执行MethodAndArgsCaller的run函数：
public class ZygoteInit {  
    ......  
  
    public static class MethodAndArgsCaller extends Exception  
            implements Runnable {  
        /** method to call */  
        private final Method mMethod;  
  
        /** argument array */  
        private final String[] mArgs;  
  
        public MethodAndArgsCaller(Method method, String[] args) {  
            mMethod = method;  
            mArgs = args;  
        }  
  
        public void run() {  
            try {  
                mMethod.invoke(null, new Object[] { mArgs });  
            } catch (IllegalAccessException ex) {  
                ......  
            } catch (InvocationTargetException ex) {  
                ......  
            }  
        }  
    }  
  
    ......  
}  

这里的成员变量mMethod和mArgs都是在前面构造异常对象时传进来的，这里的mMethod就对应android.app.ActivityThread类的main函数了，于是最后就通过下面语句执行这个函数：	
mMethod.invoke(null, new Object[] { mArgs });  

这样，android.app.ActivityThread类的main函数就被执行了。
public final class ActivityThread {  
    ......  
  
    public static final void main(String[] args) {  
        SamplingProfilerIntegration.start();  
  
        Process.setArgV0("<pre-initialized>");  
  
        Looper.prepareMainLooper();  
        if (sMainThreadHandler == null) {  
            sMainThreadHandler = new Handler();  
        }  
  
        ActivityThread thread = new ActivityThread();  
        thread.attach(false);  
  
        if (false) {  
            Looper.myLooper().setMessageLogging(new  
                LogPrinter(Log.DEBUG, "ActivityThread"));  
        }  
        Looper.loop();  
  
        if (Process.supportsProcesses()) {  
            throw new RuntimeException("Main thread loop unexpectedly exited");  
        }  
  
        thread.detach();  
        String name = (thread.mInitialApplication != null)  
            ? thread.mInitialApplication.getPackageName()  
            : "<unknown>";  
        Slog.i(TAG, "Main thread of " + name + " is now exiting");  
    }  
  
    ......  
}  
从这里我们可以看出，这个函数首先会在进程中创建一个ActivityThread对象：
ActivityThread thread = new ActivityThread();  
然后进入消息循环中：
Looper.loop();  
这样，我们以后就可以在这个进程中启动Activity或者Service了。
```	