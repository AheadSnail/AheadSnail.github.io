---
layout: pager
title: apk的启动流程
date: 2017-10-29 20:10:22
tags: [Android]
description: apk的启动流程
---

Android 系统启动流程
<!--more-->

**Android 系统启动流程，这边对应的为android源码6.0**
===

```java
手机开机会启动init.rc脚本文件，会加载编译好的init文件，这俩个可执行的文件，可以在android里面找到
init文件是 android源码中的/system/core/init文件里面编译的产物，里面提供了Android.mk文件
函数的入口为init.cpp中的main函数,其中mian函数里面有一句这样的代码 if(execv(path, args) == -1)
就会开启对应的android源码中/frameworkers/base/cmds目录下的所有的可执行的文件，注意这个目录下的文件夹都可以编译成一个可以执行的文件
其中包括了开启虚拟机的文件app_process文件夹，里面也提供了Android.mk文件，对应的编译文件app_main也是含有main函数入口的
其中的main函数中有这样的代码 
AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv));
runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
这个就是开启ZygoteInit进程，而且ZygoteInit文件为一个java文件，所以这也是虚拟机执行的第一个java文件ZygoteInit.java
我们找到对应的这个runtime文件,而对应的文件为android源码目录下的/frameworks/base/core/jni/AndroidRuntime.cpp文件其中的start方法为

/*
 * Start the Android runtime.  This involves starting the virtual machine
 * and calling the "static void main(String[] args)" method in the class
 * named by "className".
 *
 * Passes the main function two arguments, the class name and the specified
 * options string.
 */
void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote)
{

	......
	/* start the virtual machine */
    JniInvocation jni_invocation;
    jni_invocation.Init(NULL);
    JNIEnv* env;
	开启虚拟机，并且给env赋值,mJavaVm赋值，传递的是二级指针
    if (startVm(&mJavaVM, &env, zygote) != 0) {
        return;
    }
    onVmCreated(env);
	.....

    /*
     * Start VM.  This thread becomes the main thread of the VM, and will
     * not return until the VM exits.
     */
    char* slashClassName = toSlashClassName(className);
	因为上面已经给env赋值了，所以这里可以调用env对应的的FindClass方法，这里的slashClassName 就为com.android.internal.os.ZygoteInit
    jclass startClass = env->FindClass(slashClassName);
    if (startClass == NULL) {
        ALOGE("JavaVM unable to locate class '%s'\n", slashClassName);
        /* keep going */
    } else {
        jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
            "([Ljava/lang/String;)V");
        if (startMeth == NULL) {
            ALOGE("JavaVM unable to find main() in '%s'\n", className);
            /* keep going */
        } else {
            env->CallStaticVoidMethod(startClass, startMeth, strArray);

#if 0
            if (env->ExceptionCheck())
                threadExitUncaughtException(env);
#endif
        }
    }
    free(slashClassName);
}



这里先看  if (startVm(&mJavaVM, &env, zygote) != 0) 的函数实现
int AndroidRuntime::startVm(JavaVM** pJavaVM, JNIEnv** pEnv, bool zygote)
{	
	........
	
 /*
     * Retrieve the build fingerprint and provide it to the runtime. That way, ANR dumps will
     * contain the fingerprint and can be parsed.
     */
    parseRuntimeOption("ro.build.fingerprint", fingerprintBuf, "-Xfingerprint:");
	这也可以验证了在重写JNI中的onLoad方法中的返回值为什么最低要1_4以上
    initArgs.version = JNI_VERSION_1_4;
    initArgs.options = mOptions.editArray();
    initArgs.nOptions = mOptions.size();
    initArgs.ignoreUnrecognized = JNI_FALSE;

    /*
     * Initialize the VM.
     *
     * The JavaVM* is essentially per-process, and the JNIEnv* is per-thread.
     * If this call succeeds, the VM is ready, and we can start issuing
     * JNI calls.
     */
    if (JNI_CreateJavaVM(pJavaVM, pEnv, &initArgs) < 0) {
        ALOGE("JNI_CreateJavaVM failed\n");
        return -1;
    }
}
查看if (JNI_CreateJavaVM(pJavaVM, pEnv, &initArgs) < 0) 函数的执行原理,这个文件并不存在当前的文件里面，这个函数在当前引进的头文件JniInvocation.h文件
函数的实现在android源码目录下/libnativehelper/JniInvocation.cpp文件中
jint JniInvocation::JNI_CreateJavaVM(JavaVM** p_vm, JNIEnv** p_env, void* vm_args) {
  return JNI_CreateJavaVM_(p_vm, p_env, vm_args);
}

其中的JNI_CreateJavaVM_为函数的指针，这个在上面startVm之前有这样的代码  
JniInvocation jni_invocation;
jni_invocation.Init(NULL);
看 jni_invocation.Init(NULL);函数的实现， 这边传递的library为null
bool JniInvocation::Init(const char* library) {
#ifdef HAVE_ANDROID_OS
  char buffer[PROPERTY_VALUE_MAX];
#else
  char* buffer = NULL;
#endif
   执行GetLibrary方法，返回要加载的so
  library = GetLibrary(library, buffer);
  调用dlopen打开这个so，并且返回这个句柄
  handle_ = dlopen(library, RTLD_NOW);
  打开so失败
  if (handle_ == NULL) {
    if (strcmp(library, kLibraryFallback) == 0) {
      // Nothing else to try.
      ALOGE("Failed to dlopen %s: %s", library, dlerror());
      return false;
    }
    // Note that this is enough to get something like the zygote
    // running, we can't property_set here to fix this for the future
    // because we are root and not the system user. See
    // RuntimeInit.commonInit for where we fix up the property to
    // avoid future fallbacks. http://b/11463182
    ALOGW("Falling back from %s to %s after dlopen error: %s",
          library, kLibraryFallback, dlerror());
    library = kLibraryFallback;
    handle_ = dlopen(library, RTLD_NOW);
    if (handle_ == NULL) {
      ALOGE("Failed to dlopen %s: %s", library, dlerror());
      return false;
    }
  }
  成功打开so
  if (!FindSymbol(reinterpret_cast<void**>(&JNI_GetDefaultJavaVMInitArgs_),
                  "JNI_GetDefaultJavaVMInitArgs")) {
    return false;
  }
  调用FindSymbol方法，给JNI_CreateJavaVM_变量赋值，所以这个方法在这个so里面的对应的为JNI_CreateJavaVM，也就是libart.so,可以肯定的是肯定是在art文件夹里面的某一个文件
  if (!FindSymbol(reinterpret_cast<void**>(&JNI_CreateJavaVM_),
                  "JNI_CreateJavaVM")) {
    return false;
  }
  if (!FindSymbol(reinterpret_cast<void**>(&JNI_GetCreatedJavaVMs_),
                  "JNI_GetCreatedJavaVMs")) {
    return false;
  }
  return true;
}
看 library = GetLibrary(library, buffer);函数的实现 其中static const char* kLibraryFallback = "libart.so";
const char* JniInvocation::GetLibrary(const char* library, char* buffer) {
#ifdef HAVE_ANDROID_OS
  const char* default_library;
  char debuggable[PROPERTY_VALUE_MAX];
  property_get(kDebuggableSystemProperty, debuggable, kDebuggableFallback);

  if (strcmp(debuggable, "1") != 0) {
    // Not a debuggable build.
    // Do not allow arbitrary library. Ignore the library parameter. This
    // will also ignore the default library, but initialize to fallback
    // for cleanliness.
    library = kLibraryFallback;
    default_library = kLibraryFallback;
  } else {
    // Debuggable build.
    // Accept the library parameter. For the case it is NULL, load the default
    // library from the system property.
    if (buffer != NULL) {
      property_get(kLibrarySystemProperty, buffer, kLibraryFallback);
      default_library = buffer;
    } else {
      // No buffer given, just use default fallback.
      default_library = kLibraryFallback;
    }
  }
#else
  UNUSED(buffer);
  const char* default_library = kLibraryFallback;
#endif
  if (library == NULL) {
    library = default_library;
  }
  return library;
}
 调用FindSymbol方法，给JNI_CreateJavaVM_变量赋值，所以这个方法是在这个so里面的JNI_CreateJavaVM，也就是libart.so 对应的文件为
 在android源码目录下面的/art/runtime/java_vm_ext.cc文件中,证明了上面所说的一定在art文件里面的某一个文件
 extern "C" jint JNI_CreateJavaVM(JavaVM** p_vm, JNIEnv** p_env, void* vm_args) {
  ATRACE_BEGIN(__FUNCTION__);
  const JavaVMInitArgs* args = static_cast<JavaVMInitArgs*>(vm_args);
  if (IsBadJniVersion(args->version)) {
    LOG(ERROR) << "Bad JNI version passed to CreateJavaVM: " << args->version;
    ATRACE_END();
    return JNI_EVERSION;
  }
  RuntimeOptions options;
  for (int i = 0; i < args->nOptions; ++i) {
    JavaVMOption* option = &args->options[i];
    options.push_back(std::make_pair(std::string(option->optionString), option->extraInfo));
  }
  bool ignore_unrecognized = args->ignoreUnrecognized;
  if (!Runtime::Create(options, ignore_unrecognized)) {
    ATRACE_END();
    return JNI_ERR;
  }
  Runtime* runtime = Runtime::Current();
  bool started = runtime->Start();
  if (!started) {
    delete Thread::Current()->GetJniEnv();
    delete runtime->GetJavaVM();
    LOG(WARNING) << "CreateJavaVM failed";
    ATRACE_END();
    return JNI_ERR;
  }
  这边就是给JNIEnv* 赋值的过程
  *p_env = Thread::Current()->GetJniEnv();
  *p_vm = runtime->GetJavaVM();
  ATRACE_END();
  return JNI_OK;
}

当执行到  jclass startClass = env->FindClass(slashClassName);函数的声明是在jni.h文件中的，函数的定义是在jni_internal.cc文件中
找到对应的函数为
  static jclass FindClass(JNIEnv* env, const char* name) {
    CHECK_NON_NULL_ARGUMENT(name);
    Runtime* runtime = Runtime::Current();
    ClassLinker* class_linker = runtime->GetClassLinker();
    std::string descriptor(NormalizeJniClassDescriptor(name));
    ScopedObjectAccess soa(env);
    mirror::Class* c = nullptr;
    if (runtime->IsStarted()) {
      StackHandleScope<1> hs(soa.Self());
      Handle<mirror::ClassLoader> class_loader(hs.NewHandle(GetClassLoader(soa)));
      c = class_linker->FindClass(soa.Self(), descriptor.c_str(), class_loader);
    } else {
      c = class_linker->FindSystemClass(soa.Self(), descriptor.c_str());
    }
    return soa.AddLocalReference<jclass>(c);
  }
发现最终都是调用了 c = class_linker->FindClass(soa.Self(), descriptor.c_str(), class_loader); ，    ClassLinker* class_linker = runtime->GetClassLinker();函数的实现为
mirror::Class* ClassLinker::FindClass(Thread* self, const char* descriptor,
                                      Handle<mirror::ClassLoader> class_loader) {
  ......
  const size_t hash = ComputeModifiedUtf8Hash(descriptor);
  // Find the class in the loaded classes table.
  首先判断要加载的类是否已经存在classTab文件中,如果存在，直接取出
  mirror::Class* klass = LookupClass(self, descriptor, hash, class_loader.Get());
  if (klass != nullptr) {
    return EnsureResolved(self, descriptor, klass);
  }
  // Class is not yet loaded.
  if (descriptor[0] == '[') {
    return CreateArrayClass(self, descriptor, hash, class_loader);
  } else if (class_loader.Get() == nullptr) {
    // The boot class loader, search the boot class path.
    ClassPathEntry pair = FindInClassPath(descriptor, hash, boot_class_path_);
    if (pair.second != nullptr) {
      return DefineClass(self, descriptor, hash, NullHandle<mirror::ClassLoader>(), *pair.first,
                         *pair.second);
    } else {
      // The boot class loader is searched ahead of the application class loader, failures are
      // expected and will be wrapped in a ClassNotFoundException. Use the pre-allocated error to
      // trigger the chaining with a proper stack trace.
      mirror::Throwable* pre_allocated = Runtime::Current()->GetPreAllocatedNoClassDefFoundError();
      self->SetException(pre_allocated);
      return nullptr;
    }
  }
  ......
}
调用了 return DefineClass(self, descriptor, hash, NullHandle<mirror::ClassLoader>(), *pair.first,*pair.second);
mirror::Class* ClassLinker::DefineClass(Thread* self, const char* descriptor, size_t hash,
                                        Handle<mirror::ClassLoader> class_loader,
                                        const DexFile& dex_file,
                                        const DexFile::ClassDef& dex_class_def) {
  .......
  if (klass.Get() == nullptr) {
    // Allocate a class with the status of not ready.
    // Interface object should get the right size here. Regular class will
    // figure out the right size later and be replaced with one of the right
    // size when the class becomes resolved.
    klass.Assign(AllocClass(self, SizeOfClassWithoutEmbeddedTables(dex_file, dex_class_def)));
  }
  ....
  SetupClass(dex_file, dex_class_def, klass, class_loader.Get());

  // Add the newly loaded class to the loaded classes table.
  //首先将新的要记在的class，添加到classTab中
  mirror::Class* existing = InsertClass(descriptor, klass.Get(), hash);
  if (existing != nullptr) {
    // We failed to insert because we raced with another thread. Calling EnsureResolved may cause
    // this thread to block.
    return EnsureResolved(self, descriptor, existing);
  }

  // Load the fields and other things after we are inserted in the table. This is so that we don't
  // end up allocating unfree-able linear alloc resources and then lose the race condition. The
  // other reason is that the field roots are only visited from the class table. So we need to be
  // inserted before we allocate / fill in these fields.
  //加载这个class关键实现
  LoadClass(self, dex_file, dex_class_def, klass);
  if (self->IsExceptionPending()) {
    // An exception occured during load, set status to erroneous while holding klass' lock in case
    // notification is necessary.
    if (!klass->IsErroneous()) {
      mirror::Class::SetStatus(klass, mirror::Class::kStatusError, self);
    }
    return nullptr;
  }
	......
}
LoadClass(self, dex_file, dex_class_def, klass);函数的实现为：
void ClassLinker::LoadClass(Thread* self, const DexFile& dex_file,const DexFile::ClassDef& dex_class_def, Handle<mirror::Class> klass) {
  这边可以看的出来，我们的class文件，一开始是从dex文件里面获取出来的，因为一开始class文件都是存放在dex文件的，需要加载的时候，才会加载进来
  const uint8_t* class_data = dex_file.GetClassData(dex_class_def);
  if (class_data == nullptr) {
    return;  // no fields or methods - for example a marker interface
  }
  bool has_oat_class = false;
  if (Runtime::Current()->IsStarted() && !Runtime::Current()->IsAotCompiler()) {
    OatFile::OatClass oat_class = FindOatClass(dex_file, klass->GetDexClassDefIndex(),
                                               &has_oat_class);
    if (has_oat_class) {
	  加载class对应的成员变量跟成员函数
      LoadClassMembers(self, dex_file, class_data, klass, &oat_class);
    }
  }
  if (!has_oat_class) {
    LoadClassMembers(self, dex_file, class_data, klass, nullptr);
  }
}  
LoadClassMembers(self, dex_file, class_data, klass, &oat_class)；函数的实现为：
void ClassLinker::LoadClassMembers(Thread* self, const DexFile& dex_file,
                                   const uint8_t* class_data,
                                   Handle<mirror::Class> klass,
                                   const OatFile::OatClass* oat_class) {
  {
	
    // Note: We cannot have thread suspension until the field and method arrays are setup or else
    // Class::VisitFieldRoots may miss some fields or methods.
    ScopedAssertNoThreadSuspension nts(self, __FUNCTION__);
    // Load static fields.  加载静态的成员变量
    ClassDataItemIterator it(dex_file, class_data);
    const size_t num_sfields = it.NumStaticFields();
    ArtField* sfields = num_sfields != 0 ? AllocArtFieldArray(self, num_sfields) : nullptr;
    for (size_t i = 0; it.HasNextStaticField(); i++, it.Next()) {
      CHECK_LT(i, num_sfields);
      LoadField(it, klass, &sfields[i]);
    }
    klass->SetSFields(sfields);
    klass->SetNumStaticFields(num_sfields);
    DCHECK_EQ(klass->NumStaticFields(), num_sfields);
    // Load instance fields.
    const size_t num_ifields = it.NumInstanceFields();
    ArtField* ifields = num_ifields != 0 ? AllocArtFieldArray(self, num_ifields) : nullptr;
    for (size_t i = 0; it.HasNextInstanceField(); i++, it.Next()) {
      CHECK_LT(i, num_ifields);
      LoadField(it, klass, &ifields[i]);
    }
    klass->SetIFields(ifields);
    klass->SetNumInstanceFields(num_ifields);
    DCHECK_EQ(klass->NumInstanceFields(), num_ifields);
	
	
    // Load methods. 加载方法
    if (it.NumDirectMethods() != 0) {
      klass->SetDirectMethodsPtr(AllocArtMethodArray(self, it.NumDirectMethods()));
    }
    klass->SetNumDirectMethods(it.NumDirectMethods());
	
	如果class里面含有方法的时候，分配内存空间
    if (it.NumVirtualMethods() != 0) {
      klass->SetVirtualMethodsPtr(AllocArtMethodArray(self, it.NumVirtualMethods()));
    }
    klass->SetNumVirtualMethods(it.NumVirtualMethods());
    size_t class_def_method_index = 0;
    uint32_t last_dex_method_index = DexFile::kDexNoIndex;
    size_t last_class_def_method_index = 0;
    for (size_t i = 0; it.HasNextDirectMethod(); i++, it.Next()) {
      ArtMethod* method = klass->GetDirectMethodUnchecked(i, image_pointer_size_);
      LoadMethod(self, dex_file, it, klass, method);
      LinkCode(method, oat_class, class_def_method_index);
      uint32_t it_method_index = it.GetMemberIndex();
      if (last_dex_method_index == it_method_index) {
        // duplicate case
        method->SetMethodIndex(last_class_def_method_index);
      } else {
        method->SetMethodIndex(class_def_method_index);
        last_dex_method_index = it_method_index;
        last_class_def_method_index = class_def_method_index;
      }
      class_def_method_index++;
    }
    for (size_t i = 0; it.HasNextVirtualMethod(); i++, it.Next()) {
      ArtMethod* method = klass->GetVirtualMethodUnchecked(i, image_pointer_size_);
      LoadMethod(self, dex_file, it, klass, method);
      DCHECK_EQ(class_def_method_index, it.NumDirectMethods() + i);
      LinkCode(method, oat_class, class_def_method_index);
      class_def_method_index++;
    }
    DCHECK(!it.HasNext());
  }
  self->AllowThreadSuspension();
}
AllocArtMethodArray(self, it.NumVirtualMethods()) 函数的实现为：
ArtMethod* ClassLinker::AllocArtMethodArray(Thread* self, size_t length) {
  这边可以看出来，存储class里面的大小是固定的,可以通过下面的这个方法获取到
  const size_t method_size = ArtMethod::ObjectSize(image_pointer_size_);
  
  给class的方法分配内存空间，并且返回这块内存空间的首地址，length代表有多少个方法存在，所以，我们可以得出一个结论就是class里面的method的存储是连续的，所以 xpose可以根据俩个方法相
  减得到内存偏移量，一次的拷贝，解决机型适配的问题
  uintptr_t ptr = reinterpret_cast<uintptr_t>(
      Runtime::Current()->GetLinearAlloc()->Alloc(self, method_size * length));
  CHECK_NE(ptr, 0u);
  通过地址的偏移量给这块内存空间，
  for (size_t i = 0; i < length; ++i) {
    new(reinterpret_cast<void*>(ptr + i * method_size)) ArtMethod;
  }
  return reinterpret_cast<ArtMethod*>(ptr);
}
```



