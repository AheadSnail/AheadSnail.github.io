---
layout: pager
title: 'Android系统启动流程(二),ServiceManager，Zygote启动'
date: 2018-01-08 09:26:34
tags: [Android,ServiceManager,Zygote]
description:  Android 理解ServiceManager启动,Zygote启动过程
---

Android 理解ServiceManager启动,Zygote启动过程
<!--more-->

**servicemanager.rc文件的解析**
===
```java
继上一篇文章介绍 在system/core/init/builtins.cpp中实现了do_mount_all函数
static int do_mount_all(const std::vector<std::string>& args) {  
    pid_t pid;  
    int ret = -1;  
    int child_ret = -1;  
    int status;  
    struct fstab *fstab;  
  
    const char* fstabfile = args[1].c_str();  
    /* 
     * Call fs_mgr_mount_all() to mount all filesystems.  We fork(2) and 
     * do the call in the child to provide protection to the main init 
     * process if anything goes wrong (crash or memory leak), and wait for 
     * the child to finish in the parent. 
     */ 
    pid = fork(); 
	
	 /* Paths of .rc files are specified at the 2nd argument and beyond */ 解析指定路径之下的rc文件
    import_late(args, 2);

	..............  
    if (ret == FS_MGR_MNTALL_DEV_NEEDS_ENCRYPTION) {  
        ActionManager::GetInstance().QueueEventTrigger("encrypt");  
    } else if (ret == FS_MGR_MNTALL_DEV_MIGHT_BE_ENCRYPTED) {  
        property_set("ro.crypto.state", "encrypted");  
        property_set("ro.crypto.type", "block");  
        ActionManager::GetInstance().QueueEventTrigger("defaultcrypto");  
    } else if (ret == FS_MGR_MNTALL_DEV_NOT_ENCRYPTED) {  
        property_set("ro.crypto.state", "unencrypted");  
        ActionManager::GetInstance().QueueEventTrigger("nonencrypted");  //将nonencrypted加入trigger_queue_  
    } else if (ret == FS_MGR_MNTALL_DEV_NOT_ENCRYPTABLE) {  
        property_set("ro.crypto.state", "unsupported");  
        ActionManager::GetInstance().QueueEventTrigger("nonencrypted");  //将nonencrypted加入trigger_queue_  
    } else if (ret == FS_MGR_MNTALL_DEV_NEEDS_RECOVERY) {  
        /* Setup a wipe via recovery, and reboot into recovery */  
        ERROR("fs_mgr_mount_all suggested recovery, so wiping data via recovery.\n");  
        ret = wipe_data_via_recovery("wipe_data_via_recovery");  
        /* If reboot worked, there is no return. */  
    } else if (ret == FS_MGR_MNTALL_DEV_FILE_ENCRYPTED) {  
        if (e4crypt_install_keyring()) {  
            return -1;  
        }  
        property_set("ro.crypto.state", "encrypted");  
        property_set("ro.crypto.type", "file");  
  
        // Although encrypted, we have device key, so we do not need to  
        // do anything different from the nonencrypted case.  
        ActionManager::GetInstance().QueueEventTrigger("nonencrypted");  
    } else if (ret > 0) {  
        ERROR("fs_mgr_mount_all returned unexpected error %d\n", ret);  
    }  
    /* else ... < 0: error */  
  
    return ret;  
}

分析这个方法的实现
import_late(args, 2);
//大致意思就是说解析指定路径下的rc文件
/* Imports .rc files from the specified paths. Default ones are applied if none is given.
 *
 * start_index: index of the first path in the args list
 */
static void import_late(const std::vector<std::string>& args, size_t start_index) {
	//得到Parser对象
    Parser& parser = Parser::GetInstance();
    if (args.size() <= start_index) {
        // Use the default set if no path is given
		//指定要查找的路径
        static const std::vector<std::string> init_directories = {
            "/system/etc/init",
            "/vendor/etc/init",
            "/odm/etc/init"
        };
		//遍历解析每一个路径之下的文件
        for (const auto& dir : init_directories) {
            parser.ParseConfig(dir);
        }
    } else {
        for (size_t i = start_index; i < args.size(); ++i) {
            parser.ParseConfig(args[i]);
        }
    }
}


在Android 7之前的版本中，系统Native服务，不管它们的可执行文件位于系统什么位置都定义在根分区的init.*.rc文件中。这造成init＊.rc文件臃肿庞大，给维护带来了一些不便，
而且其中定义的一些服务的二进制文件根本不存在。但在Android 7.0之后，对该机制做了一些改变 。

单一的init＊.rc，被拆分，服务根据其二进制文件的位置（／system，／vendor，／odm）定义到对应分区的etc／init目录中，每个服务一个rc文件。与该服务相关的触发器、
操作等也定义在同一rc文件中。 
/system/etc/init，包含系统核心服务的定义，如SurfaceFlinger、MediaServer、Logcatd等。
/vendor/etc/init， SOC厂商针对SOC核心功能定义的一些服务。比如高通、MTK某一款SOC的相关的服务。
/odm/etc/init，OEM/ODM厂商如小米、华为、OPP其产品所使用的外设以及差异化功能相关的服务。
```

采用的是google自带的7.1的模拟器跟源码是对应上的
![结果显示](/uploads/Android启动流程二/ServiceManager文件.png)

```
通过上图可以发现在/system/etc/init 目录下面有一个ServiceManager.rc文件，对应的源码目录为frameworks/native/cmds/servicemanager/service_manager.c
```
![结果显示](/uploads/Android启动流程二/ServiceManager源码路径.png)

查看servicemanager.rc文件的内容
```
service servicemanager /system/bin/servicemanager
    class core       //class 名为core
    user system
    group system readproc
    critical     
    onrestart restart healthd
    onrestart restart zygote
    onrestart restart audioserver
    onrestart restart media
    onrestart restart surfaceflinger
    onrestart restart inputflinger
    onrestart restart drm
    onrestart restart cameraserver
    writepid /dev/cpuset/system-background/tasks

上一篇文章中有介绍到了service的解析过程，这里的class core代表的 core代表服务要启动的class名，就跟class main 一样，main 也为要启动的class名,core服务都是系统最基本的服务，
只要core服务全部启动，手机此时是可以运行的，但是却看不到东西，core服务都是系统最基本的服务，只要core服务全部启动，手机此时是可以运行的，但是却看不到东西，原因是framework没有启动。
当执行到  parser.ParseConfig(dir);的时候，就会解析servicemanager.rc中的service标签，统一的添加进ServiceManager中services_成员变量中,ServiceManager为c++的静态成员类
```

**zygote的启动**
===

```java
在system/core/init/builtins.cpp中实现了do_mount_all函数
static int do_mount_all(const std::vector<std::string>& args) 函数中执行了 import_late(args, 2);之后，继续往下执行，当执行到
ActionManager::GetInstance().QueueEventTrigger("nonencrypted");  //将nonencrypted加入trigger_queue_  就会触发下面nonencrypted 的执行，下面为nonencrypted
on nonencrypted    //触发action   
    # A/B update verifier that marks a successful boot.  
    exec - root -- /system/bin/update_verifier nonencrypted  
    class_start main         //调用do_class_start函数将class name为main的services启动起来  
    class_start late_start  

而zygote的rc文件的内容为
service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server
    class main    //class name为main  
    socket zygote stream 660 root system
    onrestart write /sys/android_power/request_state wake
    onrestart write /sys/power/state on
    onrestart restart audioserver
    onrestart restart cameraserver
    onrestart restart media
    onrestart restart netd
    writepid /dev/cpuset/foreground/tasks
	
class_start 对应的函数为
还是在system/core/init/builtins.cpp中实现了do_class_start函数   {"class_start",             {1,     1,    do_class_start}},
static int do_class_start(const std::vector<std::string>& args) {  
        /* Starting a class does not start services 
         * which are explicitly disabled.  They must 
         * be started individually. 
         */  
    ServiceManager::GetInstance().  
        ForEachServiceInClass(args[1], [] (Service* s) { s->StartIfNotDisabled(); });  
    return 0;  
}  

bool Service::StartIfNotDisabled() {  
    if (!(flags_ & SVC_DISABLED)) {  
        return Start();          //如果services不是disable,就将start()函数返回  
    } else {  
        flags_ |= SVC_DISABLED_START;  //否则,标记services为disable  
    }  
    return true;  
}  

void ServiceManager::ForEachServiceInClass(const std::string& classname,  
                                           void (*func)(Service* svc)) const {  
    for (const auto& s : services_) {   //遍历services_寻找所有name为main的servies  
        if (classname == s->classname()) {     
            func(s.get());    //如果找到就调用Start()函数  
        }  
    }  
}  

bool Service::Start() {  
    // Starting a service removes it from the disabled or reset state and  
    // immediately takes it out of the restarting state if it was in there.  
    flags_ &= (~(SVC_DISABLED|SVC_RESTARTING|SVC_RESET|SVC_RESTART|SVC_DISABLED_START));  
    time_started_ = 0;  
  
    // Running processes require no additional work --- if they're in the  
    // process of exiting, we've ensured that they will immediately restart  
    // on exit, unless they are ONESHOT.  
    if (flags_ & SVC_RUNNING) {  
        return false;  
    }  
  
    bool needs_console = (flags_ & SVC_CONSOLE);  
    if (needs_console && !have_console) {  
        ERROR("service '%s' requires console\n", name_.c_str());  
        flags_ |= SVC_DISABLED;  
        return false;  
    }  
  
    struct stat sb;  
    if (stat(args_[0].c_str(), &sb) == -1) {  
        ERROR("cannot find '%s' (%s), disabling '%s'\n",  
              args_[0].c_str(), strerror(errno), name_.c_str());  
        flags_ |= SVC_DISABLED;  
        return false;  
    }  
  
    std::string bootmode = property_get("ro.bootmode");  
    if(((!strncmp(name_.c_str(),"healthd",7))||(!strncmp(name_.c_str(),"bootanim",8))||(!strncmp(name_.c_str(),"zygote",6))||(!strncmp(name_.c_str(),"surfaceflinger",14)))&&(bootmode == "charger")){  
  
         ERROR("stopping '%s'\n",name_.c_str());  
  
        flags_ |= SVC_DISABLED;  
  
         return false;  
  
     }  
    std::string scon;  
    if (!seclabel_.empty()) {  
        scon = seclabel_;  
    } else {  
        char* mycon = nullptr;  
        char* fcon = nullptr;  
  
        INFO("computing context for service '%s'\n", args_[0].c_str());  
        int rc = getcon(&mycon);  
        if (rc < 0) {  
            ERROR("could not get context while starting '%s'\n", name_.c_str());  
            return false;  
        }  
  
        rc = getfilecon(args_[0].c_str(), &fcon);  
        if (rc < 0) {  
            ERROR("could not get context while starting '%s'\n", name_.c_str());  
            free(mycon);  
            return false;  
        }  
  
        char* ret_scon = nullptr;  
        rc = security_compute_create(mycon, fcon, string_to_security_class("process"),  
                                     &ret_scon);  
        if (rc == 0) {  
            scon = ret_scon;  
            free(ret_scon);  
        }  
        if (rc == 0 && scon == mycon) {  
            ERROR("Service %s does not have a SELinux domain defined.\n", name_.c_str());  
            free(mycon);  
            free(fcon);  
            return false;  
        }  
        free(mycon);  
        free(fcon);  
        if (rc < 0) {  
            ERROR("could not get context while starting '%s'\n", name_.c_str());  
            return false;  
        }  
    }  
    //上面都是对services进行判断,设置flags, 下面才是重点  
    NOTICE("Starting service '%s'...\n", name_.c_str());  //在kernel log中可以看打印信息  
	
	//创建一个进程
    pid_t pid = fork();     //孵化进程  
    if (pid == 0) { 	 	//pid 为0 的代表是在子进程里面执行的,也即是下面的代码都是在子进程里面执行的
        umask(077);  
  
        for (const auto& ei : envvars_) {  
            add_environment(ei.name.c_str(), ei.value.c_str());  
        }  
  
        for (const auto& si : sockets_) {  
            int socket_type = ((si.type == "stream" ? SOCK_STREAM :  
                                (si.type == "dgram" ? SOCK_DGRAM :  
                                 SOCK_SEQPACKET)));  
            const char* socketcon =  
                !si.socketcon.empty() ? si.socketcon.c_str() : scon.c_str();  
  
            int s = create_socket(si.name.c_str(), socket_type, si.perm,  
                                  si.uid, si.gid, socketcon);  
            if (s >= 0) {  
                PublishSocket(si.name, s);  
            }  
        }  
  
        std::string pid_str = StringPrintf("%d", getpid());  
        for (const auto& file : writepid_files_) {  
            if (!WriteStringToFile(pid_str, file)) {  
                ERROR("couldn't write %s to %s: %s\n",  
                      pid_str.c_str(), file.c_str(), strerror(errno));  
            }  
        }  
  
        if (ioprio_class_ != IoSchedClass_NONE) {  
            if (android_set_ioprio(getpid(), ioprio_class_, ioprio_pri_)) {  
                ERROR("Failed to set pid %d ioprio = %d,%d: %s\n",  
                      getpid(), ioprio_class_, ioprio_pri_, strerror(errno));  
            }  
        }  
  
        if (needs_console) {  
            setsid();  
            OpenConsole();  
        } else {  
            ZapStdio();  
        }  
  
        setpgid(0, getpid());  
        // As requested, set our gid, supplemental gids, and uid.  
        if (gid_) {  
            if (setgid(gid_) != 0) {  
                ERROR("setgid failed: %s\n", strerror(errno));  
                _exit(127);  
            }  
        }  
        if (!supp_gids_.empty()) {  
            if (setgroups(supp_gids_.size(), &supp_gids_[0]) != 0) {  
                ERROR("setgroups failed: %s\n", strerror(errno));  
                _exit(127);  
            }  
        }  
        if (uid_) {  
            if (setuid(uid_) != 0) {  
                ERROR("setuid failed: %s\n", strerror(errno));  
                _exit(127);  
            }  
        }  
        if (!seclabel_.empty()) {  
            if (setexeccon(seclabel_.c_str()) < 0) {  
                ERROR("cannot setexeccon('%s'): %s\n",  
                      seclabel_.c_str(), strerror(errno));  
                _exit(127);  
            }  
        }  
  
        std::vector<char*> strs;  
        for (const auto& s : args_) {  
            strs.push_back(const_cast<char*>(s.c_str()));  
        }  
        strs.push_back(nullptr);  
        if (execve(args_[0].c_str(), (char**) &strs[0], (char**) ENV) < 0) {     //执行system/bin/process程序,进入framework层  
            ERROR("cannot execve('%s'): %s\n", args_[0].c_str(), strerror(errno));  
        }  
  
        _exit(127);  
    }  
  
    if (pid < 0) {  
        ERROR("failed to start '%s'\n", name_.c_str());  
        pid_ = 0;  
        return false;  
    }  
  
    time_started_ = gettime();  
    pid_ = pid;  
    flags_ |= SVC_RUNNING;   //将services标记为running  
    if ((flags_ & SVC_EXEC) != 0) {  
        INFO("SVC_EXEC pid %d (uid %d gid %d+%zu context %s) started; waiting...\n",  
             pid_, uid_, gid_, supp_gids_.size(),  
             !seclabel_.empty() ? seclabel_.c_str() : "default");  
    }  
  
    NotifyStateChange("running");  
    return true;  
}  

调用execve函数用来执行一个可执行的文件，execve(执行文件)在父进程中fork一个子进程，在子进程中调用exec函数启动新的程序。exec函数一共有六个，其中execve为内核级系统调用，
其他(execl，execle，execlp，execv，execvp)都是调用execve的库函数。调用了execve函数就会执行system/bin/app_process程序, 就会进入framework/base/cmds/app_process/app_main.cpp的main函数.
正式进入framework层. 后文将详细分析

```

下面为app_process目录以及生成可执行文件的makefile文件的定义,可以看出生成的目标可执行文件就为app_process,还有区分32位还有64
![结果显示](/uploads/Android启动流程二/app_process可执行文件.png)

app_process 对应于7.1模拟器上的位置为
![结果显示](/uploads/Android启动流程二/app_process可执行文件的路径.png)


**serviceManager的启动**
===

ServiceManager简介

```
ServiceManager是用户空间的一个守护进程，它一直运行在后台。它的职责是管理Binder机制中的各个Server。
当Server启动时，Server会将"Server对象的名字"连同"Server对象的信息"一起注册到ServiceManager中；
而当Client需要获取Server接入点时，则通过"Server的名字"来从ServiceManager中找到对应的Server。
```

```java
在之前的init.rc文件中
on late-init
    trigger early-fs
    trigger fs            //触发
    trigger post-fs

    # Load properties from /system/ + /factory after fs mount. Place
    # this in another action so that the load will be scheduled after the prior
    # issued fs triggers have completed.
    trigger load_system_props_action

    # Now we can mount /data. File encryption requires keymaster to decrypt
    # /data, which in turn can only be loaded when system properties are present
    trigger post-fs-data
    trigger load_persist_props_action

    # Remove a file to wake up anything waiting for firmware.
    trigger firmware_mounts_complete

    trigger early-boot
    trigger boot       //触发

可以发现是先触发了trigger fs 完成了serviceManger.rc文件的解析，启动了zygote之后，触发 trigger boot
on boot
    # basic network init
	..........
    class_start core	  //调用do_class_start函数将class name为core的services启动起来  

service_manager.rc文件中	
service servicemanager /system/bin/servicemanager
    class core        //class 名为core
    user system
    group system readproc
    critical     
    onrestart restart healthd
    onrestart restart zygote
    onrestart restart audioserver
    onrestart restart media
    onrestart restart surfaceflinger
    onrestart restart inputflinger
    onrestart restart drm
    onrestart restart cameraserver
    writepid /dev/cpuset/system-background/tasks	

所以就会启动以core为函数名的service，其中就包括了servicemanager
class_start 对应的函数还是在system/core/init/builtins.cpp中实现了do_class_start函数   {"class_start",             {1,     1,    do_class_start}},
这里的启动服务的过程跟上面启动main为函数名的service是一样的，这里就不在重复了，只是调用execv函数执行的可执行的文件为servicemanager
调用execve执行system/bin/servicemanager程序, 就会进入framework/native/cmds/servicemanager/servicemanager.cpp的main函数.
```

下面为service_manager目录以及生成可执行文件的makefile文件的定义,可以看出生成的目标可执行文件就为servicemanager
![结果显示](/uploads/Android启动流程二/servicemanager源码的路径.png)

app_process 对应于7.1模拟器上的位置为
![结果显示](/uploads/Android启动流程二/serviceManager可执行文件的路径.png)

下面进入servicemanager.cpp中的main函数的入口
```java
int main()
{
    struct binder_state *bs;

	//打开/dev/binder 启动，隐射内存空间
    bs = binder_open(128*1024);
    if (!bs) {
        ALOGE("failed to open binder driver\n");
        return -1;
    }

    if (binder_become_context_manager(bs)) {
        ALOGE("cannot become context manager (%s)\n", strerror(errno));
        return -1;
    }

    selinux_enabled = is_selinux_enabled();
    sehandle = selinux_android_service_context_handle();
    selinux_status_open(true);

    if (selinux_enabled > 0) {
        if (sehandle == NULL) {
            ALOGE("SELinux: Failed to acquire sehandle. Aborting.\n");
            abort();
        }

        if (getcon(&service_manager_context) != 0) {
            ALOGE("SELinux: Failed to acquire service_manager context. Aborting.\n");
            abort();
        }
    }

    union selinux_callback cb;
    cb.func_audit = audit_callback;
    selinux_set_callback(SELINUX_CB_AUDIT, cb);
    cb.func_log = selinux_log_callback;
    selinux_set_callback(SELINUX_CB_LOG, cb);

    binder_loop(bs, svcmgr_handler);

    return 0;
}

struct binder_state *binder_open(size_t mapsize)
{
    struct binder_state *bs;
    struct binder_version vers;

    bs = malloc(sizeof(*bs));
    if (!bs) {
        errno = ENOMEM;
        return NULL;
    }

	//以只读的方式打开/dev/binder 
    bs->fd = open("/dev/binder", O_RDWR | O_CLOEXEC);
    if (bs->fd < 0) {
        fprintf(stderr,"binder: cannot open device (%s)\n",
                strerror(errno));
        goto fail_open;
    }

    if ((ioctl(bs->fd, BINDER_VERSION, &vers) == -1) ||
        (vers.protocol_version != BINDER_CURRENT_PROTOCOL_VERSION)) {
        fprintf(stderr,
                "binder: kernel driver version (%d) differs from user space version (%d)\n",
                vers.protocol_version, BINDER_CURRENT_PROTOCOL_VERSION);
        goto fail_open;
    }

    bs->mapsize = mapsize;
	//mmap(NULL, mapsize, PROT_READ, MAP_PRIVATE, bs->fd, 0)对应会调用驱动的mmap函数。第一个参数是映射内存的起始地址，NULL代表让系统自动选定地址,
	//mapsize大小是128*1024B，即128K；PROT_READ表示映射区域是可读的；MAP_PRIVATE表示建立一个写入时拷贝的私有映射，即，当进程中对该内存区域进行写入时，
	//是写入到映射的拷贝中；bs->fd是"/dev/binder"句柄；而0表示偏移。
    bs->mapped = mmap(NULL, mapsize, PROT_READ, MAP_PRIVATE, bs->fd, 0);
    if (bs->mapped == MAP_FAILED) {
        fprintf(stderr,"binder: cannot map device (%s)\n",
                strerror(errno));
        goto fail_map;
    }

    return bs;

fail_map:
    close(bs->fd);
fail_open:
    free(bs);
    return NULL;
}

```

/dev/binder在系统中的位置为
![结果显示](/uploads/Android启动流程二/dev-binder所在的位置.png)

```java
main函数中binder_loop的函数实现为，第二个参数为函数指针,函数的原型为
int svcmgr_handler(struct binder_state *bs,
                   struct binder_transaction_data *txn,
                   struct binder_io *msg,
                   struct binder_io *reply)
{
    struct svcinfo *si;
    uint16_t *s;
    size_t len;
    uint32_t handle;
    uint32_t strict_policy;
    int allow_isolated;

    //ALOGI("target=%p code=%d pid=%d uid=%d\n",
    //      (void*) txn->target.ptr, txn->code, txn->sender_pid, txn->sender_euid);

    if (txn->target.ptr != BINDER_SERVICE_MANAGER)
        return -1;

    if (txn->code == PING_TRANSACTION)
        return 0;

    // Equivalent to Parcel::enforceInterface(), reading the RPC
    // header with the strict mode policy mask and the interface name.
    // Note that we ignore the strict_policy and don't propagate it
    // further (since we do no outbound RPCs anyway).
    strict_policy = bio_get_uint32(msg);
    s = bio_get_string16(msg, &len);
    if (s == NULL) {
        return -1;
    }

    if ((len != (sizeof(svcmgr_id) / 2)) ||
        memcmp(svcmgr_id, s, sizeof(svcmgr_id))) {
        fprintf(stderr,"invalid id %s\n", str8(s, len));
        return -1;
    }

    if (sehandle && selinux_status_updated() > 0) {
        struct selabel_handle *tmp_sehandle = selinux_android_service_context_handle();
        if (tmp_sehandle) {
            selabel_close(sehandle);
            sehandle = tmp_sehandle;
        }
    }

    switch(txn->code) {
    case SVC_MGR_GET_SERVICE:
    case SVC_MGR_CHECK_SERVICE://用来响应当ServiceManager 得到对应的service
        s = bio_get_string16(msg, &len);
        if (s == NULL) {
            return -1;
        }
		
        handle = do_find_service(s, len, txn->sender_euid, txn->sender_pid);
        if (!handle)
            break;
        bio_put_ref(reply, handle);
        return 0;

    case SVC_MGR_ADD_SERVICE://用来响应当ServiceManager add对应的service
        s = bio_get_string16(msg, &len);
        if (s == NULL) {
            return -1;
        }
        handle = bio_get_ref(msg);
        allow_isolated = bio_get_uint32(msg) ? 1 : 0;
        if (do_add_service(bs, s, len, handle, txn->sender_euid,
            allow_isolated, txn->sender_pid))
            return -1;
        break;

    case SVC_MGR_LIST_SERVICES: {
        uint32_t n = bio_get_uint32(msg);

        if (!svc_can_list(txn->sender_pid, txn->sender_euid)) {
            ALOGE("list_service() uid=%d - PERMISSION DENIED\n",
                    txn->sender_euid);
            return -1;
        }
        si = svclist;
        while ((n-- > 0) && si)
            si = si->next;
        if (si) {
            bio_put_string16(reply, si->name);
            return 0;
        }
        return -1;
    }
    default:
        ALOGE("unknown code %d\n", txn->code);
        return -1;
    }

    bio_put_uint32(reply, 0);
    return 0;
}

void binder_loop(struct binder_state *bs, binder_handler func)
{
    int res;
    struct binder_write_read bwr;
    uint32_t readbuf[32];

    bwr.write_size = 0;
    bwr.write_consumed = 0;
    bwr.write_buffer = 0;

    readbuf[0] = BC_ENTER_LOOPER;
    binder_write(bs, readbuf, sizeof(uint32_t));

    for (;;) {
        bwr.read_size = sizeof(readbuf);
        bwr.read_consumed = 0;
        bwr.read_buffer = (uintptr_t) readbuf;

        res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);

        if (res < 0) {
            ALOGE("binder_loop: ioctl failed (%s)\n", strerror(errno));
            break;
        }

        res = binder_parse(bs, 0, (uintptr_t) readbuf, bwr.read_consumed, func);
        if (res == 0) {
            ALOGE("binder_loop: unexpected reply?!\n");
            break;
        }
        if (res < 0) {
            ALOGE("binder_loop: io error %d %s\n", res, strerror(errno));
            break;
        }
    }
}

int binder_parse(struct binder_state *bs, struct binder_io *bio,
                 uintptr_t ptr, size_t size, binder_handler func)
{
		......
        case BR_TRANSACTION: {
			//函数指针
            struct binder_transaction_data *txn = (struct binder_transaction_data *) ptr;
            if ((end - ptr) < sizeof(*txn)) {
                ALOGE("parse: txn too small!\n");
                return -1;
            }
            binder_dump_txn(txn);
            if (func) {
                unsigned rdata[256/4];
                struct binder_io msg;
                struct binder_io reply;
                int res;

                bio_init(&reply, rdata, sizeof(rdata), 4);
                bio_init_from_txn(&msg, txn);
				//也即是onTransate函数
                res = func(bs, txn, &msg, &reply);
                if (txn->flags & TF_ONE_WAY) {
                    binder_free_buffer(bs, txn->data.ptr.buffer);
                } else {
                    binder_send_reply(bs, &reply, txn->data.ptr.buffer, res);
                }
            }
            ptr += sizeof(*txn);
            break;
        }
	.....
    return r;
}
```

ServiceManager启动流程图
![结果显示](/uploads/Android启动流程二/ServiceManager启动流程图.jpg)

```
上面是ServiceManager的时序图。它启动之后，会先打开"/dev/binder"文件("/dev/binder"是Binder驱动注册的设备节点)。打开文件之后，再告诉Binder驱动，它是Binder的上下文管理者。
之后，就进入到了消息循环中。进入消息循环之后，会不断的从Binder的待处理事务队列中读取事务(Binder请求或反馈)，读出事务之后就进行解析，然后交给相应的进程进行处理。
若没有事务，则进入等待状态，等待被唤醒。
```

如果想要了解Kernel是怎么样处理IBinder的可以查看[IBinder详解](http://wangkuiwu.github.io/2014/09/01/Binder-Introduce/)