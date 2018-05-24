---
layout: pager
title: Hermes个人理解
date: 2018-05-24 15:00:41
tags: [Android,Hermes,IPC]
description:  Hermes个人理解
---

Hermes个人理解
<!--more-->

简介
```
Hermes 一套新颖巧妙易用的Android进程间通信IPC框架。
Hermes是一套新颖巧妙易用的Android进程间通信IPC框架。这个框架使得你不用了解IPC机制就可以进行进程间通信，像调用本地函数一样调用其他进程的函数。
```

****Hermes简单的使用****
===
```java

1. 首先在build.gradle文件中引入依赖
dependencies {
    compile 'xiaofei.library:hermes:0.7.0'
}

public class MainActivity extends Activity
{

    @Override
    protected void onCreate(Bundle savedInstanceState)
    {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        //进程A执行初始化操作
        Hermes.init(this);

        //进程A 执行 注册
        Hermes.register(UserManager.class);

        findViewById(R.id.demo).setOnClickListener(new View.OnClickListener()
        {
            @Override
            public void onClick(View view)
            {
                Intent intent = new Intent(getApplicationContext(), DemoActivity.class);
                startActivity(intent);
            }
        });
    }
}

public class DemoActivity extends Activity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_demo);

        findViewById(R.id.get_user2).setOnClickListener(new View.OnClickListener()
        {
            @Override
            public void onClick(View v)
            {
                //DemoActivity为另一个进程在清单文件中指定了process属性,这里假设为B进程，B进程执行连接操作
                Hermes.connect(getApplicationContext(), HermesService.HermesService0.class);
            }
        });

        findViewById(R.id.get_user).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {

                //获取单例对象
                IUserManager userManager = Hermes.getInstance(IUserManager.class);
                Toast.makeText(getApplicationContext(), userManager.getUser(), Toast.LENGTH_SHORT).show();
            }
        });

    }

    //销毁的时候终止连接
    @Override
    protected void onDestroy() {
        super.onDestroy();
        Hermes.disconnect(getApplicationContext());
    }
}

清单文件的书写为：
<activity
    android:name=".MainActivity"
    android:label="@string/app_name">
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />

        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
</activity>

<service android:name="xiaofei.library.hermes.HermesService$HermesService0"/>
		
<activity
    android:name=".DemoActivity"
    android:label="@string/title_activity_demo"
    android:process=":h">
</activity>

简单的介绍下上面的内容，也即是有俩个界面一个是MainActivity，在这个界面中完成了初始化以及register的操作，另一个界面DemoActivity为另一个进程的界面，在这个界面完成了连接的操作
同时可以通过简单的调用拿到另一个进程的内容
```

Herms结果演示
![结果显示](/uploads/Herms结果演示.png)

****Hermes源码分析****
===
```java

首先分析 进程A执行初始化操作
Hermes.init(this);

//初始化操作
public static void init(Context context)
{
    if (sContext != null)
    {
        return;
    }
    sContext = context.getApplicationContext();
}
可以看到初始化的时候，也没有干什么只是将获取到当前进程的ApplicationContext对象，这里不是保存Context对象，因为ApplicationContext对象的生命周期比较长，不会导致内存泄漏

2，分析 进程A 执行 注册
Hermes.register(UserManager.class);

public static void register(Class<?> clazz)
{
    //检查是否已经初始化了，就是简单的判断sContext对象是否已经完成了赋值，我们在初始化的时候已经完成了赋值
    checkInit();
    //执行注册
    TYPE_CENTER.register(clazz);
}

先分析下TYPE_CENTER 是什么东西，主要是用来干什么用的
private static final TypeCenter TYPE_CENTER = TypeCenter.getInstance();
public class TypeCenter
{
	private final ConcurrentHashMap<String, Class<?>> mAnnotatedClasses;

    //hashMap 用来缓存，防止下次要用的时候还要再获取 保存了要注册类的信息 key为要className,value为对应的Class
    private final ConcurrentHashMap<String, Class<?>> mRawClasses;

    //用来存储要注册的类方法的信息，class 代表当前要注册的类，value为HashMap，String为当前方法的字符串比如 "getUser(java.lang.String)" 这个是针对方法有注解
    private final ConcurrentHashMap<Class<?>, ConcurrentHashMap<String, Method>> mAnnotatedMethods;

    //用来存储要注册的类方法的信息，class 代表当前要注册的类，value为HashMap，String为当前方法的字符串比如 "getUser(java.lang.String)"这个是针对方法没有注解
    private final ConcurrentHashMap<Class<?>, ConcurrentHashMap<String, Method>> mRawMethods;
	...
}
可以看到TypeCenter里面还有一堆的集合，主要是用来缓存的，防止下次使用的时候再去获取，还有一些对应的获取class的类型，method等，用到的时候再来分析

所以会执行到TypeCenter 对应的类  这个方法主要是完成 注册操作,填充注册的类，已经注册的类对应的方法的信息的集合
public void register(Class<?> clazz)
{
    //检查要注册类的合法性
    TypeUtils.validateClass(clazz);
    //填充要注册的类
    registerClass(clazz);
    //填充要注册的类里面的方法信息
    registerMethod(clazz);
}
//注册class
private void registerClass(Class<?> clazz)
{
    //获取到当前类的ClassId注解
    ClassId classId = clazz.getAnnotation(ClassId.class);
    if (classId == null)
    {
        //如果注解为空 ,则获取要注册类的 className
        String className = clazz.getName();
        //如果mRawClasses 不存在className对应的key，则put一个否则不put
        mRawClasses.putIfAbsent(className, clazz);
    }
    else
    {
        //如果不为空，则获取到注解中的值
        String className = classId.value();
        //如果mRawClasses 不存在className对应的key，则put一个否则不put
        mAnnotatedClasses.putIfAbsent(className, clazz);
    }
}
可以看到上面的进行的操作首先检测要注册的类是否有ClassId 注解,有就获取到注解的值，没有就获取当前class的字符串，然后存储到集合中，这个注解的意思就是相当于取一个别名的意思，比如
我们可以在UserManager上面添加一个@ClassId("UserManagerA"),那么就是说这个UserManagerA 代表UserManger这个类

我们在调用的时候是这样调用的  Hermes.register(UserManager.class); 我们来看看UserManager.class是什么样的
@ClassId("UserManager")
public class UserManager implements IUserManager {

    private static UserManager sInstance = null;

    private UserManager() {

    }

    public static synchronized UserManager getInstance() {
        if (sInstance == null) {
            sInstance = new UserManager();
        }
        return sInstance;
    }

    @MethodId("getUser")
    @Override
    public String getUser() {
        return "Xiaofei";
    }
}

@ClassId("UserManager")
public interface IUserManager {

    @MethodId("getUser")
    String getUser();
}
这里先介绍下为什么这个UserManger为什么实现一个IUserManager接口，是这样的，我们如果要想提供内容给别人调用，那么我就必须调用register函数，对于另一方调用的人来说，他只需要一个
类似于IUserManager的接口，对于他是怎么样做到隐射的，这就需要注解ClassId了，还有MethodId,比如在IUserManager的上面的ClassId注解的内容为UserManager，另一个进程的UserManager的注解
为UserManager,这俩个是不是就对应起来了，对于MethodId注解也是类似

接下来分析  registerMethod(clazz); 该方法完成填充要注册的类方法的信息
private void registerMethod(Class<?> clazz)
{
    //获取到当前要注册的类所有的公有的方法，对于私有的当然是不允许调用的
    Method[] methods = clazz.getMethods();
    for (Method method : methods)
    {
        //获取当前方法上面的MethodId注解
        MethodId methodId = method.getAnnotation(MethodId.class);
        if (methodId == null)  //如果注解为空
        {
            //则从mRawMethods 中获取，mRawMethods为没有注解对应的存储集合  如果在mRawMethods 中存在clazz对应的key，就不会添加，否则添加
            mRawMethods.putIfAbsent(clazz, new ConcurrentHashMap<String, Method>());
            //所以这边获取一定不为空
            ConcurrentHashMap<String, Method> map = mRawMethods.get(clazz);
            //获取到当前method对应的方法名称
            String key = TypeUtils.getMethodId(method);
            //添加到集合中
            map.putIfAbsent(key, method);
        }
        else
        {
            //如果注解不为空，则获取注解的值
            //如果在mAnnotatedMethods 中存在clazz对应的key，就不会添加，否则添加
            mAnnotatedMethods.putIfAbsent(clazz, new ConcurrentHashMap<String, Method>());
            //所以这边获取一定不为空
            ConcurrentHashMap<String, Method> map = mAnnotatedMethods.get(clazz);
            //获取到当前method对应的方法名称
            String key = TypeUtils.getMethodId(method);
            //添加到集合中
            map.putIfAbsent(key, method);
        }
    }
}

接下里我们看看另一个进程，这里假设是进程B
首先执行连接操作   Hermes.connect(getApplicationContext(), HermesService.HermesService0.class);
最终会执行到 ,这里的packageName为空
public static void connectApp(Context context, String packageName, Class<? extends HermesService> service)
{
    //进程B也要执行Herms的初始化操作
    init(context);

    //进程B执行连接操作
    CHANNEL.bind(context.getApplicationContext(), packageName, service);
}
CHANNEL的定义为  private static final Channel CHANNEL = Channel.getInstance();这里先看看类的作用
public class Channel
{
	...
    //缓存的HashMap key为对应的要连接的服务，value为要连接的服务，连接成功返回的IBinder对象
    private final ConcurrentHashMap<Class<? extends HermesService>, IHermesService> mHermesServices = new ConcurrentHashMap<Class<? extends HermesService>, IHermesService>();

    //缓存的HashMap key为要连接的服务，value为连接的connection对象
    private final ConcurrentHashMap<Class<? extends HermesService>, HermesServiceConnection> mHermesServiceConnections = new ConcurrentHashMap<Class<? extends HermesService>, HermesServiceConnection>();

    //缓存的HashMap， 缓存了要连接的服务类， value为当前的连接是否正在绑定
    private final ConcurrentHashMap<Class<? extends HermesService>, Boolean> mBindings = new ConcurrentHashMap<Class<? extends HermesService>, Boolean>();

    //缓存的HashMap，缓存了要连接的服务类，value 为 当前的连接是否还可以用
    private final ConcurrentHashMap<Class<? extends HermesService>, Boolean> mBounds = new ConcurrentHashMap<Class<? extends HermesService>, Boolean>();
	
...
}
Channel类主要是用来缓存一些连接的内容，比如缓存要连接的服务Class，对应的IBinder引用，缓存当前哪些是保持连接的，当前哪些是正在连接的等

所以执行 CHANNEL.bind(context.getApplicationContext(), packageName, service);就会执行对应的函数
进程B执行连接的操作
public void bind(Context context, String packageName, Class<? extends HermesService> service)
{
    HermesServiceConnection connection;
    synchronized (this)
    {
        //首先从缓存中查找，这个service是否已经连接过了,连接过了，或者连接过了，而且当前的连接可以用就直接返回
        if (getBound(service))
        {
            return;
        }

        //如果到了这里说明当前的连接不可用，就从mBindings中判断当前的服务类是否正在绑定
        Boolean binding = mBindings.get(service);
        //如果正在绑定就直接返回
        if (binding != null && binding)
        {
            return;
        }
        //如果到了这里，说明当前的连接不可用，而且没有正在绑定,添加到缓存的集合中
        mBindings.put(service, true);//标识当前的连接正在绑定
        //构建一个ServiceConnection对象
        connection = new HermesServiceConnection(service);
        //添加到缓存的connection集合中
        mHermesServiceConnections.put(service, connection);
    }
	
    //构建一个Intent
    Intent intent;
    if (TextUtils.isEmpty(packageName))
    {
        //如果是没有传递packageName，就代表是同一个应用
        intent = new Intent(context, service);
    }
    else
    {
        //如果有传递packageName，就代表是夸应用存在
        intent = new Intent();
        intent.setClassName(packageName, service.getName());
    }
    //连接服务,那么进程A对应的这个服务就会被启动起来
    context.bindService(intent, connection, Context.BIND_AUTO_CREATE);
}
ServiceConnection实现类为
private class HermesServiceConnection implements ServiceConnection
{
        //当前要连接的服务类
        private Class<? extends HermesService> mClass;

        HermesServiceConnection(Class<? extends HermesService> service)
        {
            mClass = service;
        }

        //如果连接进程A的服务成功
        public void onServiceConnected(ComponentName className, IBinder service)
        {
            synchronized (Channel.this)
            {
                //将当前的服务类，添加到mBounds 集合中，标识当前的服务类已经连接成功可以用
                mBounds.put(mClass, true);
                //添加到mBindings集合中，标识当前的服务类，当前不是正在绑定
                mBindings.put(mClass, false);
                //获取到进程A的IBinder对象引用
                IHermesService hermesService = IHermesService.Stub.asInterface(service);
                //然后保存到mHermesServices 集合中，
                mHermesServices.put(mClass, hermesService);
                try
                {
                    //调用注册函数，
                    hermesService.register(mHermesServiceCallback, Process.myPid());
                }
                catch (RemoteException e)
                {
                    e.printStackTrace();
                    Log.e(TAG, "Remote Exception: Check whether " + "the process you are communicating with is still alive.");
                    return;
                }
            }
            //如果有设置mListener 则回调连接成功
            if (mListener != null)
            {
                mListener.onHermesConnected(mClass);
            }
        }

        //连接中断的时候.要移除对应的集合的信息
        public void onServiceDisconnected(ComponentName className)
        {
            synchronized (Channel.this)
            {
                //移除存储的IBinder引用
                mHermesServices.remove(mClass);
                //设置为连接的服务不可用
                mBounds.put(mClass, false);
                //设置为没有正在连接
                mBindings.put(mClass, false);
            }
            //回调，中断连接
            if (mListener != null)
            {
                mListener.onHermesDisconnected(mClass);
            }
        }
    }
}
当函数执行
hermesService.register(mHermesServiceCallback, Process.myPid());
mHermesServiceCallback 本质为
//进程B的注册函数回调,本身是一个IBinder
private IHermesServiceCallback mHermesServiceCallback = new IHermesServiceCallback.Stub();

所以就会调用到进程A的服务类中对应的函数
public abstract class HermesService extends Service {
    //缓存的对象
    private static final ObjectCenter OBJECT_CENTER = ObjectCenter.getInstance();

    //缓存的集合 key为当前要注册的pid，value为回调函数
    private ConcurrentHashMap<Integer, IHermesServiceCallback> mCallbacks = new ConcurrentHashMap<Integer, IHermesServiceCallback>();

    private final IHermesService.Stub mBinder = new IHermesService.Stub() {
        @Override
        public Reply send(Mail mail) {
            //进程A收到了进程B发送的Main请求
            try {
                //根据类型构建不同的Receiver对象
                Receiver receiver = ReceiverDesignator.getReceiver(mail.getObject());
                //获取当前的唯一标识pid
                int pid = mail.getPid();
                //根据当前的pid获取到一开始注册的回调函数
                IHermesServiceCallback callback = mCallbacks.get(pid);
                if (callback != null) {
                    //设置回调函数
                    receiver.setHermesServiceCallback(callback);
                }
                //
                return receiver.action(mail.getTimeStamp(), mail.getMethod(), mail.getParameters());
            } catch (HermesException e) {
                e.printStackTrace();
                return new Reply(e.getErrorCode(), e.getErrorMessage());
            }
        }

        //进程B通过远程的IBinder引用，最终调用到了进程A的对应的方法，完成注册
        @Override
        public void register(IHermesServiceCallback callback, int pid) throws RemoteException {
            //根据pid为唯一的标识，保存回调函数
            mCallbacks.put(pid, callback);
        }

        @Override
        public void gc(List<Long> timeStamps) throws RemoteException {
            OBJECT_CENTER.deleteObjects(timeStamps);
        }
	};
	@Override
	public IBinder onBind(Intent intent) {
			return mBinder;
	}

	public static class HermesService0 extends HermesService {}
	...
}

接着进程B执行  获取单例对象
IUserManager userManager = Hermes.getInstance(IUserManager.class);
//进程B获取到进程A的单例对象,第二个参数为方法对应的参数列表,有些单例确实有参数的存在
public static <T> T getInstance(Class<T> clazz, Object... parameters)
{
    //获取对象
    return getInstanceInService(HermesService.HermesService0.class, clazz, parameters);
}
//获取对象 service 值为 HermesService.HermesService0.class  clazz 值为IUserManager
public static <T> T getInstanceInService(Class<? extends HermesService> service, Class<T> clazz, Object... parameters)
{
    //因为是单例的获取，这里的methodName就统一为getInstance(),所以对于单例可以不用传递methodName
    return getInstanceWithMethodNameInService(service, clazz, "", parameters);
}

//service 要连接的服务类， clazz为进程B 的接口类对应于进程A的 类
public static <T> T getInstanceWithMethodNameInService(Class<? extends HermesService> service, Class<T> clazz, String methodName, Object... parameters)
{
    //检查当前的类是否是非空，并且是为一个接口
    TypeUtils.validateServiceInterface(clazz);

    //检查当前要连接的服务类是否已经连接，进程B也只是简单的从之前缓存的连接集合中判断是否有对应IBinder引用
    checkBound(service);

    //构建一个ObjectWrapper 对象  TYPE_OBJECT_TO_GET 标识当前获取的行为获取一个对象
    ObjectWrapper object = new ObjectWrapper(clazz, ObjectWrapper.TYPE_OBJECT_TO_GET);

    //构建一个Sender类对象
    Sender sender = SenderDesignator.getPostOffice(service, SenderDesignator.TYPE_GET_INSTANCE, object);
    //如果么有参数，构建一个空的object
    if (parameters == null)
    {
        parameters = new Object[0];
    }

    //将传递的参数，转换成Object对象数组
    int length = parameters.length;
    Object[] tmp = new Object[length + 1];
    //构建的对象数组，第一个下标的值为methodName
    tmp[0] = methodName;
    //再填充参数列表
    for (int i = 0; i < length; ++i)
    {
        tmp[i + 1] = parameters[i];
    }
    try
    {
        //执行发送,可以看到他并没有用到返回的这个对象，
        Reply reply = sender.send(null, tmp);
        if (reply != null && !reply.success())
        {
            Log.e(TAG,"Error occurs during getting instance. Error code: " + reply.getErrorCode());
            Log.e(TAG, "Error message: " + reply.getMessage());
            return null;
        }
    }
    catch (HermesException e)
    {
        e.printStackTrace();
        return null;
    }
    //在上面可以看到发送完之后，并没有用到返回的对象，而是通过代理的方式返回这个对象，这是为什么呢，因为这个对象在进程B中也不能用，而且进程B中要想调用
    //这个对象的方法也还要通过IPC的机制来通信，所以这边可以直接返回一个代理的对象，代理的对象又可以知道你下次要调用什么方法，这不是很好
    object.setType(ObjectWrapper.TYPE_OBJECT);
    return getProxy(service, object);
}

首先分析 因为我们调用的时候是通过 Hermes.getInstance(IUserManager.class); 所以这里clazz 为  IUserManager.class
ObjectWrapper object = new ObjectWrapper(clazz, ObjectWrapper.TYPE_OBJECT_TO_GET);
public ObjectWrapper(Class<?> clazz, int type)
{
    //设置ClassName，判断当前的clazz是否有含有ClassId注解，如果获取到注解的值，否则获取当前class的全类名
    setName(!clazz.isAnnotationPresent(ClassId.class), TypeUtils.getClassId(clazz));
	//存储当前要获取的类，这里为 IUserManager.class
    mClass = clazz;
	//获取到时间戳
    mTimeStamp = TimeStampGenerator.getTimeStamp();
	//获取的方式，由外面传递进来，这里为 TYPE_OBJECT_TO_GET 标识为获取单例对象，对应的type还有 TYPE_OBJECT 表示获取的为一个对象 等
    mType = type;
}

//设置类名
protected void setName(boolean isName, String name)
{
    if (name == null)
    {
        throw new IllegalArgumentException();
    }
    mIsName = isName;//标识当前mName获取是否是通过class的全类名，如果为false就表示为class的全类名
    mName = name;
}

继续分析
Sender sender = SenderDesignator.getPostOffice(service, SenderDesignator.TYPE_GET_INSTANCE, object);
public static Sender getPostOffice(Class<? extends HermesService> service, int type, ObjectWrapper object)
    {
        switch (type)
        {
            case TYPE_NEW_INSTANCE:
                return new InstanceCreatingSender(service, object);
            case TYPE_GET_INSTANCE:
                return new InstanceGettingSender(service, object);
            case TYPE_GET_UTILITY_CLASS:
                return new UtilityGettingSender(service, object);
            case TYPE_INVOKE_METHOD:
                return new ObjectSender(service, object);
            default:
                throw new IllegalArgumentException("Type " + type + " is not supported.");
        }
}
因为我们传递的type 为 TYPE_GET_INSTANCE所以这里获取的对象为 new InstanceGettingSender(service, object);
public class InstanceGettingSender extends Sender
{
    public InstanceGettingSender(Class<? extends HermesService> service, ObjectWrapper object)
    {
        super(service, object);
    }
	...
}
Sender的构造函数
public Sender(Class<? extends HermesService> service, ObjectWrapper object)
{
    mService = service; //HermesService.HermesService0.class
    mObject = object;//ObjectWrapper 
}

继续执行
//如果么有参数，构建一个空的object,这里我们调用的时候没有传递参数
if (parameters == null)
{
    parameters = new Object[0];
}
//将传递的参数，转换成Object对象数组
int length = parameters.length;
Object[] tmp = new Object[length + 1];
//构建的对象数组，第一个下标的值为methodName 这里因为我们是获取到单例的对象，这个方法比较特殊，methodName方法名可以为"",所以这里为""
tmp[0] = methodName; 
//再填充参数列表 ,这里没有参数
for (int i = 0; i < length; ++i)
{
    tmp[i + 1] = parameters[i];
}

程序继续执行 执行发送,可以看到他并没有用到返回的这个对象，
Reply reply = sender.send(null, tmp);

//根据方法名已经参数的列表,构建一个Reply对象
public synchronized final Reply send(Method method, Object[] parameters) throws HermesException
{
    //赋值当前的时间戳
    mTimeStamp = TimeStampGenerator.getTimeStamp();
    //如果没有传递参数，构建一个空的object
    if (parameters == null)
    {
        parameters = new Object[0];
    }

    //根据方法名以及参数的列表，返回参数对象的数组
    ParameterWrapper[] parameterWrappers = getParameterWrappers(method, parameters);
    //根据参数对象的数组，转成参数类型的集合，然后封装成一个MethodWrapper对象
    MethodWrapper methodWrapper = getMethodWrapper(method, parameterWrappers);
    //注册方法
    registerClass(method);
    setParameterWrappers(parameterWrappers);

    //构建一个Mail对象
    Mail mail = new Mail(mTimeStamp, mObject, methodWrapper, mParameters);
    mMethod = methodWrapper;
    //执行发送
    return CHANNEL.send(mService, mail);
}

//得到参数列表的数组
private final ParameterWrapper[] getParameterWrappers(Method method, Object[] parameters) throws HermesException
{
        //参数的长度
        int length = parameters.length;
        //构建一个数组
        ParameterWrapper[] parameterWrappers = new ParameterWrapper[length];
        //如果方法不为空
        if (method != null)
        {
            //获取到方法的参数类型列表
            Class<?>[] classes = method.getParameterTypes();
            //获取到参数列表的注解集合
            Annotation[][] parameterAnnotations = method.getParameterAnnotations();
            for (int i = 0; i < length; ++i)
            {
                if (classes[i].isInterface())
                {
                    Object parameter = parameters[i];
                    if (parameter != null)
                    {
                        parameterWrappers[i] = new ParameterWrapper(classes[i], null);
                    }
                    else
                    {
                        parameterWrappers[i] = new ParameterWrapper(null);
                    }
                    if (parameterAnnotations[i] != null && parameter != null)
                    {
                        CALLBACK_MANAGER.addCallback(mTimeStamp, i, parameter, TypeUtils.arrayContainsAnnotation(parameterAnnotations[i], WeakRef.class),
                                                     !TypeUtils.arrayContainsAnnotation(
                                                             parameterAnnotations[i], Background.class));
                    }
                }
                else if (Context.class.isAssignableFrom(classes[i]))
                {
                    parameterWrappers[i] = new ParameterWrapper(TypeUtils.getContextClass(classes[i]), null);
                }
                else
                {
                    parameterWrappers[i] = new ParameterWrapper(parameters[i]);
                }
            }
        }
        else
        {
            //如果方法为空，则当前的方法可能是获取单例的方法，这种方式，方法是可以为空的,所以对于我们获取单例的方式会进入到这里,
            for (int i = 0; i < length; ++i)
            {
                parameterWrappers[i] = new ParameterWrapper(parameters[i]);
            }
        }
        return parameterWrappers;
}
分析下这个的实现
parameterWrappers[i] = new ParameterWrapper(parameters[i]);
//将传递进来的参数对象，构建成一个ParameterWrapper 对象
public ParameterWrapper(Object object) throws HermesException
{
    if (object == null)
    {
        setName(false, "");
        mData = null;
        mClass = null;
    }
    else
    {
        //获取到参数对象的class 类型
        Class<?> clazz = object.getClass();
        mClass = clazz;
        //获取当参数对象的类名，分是否有ClassId注解还是没有
        setName(!clazz.isAnnotationPresent(ClassId.class), TypeUtils.getClassId(clazz));
        //将当前的对象使用gson转换为字符串，方便传输
        mData = CodeUtils.encode(object);
    }
}

CodeUtils.encode函数实现：
public static String encode(Object object) throws HermesException
{
     ...
    return GSON.toJson(object);
}

所以对于 getParameterWrappers(method, parameters);就是将参数对象列表，转换成 ParameterWrapper[]

对于 MethodWrapper methodWrapper = getMethodWrapper(method, parameterWrappers); 函数的实现为
//根据method ，已经参数的列表集合，获取到MethodWrapper对象
@Override
protected MethodWrapper getMethodWrapper(Method method, ParameterWrapper[] parameterWrappers) throws HermesException
{

    ParameterWrapper parameterWrapper = parameterWrappers[0];
    String methodName;
    try
    {
        methodName = CodeUtils.decode(parameterWrapper.getData(), String.class);
    }
    catch (HermesException e)
    {
        e.printStackTrace();
        throw new HermesException(ErrorCodes.GSON_DECODE_EXCEPTION, "Error occurs when decoding the method name.");
    }
    //将参数列表的集合，转换成对应的参数列表类型的集合
    int length = parameterWrappers.length;
    Class<?>[] parameterTypes = new Class[length - 1];
    for (int i = 1; i < length; ++i)
    {
        parameterWrapper = parameterWrappers[i];
        parameterTypes[i - 1] = parameterWrapper == null ? null : parameterWrapper.getClassType();
    }

	//得到参数类型集合然后构造一个MethodWrapper对象
    return new MethodWrapper(methodName, parameterTypes);
}
public MethodWrapper(String methodName, Class<?>[] parameterTypes)
{
    //设置方法名
    setName(true, methodName);
    if (parameterTypes == null)
    {
		parameterTypes = new Class<?>[0];
    }

    //感觉参数的列表类型数组，转成TypeWrapper 数组
    int length = parameterTypes.length;
    mParameterTypes = new TypeWrapper[length];
    for (int i = 0; i < length; ++i)
    {
        mParameterTypes[i] = new TypeWrapper(parameterTypes[i]);
    }
    mReturnType = null;
}
TypeWrapper构造函数为，
public TypeWrapper(Class<?> clazz)
{
    //设置参数的字符串，如果当前参数有注解就获取ClassId注解上面的内容，否则获取clazz的全类名
	setName(!clazz.isAnnotationPresent(ClassId.class), TypeUtils.getClassId(clazz));
}
所以对于getMethodWrapper(method, parameterWrappers) 所做的事就是将参数列表的集合，转换成对应的参数列表类型的集合，然后在转成TypeWrapper，最后返回一个对象MethodWrapper对象

registerClass(method); 函数实现为：
private void registerClass(Method method) throws HermesException
{
    //如果method对象为空，则返回,为空的可能为getInstance方法
    if (method == null)
    {
        return;
    }
    //如果method对象不为空，获取到当前方法的参数列表
    Class<?>[] classes = method.getParameterTypes();
    for (Class<?> clazz : classes)
    {
        if (clazz.isInterface())
        {
            TYPE_CENTER.register(clazz);
            registerCallbackMethodParameterTypes(clazz);
        }
    }
}
register所做的内容就是将参数对应的类型 clazz，获取到对应的method信息，已经className信息，存储到TYPE_CENTER 中,方便下次查找

之后构建一个Mail对象 ,这里要注意mail 是实现了Parcelable 接口的，要不然不能在IPC中传输
Mail mail = new Mail(mTimeStamp, mObject, methodWrapper, mParameters);
//Mail的构造函数
public Mail(long timeStamp, ObjectWrapper object, MethodWrapper method, ParameterWrapper[] parameters) {
    mTimeStamp = timeStamp;
    mPid = Process.myPid();
    mObject = object;
    mMethod = method;
    mParameters = parameters;
}
最后执行  return CHANNEL.send(mService, mail);
//执行发送
public Reply send(Class<? extends HermesService> service, Mail mail)
{
    //从Ibinder连接缓存中，获取到对应的Ibinder引用
    IHermesService hermesService = mHermesServices.get(service);
    try
    {
        //如果引用为空，则返回一个空的错误的对象
        if (hermesService == null)
        {
            return new Reply(ErrorCodes.SERVICE_UNAVAILABLE, "Service Unavailable: Check whether you have connected Hermes.");
        }

        //否则调用远程IBinder的send方法发成发送，此时会跳到另一个进程中完成接受操作
        return hermesService.send(mail);
    }
    catch (RemoteException e)
    {
        return new Reply(ErrorCodes.REMOTE_EXCEPTION, "Remote Exception: Check whether " + "the process you are communicating with is still alive.");
    }
}

所以对于进程B来说做了那么多事情，就是封装一些有用的信息，给进程A用来确定调用哪个类，哪个方法，传递了什么参数进来，最终封装在一起之后，本质还是通过远程的IBinder引用来通信的

所以就会调用到进程A对应的函数
private final IHermesService.Stub mBinder = new IHermesService.Stub() {
    @Override
        public Reply send(Mail mail) {
        //进程A收到了进程B发送的Main请求
        try {
            //根据类型构建不同的Receiver对象 ，对于单例来说获取到Receiver对象为 InstanceGettingReceiver
            Receiver receiver = ReceiverDesignator.getReceiver(mail.getObject());
            //获取当前的唯一标识pid
            int pid = mail.getPid();
            //根据当前的pid获取到一开始注册的回调函数
            IHermesServiceCallback callback = mCallbacks.get(pid);
            if (callback != null) {
                //设置回调函数
                receiver.setHermesServiceCallback(callback);
            }
            //
            return receiver.action(mail.getTimeStamp(), mail.getMethod(), mail.getParameters());
        } catch (HermesException e) {
            e.printStackTrace();
            return new Reply(e.getErrorCode(), e.getErrorMessage());
    }
}

public static Receiver getReceiver(ObjectWrapper objectWrapper) throws HermesException
{
        //根据接收的type获取到对应的Receiver对象
        int type = objectWrapper.getType();
        switch (type)
        {
            case ObjectWrapper.TYPE_OBJECT_TO_NEW:
                return new InstanceCreatingReceiver(objectWrapper);
            case ObjectWrapper.TYPE_OBJECT_TO_GET:
                return new InstanceGettingReceiver(objectWrapper);
            case ObjectWrapper.TYPE_CLASS:
                return new UtilityReceiver(objectWrapper);
            case ObjectWrapper.TYPE_OBJECT:
                return new ObjectReceiver(objectWrapper);
            case ObjectWrapper.TYPE_CLASS_TO_GET:
                return new UtilityGettingReceiver(objectWrapper);
            default:
                throw new HermesException(ErrorCodes.ILLEGAL_PARAMETER_EXCEPTION,
                                          "Type " + type + " is not supported.");
        }
}
因为我们在获取单列的时候传递的typ为TYPE_OBJECT_TO_GET ,所以这里获取到的Receiver对象为 new InstanceGettingReceiver(objectWrapper);
public InstanceGettingReceiver(ObjectWrapper objectWrapper) throws HermesException
{
    super(objectWrapper);
    //获取到对应的class类型
    Class<?> clazz = TYPE_CENTER.getClassType(objectWrapper);
    //检查class是否是合法的
    TypeUtils.validateAccessible(clazz);
    //赋值要返回的ObjectClass
    mObjectClass = clazz;
}

然后执行 receiver.action(mail.getTimeStamp(), mail.getMethod(), mail.getParameters());
//执行解析
public final Reply action(long methodInvocationTimeStamp, MethodWrapper methodWrapper, ParameterWrapper[] parameterWrappers) throws HermesException
{
    //得到要获取的Mehtod ,为抽象函数，交给具体的子类实现
    setMethod(methodWrapper, parameterWrappers);
    //得到参数的列表
    setParameters(methodInvocationTimeStamp, parameterWrappers);
    //利用反射调用得到具体的对象 为抽象函数，交给具体的子类实现
    Object result = invokeMethod();
    if (result == null)
    {
        return null;
    }
    else
    {
        //构建一个返回的对象，返回回去
        return new Reply(new ParameterWrapper(result));
    }
}

因为当前的类为InstanceGettingReceiver对象，所以调用对应的函数
@Override
public void setMethod(MethodWrapper methodWrapper, ParameterWrapper[] parameterWrappers) throws HermesException
    {
        //将参数的列表集合.转成对应的参数列表类型集合
        int length = parameterWrappers.length;
        Class<?>[] parameterTypes = new Class<?>[length];
        for (int i = 0; i < length; ++i)
        {
            parameterTypes[i] = TYPE_CENTER.getClassType(parameterWrappers[i]);
        }
        //获取到当前要获取的方法名
        String methodName = methodWrapper.getName();
        //得到对应的Method
        Method method = TypeUtils.getMethodForGettingInstance(mObjectClass, methodName, parameterTypes);

        //如果当前的方法不是静态的，就抛出一个异常
        if (!Modifier.isStatic(method.getModifiers()))
        {
            throw new HermesException(ErrorCodes.METHOD_GET_INSTANCE_NOT_STATIC,
                                      "Method " + method.getName() + " of class " + mObjectClass.getName() + " is not static. " + "Only the static method can be invoked to get an instance.");
        }
        //检查方法的合法性
        TypeUtils.validateAccessible(method);
		//最终赋值给mMethod
        mMethod = method;
 }

执行 setParameters(methodInvocationTimeStamp, parameterWrappers);
//将传递的ParameterWrapper对象数组，变成Object数组，至于为什么要获取到Object数组，是因为我们在反射的时候需要传递参数
private void setParameters(long methodInvocationTimeStamp, ParameterWrapper[] parameterWrappers) throws HermesException
    {
        if (parameterWrappers == null)
        {
            mParameters = null;
        }
        else
        {
		    //根据传递的参数的个数，构造一个对应长度的数组
            int length = parameterWrappers.length;
            mParameters = new Object[length];
            for (int i = 0; i < length; ++i)
            {
				//遍历获取到里面的值,然后填充到对象数组中
                ParameterWrapper parameterWrapper = parameterWrappers[i];
                if (parameterWrapper == null)
                {
                    mParameters[i] = null;
                }
                else
                {
                    Class<?> clazz = TYPE_CENTER.getClassType(parameterWrapper);
                    if (clazz != null && clazz.isInterface())//如果参数的类型为接口
                    {
                        registerCallbackReturnTypes(clazz); //****
                        mParameters[i] = getProxy(clazz, i, methodInvocationTimeStamp);
                        HERMES_CALLBACK_GC.register(mCallback, mParameters[i], methodInvocationTimeStamp, i);
                    }
                    else if (clazz != null && Context.class.isAssignableFrom(clazz)) //如果获取的是Herms
                    {
                        mParameters[i] = Hermes.getContext();
                    }
                    else
                    {
						//其他的情况
                        String data = parameterWrapper.getData();
                        if (data == null)
                        {
                            mParameters[i] = null;
                        }
                        else
                        {
							//使用Gson来解析传递的json变成对应的对象
                            mParameters[i] = CodeUtils.decode(data, clazz);
                        }
                    }
                }
            }
        }		
} 
 
Object result = invokeMethod();执行，会由具体的子类来实现
@Override
protected Object invokeMethod() throws HermesException
{
    Exception exception;
    try
    {
        //利用反射的形式调用
        Object object = mMethod.invoke(null, getParameters());
        //将构造的结果存储起来,防止下次在用到,就可以直接取，而不用反射的方式获取到
        OBJECT_CENTER.putObject(getObjectTimeStamp(), object);
        return null;
    }
	...
}

返回结果之后  return new Reply(new ParameterWrapper(result)); 要注意 这里的Reply 是实现了Parcelable 接口的，要不然不能在IPC中传输
将获取到的对象，利用ParameterWrapper对象封装
//将传递进来的参数对象，构建成一个ParameterWrapper 对象
public ParameterWrapper(Object object) throws HermesException
{
        if (object == null)
        {
            setName(false, "");
            mData = null;
            mClass = null;
        }
        else
        {
            //获取到参数对象的class 类型
            Class<?> clazz = object.getClass();
            mClass = clazz;
            //获取当参数对象的类名，分是否有ClassId注解还是没有
            setName(!clazz.isAnnotationPresent(ClassId.class), TypeUtils.getClassId(clazz));
            //将当前的对象使用gson转换为字符串
            mData = CodeUtils.encode(object);
        }
}
public Reply(ParameterWrapper parameterWrapper) {
    //获取到返回对象的类型
    Class<?> clazz = TYPE_CENTER.getClassType(parameterWrapper);
	//将返回的对象，利用gson，转成字符串,方便传输
    mResult = CodeUtils.decode(parameterWrapper.getData(), clazz);
	//标识当前请求成功
    mErrorCode = ErrorCodes.SUCCESS;
	//错误信息为空
    mErrorMessage = null;
    mClass = new TypeWrapper(clazz);
	...
}

当上面执行了返回之后，这边就能得到这个Reply这个结果了，
Reply reply = sender.send(null, tmp);
判断是否是正常的执行，如果不是打印错误信息
if (reply != null && !reply.success())
{
    Log.e(TAG,"Error occurs during getting instance. Error code: " + reply.getErrorCode());
    Log.e(TAG, "Error message: " + reply.getMessage());return null;
}

//在上面可以看到发送完之后，并没有用到返回的对象，而是通过代理的方式返回这个对象，这是为什么呢，因为这个对象在进程B中也不能用，而且进程B中要想调用
//这个对象的方法也还要通过IPC的机制来通信，所以这边可以直接返回一个代理的对象，代理的对象又可以知道你下次要调用什么方法，这不是很好
object.setType(ObjectWrapper.TYPE_OBJECT);
return getProxy(service, object);

看看getProxy(service, object)实现：
//通过动态代理的方式，返回一个代理的对象，这样我们得到的其实是代理的对象，这样当我们触发操作的时候，代理对象通过HermesInvocationHandler是能知道我们触发了什么操作 
private static <T> T getProxy(Class<? extends HermesService> service, ObjectWrapper object)
{
    Class<?> clazz = object.getObjectClass();
    //构建一个动态代理,并返回
    T proxy = (T) Proxy.newProxyInstance(clazz.getClassLoader(), new Class<?>[]{clazz}, new HermesInvocationHandler(service, object));
    HERMES_GC.register(service, proxy, object.getTimeStamp());
    return proxy;
}

HermesInvocationHandler实现为：

public class HermesInvocationHandler implements InvocationHandler {
    private static final String TAG = "HERMES_INVOCATION";
    private Sender mSender;

    //因为返回的是一个动态对象，所以当下次再触发什么操作的时候，这边就可以接受到，再次完成ipc的通信
    public HermesInvocationHandler(Class<? extends HermesService> service, ObjectWrapper object) {
        //构建一个Sender对象,只是这里的type变成了SenderDesignator.TYPE_INVOKE_METHOD ，这个逻辑就不看了
        mSender = SenderDesignator.getPostOffice(service, SenderDesignator.TYPE_INVOKE_METHOD, object);
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] objects) {
        try {
            //通过AIDL完成通信,这个逻辑上面也有...
            Reply reply = mSender.send(method, objects);
            if (reply == null) {
                return null;
            }
			
            //得到返回的结果，返回
            if (reply.success()) {
                return reply.getResult();//这里获取到要传递的对象，相当于俩个进程都有这样的对象存在，但是这俩个对象是不想关的。。
            } else {
                Log.e(TAG, "Error occurs. Error " + reply.getErrorCode() + ": " + reply.getMessage());
                return null;
            }
        } catch (HermesException e) {
            e.printStackTrace();
            Log.e(TAG, "Error occurs. Error " + e.getErrorCode() + ": " + e.getErrorMessage());
            return null;
        }
    }
}

所以最终这边能获取到 userManager.getUser() 另一个进程返回的值
Toast.makeText(getApplicationContext(), userManager.getUser(), Toast.LENGTH_SHORT).show();

```

总结：对于Hermes 多进程的调用，只不过是使用了IPC的机制，对于一个存在进程A的对象来说，如果进程B要想获取到就要通过IPC的机制获取到远端的IBInder引用，对于进程B想要操作进程A的这个对象
都是通过IBinder的机制来传递要修改的内容，另一端获取到了要修改的值，然后在利用反射的机制来修改当前进程的这个对象，对于返回当前进程的对象，虽然另一个进程确实得到了对象，但是这俩个
对象没有任务的关联，对于这个进程的对象来说，只是一个封装了数据的bean，如果这个进程要修改这个对象的值，就要通过IPC的机制来做到
