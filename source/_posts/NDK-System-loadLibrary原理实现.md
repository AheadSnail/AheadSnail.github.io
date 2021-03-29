---
title: NDK System.loadLibrary原理实现
date: 2017-10-04 17:58:11
tags: [Android,NDK,SystemLoad]
description: NDK System.loadLibrary原理实现
---
1.首先我们在java代码里面写了这样的代码
```java
 static
 {
    System.loadLibrary("native-lib");
 }
 然后会调用到
public static void loadLibrary(String libname) { libname为传递的so的名称
        Runtime.getRuntime().loadLibrary0(VMStack.getCallingClassLoader(), libname);
}
之后执行到Runtime中的loadLibrary0方法,第一个参数表示为当前类加载，第二个参数为要加载的so的名称
第二个实例对象，我们可以打印出来，发现这个实例对象为PathClassLoader
Log.d(TAG, " Classloader: "+this.getClassLoader().toString());
10-04 10:53:09.020 4451-4451/com.example.administrator.ndksystemloaddemo D/MainActivity:  Classloader: dalvik.system.PathClassLoader
[DexPathList[[zip file "/data/app/com.example.administrator.ndksystemloaddemo-2/base.apk", zip file "/data/app/com.example.administrator.ndksystemloaddemo-2/split_lib_dependencies_apk.apk", 
 nativeLibraryDirectories=[/data/app/com.example.administrator.ndksystemloaddemo-2/lib/x86, /data/app/com.example.administrator.ndksystemloaddemo-2/base.apk!/lib/x86, 
 /data/app/com.example.administrator.ndksystemloaddemo-2/split_lib_dependencies_apk.apk!/lib/x86, /data/app/com.example.administrator.ndksystemloaddemo-2/split_lib_slice_0_apk.apk!/lib/x86, 
 /data/app/com.example.administrator.ndksystemloaddemo-2/split_lib_slice_1_apk.apk!/lib/x86, /data/app/com.example.administrator.ndksystemloaddemo-2/split_lib_slice_2_apk.apk!/lib/x86, 
 /data/app/com.example.administrator.ndksystemloaddemo-2/split_lib_slice_3_apk.apk!/lib/x86, /data/app/com.example.administrator.ndksystemloaddemo-2/split_lib_slice_4_apk.apk!/lib/x86, 
 /data/app/com.example.administrator.ndksystemloaddemo-2/split_lib_slice_5_apk.apk!/lib/x86, /data/app/com.example.administrator.ndksystemloaddemo-2/split_lib_slice_6_apk.apk!/lib/x86,
 /data/app/com.example.administrator.ndksystemloaddemo-2/split_lib_slice_7_apk.apk!/lib/x86, /data/app/com.example.administrator.ndksystemloaddemo-2/split_lib_slice_8_apk.apk!/lib/x86, 
 /data/app/com.example.administrator.ndksystemloaddemo-2/split_lib_slice_9_apk.apk!/lib/x86, /system/lib, /vendor/lib]]]

synchronized void loadLibrary0(ClassLoader loader, String libname) {
        if (libname.indexOf((int)File.separatorChar) != -1) {
            throw new UnsatisfiedLinkError(
    "Directory separator should not appear in library name: " + libname);
        }
        String libraryName = libname;
        if (loader != null) {
			1,首先查找到对应的要加载so的全路径,传递的library
            String filename = loader.findLibrary(libraryName);
            if (filename == null) {
                // It's not necessarily true that the ClassLoader used
                // System.mapLibraryName, but the default setup does, and it's
                // misleading to say we didn't find "libMyLibrary.so" when we
                // actually searched for "liblibMyLibrary.so.so".
                throw new UnsatisfiedLinkError(loader + " couldn't find \"" +
                                               System.mapLibraryName(libraryName) + "\"");
            }
			2.加载so
            String error = doLoad(filename, loader);
            if (error != null) {
                throw new UnsatisfiedLinkError(error);
            }
            return;
        }
		
		//如果classLoad为空,就从系统的路径下查找getLibPaths()
        String filename = System.mapLibraryName(libraryName);
        List<String> candidates = new ArrayList<String>();
        String lastError = null;
        for (String directory : getLibPaths()) {
            String candidate = directory + filename;
            candidates.add(candidate);

            if (IoUtils.canOpenReadOnly(candidate)) {
                String error = doLoad(candidate, loader);
                if (error == null) {
                    return; // We successfully loaded the library. Job done.
                }
                lastError = error;
            }
        }

        if (lastError != null) {
            throw new UnsatisfiedLinkError(lastError);
        }
        throw new UnsatisfiedLinkError("Library " + libraryName + " not found; tried " + candidates);
    }
	
	getLibPaths() 函数的实现为，System.getProperty("java.library.path");发现也是查找这个，
	Log.d(TAG,"system librarPath :"+System.getProperty("java.library.path"));
	输出结果为D/MainActivity: system librarPath :/system/lib:/vendor/lib
		
	private String[] getLibPaths() {
        if (mLibPaths == null) {
            synchronized(this) {
                if (mLibPaths == null) {
                    mLibPaths = initLibPaths();
                }
            }
        }
        return mLibPaths;
    }

    private static String[] initLibPaths() {
        String javaLibraryPath = System.getProperty("java.library.path");
        if (javaLibraryPath == null) {
            return EmptyArray.STRING;
        }
        String[] paths = javaLibraryPath.split(":");
        // Add a '/' to the end of each directory so we don't have to do it every time.
        for (int i = 0; i < paths.length; ++i) {
            if (!paths[i].endsWith("/")) {
                paths[i] += "/";
            }
        }
        return paths;
    }
	
 先看这里面是怎么查找到我们的so的路径的 ，这个loader的实例对象为PathClassLoader
 String filename = loader.findLibrary(libraryName);
 public class PathClassLoader extends BaseDexClassLoader {
    /**
     * Creates a {@code PathClassLoader} that operates on a given list of files
     * and directories. This method is equivalent to calling
     * {@link #PathClassLoader(String, String, ClassLoader)} with a
     * {@code null} value for the second argument (see description there).
     *
     * @param dexPath the list of jar/apk files containing classes and
     * resources, delimited by {@code File.pathSeparator}, which
     * defaults to {@code ":"} on Android
     * @param parent the parent class loader
     */
    public PathClassLoader(String dexPath, ClassLoader parent) {
        super(dexPath, null, null, parent);
    }
}
找到了BaseDexClassLoader中有实现这个的方法
@Override
public String findLibrary(String name) {
    return pathList.findLibrary(name);
}
其中的这个pathList为BaseDexClassLoader中的一个成员变量，并且在构造函数里面完成赋值操作
private final DexPathList pathList;
所以也就找到DexPathList中的findLibrary实现
public String findLibrary(String libraryName) {
        String fileName = System.mapLibraryName(libraryName);
		//nativeLibraryPathElements 这个值代表的是什么?
        for (Element element : nativeLibraryPathElements) {
            String path = element.findNativeLibrary(fileName);
			遍历查找这个path对应的路径，直到找到为止
            if (path != null) {
                return path;
            }
        }

        return null;
    }
nativeLibraryPathElements 这个值代表的是什么?
其中的这个pathList为BaseDexClassLoader中的一个成员变量，并且在构造函数里面完成赋值操作
public BaseDexClassLoader(String dexPath, File optimizedDirectory,
            String librarySearchPath, ClassLoader parent) {
        super(parent);
       this.pathList = new DexPathList(this, dexPath, librarySearchPath, optimizedDirectory);
}
this.pathList = new DexPathList(this, dexPath, librarySearchPath, optimizedDirectory);函数的实现为
nativeLibraryPathElements 在这里完成了赋值操作
public DexPathList(ClassLoader definingContext, String dexPath,
            String librarySearchPath, File optimizedDirectory) {

        this.dexElements = makeDexElements(splitDexPath(dexPath), optimizedDirectory,
                                           suppressedExceptions, definingContext);

        // Native libraries may exist in both the system and
        // application library paths, and we use this search order:
        //
        //   1. This class loader's library path for application libraries (librarySearchPath):
        //   1.1. Native library directories
        //   1.2. Path to libraries in apk-files
        //   2. The VM's library path from the system property for system libraries
        //      also known as java.library.path
        //
        // This order was reversed prior to Gingerbread; see http://b/2933456.
		根据我们传递的dex 的path，得到一组path路径
        this.nativeLibraryDirectories = splitPaths(librarySearchPath, false);
		
		我们想知道System.getProperty("java.library.path")到底是什么，可以打印输出
		这也发现了一个另外一个可以存放so的地方就是system/lib,或者是/vendor/lib下，也是可以直接放so的
		Log.d(TAG,"system librarPath :"+System.getProperty("java.library.path"));
		输出结果为D/MainActivity: system librarPath :/system/lib:/vendor/lib
		
        this.systemNativeLibraryDirectories =
                splitPaths(System.getProperty("java.library.path"), true);
		
		然后将上面的路径加进allNativeLibraryDirectories集合中
        List<File> allNativeLibraryDirectories = new ArrayList<>(nativeLibraryDirectories);
        allNativeLibraryDirectories.addAll(systemNativeLibraryDirectories);

		给nativeLibraryPathElements赋值 发现他的内容有allNativeLibraryDirectories这个在上面完成了赋值
        this.nativeLibraryPathElements = makePathElements(allNativeLibraryDirectories,
                                                          suppressedExceptions,
                                                          definingContext);

        if (suppressedExceptions.size() > 0) {
            this.dexElementsSuppressedExceptions =
                suppressedExceptions.toArray(new IOException[suppressedExceptions.size()]);
        } else {
            dexElementsSuppressedExceptions = null;
        }
    }
 ```
 **加载so**
 ===
 ```java
 String error = doLoad(filename, loader); 函数的实现为:
private String doLoad(String name, ClassLoader loader) {
    中有这样的关键代码
    synchronized (this) {
        return nativeLoad(name, loader, librarySearchPath);
    }
}
可以发现这是一个native方法
// TODO: should be synchronized, but dalvik doesn't support synchronized internal natives.
private static native String nativeLoad(String filename, ClassLoader loader,String librarySearchPath);
对应的c文件的路径为源码路径下的libcore\ojluni\src\main\native\Runtime.c 

这边是采用动态注册的方式，所以没有包括包名
static JNINativeMethod gMethods[] = {
  NATIVE_METHOD(Runtime, freeMemory, "!()J"),
  NATIVE_METHOD(Runtime, totalMemory, "!()J"),
  NATIVE_METHOD(Runtime, maxMemory, "!()J"),
  NATIVE_METHOD(Runtime, gc, "()V"),
  NATIVE_METHOD(Runtime, nativeExit, "(I)V"),
  NATIVE_METHOD(Runtime, nativeLoad,
                "(Ljava/lang/String;Ljava/lang/ClassLoader;Ljava/lang/String;)"
                    "Ljava/lang/String;"),
};

JNIEXPORT jstring JNICALL
Runtime_nativeLoad(JNIEnv* env, jclass ignored, jstring javaFilename,
                   jobject javaLoader, jstring javaLibrarySearchPath)
{
    return JVM_NativeLoad(env, javaFilename, javaLoader, javaLibrarySearchPath);
}
JVM_NativeLoad(env, javaFilename, javaLoader, javaLibrarySearchPath);可以通过全局搜索，最终在art/runtime/openjdkjvm/OpenjdkJvm.cc

JNIEXPORT jstring JVM_NativeLoad(JNIEnv* env,
                                 jstring javaFilename,
                                 jobject javaLoader,
                                 jstring javaLibrarySearchPath) {
  ScopedUtfChars filename(env, javaFilename);
  if (filename.c_str() == NULL) {
    return NULL;
  }

  std::string error_msg;
  {
	得到JVM指针，这里要注意一般一个应用程序对应一个进程，也就对应的有一个JVM对象，
	而JNIEnv 在每一个线程里面都有一个这样的对象，所以如果将加载过的so的信息保存在JVM里面，这样每一个JNIENV都能得到
    art::JavaVMExt* vm = art::Runtime::Current()->GetJavaVM();
	
    bool success = vm->LoadNativeLibrary(env,
                                         filename.c_str(),
                                         javaLoader,
                                         javaLibrarySearchPath,
                                         &error_msg);
    if (success) {
      return nullptr;
    }
  }
  
  vm->LoadNativeLibrary()函数实现为
  bool JavaVMExt::LoadNativeLibrary(JNIEnv* env,
                                  const std::string& path,
                                  jobject class_loader,
                                  jstring library_path,
                                  std::string* error_msg) {
  error_msg->clear();

  // See if we've already loaded this library.  If we have, and the class loader
  // matches, return successfully without doing anything.
  // TODO: for better results we should canonicalize the pathname (or even compare
  // inodes). This implementation is fine if everybody is using System.loadLibrary.
  //定义一个ShareLibrary* 指针
  SharedLibrary* library;
  Thread* self = Thread::Current();
  {
    // TODO: move the locking (and more of this logic) into Libraries.
    MutexLock mu(self, *Locks::jni_libraries_lock_);
	libraries_ 为一个成员变量JavaVMExt,std::unique_ptr<Libraries> libraries_;这里为一个智能指针
	并且在构造函数中初始了,libraries_(new Libraries), 
	这里的作用就是，防止存储的是shareLibrary 指针对象,这里先根据path去判断当前是否已经有加载过这个so
    library = libraries_->Get(path);
  }
  void* class_loader_allocator = nullptr;
  {
    ScopedObjectAccess soa(env);
    // As the incoming class loader is reachable/alive during the call of this function,
    // it's okay to decode it without worrying about unexpectedly marking it alive.
    mirror::ClassLoader* loader = soa.Decode<mirror::ClassLoader*>(class_loader);
	
    ClassLinker* class_linker = Runtime::Current()->GetClassLinker();
    if (class_linker->IsBootClassLoader(soa, loader)) {
      loader = nullptr;
      class_loader = nullptr;
    }

    class_loader_allocator = class_linker->GetAllocatorForClassLoader(loader);
    CHECK(class_loader_allocator != nullptr);
  }
  
  如果library 不为空
  if (library != nullptr) {
    // Use the allocator pointers for class loader equality to avoid unnecessary weak root decode.
    if (library->GetClassLoaderAllocator() != class_loader_allocator) {
      // The library will be associated with class_loader. The JNI
      // spec says we can't load the same library into more than one
      // class loader.
      StringAppendF(error_msg, "Shared library \"%s\" already opened by "
          "ClassLoader %p; can't open in ClassLoader %p",
          path.c_str(), library->GetClassLoader(), class_loader);
      LOG(WARNING) << error_msg;
      return false;
    }
	这就代表已经加载过这个so了，就没有必要重新的加载了,下面直接返回true，表示记载成功
    VLOG(jni) << "[Shared library \"" << path << "\" already loaded in "
              << " ClassLoader " << class_loader << "]";
    if (!library->CheckOnLoadResult()) {
      StringAppendF(error_msg, "JNI_OnLoad failed on a previous attempt "
          "to load \"%s\"", path.c_str());
      return false;
    }
    return true;
  }

  这里library 为空，第一次加载so
  // Open the shared library.  Because we're using a full path, the system
  // doesn't have to search through LD_LIBRARY_PATH.  (It may do so to
  // resolve this library's dependencies though.)

  // Failures here are expected when java.library.path has several entries
  // and we have to hunt for the lib.

  // Below we dlopen but there is no paired dlclose, this would be necessary if we supported
  // class unloading. Libraries will only be unloaded when the reference count (incremented by
  // dlopen) becomes zero from dlclose.

  Locks::mutator_lock_->AssertNotHeld(self);
  得到要加载的so的全路径
  const char* path_str = path.empty() ? nullptr : path.c_str();
  加载这个so,并且得到这个加载so的返回值，是否成功
  void* handle = android::OpenNativeLibrary(env,
                                            runtime_->GetTargetSdkVersion(),
                                            path_str,
                                            class_loader,
                                            library_path);

  bool needs_native_bridge = false;
  if (handle == nullptr) {
    if (android::NativeBridgeIsSupported(path_str)) {
      handle = android::NativeBridgeLoadLibrary(path_str, RTLD_NOW);
      needs_native_bridge = true;
    }
  }

  VLOG(jni) << "[Call to dlopen(\"" << path << "\", RTLD_NOW) returned " << handle << "]";

  如果handle为nullptr标识加载so失败
  if (handle == nullptr) {
    *error_msg = dlerror();
    VLOG(jni) << "dlopen(\"" << path << "\", RTLD_NOW) failed: " << *error_msg;
    return false;
  }
  下面就表示已经成功打开了这个so
  判断是否有异常产生
  if (env->ExceptionCheck() == JNI_TRUE) {
    LOG(ERROR) << "Unexpected exception:";
    env->ExceptionDescribe();
    env->ExceptionClear();
  }
  // Create a new entry.
  // TODO: move the locking (and more of this logic) into Libraries.
  
  bool created_library = false;
  {
    // Create SharedLibrary ahead of taking the libraries lock to maintain lock ordering.
	智能指针,创建一个SharedLibrary指针对象new_library，将成功打开的结果重新的封装成一个library指针对象
    std::unique_ptr<SharedLibrary> new_library(
        new SharedLibrary(env, self, path, handle, class_loader, class_loader_allocator));
    MutexLock mu(self, *Locks::jni_libraries_lock_);
    library = libraries_->Get(path);
    if (library == nullptr) {  // We won race to get libraries_lock.
      library = new_library.release();
	  将library添加到了libraries_里面，以后再次访问这个so就不用重新的打开了
      libraries_->Put(path, library);
      created_library = true;
    }
  }
  if (!created_library) {
    LOG(INFO) << "WOW: we lost a race to add shared library: "
        << "\"" << path << "\" ClassLoader=" << class_loader;
    return library->CheckOnLoadResult();
  }
  VLOG(jni) << "[Added shared library \"" << path << "\" for ClassLoader " << class_loader << "]";

  bool was_successful = false;
  定义了一个void *类型的指针
  void* sym;
  if (needs_native_bridge) {
    library->SetNeedsNativeBridge();
  }
  再成功加载了so之后，找到JNI_OnLoad函数
  sym = library->FindSymbol("JNI_OnLoad", nullptr);
  if (sym == nullptr) {
    VLOG(jni) << "[No JNI_OnLoad found in \"" << path << "\"]";
    was_successful = true;
  } else {
    // Call JNI_OnLoad.  We have to override the current class
    // loader, which will always be "null" since the stuff at the
    // top of the stack is around Runtime.loadLibrary().  (See
    // the comments in the JNI FindClass function.)
    ScopedLocalRef<jobject> old_class_loader(env, env->NewLocalRef(self->GetClassLoaderOverride()));
    self->SetClassLoaderOverride(class_loader);

    VLOG(jni) << "[Calling JNI_OnLoad in \"" << path << "\"]";
    typedef int (*JNI_OnLoadFn)(JavaVM*, void*);
	将这个void* 类型的变量强制的转换成一个JNI_OnLoadFn 类型的指针函数指针
    JNI_OnLoadFn jni_on_load = reinterpret_cast<JNI_OnLoadFn>(sym);
	这里执行了这个函数指针，这也就是为什么Jni头文件里面有这样的JNIEXPORT jint JNICALL JNI_OnLoad(JavaVM* vm, void* reserved);
	下面就是调用了这个函数，将结果传递过来，这里只会在加载so第一次调用
    int version = (*jni_on_load)(this, nullptr);

    if (runtime_->GetTargetSdkVersion() != 0 && runtime_->GetTargetSdkVersion() <= 21) {
      fault_manager.EnsureArtActionInFrontOfSignalChain();
    }

    self->SetClassLoaderOverride(old_class_loader.get());

	
	当加载so成功之后，还要检查jni的版本号IsBadJniVersion(version) return version != JNI_VERSION_1_2 && version != JNI_VERSION_1_4 && version != JNI_VERSION_1_6;

    if (version == JNI_ERR) {
      StringAppendF(error_msg, "JNI_ERR returned from JNI_OnLoad in \"%s\"", path.c_str());
    } else if (IsBadJniVersion(version)) {
      StringAppendF(error_msg, "Bad JNI version returned from JNI_OnLoad in \"%s\": %d",
                    path.c_str(), version);
      // It's unwise to call dlclose() here, but we can mark it
      // as bad and ensure that future load attempts will fail.
      // We don't know how far JNI_OnLoad got, so there could
      // be some partially-initialized stuff accessible through
      // newly-registered native method calls.  We could try to
      // unregister them, but that doesn't seem worthwhile.
    } else {
      was_successful = true;
    }
    VLOG(jni) << "[Returned " << (was_successful ? "successfully" : "failure")
              << " from JNI_OnLoad in \"" << path << "\"]";
  }
  标识已经加载成功
  library->SetResult(was_successful);
  return was_successful;
}

android::OpenNativeLibrary()函数的实现为： 路径为native_loader.cpp文件中
void* OpenNativeLibrary(JNIEnv* env,
                        int32_t target_sdk_version,
                        const char* path,
                        jobject class_loader,
                        jstring library_path) {
#if defined(__ANDROID__)
  UNUSED(target_sdk_version);
  if (class_loader == nullptr) {
    return dlopen(path, RTLD_NOW);
  }

  std::lock_guard<std::mutex> guard(g_namespaces_mutex);
  android_namespace_t* ns = g_namespaces->FindNamespaceByClassLoader(env, class_loader);

  if (ns == nullptr) {
    // This is the case where the classloader was not created by ApplicationLoaders
    // In this case we create an isolated not-shared namespace for it.
    ns = g_namespaces->Create(env, class_loader, false, library_path, nullptr);
    if (ns == nullptr) {
      return nullptr;
    }
  }

  android_dlextinfo extinfo;
  extinfo.flags = ANDROID_DLEXT_USE_NAMESPACE;
  extinfo.library_namespace = ns;

  return android_dlopen_ext(path, RTLD_NOW, &extinfo);
#else
  UNUSED(env, target_sdk_version, class_loader, library_path);
  return dlopen(path, RTLD_NOW);
#endif
}

执行 android_dlopen_ext(path, RTLD_NOW, &extinfo); 路径为dlfcn.cpp
void* android_dlopen_ext(const char* filename, int flags, const android_dlextinfo* extinfo) {
  void* caller_addr = __builtin_return_address(0);
  return dlopen_ext(filename, flags, extinfo, caller_addr);
}

static void* dlopen_ext(const char* filename, int flags,
                        const android_dlextinfo* extinfo, void* caller_addr) {
  ScopedPthreadMutexLocker locker(&g_dl_mutex);
  void* result = do_dlopen(filename, flags, extinfo, caller_addr);
  if (result == nullptr) {
    __bionic_format_dlerror("dlopen failed", linker_get_error_buffer());
    return nullptr;
  }
  return result;
}
do_dlopen(filename, flags, extinfo, caller_addr);函数实现为 路径为linker.cpp文件
void* do_dlopen(const char* name, int flags, const android_dlextinfo* extinfo,
                  void* caller_addr) {
  .......
  ProtectedDataGuard guard;
  soinfo* si = find_library(ns, translated_name, flags, extinfo, caller);
  if (si != nullptr) {
    si->call_constructors();
    return si->to_handle();
  }

  return nullptr;
}
si->to_handle()函数实现为

 ```