---
layout: pager
title: Xutils Ioc实现
date: 2018-06-01 11:10:35
tags: [Android,Xutils,Ioc]
description:  Xutils Ioc实现
---

### 概述

> Xutils Ioc实现

<!--more-->

### 什么是IOC
> Ioc也叫 控制反转(Inversion of Control，英文缩写为IoC)把创建对象的权利交给框架,是框架的重要特征，并非面向对象编程的专用术语。它包括依赖注入(Dependency Injection，简称DI)和依赖查找(Dependency Lookup)。通俗点的意思就是说把创建对象的权利交给框架来实现，比如这个事情本身是我做的，但是我可以交给你来完成

### Xutils Ioc使用
```gradle
使用Gradle构建时添加一下依赖即可:
compile 'org.xutils:xutils:3.5.0'
```
需要的权限
```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
```
Java代码的编写
```java
//首先在BaseActivity里面执行注册
public class BaseActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        x.view().inject(this);
    }
}

之后就可以在子类里面通过下面注解的形式来填充操作，比如设置setContentView,findViewById的操作，甚至事件的操作，

@ContentView(R.layout.activity_main)//setContentView的操作
public class MainActivity extends BaseActivity {

    @ViewInject(R.id.container)
    private ViewPager mViewPager;

    @ViewInject(R.id.toolbar)
    private Toolbar toolbar;

    @ViewInject(R.id.tabs) //findViewById的操作
    private TabLayout tabLayout;
	
    ...
	
    @Event(R.id.btn_test_db)//事件的绑定操作
    private void onTestDbClick(View view) {
	
    }
}

下面 分析 xUtils中的IOC实现
```

### Xutils Ioc分析

```java
1.@ContentView(R.layout.activity_main) 原理实现：

ContentView注解的声明为：

//标识注解用在类上面
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface ContentView {
    //注解的参数
    int value();
}

首先我们在BaseActivity中执行了注册
public class BaseActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        x.view().inject(this);
    }
}

执行了 x.view().inject(this);的操作，最终会执行到这个方法
@Override
public void inject(Activity activity) {
    //获取Activity的ContentView的注解
    Class<?> handlerType = activity.getClass();
    try {
        //获取注注解
        ContentView contentView = findContentView(handlerType);
        if (contentView != null) {
            //如果不为空，就获取到注解参数的内容
            int viewId = contentView.value();
            if (viewId > 0) {
                //获取到setContentView的 Method
                Method setContentViewMethod = handlerType.getMethod("setContentView", int.class);
                //然后通过反射的方式来调用，这样就相当于是调用了setContentView(viewId)
                setContentViewMethod.invoke(activity, viewId);
            }
        }
    } catch (Throwable ex) {
        LogUtil.e(ex.getMessage(), ex);
    }

    //对于ViewInject 已经 Event 注解原理实现的地方，后面会介绍到：
    injectObject(activity, handlerType, new ViewFinder(activity));
}

findContentView(handlerType)的实现：
//首先从当前类中查找 是否有 ContentView.class 注解，如果没有，再从父类中查找 
private static ContentView findContentView(Class<?> thisCls) {
    if (thisCls == null || IGNORED.contains(thisCls)) {
        return null;
    }
    ContentView contentView = thisCls.getAnnotation(ContentView.class);
    if (contentView == null) {
        return findContentView(thisCls.getSuperclass());
    }
    return contentView;
}

所以对于@ContentView(R.layout.activity_main) 所做的事无非就是获取到注解上面传递的参数，然后通过反射的方式调用setContentView来设置布局

分析 @ViewInject(R.id.toolbar)
private Toolbar toolbar;

ViewInject注解的声明为：

//标识这个注解可以用在成员变量上
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface ViewInject {

    int value();

    /* parent view id */
    int parentId() default 0;
}

前面分析到了这个地方  injectObject(activity, handlerType, new ViewFinder(activity)); ，现在就可以接着分析了,下面是这个函数的关键代码

// inject view
Field[] fields = handlerType.getDeclaredFields();
if (fields != null && fields.length > 0) {
    for (Field field : fields) {
        //获取到成员变量的class类型，判断注入的字段的是否合法，这里不允许有静态字段，final字段，基本类型，数组类型
        Class<?> fieldType = field.getType();
        if (
            /* 不注入静态字段 */     Modifier.isStatic(field.getModifiers()) ||
            /* 不注入final字段 */    Modifier.isFinal(field.getModifiers()) ||
            /* 不注入基本类型字段 */  fieldType.isPrimitive() || /* 不注入数组类型字段 */  fieldType.isArray())
        {
            continue;
        }
		
        //如果viewInject不为空，则说明当前的字段field字段含有ViewInject的注解
        ViewInject viewInject = field.getAnnotation(ViewInject.class);
        if (viewInject != null) {
            try {
                //viewInject.value() 获取到注解的值,通过持有Activity的引用调用findViewById来获取到对应的View
                View view = finder.findViewById(viewInject.value(), viewInject.parentId());
                if (view != null) {
                    //然后将获取到的view，通过反射的方式给这个字段赋值，这样这个字段就有值了，可以正常使用了
                    field.setAccessible(true);
                    field.set(handler, view);
                } else {
                        throw new RuntimeException("Invalid @ViewInject for "+ handlerType.getSimpleName() + "." + field.getName());
                    }
                } catch (Throwable ex) {
                    LogUtil.e(ex.getMessage(), ex);
            }
        }
    }
} // end inject view

finder.findViewById(viewInject.value(), viewInject.parentId()); 函数的实现：

先来看看ViewFinder主要是用来执行findViewById，至于找的过程也是很简单的。通过持有Activity，或者View的 引用，调用对应的findViewById的方式来获取，至于初始化的地方为：
injectObject(activity, handlerType, new ViewFinder(activity))，发现没有这里传递进了activity的引用

final class ViewFinder {

    private View view;
    private Activity activity;

    public ViewFinder(View view) {
        this.view = view;
    }

    public ViewFinder(Activity activity) {
        this.activity = activity;
    }

    public View findViewById(int id) {
        if (view != null) return view.findViewById(id);
        if (activity != null) return activity.findViewById(id);
        return null;
    }

    public View findViewByInfo(ViewInfo info) {
        return findViewById(info.value, info.parentId);
    }

    //可以发现findViewById还是通过传递进来的activity或者View findViewById来实现的。。。这里为什么不用反射，估计是因为反射的性能不好，所以用原始的方式来调用就最好
    public View findViewById(int id, int pid) {
        View pView = null;
        if (pid > 0) {
            pView = this.findViewById(pid);
        }

        View view = null;
        if (pView != null) {
            view = pView.findViewById(id);
        } else {
            view = this.findViewById(id);
        }
        return view;
    }
}

所以对于@ViewInject(R.id.toolbar) 所做的事无非就是通过持有要注册Activity对象引用，然后通过findViewById的方式来获取到对应view，然后获取到ViewInject注解的值，
找到对应的成员变量,然后通过反射的方式将这个view的值设置到这个字段上面，这样就完成了赋值操作，其实findViewById完全也可以通过反射的方式来获取，但是这样性能就比较低
由于是对于大量的findViewById操作


3 分析事件的绑定 @Event(R.id.btn_test_db)//事件的绑定操作
private void onTestDbClick(View view) {
}

Event注解的声明为：

/**
 * 事件注解.
 * 被注解的方法必须具备以下形式:
 * 1. private 修饰
 * 2. 返回值类型没有要求
 * 3. 参数签名和type的接口要求的参数签名一致.
 * Author: wyouflf
 * Date: 13-9-9
 * Time: 下午12:43
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Event {

    /**
     * 控件的id集合, id小于1时不执行ui事件绑定.
     *
     * @return
     */
    int[] value();

    /**
     * 控件的parent控件的id集合, 组合为(value[i], parentId[i] or 0).
     *
     * @return
     */
    int[] parentId() default 0;

    /**
     * 事件的listener, 默认为点击事件.
     *
     * @return
     */
    Class<?> type() default View.OnClickListener.class;

    /**
     * 事件的setter方法名, 默认为set+type#simpleName.
     *
     * @return
     */
    String setter() default "";

    /**
     * 如果type的接口类型提供多个方法, 需要使用此参数指定方法名.
     *
     * @return
     */
    String method() default "";
}


// inject event 方法实现
//获取到所有的方法
Method[] methods = handlerType.getDeclaredMethods();
if (methods != null && methods.length > 0) {
    for (Method method : methods) {

    //如果方法为静态或者为私有的，就会跳过
    if (Modifier.isStatic(method.getModifiers()) || !Modifier.isPrivate(method.getModifiers())) {
        continue;
    }

    //检查当前方法是否是event注解的方法
    Event event = method.getAnnotation(Event.class);
    if (event != null) {
        try {
            // id数组参数
            int[] values = event.value();
            int[] parentIds = event.parentId();
            int parentIdsLen = parentIds == null ? 0 : parentIds.length;
            //循环所有id，生成ViewInfo并添加代理反射
            for (int i = 0; i < values.length; i++) {
                int value = values[i];
                if (value > 0) {
                    ViewInfo info = new ViewInfo();
                    info.value = value;
                    info.parentId = parentIdsLen > i ? parentIds[i] : 0;
                    method.setAccessible(true);
                    EventListenerManager.addEventMethod(finder, info, event, handler, method);
                }
            }
        } catch (Throwable ex) {
            LogUtil.e(ex.getMessage(), ex);
        }
    }
  }
} // end inject event

EventListenerManager.addEventMethod(finder, info, event, handler, method);函数的实现为：

public static void addEventMethod(
            //根据页面或view holder生成的ViewFinder
            ViewFinder finder,
            //根据当前注解ID生成的ViewInfo
            ViewInfo info,
            //注解对象
            Event event,
            //页面或view holder对象
            Object handler,
            //当前注解方法
            Method method) {
        try {
		
            //根据id获取到的View，这里是通过finder来获取到，上面已经分析过了finder.findViewByInfo,无非就是持有Acvitity的引用然后调用findViewById找到对应的View
            View view = finder.findViewByInfo(info);

            if (view != null) {
                // 注解中定义的接口，比如Event注解默认的接口为View.OnClickListener
                Class<?> listenerType = event.type();
                // 默认为空，注解接口对应的Set方法，比如setOnClickListener方法
                String listenerSetter = event.setter();
                if (TextUtils.isEmpty(listenerSetter)) {
                    listenerSetter = "set" + listenerType.getSimpleName();
                }

                String methodName = event.method();

                boolean addNewMethod = false;
                /*
                 *根据View的ID和当前的接口类型获取已经缓存的接口实例对象，
                 *比如根据View.id和View.OnClickListener.class两个键获取这个View的OnClickListener对象
                 */
                Object listener = listenerCache.get(info, listenerType);
                DynamicHandler dynamicHandler = null;
                /*
                 * 如果接口实例对象不为空
                 * 获取接口对象对应的动态代理对象
                 * 如果动态代理对象的handler和当前handler相同
                 * 则为动态代理对象添加代理方法
                 */
                if (listener != null) {
                    dynamicHandler = (DynamicHandler) Proxy.getInvocationHandler(listener);
                    //判断当前是否已经添加过，如果为true的化，为之前已经添加过
                    addNewMethod = handler.equals(dynamicHandler.getHandler());
                    if (addNewMethod) {
                        dynamicHandler.addMethod(methodName, method);
                    }
                }

                // 如果还没有注册此代理
                if (!addNewMethod) {
                    dynamicHandler = new DynamicHandler(handler);
                    dynamicHandler.addMethod(methodName, method);
                    // 生成的代理对象实例，比如View.OnClickListener的实例对象
                    listener = Proxy.newProxyInstance(
                            listenerType.getClassLoader(),
                            new Class<?>[]{listenerType},
                            dynamicHandler);
                    //缓存起来		
                    listenerCache.put(info, listenerType, listener);
                }

                //相当于给view调用setOnClickListner(listener)，
                Method setEventListenerMethod = view.getClass().getMethod(listenerSetter, listenerType);
                setEventListenerMethod.invoke(view, listener);
            }
        } catch (Throwable ex) {
            LogUtil.e(ex.getMessage(), ex);
        }
}
我们看看DynamicHandler的实现为：
public static class DynamicHandler implements InvocationHandler {

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        ....
        try {
                return method.invoke(handler, args);
            } catch (Throwable ex) {
                throw new RuntimeException("invoke method error:" + handler.getClass().getName() + "#" + method.getName(), ex);
            }
        ...	
    }	
}

可以发现事件的绑定主要是通过注解id，找到对应的View，然后根据注解上面要设置的listener的三要素，找到对应的Method，然后利用反射的形式设置给这个view一个监听
只不过这个监听对象是通过动态代理获取到的，当真正触发操作的时候，再利用反射的方式回调到我们创建的方法上面

```

### 手写Xutils的Ioc

```java
用来填充布局的注解
@Retention(RetentionPolicy.RUNTIME)
//标识注解用在类上面
@Target({ElementType.TYPE})
public @interface ContentView
{
    //注解的参数，默认值为-1
    int value() default -1;
}

用来变量的赋值操作的注解
@Retention(RetentionPolicy.RUNTIME)
//标识方法要作用在成员变量上面
@Target(ElementType.FIELD)
public @interface ViewJect
{
    int value() default -1;
}

用来标识一个事件的三要素 注解,注意这个注解是做用到另一个注解上面的
@Retention(RetentionPolicy.RUNTIME)
//标识这个注解要在另一个注解上面使用
@Target(ElementType.ANNOTATION_TYPE)
public @interface EventBase
{
    /**
     * 事件的名称， 比如 setOnClickListener
     * @return
     */
    String eventStr();

    /**
     * 事件的类型 ，比如 new View.OnClickListener()
     * @return
     */
    Class<?> eventType();


    /**
     * 事件的回调方法 比如  onClick
     * @return
     */
    String  eventMethod();

}

@Retention(RetentionPolicy.RUNTIME)
//声明这个注解要在方法上面使用
@Target(ElementType.METHOD)
@EventBase(eventStr = "setOnClickListener",eventType = View.OnClickListener.class,eventMethod = "onClick")
public @interface onClick
{
    int[] value();//定义整形数组属性，用来存储多个的点击事件，这里可以传递多个view的Id进来
}

@Retention(RetentionPolicy.RUNTIME)
//声明这个注解要在方法上面使用
@Target(ElementType.METHOD)
@EventBase(eventStr = "setOnLongClickListener",eventType = View.OnLongClickListener.class,eventMethod = "onLongClick")
public @interface onLongClick
{
    int[] value();//定义整形数组属性，用来存储多个的点击事件，这里可以传递多个view的Id进来
}


基类的使用
public class BaseActivity extends AppCompatActivity
{
    @Override
    protected void onCreate(Bundle savedInstanceState)
    {
        super.onCreate(savedInstanceState);

        //注入
        InjectUtils.inject(this);
    }
}

子类的使用
@ContentView(R.layout.activity_main)
public class MainActivity extends BaseActivity
{

    @ViewJect(R.id.but1)
    public Button mBut1;

    @ViewJect(R.id.but2)
    public Button mBut2;

    @Override
    protected void onCreate(Bundle savedInstanceState)
    {
        super.onCreate(savedInstanceState);

        Toast.makeText(MainActivity.this,"---->"+ mBut1,Toast.LENGTH_LONG).show();
    }

    @onClick({R.id.but1,R.id.but2})
    public void MyClick(View view)
    {
        Toast.makeText(MainActivity.this,"---->"+ mBut1,Toast.LENGTH_LONG).show();
    }
}

/**
 * Project Name: MtXutilsIoc
 * File Name:    InjectUtils.java
 * ClassName:    InjectUtils
 *
 * Description: IOC容器 控制反转(Inversion of Control，英文缩写为IoC)把创建对象的权利交给框架,是框架的重要特征，并非面向对象编程的专用术语。
 * 它包括依赖注入(Dependency Injection，简称DI)和依赖查找(Dependency Lookup)。
 *
 * @author Zhangyuhui
 * @date 2018年06月01日 9:19
 *
 * Copyright (c) 2018年, 4399 Network CO.ltd. All Rights Reserved.
 */
public class InjectUtils
{

    /**
     * 提供注入的方法
     * @param object 要注入的对象
     */
    public static void inject(Object object)
    {
        injectContentView(object);
        injectView(object);
        injectEvent(object);
    }

    /**
     * 注入事件
     * @param object 要注入的对象
     */
    private static void injectEvent(Object object)
    {
        Class<?> objectClass = object.getClass();
        //注意这里只是用来检查公有的方法,不会去检查私有的方法
        Method[] methods = objectClass.getMethods();
        for(Method method : methods)
        {
            //获取当当前方法上面的所有的注解，因为一个方法上面有可能有多个注解，这是有可能的
            Annotation[] annotations = method.getAnnotations();
            //遍历方法上面的每一个注解
            for (Annotation annotation:annotations)
            {
                //获取到当前注解的类型，注解也是一个类，获取到注解的类型，也就可以获取到当前class上面有么有这个注解,获取注解的类型要用annotionType()，
                //而不能用getClass
                Class<? extends Annotation> annotationClass = annotation.annotationType();
                //获取到当前的注解类型再去获取这个类型上面的EventBase这个注解
                EventBase eventBase = annotationClass.getAnnotation(EventBase.class);
                if(eventBase != null)
                {

                    //获取到annotationClass 注解的value数组，里面存储了当前要设置事件的对象id
                    try
                    {
                        //获取到value的method,这里为什么不直接强转为onClick注解，是因为方便扩展，因为当有一个事件是onLongClick的时候，所以不能写死
                        Method valueMethod = annotationClass.getMethod("value");
                        int[] viewId= (int[]) valueMethod.invoke(annotation);
                        for (int id:viewId)
                        {
                            //得到id对应的View 对象
                            Method findViewById=objectClass.getMethod("findViewById",int.class);
                            View view= (View) findViewById.invoke(object,id);
                            if(view==null)
                            {
                                continue;
                            }
                            //类似以 setOnClickListener
                            String eventStr = eventBase.eventStr();
                            //类似于  View.OnClickListener.class
                            Class<?> eventType = eventBase.eventType();
                            //类似于 onClick
                            String eventMethod = eventBase.eventMethod();
                            //获取到view上面的 setOnClickListener MethodId
                            Method enventMethod = view.getClass().getMethod(eventStr,eventType);

                            EventInvoketion eventInvoketion = new EventInvoketion(method,object);
                            Object newProxyInstance = Proxy.newProxyInstance(annotationClass.getClassLoader(), new Class[]{eventType}, eventInvoketion);
                            //设置对应的setOnClickListener 监听
                            enventMethod.invoke(view,newProxyInstance);
                        }
                    }
                    catch (Exception e)
                    {
                        e.printStackTrace();
                    }
                }
            }
        }

    }

    /**
     * 注入对象
     * @param object
     */
    private static void injectView(Object object)
    {
        Class<?> objectClass = object.getClass();
        //首先一定要是Activity的子类
        if(object instanceof Activity)
        {
            //注意这里获取的是public 的成员变量，如果是私有的这里不能实现
            Field[] declaredFields = objectClass.getFields();
            for (Field field : declaredFields)
            {
                //判断是否含有ViewJect注解
                ViewJect viewJect = field.getAnnotation(ViewJect.class);
                if(viewJect != null)
                {
                    int value = viewJect.value();
                    try
                    {
                        Method method = objectClass.getMethod("findViewById", int.class);
                        View view = (View) method.invoke(object, value);
                        //在通过反射的方式设置值到成员变量上
                        field.setAccessible(true);
                        field.set(object,view);
                    }
                    catch (Exception e)
                    {
                        e.printStackTrace();
                    }
                }
            }
        }
    }

    /**
     * 注入ContentView实现
     * @param object
     */
    private static void injectContentView(Object object)
    {
        Class<?> objectClass = object.getClass();
        //首先一定要是Activity的子类
        if(object instanceof Activity)
        {
            ContentView contentView = objectClass.getAnnotation(ContentView.class);
            if(contentView != null)
            {
                //代表这个类有这个注解的存在,获取到注解上面的值
                int value = contentView.value();
                try
                {
                    //得到setContentView的MethodId
                    Method method = objectClass.getMethod("setContentView", int.class);
                    //使用反射的方式调用这个方法
                    method.invoke(object,value);
                }
                catch (Exception e)
                {
                    e.printStackTrace();
                }
            }
        }
    }
}

/**
 * Project Name: MtXutilsIoc
 * File Name:    EventInvoketion.java
 * ClassName:    EventInvoketion
 *
 * Description: 动态代理反射回去
 *
 * @author Zhangyuhui
 * @date 2018年06月01日 10:26
 *
 * Copyright (c) 2018年, 4399 Network CO.ltd. All Rights Reserved.
 */
public class EventInvoketion implements InvocationHandler
{
    private Method methodId;
    private Object object;

    public EventInvoketion(Method methodId, Object object)
    {
        this.methodId = methodId;
        this.object = object;
    }

    @Override
    public Object invoke(Object o, Method method, Object[] objects) throws Throwable
    {
        return methodId.invoke(object,objects);
    }
}

```

运行结果：

![结果显示](/uploads/XUtils/click.gif)

![结果显示](/uploads/XUtils/longClick.gif)
