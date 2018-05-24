---
layout: pager
title: EventBus个人理解
date: 2018-05-22 16:43:16
tags: [Android,EventBus]
description:  EventBus个人理解
---

EventBus个人理解
<!--more-->

简介
```
当我们进行项目开发的时候，往往是需要应用程序的各组件、组件与后台线程间进行通信，比如在子线程中进行请求数据，当数据请求完毕后通过Handler或者是广播通知UI，
而两个Fragment之家可以通过Listener进行通信等等。当我们的项目越来越复杂，使用Intent、Handler、Broadcast进行模块间通信、模块与后台线程进行通信时，代码量大，而且高度耦合。
此时EventBus就可以很好的解决这个问题，这个框架目前在github上面的start数量已经达到了18k，可谓是一个大家都认同的牛逼的项目，所以我们就有必要去学习了解下人家的牛逼之处
```

****EventBus简单的使用****
===
```java
1.首先我们需要在接收方法的类里面注册，对应的代码为  EventBus.getDefault().register(this);
2.如果当前不需要接收方法了就有必要，取消注册，要不然会造成内存泄漏 对应的代码为  EventBus.getDefault().unregister(this);	
3.我们需要写一个函数用来响应接收，函数上面一定要有注解  @Subscribe(threadMode = ThreadMode.MAIN)，threadMode可以指定接受所处的线程，而且要注意这个接收的函数，只能有一个参数
如果你有多个参数，就有必要封装成一个对象，再传这个对象既可
4.在发送的另一方只要 执行这样的代码 EventBus.getDefault().post(Object obj);，就可以根据要发送的Object的类型，找到可以接受的方法，完成接收

下面是整体的demo
public class MainActivity extends Activity
{
    @Override
    public void onCreate(Bundle savedInstanceState)
    {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
		//注册
        EventBus.getDefault().register(this);

        findViewById(R.id.but).setOnClickListener(new View.OnClickListener()
        {
            @Override
            public void onClick(View view)
            {
                startActivity(new Intent(MainActivity.this,SecondActivity.class));
            }
        });
    }
	//接受
    @Subscribe(threadMode = ThreadMode.MAIN)
    public void onMessageEvent(String info) {
        Toast.makeText(MainActivity.this,info,Toast.LENGTH_SHORT).show();
    }


    @Override
    protected void onDestroy()
    {
        super.onDestroy();
		//取消注册
        EventBus.getDefault().unregister(this);
    }
}
 
public class SecondActivity extends Activity
{
    @Override
    public void onCreate(Bundle savedInstanceState)
    {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_second);

        findViewById(R.id.but2).setOnClickListener(new View.OnClickListener()
        {
            @Override
            public void onClick(View view)
            {
                EventBus.getDefault().post("Hello Form you");
            }
        });
    }
} 
```
结果演示
![结果显示](/uploads/EventDemo执行结果.png)

****EventBus源码分析****
===
```java
我们这样采用默认的配置，也即是我们测试程序demo这样的默认配置来分析源码

首先分析
EventBus.getDefault().register(this);

首先利用建造者模式，构建一个EventBus对象
private static final EventBusBuilder DEFAULT_BUILDER = new EventBusBuilder();
public EventBus() {
    this(DEFAULT_BUILDER);
}
EventBus(EventBusBuilder builder) {
    logger = builder.getLogger();
    subscriptionsByEventType = new HashMap<>();
    typesBySubscriber = new HashMap<>();
    stickyEvents = new ConcurrentHashMap<>();
    //获取到builder 中的MainThreadSupport对象，即为AndroidHandlerMainThreadSupport对象
    mainThreadSupport = builder.getMainThreadSupport();
    //获取到的即是 AndroidHandlerMainThreadSupport中的对应的函数
    mainThreadPoster = mainThreadSupport != null ? mainThreadSupport.createPoster(this) : null;
    backgroundPoster = new BackgroundPoster(this);
    asyncPoster = new AsyncPoster(this);
    indexCount = builder.subscriberInfoIndexes != null ? builder.subscriberInfoIndexes.size() : 0;
    //subscriberInfoIndexes 默认为空对象，strictMethodVerification 默认为false, ignoreGeneratedIndex默认也为false
    subscriberMethodFinder = new SubscriberMethodFinder(builder.subscriberInfoIndexes,builder.strictMethodVerification, builder.ignoreGeneratedIndex);
    logSubscriberExceptions = builder.logSubscriberExceptions;
    logNoSubscriberMessages = builder.logNoSubscriberMessages;
    sendSubscriberExceptionEvent = builder.sendSubscriberExceptionEvent;
    sendNoSubscriberEvent = builder.sendNoSubscriberEvent;
    throwSubscriberException = builder.throwSubscriberException;
    eventInheritance = builder.eventInheritance;
    executorService = builder.executorService;
}
然后调用对应的register(this)函数
public void register(Object subscriber) {
    Class<?> subscriberClass = subscriber.getClass();
    //查找当前class 对应的 已经父类所有的含有Subscriber 注解的方法，返回List<SubscriberMethod>的封装
    List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
    synchronized (this) {
        //遍历集合
        for (SubscriberMethod subscriberMethod : subscriberMethods) {
            subscribe(subscriber, subscriberMethod);
        }
    }
}

先来分析  List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);函数的实现为SubscriberMethodFinder 中的

class SubscriberMethodFinder {
...
	//缓存key为对应的class ，value为对应的含有注解的方法的集合
	private static final Map<Class<?>, List<SubscriberMethod>> METHOD_CACHE = new ConcurrentHashMap<>();

	//通过EventBus 传递进来，默认也是为null 的
	private List<SubscriberInfoIndex> subscriberInfoIndexes;

	//默认为false
	private final boolean strictMethodVerification;

	//默认为false
	private final boolean ignoreGeneratedIndex;

	//缓存的FindState的大小
	private static final int POOL_SIZE = 4;
	//缓存的 FindState 数组
	private static final FindState[] FIND_STATE_POOL = new FindState[POOL_SIZE];

	//找到class对应的List<SubscriberMethod>集合
	List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
		//首先根据key 从缓存的集合中获取
		List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
		//如果获取到了，就直接返回
		if (subscriberMethods != null) {
			return subscriberMethods;
		}	

		//如果没有获取到，则会进入这里面
		//ignoreGeneratedIndex 默认为false
		if (ignoreGeneratedIndex) {
			subscriberMethods = findUsingReflection(subscriberClass);
        } else {
            //默认会进入这里面
            subscriberMethods = findUsingInfo(subscriberClass);
        }

        if (subscriberMethods.isEmpty()) {
            throw new EventBusException("Subscriber " + subscriberClass + " and its super classes have no public methods with the @Subscribe annotation");
        } else {
         //将当前的class 已经对应的list<SubscriberMethod>填充到缓存中,并返回这个集合
        METHOD_CACHE.put(subscriberClass, subscriberMethods);
        return subscriberMethods;
		`}
	}
 ...

  根据上面的注释以及代码，可以看到首先会获取到当前注册对象的class类型，从缓存的集合中 METHOD_CACHE获取，如果获取不到，则新建一个，然后执行下面的代码
  subscriberMethods = findUsingInfo(subscriberClass);
 
  //根据class 找到对应的含有注解的集合
    private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
	
        //准备FindState,如果缓存中有就直接获取，返回这个缓存，如果没有，就新建一个
        FindState findState = prepareFindState();
        //初始化findState
        findState.initForSubscriber(subscriberClass);
        while (findState.clazz != null) {
            ....
            findUsingReflectionInSingleClass(findState);
			..
            //检查父类的注解的情况
            findState.moveToSuperclass();
        }
        //根据FindState 得到List<SubscriberMethod>集合
        return getMethodsAndRelease(findState);
    } 
}

//准备FindState,如果缓存中有就直接获取，返回这个缓存，如果没有，就新建一个
private FindState prepareFindState() {
        synchronized (FIND_STATE_POOL) {
            for (int i = 0; i < POOL_SIZE; i++) {
                FindState state = FIND_STATE_POOL[i];
                if (state != null) {
                    FIND_STATE_POOL[i] = null;
                    return state;
                }
            }
        }
    return new FindState();
}

下面是FindState的类的定义
static class FindState {
        //当前class上面所有的含有Subscribe注解的方法的集合,包括所有的父类的含有这个注解的
        final List<SubscriberMethod> subscriberMethods = new ArrayList<>();
        final Map<Class, Object> anyMethodByEventType = new HashMap<>();
        final Map<String, Class> subscriberClassByMethodKey = new HashMap<>();
        final StringBuilder methodKeyBuilder = new StringBuilder(128);

        Class<?> subscriberClass;
        Class<?> clazz;
        boolean skipSuperClasses;
        SubscriberInfo subscriberInfo;

        //初始化
        void initForSubscriber(Class<?> subscriberClass) {
            this.subscriberClass = clazz = subscriberClass;
            skipSuperClasses = false;
            subscriberInfo = null;
        }
...
}

然后执行到  findUsingReflectionInSingleClass(findState);
通过反射的方式获取
private void findUsingReflectionInSingleClass(FindState findState) {
        Method[] methods;
        try {
            // This is faster than getMethods, especially when subscribers are fat classes like Activities
            methods = findState.clazz.getDeclaredMethods();
        } catch (Throwable th) {
            // Workaround for java.lang.NoClassDefFoundError, see https://github.com/greenrobot/EventBus/issues/149
            methods = findState.clazz.getMethods();
            findState.skipSuperClasses = true;
        }
        //获取到所有的方法
        for (Method method : methods) {
            int modifiers = method.getModifiers();
            if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
                //获取到方法的参数类型
                Class<?>[] parameterTypes = method.getParameterTypes();
                //判断参数的个数，EventBus只支持一个参数
                if (parameterTypes.length == 1) {
                    //获取到参数上面的Subscribe的注解
                    Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                    //如果注解不为空，说明当前的这个方法是有能力接受内容的
                    if (subscribeAnnotation != null) {
                        Class<?> eventType = parameterTypes[0];
                        if (findState.checkAdd(method, eventType)) {
                            //获取到Subscribe上注解的值
                            ThreadMode threadMode = subscribeAnnotation.threadMode();
                            //构建一个SubscriberMethod,然后添加到集合中
                            findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode, subscribeAnnotation.priority(), subscribeAnnotation.sticky()));
                        }
                    }
                //EventBus只支持一个参数
                } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                    String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                    throw new EventBusException("@Subscribe method " + methodName +
                            "must have exactly 1 parameter but has " + parameterTypes.length);
                }
            } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                throw new EventBusException(methodName +
                        " is a illegal @Subscribe method: must be public, non-static, and non-abstract");
            }
        }
}
上面的函数就是说获取当前要注册的类里面所有的方法，检查是否有Subscriber注解，如果有注解，就判断参数类型是否是只有一个，如果都满足则拼接成一个SubscriberMethod对象，添加到subscriberMethods	
findState.subscriberMethods的本质为 final List<SubscriberMethod> subscriberMethods = new ArrayList<>();

SubscriberMethod类的定义为
public class SubscriberMethod {
    final Method method;
    //Subscriber上面的ThreadMode 的值
    final ThreadMode threadMode;
    //参数的类型
    final Class<?> eventType;
    //优先级
    final int priority;
    //是否是占性事件
    final boolean sticky;
	//方法名
    String methodString;

    public SubscriberMethod(Method method, Class<?> eventType, ThreadMode threadMode, int priority, boolean sticky) {
        this.method = method;
        this.threadMode = threadMode;
        this.eventType = eventType;
        this.priority = priority;
        this.sticky = sticky;
    }
...
}

while循环继续执行  findState.moveToSuperclass();
//跳过父类的检查
void moveToSuperclass() {
    if (skipSuperClasses) {
            clazz = null;
        } else {
            //获取到父类的class
            clazz = clazz.getSuperclass();
            String clazzName = clazz.getName();
            /** Skip system classes, this just degrades performance. */
            //简单的判断如果当前的class 是以java. 或者javax. 或者android.的跳过父类
            if (clazzName.startsWith("java.") || clazzName.startsWith("javax.") || clazzName.startsWith("android.")) {
                clazz = null;
        }
    }
}
这个函数的作用就是，我们不止需要检查我们当前的类，还需要去检查父类的情况，当然这个父类里面如果是系统的类，我们是没有必要去检查的，因为他们不可能有这个Subscriber注解的存在

所以当while循环执行完毕之后，我们就获取到了当前类以及父类含有Subscriber注解情况的集合
然后执行 
//根据FindState 得到List<SubscriberMethod>集合
return getMethodsAndRelease(findState);

//根据FindState 得到List<SubscriberMethod>集合
private List<SubscriberMethod> getMethodsAndRelease(FindState findState) {
    //获取到findState 中的subscriberMethods 集合的内容
    List<SubscriberMethod> subscriberMethods = new ArrayList<>(findState.subscriberMethods);
    //回收这个FindState对象
    findState.recycle();
    //当前的FindState 对象 放入缓存池中
    synchronized (FIND_STATE_POOL) {
        for (int i = 0; i < POOL_SIZE; i++) {
            if (FIND_STATE_POOL[i] == null) {
                FIND_STATE_POOL[i] = findState;
                break;
            }
        }
    }
    //返回这个集合
    return subscriberMethods;
}

最后执行
//将当前的class 已经对应的list<SubscriberMethod>填充到缓存中,并返回这个集合
METHOD_CACHE.put(subscriberClass, subscriberMethods);
return subscriberMethods;

register函数继续执行
public void register(Object subscriber) {
    Class<?> subscriberClass = subscriber.getClass();
    //查找当前class 对应的 已经父类所有的含有Subscriber 注解的方法，返回List<SubscriberMethod>的封装
    List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
    synchronized (this) {
        //遍历集合
		for (SubscriberMethod subscriberMethod : subscriberMethods) {
            subscribe(subscriber, subscriberMethod);
        }
    }
}

subscribe(subscriber, subscriberMethod);函数的实现：
// Must be called in synchronized block
private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
        //获取到参数的类型
        Class<?> eventType = subscriberMethod.eventType;
        //构建一个Subsciption对象，里面存储了subscriber 订阅者，以及 SubscriberMethod
        Subscription newSubscription = new Subscription(subscriber, subscriberMethod);

        //根据当前参数的type的从集合中获取Subscription 集合
        CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
        if (subscriptions == null) {
            //如果获取为空，则构建一个新的，然后添加到subscriptionsByEventType 中
            subscriptions = new CopyOnWriteArrayList<>();
            subscriptionsByEventType.put(eventType, subscriptions);
        } else {
            if (subscriptions.contains(newSubscription)) {
                throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
                        + eventType);
            }
        }
        //获取到当前type对应的Subscription 集合，遍历集合中每一个元素的priority 跟当前的priority对比，重新的构建一个Subscription 集合
        int size = subscriptions.size();
        for (int i = 0; i <= size; i++) {
            if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
                subscriptions.add(i, newSubscription);
                break;
            }
        }

        //根据当前的订阅者从集合typesBySubscriber 中获取到对应的List<Class>>集合
        List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
        //如果获取为空，则构建一个新的集合
        if (subscribedEvents == null) {
            subscribedEvents = new ArrayList<>();
            typesBySubscriber.put(subscriber, subscribedEvents);
        }
        //往集合中添加参数的类型
        subscribedEvents.add(eventType);
        ...
}
上面函数中几个map集合的定义为
//缓存的集合 key为对应的post参数的类型，value为对应的class集合包含当前的post参数的类型，已经父类的接口
private static final Map<Class<?>, List<Class<?>>> eventTypesCache = new HashMap<>();

//缓存的集合 key为参数的类型 value为Subscription 集合
private final Map<Class<?>, CopyOnWriteArrayList<Subscription>> subscriptionsByEventType;

//缓存的集合 key为 订阅者的对象，value为含有Subscriber注解方法的参数的type集合
private final Map<Object, List<Class<?>>> typesBySubscriber;


总结下：其实register函数，目的就是为了维护几个集合，这几个集合里面存储了必要的信息，比如订阅者对象，订阅者类包含的所有的订阅函数的集合，订阅的参数等,目的就是为了下一步post的时候
从对应的集合中能找到对应的值，然后根据反射调用对应的方法


下面解析分析
EventBus.getDefault().post("Hello Form you");
/** Posts the given event to the event bus. */
//发送一个事件，要注意的是EventBus只支持一个参数,如果有多个参数需要封装成一个对象，再来传递
public void post(Object event) {
        //从ThreadLocal中获取到PostingThreadState对象
        PostingThreadState postingState = currentPostingThreadState.get();
        //获取到当前的eventQueue事件的队列
        List<Object> eventQueue = postingState.eventQueue;
        //添加当前的事件
        eventQueue.add(event);

        //默认为false
        if (!postingState.isPosting) {
            //判断是否是主线程
            postingState.isMainThread = isMainThread();
            postingState.isPosting = true;
            //判断是否是取消状态
            if (postingState.canceled) {
                throw new EventBusException("Internal error. Abort state was not reset");
            }
            //如果不是取消状态，发送事件
            try {
                //发送队列事件的内容
                while (!eventQueue.isEmpty()) {
                    postSingleEvent(eventQueue.remove(0), postingState);
                }
            } finally {
                postingState.isPosting = false;
                postingState.isMainThread = false;
            }
        }
}

PostingThreadState postingState = currentPostingThreadState.get();使用ThreadLocal来获取数据，第一次化会创建一个 new PostingThreadState();对象

//使用ThreadLocal来保存数据,速度比较快
private final ThreadLocal<PostingThreadState> currentPostingThreadState = new ThreadLocal<PostingThreadState>() {
    @Override
    protected PostingThreadState initialValue() {
        return new PostingThreadState();
    }
};
PostingThreadStated对象的定义为:
final static class PostingThreadState {
    //当前的事件的队列
    final List<Object> eventQueue = new ArrayList<>();
    //true
    boolean isPosting;
    //标识是否是在主线程
    boolean isMainThread;
    //标识找到的Subscription对象
    Subscription subscription;
    //标识当前的参数对象
    Object event;
    //是否是取消状态
    boolean canceled;
}

isMainThread();函数的实现为：
//简单的封装
class AndroidHandlerMainThreadSupport implements MainThreadSupport {

    private final Looper looper;

    public AndroidHandlerMainThreadSupport(Looper looper) {
        this.looper = looper;
    }

    //判断是否是在主线程
    @Override
    public boolean isMainThread() {
        return looper == Looper.myLooper(); //判断是否在主线程也即是简单的当前线程的Looper对象跟主线程的Looper对象是否一样，每一个线程都有一个Looper对象
    }

    @Override
    public Poster createPoster(EventBus eventBus) {
      return new HandlerPoster(eventBus, looper, 10);
	}
}

继续执行 发送队列事件的内容
while (!eventQueue.isEmpty()) {
    postSingleEvent(eventQueue.remove(0), postingState);
}

//发送单一的事件
private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
        //获取到事件的类型,也即是参数的类型
        Class<?> eventClass = event.getClass();
        //标识是否找到了订阅者
        boolean subscriptionFound = false;
        //eventInheritance 默认为true
        if (eventInheritance) {
            //获取到当前事件的继承关系的集合，也即是说可以传递父类的对象也是允许的
            List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);
            int countTypes = eventTypes.size();
            for (int h = 0; h < countTypes; h++) {
                Class<?> clazz = eventTypes.get(h);
                //发送单一事件
                subscriptionFound |= postSingleEventForEventType(event, postingState, clazz);
            }
        } else {
            subscriptionFound = postSingleEventForEventType(event, postingState, eventClass);
        }
        if (!subscriptionFound) {
            if (logNoSubscriberMessages) {
                logger.log(Level.FINE, "No subscribers registered for event " + eventClass);
            }
            if (sendNoSubscriberEvent && eventClass != NoSubscriberEvent.class &&
                    eventClass != SubscriberExceptionEvent.class) {
                post(new NoSubscriberEvent(this, event));
            }
        }
}

//从缓存的集合中获取符合这个参数类型的集合
private static List<Class<?>> lookupAllEventTypes(Class<?> eventClass) {
        synchronized (eventTypesCache) {
            //根据当前的参数的类型从eventTypesCache中获取到对应的List<Class<?>> 集合
            List<Class<?>> eventTypes = eventTypesCache.get(eventClass);
            //如果为空，则构建一个集合
            if (eventTypes == null) {
                eventTypes = new ArrayList<>();
                Class<?> clazz = eventClass;
                while (clazz != null) {
                    eventTypes.add(clazz);
                    addInterfaces(eventTypes, clazz.getInterfaces());
                    clazz = clazz.getSuperclass();
                }
                //填充进集合中
                eventTypesCache.put(eventClass, eventTypes);
            }
            return eventTypes;
        }
}


postSingleEventForEventType函数的实现为：
//第一个参数代表参数的对象， 最后一个参数为参数的类型
private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
        CopyOnWriteArrayList<Subscription> subscriptions;
        synchronized (this) {
            //首先从缓存的集合 key为参数的类型 value为Subscription 集合
            subscriptions = subscriptionsByEventType.get(eventClass);
        }
        //如果获取到的 Subscription集合不为空,遍历集合的内容
        if (subscriptions != null && !subscriptions.isEmpty()) {
            //注意这里是遍历符合的条件的所有的subscriptions的集合
            for (Subscription subscription : subscriptions) {
                //给postingState 赋值
                postingState.event = event;
                postingState.subscription = subscription;
                boolean aborted = false;
                try {
                    //发送事件
                    postToSubscription(subscription, event, postingState.isMainThread);
                    aborted = postingState.canceled;
                } finally {
                    postingState.event = null;
                    postingState.subscription = null;
                    postingState.canceled = false;
                }
                if (aborted) {
                    break;
                }
            }
            return true;
        }
        return false;
}

//发送事件 ,这里要比较isMainThread 的情况   subscription 代表为当前事件的封装，里面有订阅者对象
private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
        //根据当前的含有Subscriber注解方法的threadMode的情况来执行不同的情况
        switch (subscription.subscriberMethod.threadMode) {
            case POSTING://直接发送,这是默认的设置，这种设置减少了线程切换的开销
                invokeSubscriber(subscription, event);
                break;
            case MAIN://如果当前订阅的方法是在主线程
                if (isMainThread) {//并且当前触发事件的也是在主线程，那么直接调用
                    invokeSubscriber(subscription, event);
                } else {
                    //如果订阅的方法是在主线程，而触发事件的不是在主线程，则要进行线程的切换
                    mainThreadPoster.enqueue(subscription, event);
                }
                break;
            case MAIN_ORDERED:
                //如果mainThreadPoster不为空，则直接切换到主线程中
                if (mainThreadPoster != null) {
                    mainThreadPoster.enqueue(subscription, event);
                } else {
                    // temporary: technically not correct as poster not decoupled from subscriber
                    invokeSubscriber(subscription, event);
                }
                break;
            case BACKGROUND://如果当前订阅者是在子线程
                if (isMainThread) {//而发布事件的是处于主线程，也要将主线程切换到子线程
                    backgroundPoster.enqueue(subscription, event);
                } else {
                    //如果同处于同一个线程直接调用
                    invokeSubscriber(subscription, event);
                }
                break;
            case ASYNC:
                //异步的
                asyncPoster.enqueue(subscription, event);
                break;
            default:
                throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
        }
}

//执行订阅的方法
void invokeSubscriber(Subscription subscription, Object event) {
    try {
        //也即是通过反射的方式来实现的,什么参数都有
        subscription.subscriberMethod.method.invoke(subscription.subscriber, event);
    } catch (InvocationTargetException e) {
        handleSubscriberException(subscription, event, e.getCause());
    } catch (IllegalAccessException e) {
        throw new IllegalStateException("Unexpected exception", e);
    }
}

可以看到最终都是通过反射的方式来调用对应的方法，这里分析下线程的切换

如果订阅的方法是在主线程，而触发事件的不是在主线程，则要进行线程的切换 ,及时要切换到主线程
mainThreadPoster.enqueue(subscription, event);

首先来分析下mainThradPoster对象代表的是什么，他在构建EventBus的时候就进行了初始化
//获取到builder 中的MainThreadSupport对象，即为AndroidHandlerMainThreadSupport对象
mainThreadSupport = builder.getMainThreadSupport();
//获取到的即是 AndroidHandlerMainThreadSupport中的对应的函数
mainThreadPoster = mainThreadSupport != null ? mainThreadSupport.createPoster(this) : null;

所以当执行了 mainThreadSupport.createPoster(this)的时候，就会执行到对应的函数 也即是new HandlerPoster(eventBus, looper, 10); ,这里要注意这里的looper对象为主线程的Loopder对象

mainThreadSupport = builder.getMainThreadSupport()赋值的函数实现为：
MainThreadSupport getMainThreadSupport() {
        //默认为空
        if (mainThreadSupport != null) {
            return mainThreadSupport;
        } else if (AndroidLogger.isAndroidLogAvailable()) {//所以会进入这里面
            //获取到AndroidMainLooper对象
            Object looperOrNull = getAndroidMainLooperOrNull();
            return looperOrNull == null ? null : new MainThreadSupport.AndroidHandlerMainThreadSupport((Looper) looperOrNull);
        } else {
            return null;
        }
    }

    Object getAndroidMainLooperOrNull() {
        try {
            return Looper.getMainLooper();
        } catch (RuntimeException e) {
            // Not really a functional Android (e.g. "Stub!" maven dependencies)
            return null;
        }
}
	
//简单的封装
class AndroidHandlerMainThreadSupport implements MainThreadSupport {

        private final Looper looper;

        public AndroidHandlerMainThreadSupport(Looper looper) {
            this.looper = looper;
        }

        //判断是否是在主线程
        @Override
        public boolean isMainThread() {
            return looper == Looper.myLooper();
        }

        @Override
        public Poster createPoster(EventBus eventBus) {
            return new HandlerPoster(eventBus, looper, 10);
        }
}

所以当执行 mainThreadPoster.enqueue(subscription, event);的时候就会执行对应的函数
//添加队列
public void enqueue(Subscription subscription, Object event) {
    //构建一个PendingPost对象
    PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
    synchronized (this) {
            //添加队列
            queue.enqueue(pendingPost);
            if (!handlerActive) {
                handlerActive = true;
                //发送消息
				if (!sendMessage(obtainMessage())) {
                    throw new EventBusException("Could not send handler message");
            }
        }
    }
}

PendingPost.obtainPendingPost(subscription, event);函数的实现为：
//构建一个PendingPost对象
static PendingPost obtainPendingPost(Subscription subscription, Object event) {
        synchronized (pendingPostPool) {
            //如果PendingPost集合中有内容，移除最后一个，则为复用,将
            int size = pendingPostPool.size();
            if (size > 0) {
                PendingPost pendingPost = pendingPostPool.remove(size - 1);
                pendingPost.event = event;
                pendingPost.subscription = subscription;
                pendingPost.next = null;
                return pendingPost;
        }
    }
    //如果集合中没有，则新建一个PendingPost对象
    return new PendingPost(event, subscription);
}

然后执行发送消息 sendMessage(obtainMessage() ，所以handlerMessage接受到回调
@Override
public void handleMessage(Message msg) {
        boolean rescheduled = false;
        try {
            long started = SystemClock.uptimeMillis();
            while (true) {
                //从队列里面取出当前的PendingPost对象
                PendingPost pendingPost = queue.poll();
                if (pendingPost == null) {
                    synchronized (this) {
                        // Check again, this time in synchronized
                        pendingPost = queue.poll();
                        if (pendingPost == null) {
                            handlerActive = false;
                            return;
                        }
                    }
                }
                //执行方法
                eventBus.invokeSubscriber(pendingPost);
                long timeInMethod = SystemClock.uptimeMillis() - started;
                if (timeInMethod >= maxMillisInsideHandleMessage) {
                    if (!sendMessage(obtainMessage())) {
                        throw new EventBusException("Could not send handler message");
                    }
                    rescheduled = true;
                    return;
                }
            }
        } finally {
            handlerActive = rescheduled;
        }
}

最终执行
eventBus.invokeSubscriber(pendingPost);
void invokeSubscriber(PendingPost pendingPost) {
    Object event = pendingPost.event;
    Subscription subscription = pendingPost.subscription;
    PendingPost.releasePendingPost(pendingPost);
    if (subscription.active) {
        invokeSubscriber(subscription, event);
    }
}

//执行订阅的方法
void invokeSubscriber(Subscription subscription, Object event) {
    try {
        //也即是通过反射的方式来实现的,什么参数都有
        subscription.subscriberMethod.method.invoke(subscription.subscriber, event);
    } catch (InvocationTargetException e) {
        handleSubscriberException(subscription, event, e.getCause());
    } catch (IllegalAccessException e) {
        throw new IllegalStateException("Unexpected exception", e);
    }
}

所以他最终通过主线程的handler来将线程切换到主线程来

接下来分析下子线程的情况
if (isMainThread) {//而发布事件的是处于主线程，也要将主线程切换到子线程
    backgroundPoster.enqueue(subscription, event);
} else {
    //如果同处于同一个线程直接调用
   invokeSubscriber(subscription, event);
}

backgroundPoster.enqueue(subscription, event);
backgroundPoster对象为   backgroundPoster = new BackgroundPoster(this);本质为一个Runnable对象,下面是具体的定义

final class BackgroundPoster implements Runnable, Poster {
    //事件的队列
    private final PendingPostQueue queue;
    private final EventBus eventBus;

    //标识线程池是否正在运行
    private volatile boolean executorRunning;

    BackgroundPoster(EventBus eventBus) {
        this.eventBus = eventBus;
        queue = new PendingPostQueue();
    }
	...
}

所以当执行了 backgroundPoster.enqueue(subscription, event);的时候，就会执行到对应的函数
public void enqueue(Subscription subscription, Object event) {
        //构建一个PendingPost对象
        PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
        synchronized (this) {
            //添加到队列中
            queue.enqueue(pendingPost);
            //如果当前线程池没有正在运行，则运行当前的BackgroundPoster
            if (!executorRunning) {
                executorRunning = true;
                eventBus.getExecutorService().execute(this);
            }
        }
}

当执行到了 eventBus.getExecutorService().execute(this);
获取到EventBus中的ExecutorService对象，这个对象的赋值地方为 executorService = builder.executorService;
ExecutorService executorService = DEFAULT_EXECUTOR_SERVICE;
//线程池对象,用来执行线程间的切换操作
private final static ExecutorService DEFAULT_EXECUTOR_SERVICE = Executors.newCachedThreadPool();

所以最终这里获取到的是build中默认的ExecutorService对象即为DEFAULT_EXECUTOR_SERVICE 所以通过execute 提交线程，让线程执行，最终会回调执行run方法
@Override
    public void run() {
        try {
            try {
                //子线程执行了
                while (true) {
                    //获取到队列里面的内容
                    PendingPost pendingPost = queue.poll(1000);
                    if (pendingPost == null) {
                        synchronized (this) {
                            // Check again, this time in synchronized
                            pendingPost = queue.poll();
                            if (pendingPost == null) {
                                executorRunning = false;
                                return;
                            }
                        }
                    }
                    //回调执行
                    eventBus.invokeSubscriber(pendingPost);
                }
            } catch (InterruptedException e) {
                eventBus.getLogger().log(Level.WARNING, Thread.currentThread().getName() + " was interruppted", e);
            }
        } finally {
            executorRunning = false;
        }
}

又回到了这里  eventBus.invokeSubscriber(pendingPost); 上面已经介绍了，这里就不介绍了

总结主线程切换到子线程是通过ExecutorService 提交一个任务的方式，来切换到子线程
```

总结：EventBus的巧妙之处就是那几个集合存储的内容，当post事件发生的时候，可以根据这几个集合的内容找到对应的函数，还有订阅者对象，然后利用反射的方式来调用，主线程的切换是通过
主线程的Handler方式来做到的，子线程的切换是通过ExecutorService的方式来切换的


