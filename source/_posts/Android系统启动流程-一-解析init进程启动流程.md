---
layout: pager
title: Android系统启动流程(一) 解析init进程启动流程
date: 2018-01-05 16:07:02
tags: [Android,Init,Init.rc]
description:  Android 理解Init进程启动,理解Init.rc文件的解析过程
---

Android 理解Init进程启动,理解Init.rc文件的解析过程
<!--more-->

1.init简介
```
init进程是Android系统中用户空间的第一个进程，本身是一个可执行的文件，文件所在的目录系统的根目录下面,是一个具有可以执行权限的文件
并且由linux底层启动， 作为第一个进程，它被赋予了很多极其重要的工作职责，比如创建zygote(孵化器)和属性服务等。
init进程是由多个源文件共同组成的，这些文件位于源码目录system/core/init。本文将基于Android7.1源码来分析Init进程
```
通过下图可知，init文件具有可执行的权限，所以他是一个可执行的文件
![结果显示](/uploads/Android启动流程Init解析/Init文件所在的目录.png)

源码所在的目录为system/core/init/init.cpp 
```java
int main(int argc, char** argv) {
	...
	const BuiltinFunctionMap function_map;
    Action::set_function_map(&function_map);
    Parser& parser = Parser::GetInstance();
    parser.AddSectionParser("service",std::make_unique<ServiceParser>());
    parser.AddSectionParser("on", std::make_unique<ActionParser>());
    parser.AddSectionParser("import", std::make_unique<ImportParser>());
    parser.ParseConfig("/init.rc");
...
}
```

parser.ParseConfig("/init.rc")设计到了rc文件的解析，这里先了解下rc文件
```
init.rc文件所在目录为 /system/core/rootdir/init.rc
init.rc是由一种被称为＂Android初始化语言＂（Android Init Language，这里简称为AIL）的脚本写成的文件．该语言是由语句组成的，主要包含了五种类型的语句:
Action
Commands
Services
Options
Import
在init.rc文件中一条语句通常占用一行，单词之间是用空格符来相隔的。如果一行写不下，可以在行尾加上反斜杠，来连接下一行。也就是说，可以用反斜杠将多行代码连接成一行代码。
并且使用#来进行注释。在init.rc中分成三个部分（Section），而每一部分的开头需要指定on（Actions）、service（Services）或import。
也就是说，每一个Actions, import或 Services确定一个Section。而所有的Commands和Options只能属于最近定义的Section。如果Commands和 Options在第一个Section之前被定义，
它们将被忽略。Actions和Services的名称必须唯一。如果有两个或多个Actions或Services拥有同样的名称，那么init在执行它们时将抛出错误，并忽略这些Action和Service。
```

下面将service, on, import设置为三个Section.

```java
Parser& parser = Parser::GetInstance();  
//构建ActionParser,ServiceParser,ImportParser对象,并添加到parser成员变量中
//std::make_unique<ServiceParser>() 这个为c++的智能指针，是用来构建对象的，会调用对应类的构造函数，完成初始化
parser.AddSectionParser("service",std::make_unique<ServiceParser>());  
parser.AddSectionParser("on", std::make_unique<ActionParser>());  
parser.AddSectionParser("import", std::make_unique<ImportParser>()); 
section_parsers_成员变量的声明为,大致功能就相当于是java中的map类似
std::map<std::string, std::unique_ptr<SectionParser>> section_parsers_; 
```

下面来看一下Action,Service,Import都是该怎么定义的

```java
在system/core/init/readme.txt中详细说明
Action的格式如下：
on <trigger> [&& <trigger>]*     //设置触发器  
   <command>  
   <command>                  //动作触发之后要执行的命令  
   <command>  
Triggers                    //对trigger的详细讲解  
 
Triggers are strings which can be used to match certain kinds of  
events and used to cause an action to occur.  
  
Triggers are subdivided into event triggers and property triggers.  
  
Event triggers are strings triggered by the 'trigger' command or by  
the QueueEventTrigger() function within the init executable.  These  
take the form of a simple string such as 'boot' or 'late-init'.  
  
Property triggers are strings triggered when a named property changes  
value to a given new value or when a named property changes value to  
any new value.  These take the form of 'property:<name>=<value>' and  
'property:<name>=*' respectively.  Property triggers are additionally  
evaluated and triggered accordingly during the initial boot phase of  
init.  
  
An Action can have multiple property triggers but may only have one  
event trigger.  
  
For example:  
'on boot && property:a=b' defines an action that is only executed when  
the 'boot' event trigger happens and the property a equals b.  
  
'on property:a=b && property:c=d' defines an action that is executed  
at three times,  
   1) During initial boot if property a=b and property c=d  
   2) Any time that property a transitions to value b, while property  
      c already equals d.  
   3) Any time that property c transitions to value d, while property  
      a already equals b.  

在init.cpp中设置的trigger有early-init, init, late-init等, 当trigger被触发时就执行command,
我们来看一个标准的Action:
on early-init     //trigger为early-init,在init.cpp的main函数中设置过  
    # Set init and its forked children's oom_adj.  
    write /proc/1/oom_score_adj -1000   //调用do_write函数, 写入oom_score_adj为-1000  
  
    # Disable sysrq from keyboard  
    write /proc/sys/kernel/sysrq 0  
  
    # Set the security context of /adb_keys if present.  
    restorecon /adb_keys       //为adb_keys 重置安全上下文  
  
    # Shouldn't be necessary, but sdcard won't start without it. http://b/22568628.  
    mkdir /mnt 0775 root system    //创建mnt目录  
  
    # Set the security context of /postinstall if present.  
    restorecon /postinstall  
  
    start ueventd       //调用函数do_start, 启动服务uevent,  

下面对所有命令详细讲解.
bootchart_init    //初始化bootchart,用于获取开机过程系统信息  
   Start bootcharting if configured (see below).  
   This is included in the default init.rc.  
  
chmod <octal-mode> <path>  //改变文件的权限  
   Change file access permissions.  
  
chown <owner> <group> <path>  //改变文件的群组  
   Change file owner and group.  
  
class_start <serviceclass>   //启动所有具有特定class的services  
   Start all services of the specified class if they are  
   not already running.  
  
class_stop <serviceclass>      //将具有特定class的所有运行中的services给停止或者diasble  
   Stop and disable all services of the specified class if they are  
   currently running.  
  
class_reset <serviceclass>   //先将services stop掉, 之后可能会通过class_start再重新启动起来  
   Stop all services of the specified class if they are  
   currently running, without disabling them. They can be restarted  
   later using class_start.  
  
copy <src> <dst>    //复制文件  
   Copies a file. Similar to write, but useful for binary/large  
   amounts of data.  
  
domainname <name>      
   Set the domain name.  
  
enable <servicename>  //如果services没有特定disable,就将他设为enable  
   Turns a disabled service into an enabled one as if the service did not  
   specify disabled.          
   If the service is supposed to be running, it will be started now.  
   Typically used when the bootloader sets a variable that indicates a specific  
   service should be started when needed. E.g.  
     on property:ro.boot.myfancyhardware=1  
        enable my_fancy_service_for_my_fancy_hardware  
  
exec [ <seclabel> [ <user> [ <group> ]* ] ] -- <command> [ <argument> ]*    //创建执行程序.比较重要,后面启动service要用到  
   Fork and execute command with the given arguments. The command starts  
   after "--" so that an optional security context, user, and supplementary  
   groups can be provided. No other commands will be run until this one  
export <name> <value>      //在全局设置环境变量  
   Set the environment variable <name> equal to <value> in the  
   global environment (which will be inherited by all processes  
   started after this command is executed)  
  
hostname <name>       //设置主机名称  
   Set the host name.  
  
ifup <interface>       //启动网络接口  
   Bring the network interface <interface> online.  
  
insmod <path>    //在某个路径安装一个模块  
   Install the module at <path>  
  
load_all_props    //加载所有的配置  
   Loads properties from /system, /vendor, et cetera.  
   This is included in the default init.rc.  
  
load_persist_props  //当data加密时加载一些配置  
   Loads persistent properties when /data has been decrypted.  
   This is included in the default init.rc.  
  
loglevel <level>       //设置kernel log level  
   Sets the kernel log level to level. Properties are expanded within <level>.  
  
mkdir <path> [mode] [owner] [group]    //创建文件夹  
   Create a directory at <path>, optionally with the given mode, owner, and  
   group. If not provided, the directory is created with permissions 755 and  
   owned by the root user and root group. If provided, the mode, owner and group  
   will be updated if the directory exists already.  
  
mount_all <fstab> [ <path> ]*      //挂载  
   Calls fs_mgr_mount_all on the given fs_mgr-format fstab and imports .rc files  
   at the specified paths (e.g., on the partitions just mounted). Refer to the  
   section of "Init .rc Files" for detail.  
  
mount <type> <device> <dir> [ <flag> ]* [<options>]   //在dir文件夹下面挂载设备  
   Attempt to mount the named device at the directory <dir>  
   <device> may be of the form mtd@name to specify a mtd block  
   device by name.  
   <flag>s include "ro", "rw", "remount", "noatime", ...  
   <options> include "barrier=1", "noauto_da_alloc", "discard", as  
   a comma separated string, eg: barrier=1,noauto_da_alloc  
  
powerctl  
   Internal implementation detail used to respond to changes to the  
   "sys.powerctl" system property, used to implement rebooting.  
restart <service>          //重启服务  
   Like stop, but doesn't disable the service.  
  
restorecon <path> [ <path> ]*      //重置文件的安全上下文  
   Restore the file named by <path> to the security context specified  
   in the file_contexts configuration.  
   Not required for directories created by the init.rc as these are  
   automatically labeled correctly by init.  
  
restorecon_recursive <path> [ <path> ]*//一般都是 selinux完成初始化之后又创建、或者改变的目录  
   Recursively restore the directory tree named by <path> to the  
   security contexts specified in the file_contexts configuration.  
  
rm <path>         //删除文件  
   Calls unlink(2) on the given path. You might want to  
   use "exec -- rm " instead (provided the system partition is  
   already mounted).  
  
rmdir <path>     //删除文件夹  
   Calls rmdir(2) on the given path.  
  
setprop <name> <value>    //设置属性  
   Set system property <name> to <value>. Properties are expanded  
   within <value>.  
  
setrlimit <resource> <cur> <max>  
   Set the rlimit for a resource.  
  
start <service>     //如果service没有启动,就将他启动起来  
   Start a service running if it is not already running.  
  
stop <service> //将运行的服务停掉  
   Stop a service from running if it is currently running.  
  
swapon_all <fstab>  
   Calls fs_mgr_swapon_all on the given fstab file.  
  
symlink <target> <path>  
   Create a symbolic link at <path> with the value <target>  
  
sysclktz <mins_west_of_gmt>  
   Set the system clock base (0 if system clock ticks in GMT)  
trigger <event>      //触发一个事件  
   Trigger an event.  Used to queue an action from another  
   action.  
  
verity_load_state  
   Internal implementation detail used to load dm-verity state.  
  
verity_update_state <mount_point>  
   Internal implementation detail used to update dm-verity state and  
   set the partition.<mount_point>.verified properties used by adb remount  
   because fs_mgr can't set them directly itself.  
  
wait <path> [ <timeout> ]     //等待  
   Poll for the existence of the given file and return when found,  
   or the timeout has been reached. If timeout is not specified it  
   currently defaults to five seconds.  
  
write <path> <content>   //写文件  
   Open the file at <path> and write a string to it with write(2).  
   If the file does not exist, it will be created. If it does exist,  
   it will be truncated. Properties are expanded within <content>.  
这些命令在执行时会通过在BuiltinFunctionMap中的对应关系找到自己对应的函数.

Services格式如下:
service <name> <pathname> [ <argument> ]*   //service的名字,启动路径,以及参数  
   <option>       //option影响什么时候,如何启动services  
   <option>  

Options        //所有的options命令如下  
-------  
Options are modifiers to services.  They affect how and when init  
runs the service.  
  
critical     //表示该service非常重要,如果退出四次以上在4分钟内,设备就会重启进入recovery模式  
  This is a device-critical service. If it exits more than four times in  
  four minutes, the device will reboot into recovery mode.  
  
disabled    //该services不能通过class启动,只能通过name将他启动  
  This service will not automatically start with its class.  
  It must be explicitly started by name.  
  
setenv <name> <value>   //设置环境变量  
  Set the environment variable <name> to <value> in the launched process.  
  
socket <name> <type> <perm> [ <user> [ <group> [ <seclabel> ] ] ]  
  Create a unix domain socket named /dev/socket/<name> and pass  
  its fd to the launched process.  <type> must be "dgram", "stream" or "seqpacket".  
  User and group default to 0.  
  'seclabel' is the SELinux security context for the socket.  
  It defaults to the service security context, as specified by seclabel or  
  computed based on the service executable file security context.  
  
user <username>  
  Change to username before exec'ing this service.  
  Currently defaults to root.  (??? probably should default to nobody)  
  As of Android M, processes should use this option even if they  
  require linux capabilities.  Previously, to acquire linux  
  capabilities, a process would need to run as root, request the  
  capabilities, then drop to its desired uid.  There is a new  
  mechanism through fs_config that allows device manufacturers to add  
  linux capabilities to specific binaries on a file system that should  
  be used instead. This mechanism is described on  
  http://source.android.com/devices/tech/config/filesystem.html.  When  
  using this new mechanism, processes can use the user option to  
  select their desired uid without ever running as root.  
  
group <groupname> [ <groupname> ]*  
  Change to groupname before exec'ing this service.  Additional  
  groupnames beyond the (required) first one are used to set the  
  supplemental groups of the process (via setgroups()).  
  Currently defaults to root.  (??? probably should default to nobody)  
  
seclabel <seclabel>  
  Change to 'seclabel' before exec'ing this service.  
  Primarily for use by services run from the rootfs, e.g. ueventd, adbd.  
  Services on the system partition can instead use policy-defined transitions  
  based on their file security context.  
  If not specified and no transition is defined in policy, defaults to the init context.  
  
oneshot   
  Do not restart the service when it exits.  
  
class <name>   //一个特定的name, 所有有这个name的service统一管理,一起启动,一起stop  
  Specify a class name for the service.  All services in a  
  named class may be started or stopped together.  A service  
  is in the class "default" if one is not specified via the  
  class option.  
  
onrestart      //当服务重启时执行  
  Execute a Command (see below) when service restarts.  
  
writepid <file...>  
  Write the child's pid to the given files when it forks. Meant for  
  cgroup/cpuset usage.  
下面看一下zygote服务示例代码位置system/core/rootdir/init.zygote64.rc;
service的名字为zygote, 启动路径为手机中/system/bin/app_process64后面的都是启动参数.
service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server   
    class main      //zygote的name为main,和class name为main的一块被启动  
    socket zygote stream 660 root system   //为zygote创建socket  
    onrestart write /sys/android_power/request_state wake  //当zygote重启时执行下面动作  
    onrestart write /sys/power/state on  
    onrestart restart audioserver  
    onrestart restart cameraserver  
    onrestart restart media  
    onrestart restart netd  
    writepid /dev/cpuset/foreground/tasks /dev/stune/foreground/tasks  
对Action与Services了解之后就可以开始解析init.rc文件了.

```
 
在分析init进程时知道解析init.rc文件的入口在init.cpp的main函数中

``` java
parser.ParseConfig("/init.rc");  
解析init.rc的核心在system/core/init/init_parse.cpp文件中
bool Parser::ParseConfig(const std::string& path) {  
    if (is_dir(path.c_str())) {       //如果路径为文件夹,就调用解析文件夹的函数  
        return ParseConfigDir(path);     
    }  
    return ParseConfigFile(path);  //解析init.rc文件  
}  

bool Parser::ParseConfigFile(const std::string& path) {  
    INFO("Parsing file %s...\n", path.c_str());  //根据之前讲解klog一文,可知该log打印不出  
    Timer t;      //利用Timer计时,解析文件耗时多长时间  
    std::string data;  
    if (!read_file(path.c_str(), &data)) {   //读取文件  
        return false;  
    }  
  
    data.push_back('\n'); // TODO: fix parse_config.  
    ParseData(path, data);     //解析内容  
    for (const auto& sp : section_parsers_) {  
        sp.second->EndFile(path);       //解析完init.rc文件后, 调用Import_parse.cpp的EndFile函数,解析引用的rc文件  
    }  
  
    // Turning this on and letting the INFO logging be discarded adds 0.2s to  
    // Nexus 9 boot time, so it's disabled by default.  
    if (false) DumpState();  
  
    NOTICE("(Parsing %s took %.2fs.)\n", path.c_str(), t.duration());  //打印出解析哪个文件,花费多长时间, 来查找耗时点  
    return true;  
}  
void Parser::ParseData(const std::string& filename, const std::string& data) {  
    //TODO: Use a parser with const input and remove this copy  
    std::vector<char> data_copy(data.begin(), data.end());     //数据copy  
    data_copy.push_back('\0');  
  
    parse_state state;  
    state.filename = filename.c_str();  
    state.line = 0;  
    state.ptr = &data_copy[0];  
    state.nexttoken = 0;  
  
    SectionParser* section_parser = nullptr;  
    std::vector<std::string> args;  
  
    for (;;) {        //循环解析init.rc文件  
        switch (next_token(&state)) {  //通过nextToken函数获得需要解析的这一行是文本内容还是action,services  
        case T_EOF:            //文件解析结束 
            if (section_parser) {  
                section_parser->EndSection();     
            }  
            return;  
        case T_NEWLINE:      //解析新的一行, 可能是一个action,services或者import  
            state.line++;  
            if (args.empty()) {  
                break;  
            }  
            if (section_parsers_.count(args[0])) {  
                if (section_parser) {  
                    section_parser->EndSection();   //section解析结束  EndSection  类似java中的多态，会调用真正实例的方法
                }  
                section_parser = section_parsers_[args[0]].get();  
                std::string ret_err;  
                if (!section_parser->ParseSection(args, &ret_err)) {  //解析Action,Service, Import 三个Section ParseSection 类似java中的多态，会调用真正实例的方法
                    parse_error(&state, "%s\n", ret_err.c_str());  
                    section_parser = nullptr;  
                }  
            } else if (section_parser) {  
                std::string ret_err;      //解析section的内容  
                if (!section_parser->ParseLineSection(args, state.filename,state.line, &ret_err)) //  ParseLineSection  类似java中的多态，会调用真正实例的方法
				{ 
                    parse_error(&state, "%s\n", ret_err.c_str());  
                }  
            }  
            args.clear();  
            break;  
        case T_TEXT:  
            args.emplace_back(state.text);     //将文本放入args中  
            break;  
        }  
    }  
}  

解析Action 即为on标签后面的内容 比如on early-init 解析的即为early-init
调用ActionParser的ParseSection函数,代码位置system/core/init/action.cpp
bool ActionParser::ParseSection(const std::vector<std::string>& args, std::string* err) {  
    std::vector<std::string> triggers(args.begin() + 1, args.end());  //获取trigger, 截取on后面的字符串  
    if (triggers.size() < 1) {  
        *err = "actions must have a trigger";    //检查是否存在trigger  
        return false;  
    }  
	
    c++的智能指针构建对象，调用对应的构造函数 即为Action::Action(bool oneshot) : oneshot_(oneshot){}
    auto action = std::make_unique<Action>(false);  
    if (!action->InitTriggers(triggers, err)) {   //将triggers放入event_trigger_  event_trigger_为一个string对象后面为他的声明 std::string event_trigger_;
        return false;  
    }  
    action_ = std::move(action);   //赋值action_  
    return true;  
}  

解析Action中的每一个Command命令 比如 write /proc/1/oom_score_adj -1000 
bool ActionParser::ParseLineSection(const std::vector<std::string>& args,  
                                    const std::string& filename, int line,  
                                    std::string* err) const {  
    return action_ ? action_->AddCommand(args, filename, line, err) : false;  //action_为true ,已经赋过值  
}  

下面的方法中有这样的变量function_map_ 这里先了解这个变量
function_map_ 类型为static const KeywordMap<BuiltinFunction>* function_map_;
并且在Action.h头文件中有这样的方法
static void set_function_map(const KeywordMap<BuiltinFunction>* function_map) {
        function_map_ = function_map;
}
方法的调用为
const BuiltinFunctionMap function_map;
Action::set_function_map(&function_map);
而BuildinFunctionMap的类型声明在/system/core/init/builtins.cpp文件中
BuiltinFunctionMap::Map& BuiltinFunctionMap::map() const {
    constexpr std::size_t kMax = std::numeric_limits<std::size_t>::max();
    static const Map builtin_functions = {
        {"bootchart_init",          {0,     0,    do_bootchart_init}},
        {"chmod",                   {2,     2,    do_chmod}},
        {"chown",                   {2,     3,    do_chown}},
        {"class_reset",             {1,     1,    do_class_reset}},
        {"class_start",             {1,     1,    do_class_start}},
        {"class_stop",              {1,     1,    do_class_stop}},
        {"copy",                    {2,     2,    do_copy}},
        {"domainname",              {1,     1,    do_domainname}},
        {"enable",                  {1,     1,    do_enable}},
        {"exec",                    {1,     kMax, do_exec}},
        {"export",                  {2,     2,    do_export}},
        {"hostname",                {1,     1,    do_hostname}},
        {"ifup",                    {1,     1,    do_ifup}},
        {"init_user0",              {0,     0,    do_init_user0}},
        {"insmod",                  {1,     kMax, do_insmod}},
        {"installkey",              {1,     1,    do_installkey}},
        {"load_persist_props",      {0,     0,    do_load_persist_props}},
        {"load_system_props",       {0,     0,    do_load_system_props}},
        {"loglevel",                {1,     1,    do_loglevel}},
        {"mkdir",                   {1,     4,    do_mkdir}},
        {"mount_all",               {1,     kMax, do_mount_all}},
        {"mount",                   {3,     kMax, do_mount}},
        {"umount",                  {1,     1,    do_umount}},
        {"powerctl",                {1,     1,    do_powerctl}},
        {"restart",                 {1,     1,    do_restart}},
        {"restorecon",              {1,     kMax, do_restorecon}},
        {"restorecon_recursive",    {1,     kMax, do_restorecon_recursive}},
        {"rm",                      {1,     1,    do_rm}},
        {"rmdir",                   {1,     1,    do_rmdir}},
        {"setprop",                 {2,     2,    do_setprop}},
        {"setrlimit",               {3,     3,    do_setrlimit}},
        {"start",                   {1,     1,    do_start}},
        {"stop",                    {1,     1,    do_stop}},
        {"swapon_all",              {1,     1,    do_swapon_all}},
        {"symlink",                 {2,     2,    do_symlink}},
        {"sysclktz",                {1,     1,    do_sysclktz}},
        {"trigger",                 {1,     1,    do_trigger}},
        {"verity_load_state",       {0,     0,    do_verity_load_state}},
        {"verity_update_state",     {0,     0,    do_verity_update_state}},
        {"wait",                    {1,     2,    do_wait}},
        {"write",                   {2,     2,    do_write}},
    };
    return builtin_functions;
}

bool Action::AddCommand(const std::vector<std::string>& args,  
                        const std::string& filename, int line, std::string* err) {  
    if (!function_map_) { //上面的分析得知，function_map_是不为空的         
        *err = "no function map available";  
        return false;  
    }  
  
    if (args.empty()) {  
        *err = "command needed, but not provided";  
        return false;  
    }  
	
	//根据命令找到对应的函数 ,比如 write /proc/1/oom_score_adj -1000  那么在 function_map中有这样的对应关系 {"write",2,2, do_write}},这里得到的就为do_write函数指针
    auto function = function_map_->FindFunction(args[0], args.size() - 1, err);   
    if (!function) {  
        return false;  
    }  
  
	//将命令保存到action中
    AddCommand(function, args, filename, line);  
    return true;  
}  

void Action::AddCommand(BuiltinFunction f,  
                        const std::vector<std::string>& args,  
                        const std::string& filename, int line) {  
						
	//commands_的类型为 std::vector<Command> commands_，就是类似java中向量集合
	//emplace_back是他特有的方法，这个方法调用会先去构造一个Command的对象，后面的参数即为要构造这个对象的初始化参数
    commands_.emplace_back(f, args, filename, line);  //将对应函数, 参数,文件名先构造一个Command对象，然后放入commans_ 集合中
}  

command的构造函数
Command::Command(BuiltinFunction f, const std::vector<std::string>& args,
                 const std::string& filename, int line)
    : func_(f), args_(args), filename_(filename), line_(line) {
}

当解析到on标签结尾的时候，会调用这个方法
void ActionParser::EndSection() {  
    if (action_ && action_->NumCommands() > 0) {  
		//ActionManager主要是用来存储这些Action对象的里面有actions_ 集合用来存储所有的action
        ActionManager::GetInstance().AddAction(std::move(action_));  //将当前解析的action添加到action_集合中
    }  
}  

void ActionManager::AddAction(std::unique_ptr<Action> action) {  
    auto old_action_it =  
        std::find_if(actions_.begin(), actions_.end(),  
                     [&action] (std::unique_ptr<Action>& a) {  
                         return action->TriggersEqual(*a);  
                     });  
  
    if (old_action_it != actions_.end()) {  
        (*old_action_it)->CombineAction(*action);  
    } else {  
        actions_.emplace_back(std::move(action));   //将所有的action加入actions_列表  actions_声明为 std::vector<std::unique_ptr<Action>> actions_;类似一个集合
    }  
}  
```

解析Services

```java
init.rc文件中有这样的service声明
service ueventd /sbin/ueventd
    class core
    critical
    seclabel u:r:ueventd:s0

调用ServiceParser的ParseSection函数,代码位置system/core/init/service.cpp
bool ServiceParser::ParseSection(const std::vector<std::string>& args,std::string* err) {  
    if (args.size() < 3) {    //判断service是否有name与可执行程序  
        *err = "services must have a name and a program";  
        return false;  
    }  
  
    const std::string& name = args[1]; // 就像解析 service ueventd /sbin/ueventd 中得到ueventd
    if (!IsValidName(name)) { //检查name是否可用  
        *err = StringPrintf("invalid service name '%s'", name.c_str());  
        return false;  
    }  
  
    std::vector<std::string> str_args(args.begin() + 2, args.end()); //获取执行程序与参数  
	//指针指针构建对象，并完成对应的构造函数,这里得到一个service对象
    service_ = std::make_unique<Service>(name, "default", str_args);   //给service_赋值  
    return true;  
}  

解析service的内容 比如解析 class core
bool ServiceParser::ParseLineSection(const std::vector<std::string>& args,  
                                     const std::string& filename, int line, std::string* err) const {  
    return service_ ? service_->HandleLine(args, err) : false;  //service_为true, 调用HandleLine  
}  

Service::OptionHandlerMap::Map& Service::OptionHandlerMap::map() const {  
    constexpr std::size_t kMax = std::numeric_limits<std::size_t>::max();  
    static const Map option_handlers = {   //option对应的函数  
        {"class",       {1,     1,    &Service::HandleClass}},  
        {"console",     {0,     0,    &Service::HandleConsole}},  
        {"critical",    {0,     0,    &Service::HandleCritical}},  
        {"disabled",    {0,     0,    &Service::HandleDisabled}},  
        {"group",       {1,     NR_SVC_SUPP_GIDS + 1, &Service::HandleGroup}},  
        {"ioprio",      {2,     2,    &Service::HandleIoprio}},  
        {"keycodes",    {1,     kMax, &Service::HandleKeycodes}},  
        {"oneshot",     {0,     0,    &Service::HandleOneshot}},  
        {"onrestart",   {1,     kMax, &Service::HandleOnrestart}},  
        {"seclabel",    {1,     1,    &Service::HandleSeclabel}},  
        {"setenv",      {2,     2,    &Service::HandleSetenv}},  
        {"socket",      {3,     6,    &Service::HandleSocket}},  
        {"user",        {1,     1,    &Service::HandleUser}},  
        {"writepid",    {1,     kMax, &Service::HandleWritepid}},  
    };  
    return option_handlers;  
}  
  
bool Service::HandleLine(const std::vector<std::string>& args, std::string* err) {  
    if (args.empty()) {  
        *err = "option needed, but not provided";  
        return false;  
    }  
  
    static const OptionHandlerMap handler_map;   //获得option对应的函数表  ,上面即为这个的函数表
	//根据option获取对应的函数名 对于class core 来说，对应的即为{"class",{1,1,&Service::HandleClass}} ,&Service::HandleClass即为要找的函数指针
    auto handler = handler_map.FindFunction(args[0], args.size() - 1, err); 
    if (!handler) {  
        return false;  
    }  
    //执行对应的函数
    return (this->*handler)(args, err);     
}  
Service::HandleClass函数的圆形为,其实就是解析到core赋值给classname_ 对于class core 来说
bool Service::HandleClass(const std::vector<std::string>& args, std::string* err) {
    classname_ = args[1];
    return true;
}

service文件解析到了尾
void ServiceParser::EndSection() {  
    if (service_) { 
		//ServiceManager  是用来专门保存service的
        ServiceManager::GetInstance().AddService(std::move(service_));  
    }  
}  

void ServiceManager::AddService(std::unique_ptr<Service> service) {  
    Service* old_service = FindServiceByName(service->name());  
    if (old_service) {    //service已经被定义过了就抛弃  
        ERROR("ignored duplicate definition of service '%s'",  
              service->name().c_str());  
        return;  
    } 
	
	//services_的声明为  std::vector<std::unique_ptr<Service>> services_ 类似集合
    services_.emplace_back(std::move(service));  //将service添加services_列表  
}  
```

解析Import

```java
调用ImportParser的ParseSection函数,代码位置system/core/init/import_parser.cpp
bool ImportParser::ParseSection(const std::vector<std::string>& args,  
                                std::string* err) {  
    if (args.size() != 2) {  
        *err = "single argument needed for import\n";  
        return false;  
    }  
  
    std::string conf_file;  
    bool ret = expand_props(args[1], &conf_file); //获取引用的conf_file文件,   
    if (!ret) {  
        *err = "error while expanding import";  
        return false;  
    }  
  
    INFO("Added '%s' to import list\n", conf_file.c_str());  
    imports_.emplace_back(std::move(conf_file));   //将所有的conf_file添加到imports_列表  
    return true;  
} 
``` 

init.cpp 中的main方法继续往下面执行

``` java
 ....
 //添加对应的trigger
 ActionManager& am = ActionManager::GetInstance();
 am.QueueEventTrigger("early-init");
 // Trigger all the boot actions to get us started.
 am.QueueEventTrigger("init");
 am.QueueEventTrigger("late-init");
 ....
 
 static ActionManager instance;因为声明为静态的，那获取到的其实就是上面解析Action用来保存Action的同一个Manager
 ActionManager& ActionManager::GetInstance() {
    static ActionManager instance;
    return instance;
}
am.QueueEventTrigger("early-init");实现为
void ActionManager::QueueEventTrigger(const std::string& trigger) {
	// trigger_queue_ 成员的声明为 std::queue<std::unique_ptr<Trigger>> trigger_queue_; 相当于一个集合，用来存储Trigger对象
	//std::make_unique<EventTrigger>(trigger)回构建这个EventTrigger对象，EventTrigger继承于Trigger
    trigger_queue_.push(std::make_unique<EventTrigger>(trigger));
}
class EventTrigger : public Trigger {
public:
    EventTrigger(const std::string& trigger) : trigger_(trigger) {
    }
    bool CheckTriggers(const Action& action) const override {
        return action.CheckEventTrigger(trigger_);
    }
private:
    const std::string trigger_;
};
经过上面的操作那么trigger_queue_就有了对应的EventTrigger实例
init.cpp 中的main方法继续往下面执行
while (true) {
        if (!waiting_for_exec) {
            am.ExecuteOneCommand(); 启动action
            restart_processes();
        }

        int timeout = -1;
        if (process_needs_restart) {
            timeout = (process_needs_restart - gettime()) * 1000;
            if (timeout < 0)
                timeout = 0;
        }

        if (am.HasMoreCommands()) {
            timeout = 0;
        }

        bootchart_sample(&timeout);

        epoll_event ev;
        int nr = TEMP_FAILURE_RETRY(epoll_wait(epoll_fd, &ev, 1, timeout));
        if (nr == -1) {
            ERROR("epoll_wait failed: %s\n", strerror(errno));
        } else if (nr == 1) {
            ((void (*)()) ev.data.ptr)();
        }
    }
}

 void ActionManager::ExecuteOneCommand() {  
    // Loop through the trigger queue until we have an action to execute  
	
	//当前的可执行action队列为空,默认是为空 trigger_queue_队列不为空   trigger_queue_不为空，比如上面添加了三个early-init,init,late-init
    while (current_executing_actions_.empty() && !trigger_queue_.empty()) { 
	 //循环遍历action_队列,包含了所有需要执行的命令,解析init.rc获得  ,不为空
        for (const auto& action : actions_) {          
		//获取队头的trigger, 检查actions_列表中的action的trigger,对比是否相同 比如init.rc文件中有 on early-init ,跟上面的trigger_queue_中的元素匹配的上，就表示当前的action需要执行
            if (trigger_queue_.front()->CheckTriggers(*action)) { 
				//将所有具有同一trigger的action加入当前可执行action队列  
                current_executing_actions_.emplace(action.get());  
            }  
        }  
        trigger_queue_.pop();   //将队头trigger出栈  
    }  
  
    if (current_executing_actions_.empty()) {//当前可执行的actions队列为空就返回  
        return;  
    }  
  
    auto action = current_executing_actions_.front(); //获取当前可执行actions队列的首个action  
  
    if (current_command_ == 0) {  
        std::string trigger_name = action->BuildTriggersString();  
        INFO("processing action (%s)\n", trigger_name.c_str());  
    }  
  
    action->ExecuteOneCommand(current_command_);      //执行当前的命令  
  
    // If this was the last command in the current action, then remove  
    // the action from the executing list.  
    // If this action was oneshot, then also remove it from actions_.  
    ++current_command_;    //不断叠加,将action_中的所有命令取出  
    if (current_command_ == action->NumCommands()) {  
        current_executing_actions_.pop();  
        current_command_ = 0;  
        if (action->oneshot()) {  
            auto eraser = [&action] (std::unique_ptr<Action>& a) {  
                return a.get() == action;  
            };  
            actions_.erase(std::remove_if(actions_.begin(), actions_.end(), eraser));  
        }  
    }  
}  

根据trigger找到对应的action就开始执行了
void Action::ExecuteOneCommand(std::size_t command) const {  
    ExecuteCommand(commands_[command]);   //从commands_队列取出命令对应的函数  
}  
void Action::ExecuteCommand(const Command& command) const {  
    Timer t;      //使用timer计时  
    int result = command.InvokeFunc();  
  
    if (klog_get_level() >= KLOG_INFO_LEVEL) {  
        std::string trigger_name = BuildTriggersString();  
        std::string cmd_str = command.BuildCommandString();  
        std::string source = command.BuildSourceString();  
  
        INFO("Command '%s' action=%s%s returned %d took %.2fs\n",  
             cmd_str.c_str(), trigger_name.c_str(), source.c_str(),  
             result, t.duration());  //可以将该log级别改为NOTICE,可以输出当前执行命令的信息以及执行时间输出, 定位开机问题  
    }  
}  
int Command::InvokeFunc() const {  
    std::vector<std::string> expanded_args;  
    expanded_args.resize(args_.size());  
    expanded_args[0] = args_[0];      
    for (std::size_t i = 1; i < args_.size(); ++i) {  
        if (!expand_props(args_[i], &expanded_args[i])) {  
            ERROR("%s: cannot expand '%s'\n", args_[0].c_str(), args_[i].c_str());  
            return -EINVAL;  
        }  
    }  
  
    return func_(expanded_args);  //执行获得的函数  就是解析action的时候每一条command 关键字对应的函数指针
}  

由于根据android7.0 init进程可以知道第一个触发的动作为early-init
在init.rc中early-init的命令如下:
on early-init       //触发器为early-init, 首先执行  
    # Set init and its forked children's oom_adj.  
    write /proc/1/oom_score_adj -1000  //调用do_write函数  
  
    # Disable sysrq from keyboard  
    write /proc/sys/kernel/sysrq 0    
  
    # Set the security context of /adb_keys if present.  
    restorecon /adb_keys      //调用do_restorecon函数  
  
    # Shouldn't be necessary, but sdcard won't start without it. http://b/22568628.  
    mkdir /mnt 0775 root system  //调用do_mkdir函数  
  
    # Set the security context of /postinstall if present.  
    restorecon /postinstall   
  
    start ueventd   //调用do_start函数启动ueventd服务  

service ueventd /sbin/ueventd  
    class core      //class name为core  
    critical          //重要服务  
    seclabel u:r:ueventd:s0 
	
之后会相继触发init, late-init等.trigger_queue_中有添加 init,late-init

on late-init       
    trigger early-fs    //触发early-fs  
    trigger fs             //触发fs  
    trigger post-fs    //触发post-fs  
  
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
    trigger boot  
	
on fs         //触发fs就会执行如下命令  
    ubiattach 0 ubipac  
    exec /sbin/resize2fs -ef /fstab.${ro.hardware}  
    mount_all /fstab.${ro.hardware}     //执行do_mount_all函数  
    mount pstore pstore /sys/fs/pstore  
    setprop ro.crypto.fuse_sdcard true  
        symlink /system/res /res  
        symlink /system/bin /bin  
		
在system/core/init/builtins.cpp中实现了do_mount_all函数
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
	
	 /* Paths of .rc files are specified at the 2nd argument and beyond */
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

``` 

今天就到这里吧，下回讲解 ServiceManager启动，Zygote启动流程......