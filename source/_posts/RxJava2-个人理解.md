---
layout: pager
title: RxJava2 个人理解
date: 2018-05-31 09:34:23
tags: [Android,RxJava,RxAndroid]
description:  RxJava2 个人理解
---

RxJava2个人理解
<!--more-->

****简介****
===
```
前面一篇文章中，分析了RxJava的简单的使用，这篇文章将会分析部分的源码，这里不会分析所有的操作符
1.Create操作符
2.Map操作符
3.subscribeOn操作符
```

****Create操作符****
===
```java
我们通过下面的方式来使用Create操作符:

Observable.create(new ObservableOnSubscribe<Integer>() {
            @Override
            public void subscribe(@NonNull ObservableEmitter<Integer> e) throws Exception {
                System.out.println("Observable emit 1");
                e.onNext(1);
                System.out.println("Observable emit 2");
                e.onNext(2);
                System.out.println("Observable emit 3");
                e.onNext(3);
                e.onComplete();
                System.out.println("Observable emit 4");
                e.onNext(4);
            }
        }).subscribe(new Observer<Integer>() {

            private int i;
            private Disposable mDisposable;

            //其实传递的对象为发射器对象，不过他实现了Disposable接口，这个接口可以用来中断事件
            @Override
            public void onSubscribe(@NonNull Disposable d) {
                System.out.println("onSubscribe : " + d.isDisposed());
                mDisposable = d;
            }

            @Override
            public void onNext(@NonNull Integer integer) {
                System.out.println("onNext : value : " + integer);
                i++;
                if (i == 2) {
                    // 在RxJava 2.x 中，新增的Disposable可以做到切断的操作，让Observer观察者不再接收上游事件
                    mDisposable.dispose();
                    System.out.println("onNext : isDisposable : " + mDisposable.isDisposed());
                }
            }

            @Override
            public void onError(@NonNull Throwable e) {
                System.out.println("onError : value : " + e.getMessage());
            }

            @Override
            public void onComplete() {
                System.out.println("onComplete");
            }
});


首先分析 Observable.create(new ObservableOnSubscribe<Integer>())的时候发生的事情
public static <T> Observable<T> create(ObservableOnSubscribe<T> source) {
    //检查对象是否为空，如果对象为空就会抛出一个空指针异常，当然也就不会往下走,如果不往下走，那么就是没有异常
    ObjectHelper.requireNonNull(source, "source is null");

    //首先构造一个ObservableCreate对象,里面存储了ObservableOnSubscribe 对象
    return RxJavaPlugins.onAssembly(new ObservableCreate<T>(source));
}

ObservableCreate 继承了Observable类，所以本身是一个被观察者对象
public final class ObservableCreate<T> extends Observable<T> {
    //保存成员变量，也即是在开始创建的匿名对象
    final ObservableOnSubscribe<T> source;

    public ObservableCreate(ObservableOnSubscribe<T> source) {
        this.source = source;
    }
...
}

所以Observable.create(new ObservableOnSubscribe<Integer>()) 此时就会返回一个ObservableCreate对象，本身是一个观察者对象
然后执行.subscribe(new Observer<Integer>()); 所以就会执行到ObservableCreate对象对应的函数,进过发现在ObservableCreate中并没有这个方法，但是他继承了Observable，所以这个方法是在父类的

这里传递的参数为 Observer<? super T> observer ,为什么不能传递 Observer<? extends T> observer ，具体怎么解释就不知道了，主要记住，如果是传递参数的化，就要用super ，如果是返回值的化
就要使用extends ，记住就好了....
public final void subscribe(Observer<? super T> observer) {
    //检查观察者是否为空，如果为空，会抛出异常
    ObjectHelper.requireNonNull(observer, "observer is null");
    try {
        //判断是否有设置插件的onObservableSubscribe ，如果没有就直接返回 observer 对象
        observer = RxJavaPlugins.onSubscribe(this, observer);

        //检查对象为空
        ObjectHelper.requireNonNull(observer, "Plugin returned null Observer");

        //subscribeActual 为抽象函数，由具体的子类来实现,同时传递观察者进去
        subscribeActual(observer);
    } catch (NullPointerException e) { // NOPMD
        throw e;
    } catch (Throwable e) {
        Exceptions.throwIfFatal(e);
        // can't call onError because no way to know if a Disposable has been set or not
        // can't call onSubscribe because the call might have set a Subscription already
        RxJavaPlugins.onError(e);

        NullPointerException npe = new NullPointerException("Actually not, but can't throw other exceptions due to RS");
        npe.initCause(e);
        throw npe;
    }
}
subscribeActual 函数的声明：
protected abstract void subscribeActual(Observer<? super T> observer);

当执行到 subscribeActual(observer); 因为是一个抽象函数，所以就会由子类来实现，当前的类为ObservableCreate 对象，所以执行对应的函数

这里传递的参数 observer为我们创建的匿名观察者对象，这里传递了进来,看到没有，这里也是使用了super关键字，因为当前是按参数传递的所以要用super
@Override
protected void subscribeActual(Observer<? super T> observer) { 
    //将观察者对象封装成一个CreateEmitter 对象,这个类为发射器类，用来发射事件的,至于为什么要封装成一个CreateEmitter对象，是因为
    //这个对象具有中断事件的能力，具备原子性的操作
    CreateEmitter<T> parent = new CreateEmitter<T>(observer);

    //observer为我们的观察者对象，这里也即是回调执行对应的方法 onSubscribe同时将发射器对象传递过来，
    observer.onSubscribe(parent);
    try {
        //source为我们创建的匿名内部类ObservableOnSubscribe，所以回调执行对应的函数subscribe,同时将这个发射器对象传递过去，这样就能执行事件的发送
        source.subscribe(parent);
    } catch (Throwable ex) {
        Exceptions.throwIfFatal(ex);
        parent.onError(ex);
    }
}

首先创建  new CreateEmitter<T>(observer);
//CreateEmitter内部类实现了Disposable接口，该接口有中断事件的能力,同事实现了ObservableEmitter接口，既有发射事件的能力
//同时继承了AtomicReference   AtomicReference是作用是对"对象"进行原子操作。
static final class CreateEmitter<T>
            extends AtomicReference<Disposable>
            implements ObservableEmitter<T>, Disposable {
			
        private static final long serialVersionUID = -3434801548987643227L;

        //保存观察者对象，这里的观察者对象即为外面创建的匿名内部类对象
        final Observer<? super T> observer;

        //构建一个发射器对象，同时保存观察者对象
        CreateEmitter(Observer<? super T> observer) {
            this.observer = observer;
        }
	....
}

然后执行  observer.onSubscribe(parent);，因为这里的observer为我们创建的匿名内部类，所以就会回调回去，也即执行到了

//其实传递的对象为发射器对象，不过他实现了Disposable接口，这个接口可以用来中断事件
@Override
public void onSubscribe(@NonNull Disposable d) {
    System.out.println("onSubscribe : " + d.isDisposed());
    mDisposable = d;
}

然后执行  source.subscribe(parent); 这里的source为我们一开始创建ObservableCreate对象传递进来的匿名ObservableOnSubscribe对象，所以就会执行对应的函数
new ObservableOnSubscribe<Integer>() {
    @Override
    public void subscribe(@NonNull ObservableEmitter<Integer> e) throws Exception {
        System.out.println("Observable emit 1");
        e.onNext(1);
        System.out.println("Observable emit 2");
        e.onNext(2);
        System.out.println("Observable emit 3");
        e.onNext(3);
        e.onComplete();
        System.out.println("Observable emit 4");
        e.onNext(4);
	}
});

之后执行   e.onNext(1); 这里的e为ObservableEmitter对象，当前也即为CreateEmitter对象，所以就会执行对应的函数
//执行事件的发送
@Override
public void onNext(T t) {
    //注意这里不允许发射空事件
    if (t == null) {
        onError(new NullPointerException("onNext called with null. Null values are generally not allowed in 2.x operators and sources."));
        return;
    }
    //如果当前事件没有被中断
    if (!isDisposed()) {
        //最终回调执行 我们创建的观察者对象 ，所以观察者对象的OnNext函数就会被执行，这就达到了类似观察者，被观察者的效果
        observer.onNext(t);
    }
}

//判断当前事件是否有设置事件的中断
@Override
public boolean isDisposed() {
    return DisposableHelper.isDisposed(get());
}

public static boolean isDisposed(Disposable d) {
    return d == DISPOSED;
}

执行 observer.onNext(t); 这里的observer对象也即是我们创建的匿名的观察者，所以就会回调执行我们的方法
@Override
public void onNext(@NonNull Integer integer) {
    System.out.println("onNext : value : " + integer);
    i++;
    if (i == 2) {
        // 在RxJava 2.x 中，新增的Disposable可以做到切断的操作，让Observer观察者不再接收上游事件
        mDisposable.dispose();
        System.out.println("onNext : isDisposable : " + mDisposable.isDisposed());
    }
}

至此事件的发送至接收已经完成，当i==2的时候，执行 mDisposable.dispose(); 这里的mDisposable 为我们在回调执行onSubscribe的时候，保存下来的，本质的对象为 CreateEmitter
public void onSubscribe(@NonNull Disposable d) {
    System.out.println("onSubscribe : " + d.isDisposed());
    mDisposable = d;
}

所以当执行  mDisposable.dispose();的时候，就会执行对应的函数
//中断事件的接收
@Override
public void dispose() {
    DisposableHelper.dispose(this);
}

设置中断事件，原子性的对象，也即是将当前的对象设置为DISPOSED 对象
public static boolean dispose(AtomicReference<Disposable> field) {
    // 获取当前值。
    Disposable current = field.get();
    Disposable d = DISPOSED;
    //如果当前的值不是DISPOSED
    if (current != d) {
        //以原子方式设置为给定值，并返回旧值。 对于原子的操作，也即设计到了CAS，这方面的内容自己去百度
        current = field.getAndSet(d);
        if (current != d) {
            if (current != null) {
                current.dispose();
            }
            return true;
        }
    }
    return false;
}

所以当设置了事件的中断之后，下次再接受的时候，isDisposed就会返回true，所以就不会执行observer.onNext()操作，也即是做到了事件中断了
if (!isDisposed()) {
    //最终回调执行 我们创建的观察者对象 ，所以观察者对象的OnNext函数就会被执行，这就达到了类似观察者，被观察者的效果
    observer.onNext(t);
}
```
流程分析：
![结果显示](/uploads/RxJava简单使用/create操作符流程.png)

****Map操作符****
===
```java
我们通过下面的方式来使用Map操作符

//Map操作符，可以针对上游发送的每一个事件应用一个函数，使得每一个事件都按照指定的函数去变化
Observable.create(new ObservableOnSubscribe<Integer>() {
            @Override
            public void subscribe(@NonNull ObservableEmitter<Integer> e) throws Exception {
                e.onNext(1);
                e.onNext(2);
                e.onNext(3);
            }
        }).map(new Function<Integer, String>() {
            @Override
            public String apply(@NonNull Integer integer) throws Exception {
                return "This is result " + integer;
            }
        }).subscribe(new Observer<String>()
        {
            @Override
            public void onSubscribe(Disposable d)
            {

            }

            @Override
            public void onNext(String s)
            {
                System.out.println(s);
            }

            @Override
            public void onError(Throwable e)
            {

            }

            @Override
            public void onComplete()
            {

            }
});

首先执行Observable.create(new ObservableOnSubscribe<Integer>(),这里上面已经分析过了，无非就是返回一个ObserverCreate对象，本身为一个Observable观察者对象
当执行.map(new Function<Integer, String>()

//<R>泛型要先定义，对于<T>可以直接使用，因为Observable本身是一个泛型类 ，这里注意一下Function<? super T, ? extends R> mapper，根据前面的介绍知道，super可以用来传递参数，extends
用来返回值，这里用到了俩个，这里是因为Function 泛型的定义，可以知道R是作为返回值的，所以用extends，T是用来做参数的，所以用super
public interface Function<T, R> {
    /**
     * Apply some calculation to the input value and return some other value.
     * @param t the input value
     * @return the output value
     * @throws Exception on error
     */
    R apply(@NonNull T t) throws Exception;
}

public final <R> Observable<R> map(Function<? super T, ? extends R> mapper) {
    //检查Map对象不能为空,为空会抛出异常
    ObjectHelper.requireNonNull(mapper, "mapper is null");
    //首先构造一个ObservableMap对象,ObservableMap本身为一个Observable，同时保存了当前的被观察者，还有Function对象
    //又相当于是一个中间者的样子,这里的this为 ObservableCreate对象
    return RxJavaPlugins.onAssembly(new ObservableMap<T, R>(this, mapper));
}

当执行到 new ObservableMap<T, R>(this, mapper)
public final class ObservableMap<T, U> extends AbstractObservableWithUpstream<T, U> {
    //保存Function对象，对于参数里面的泛型中使用super,或者extends 关键字，如果是传递参数就用super，如果是获取到内容，就用extends
    final Function<? super T, ? extends U> function;

    //构造ObservableMap对象，保存观察者对象 source
    public ObservableMap(ObservableSource<T> source, Function<? super T, ? extends U> function) {
        super(source);
        this.function = function;
    }
	....
}
abstract class AbstractObservableWithUpstream<T, U> extends Observable<U> implements HasUpstreamObservableSource<T> {
    /** The source consumable Observable. */
    //源的观察者对象
    protected final ObservableSource<T> source;

    /**
     * Constructs the ObservableSource with the given consumable.
     * @param source the consumable Observable
     */
    //父类的构造函数，保存源的观察者对象
    AbstractObservableWithUpstream(ObservableSource<T> source) {
        this.source = source;
    }

    @Override
    public final ObservableSource<T> source() {
        return source;
    }
}

可以看出ObservableMap类继承了Observable ，本身是一个被观察者对象，所以对于map操作返回了一个ObservableMap对象，这个对象本身为一个被观察者对象，同时这个对象里面保存了map里面匿名的
Function对象,还有上一个操作符create创建的ObservableCreate对象

之后执行.subscribe(new Observer<String>()，因为当前的被观察者对象为ObservableMap对象，本身为一个Observable，所以会执行对应的subscribe函数，经前面介绍可知，这个最终会调用子类的
subscribeActual函数，所以就会调用到ObservableMap中对应的函数

这里因为是传递参数，所以可以用super关键字，这里传递的对象t为我们创建的匿名观察者对象
@Override
public void subscribeActual(Observer<? super U> t) {
    //这里保存的source为ObservableCreate对象，首先构造一个 MapObserver对象 ,所以会调用对应的函数
    source.subscribe(new MapObserver<T, U>(t, function));
}

首先执行 new MapObserver<T, U>(t, function)

static final class MapObserver<T, U> extends BasicFuseableObserver<T, U> {

    //保存Map 操作符中创建的 匿名 Function对象
    final Function<? super T, ? extends U> mapper;

    //构造函数的执行，将传递的参数保存下来 ,这里的actual 为Observer对象,也即是我们创建的匿名观察者对象，
    MapObserver(Observer<? super U> actual, Function<? super T, ? extends U> mapper) {
        super(actual);
        this.mapper = mapper;
    }
	...	
}

public abstract class BasicFuseableObserver<T, R> implements Observer<T>, QueueDisposable<R> {
    //构造函数，保存观察者对象
    public BasicFuseableObserver(Observer<? super R> actual) {
        this.actual = actual;
    }
	....
}

所以可以知道 MapObserver类，实现了Observer接口，还有Disposal接口,本身是一个观察者对象,他持有上一个观察者对象这里也即是创建的匿名观察者对象，以及Map操作符中创建的Function对象

之后执行  source.subscribe(new MapObserver<T, U>(t, function));，因为这里的source保存的为上一个操作符中创建的被观察者对象，这里即为observableCreate对象，又因为subscribe为抽象函数
最终会执行子类的subscribeActual函数

@Override
protected void subscribeActual(Observer<? super T> observer) {//此时传递的observer对象即为刚刚创建的MapObserver 对象
    //将观察者对象封装成一个CreateEmitter 对象,这个类为发射器类，用来发射事件的,至于为什么要封装成一个CreateEmitter对象，是因为
    //这个对象具有中断事件的能力，具备原子性的操作
    CreateEmitter<T> parent = new CreateEmitter<T>(observer);

    //observer为我们的观察者对象，这里也即是回调执行对应的方法 onSubscribe同时将发射器对象传递过来，
    observer.onSubscribe(parent);
    try {
        //source为我们创建的匿名内部类ObservableOnSubscribe，所以回调执行对应的函数subscribe,同时将这个发射器对象传递过去，这样就能执行事件的发送
        source.subscribe(parent);
    } catch (Throwable ex) {
        Exceptions.throwIfFatal(ex);
        parent.onError(ex);
    }
}

//CreateEmitter内部类实现了Disposable接口，该接口有中断事件的能力,同事实现了ObservableEmitter接口，既有发射事件的能力
//同时继承了AtomicReference   AtomicReference是作用是对"对象"进行原子操作。
static final class CreateEmitter<T> extends AtomicReference<Disposable> implements ObservableEmitter<T>, Disposable {

    //保存观察者对象,这里即为MapObserver对象
    final Observer<? super T> observer;

    //构建一个发射器对象，同时保存观察者对象
    CreateEmitter(Observer<? super T> observer) {
        this.observer = observer;
    }
	...	
}

之后执行  observer.onSubscribe(parent);这里的observer 对象为 MapObserver 对象，所以会执行对应的函数,经过发现MapObserver 中并没有这个函数，但是他的父类有实现，下面为函数的定义
//onSubscribe回调执行
public final void onSubscribe(Disposable s) {
    //判断Disposable的合法性
    if (DisposableHelper.validate(this.s, s)) {
        this.s = s;
        if (s instanceof QueueDisposable) {
            this.qs = (QueueDisposable<T>)s;
        }

        if (beforeDownstream()) {
            //转到真正的观察者对象，执行onSubscribe函数
            actual.onSubscribe(this);
            afterDownstream();
        }
    }
}

protected boolean beforeDownstream() {
    return true;
}

所以就会执行到 actual.onSubscribe(this); ,这里的actual 为Observer对象,也即是我们创建的匿名观察者对象，所以就会回调执行对应的函数
@Override
public void onSubscribe(Disposable d)
{
    System.out.println("Disposable");
}
		
observableCreate 中的 subscribeActual函数继续往下执行

//source为我们创建的匿名内部类ObservableOnSubscribe，所以回调执行对应的函数subscribe,同时将这个发射器对象传递过去，这样就能执行事件的发送
source.subscribe(parent);	

所以就会执行到 我们创建的匿名对象中对应的函数
new ObservableOnSubscribe<Integer>() {
    @Override
    public void subscribe(@NonNull ObservableEmitter<Integer> e) throws Exception {
        e.onNext(1);
        e.onNext(2);
        e.onNext(3);
	}	
});
			
当执行到 e.onNext(1); 的时候：,这里的e 对象为 CreateEmitter ，所以会执行对应的函数
//执行事件的发送
@Override
public void onNext(T t) {
    //注意这里不允许发射空事件
    if (t == null) {
        onError(new NullPointerException("onNext called with null. Null values are generally not allowed in 2.x operators and sources."));
        return;
    }
    //如果当前事件没有被中断
    if (!isDisposed()) {
        //最终回调执行 我们创建的观察者对象 ，所以观察者对象的OnNext函数就会被执行，这就达到了类似观察者，被观察者的效果
        observer.onNext(t);
    }
}

当执行到 observer.onNext(t); 这里的observer对象为我们map创建的的MapObserver对象，所以会执行对应的函数
@Override
public void onNext(T t) {
    ...
    U v;
    try {
        //这里的mapper为我们创建的map匿名内部类，这里回调执行对应的函数,并获取到结果
        v = ObjectHelper.requireNonNull(mapper.apply(t), "The mapper function returned a null value.");
    } catch (Throwable ex) {
        fail(ex);
        return;
    }
    //获取到结果之后，在转到 LambdaObserver 对应的onNext方法中
    actual.onNext(v);
}

当执行到 mapper.apply(t) 这里的mapper为我们创建的map匿名内部类，这里回调执行对应的函数
new Function<Integer, String>() {
    @Override
    public String apply(@NonNull Integer integer) throws Exception {
        return "This is result " + integer;
    }			
});

当得到mapper函数转换之后的结果之后，调用 actual.onNext(v);这里的actual即为我们创建的匿名的观察者对象，所以就会回调执行
@Override
public void onNext(String s)
{
    System.out.println(s);
}

至此事件从发送到接受都已经完成

总结：从Map的流程可以看出，创建了多个观察者，已经多个被观察者对象，为什么要这样做呢，原因1：因为要保持链式的调度，而且返回的不是同一个对象，而是同一类型的对象，而且后面一个对象会持有
前面一个对象的引用，这样做到了层层的相扣的样子，比如一开始使用create操作符产生一个ObservableCreate被观察者对象，然后经过map操作符，返回一个MapObservable对象，同时他持有上一个的引用
至于观察者来说也是一样的，当订阅发生的时候，就可以通过这种层层相扣，找到最始的源，执行对应的回调，事件的发送也是一样，最终找到最开始的那个观察者对象，这估计就是层层相扣的原因
```

流程分析：
![结果显示](/uploads/RxJava简单使用/Map流程图.png)

****subscribeOn(线程切换)操作符****
===
```java
我们通过下面的方式来使用subscribeOn操作符:

Observable.create(new ObservableOnSubscribe<String>()
        {
            @Override
            public void subscribe(ObservableEmitter<String> emitter) throws Exception
            {
                emitter.onNext("1");
                emitter.onNext("2");
                emitter.onNext("3");
            }
        }).subscribeOn(Schedulers.io())
        .subscribe(new Observer<String>()
        {
             @Override
             public void onSubscribe(Disposable d)
             {

             }

             @Override
             public void onNext(String s)
             {
                 System.out.println(s);
             }

             @Override
             public void onError(Throwable e)
             {

             }

             @Override
             public void onComplete()
             {

             }
});

首先执行 Observable.create(new ObservableOnSubscribe<String>()，前面已经分析过了，这里会返回一个ObservableCreate对象，这个是一个继承了Observable，本身是一个被观察者
然后执行.subscribeOn(Schedulers.io())
先来分析 看看Schedulers.io() 具体干了什么

public final class Schedulers {
    @NonNull
    static final Scheduler SINGLE;
    @NonNull
    static final Scheduler COMPUTATION;
    @NonNull
    static final Scheduler IO;
    @NonNull
    static final Scheduler TRAMPOLINE;
    @NonNull
    static final Scheduler NEW_THREAD;

    static final class SingleHolder {
        static final Scheduler DEFAULT = new SingleScheduler();
    }

    static final class ComputationHolder {
        static final Scheduler DEFAULT = new ComputationScheduler();
    }

    static final class IoHolder {
        static final Scheduler DEFAULT = new IoScheduler();
    }

    static final class NewThreadHolder {
        static final Scheduler DEFAULT = new NewThreadScheduler();
    }

    static {
        SINGLE = RxJavaPlugins.initSingleScheduler(new SingleTask());

        COMPUTATION = RxJavaPlugins.initComputationScheduler(new ComputationTask());

        IO = RxJavaPlugins.initIoScheduler(new IOTask());

        TRAMPOLINE = TrampolineScheduler.instance();

        NEW_THREAD = RxJavaPlugins.initNewThreadScheduler(new NewThreadTask());
    }
	...
}
可以看到Sechedulers.Io是这个类中的一个静态的成员变量 static final Scheduler IO，并且赋值操作发生在静态代码块内 IO = RxJavaPlugins.initIoScheduler(new IOTask());
分析下RxJavaPlugins.initIoScheduler(new IOTask())发生了什么 ：首先构造一个IOTask对象 new IOTask(),实现了Callable接口

static final class IOTask implements Callable<Scheduler> {
    @Override
    public Scheduler call() throws Exception {
        return IoHolder.DEFAULT;
    }
}

public static Scheduler initIoScheduler(@NonNull Callable<Scheduler> defaultScheduler) {
    //检查对象是否为空,如果为空，抛出异常
    ObjectHelper.requireNonNull(defaultScheduler, "Scheduler Callable can't be null");
    //默认 onInitIoHandler为空，所以f为空
    Function<? super Callable<Scheduler>, ? extends Scheduler> f = onInitIoHandler;
    if (f == null) {
        //所以会进入这里面
        return callRequireNonNull(defaultScheduler);
    }
    return applyRequireNonNull(f, defaultScheduler);
}

static Scheduler callRequireNonNull(@NonNull Callable<Scheduler> s) {
    try {
        //检查s.call()函数返回的对象不能为空，如果为空会抛出异常
        return ObjectHelper.requireNonNull(s.call(), "Scheduler Callable result can't be null");
    } catch (Throwable ex) {
        throw ExceptionHelper.wrapOrThrow(ex);
    }
}

所以当执行s.call()的时候，这里的s为 创建的IOTask，所以会执行对应的call函数
static final class IOTask implements Callable<Scheduler> {
    @Override
    public Scheduler call() throws Exception {
        return IoHolder.DEFAULT;
    }
}

static final class IoHolder {
    static final Scheduler DEFAULT = new IoScheduler();
}

所以最终这个Scheduler.IO最终获取到的对象为 IoScheduler 对象

public IoScheduler() {
    this(WORKER_THREAD_FACTORY);
}

所以当执行 subscribeOn(Schedulers.io())  ，传递的scheduler 对象就为 IoScheduler对象
public final Observable<T> subscribeOn(Scheduler scheduler) {
    //检查对象scheduler 是否为空，如果为空就抛出异常
    ObjectHelper.requireNonNull(scheduler, "scheduler is null");
    //构建一个ObservableSubscribeOn对象，本身是一个Observable对象
    return RxJavaPlugins.onAssembly(new ObservableSubscribeOn<T>(this, scheduler));
}

public final class ObservableSubscribeOn<T> extends AbstractObservableWithUpstream<T, T> {
    //保存Scheduler对象，也即为IoScheduler对象
    final Scheduler scheduler;

    //构造对象，保存原先的被观察者对象，以及线程调度对象,这里的被观察者对象为前一个操作符create对象返回的ObservableCreate对象
    public ObservableSubscribeOn(ObservableSource<T> source, Scheduler scheduler) {
        super(source);
        this.scheduler = scheduler;
    }
	....
}

abstract class AbstractObservableWithUpstream<T, U> extends Observable<U> implements HasUpstreamObservableSource<T> {

    /** The source consumable Observable. */
    //源的观察者对象
    protected final ObservableSource<T> source;
	
    /**
     * Constructs the ObservableSource with the given consumable.
     * @param source the consumable Observable
     */
    //父类的构造函数，保存源的观察者对象
    AbstractObservableWithUpstream(ObservableSource<T> source) {
        this.source = source;
    }

    @Override
    public final ObservableSource<T> source() {
        return source;
    }

}

ObservableSubscribeOn对象，本身继承了Observable，为一个被观察者,他持有了IoSchedler ，以及 前一个操作符create对象返回的ObservableCreate对象

之后执行subscribe(new Observer<String>());,这里的对象为刚刚返回的 ObservableSubscribeOn对象 ,经前面介绍可知，subscribe 为一个抽象函数，这个最终会调用子类的 subscribeActual，所以会调用

@Override
public void subscribeActual(final Observer<? super T> s) {//这里传递的s为一个真正的观察者对象，也即是我们创建的匿名的观察者对象
    //构建一个SubscribeOnObserver 对象,本身为一个观察者，保存原有的观察者对象
    final SubscribeOnObserver<T> parent = new SubscribeOnObserver<T>(s);

    //回调原的onSubscribe函数
    s.onSubscribe(parent);

    parent.setDisposable(scheduler.scheduleDirect(new SubscribeTask(parent)));
}

首先构造一个new SubscribeOnObserver<T>(s);

static final class SubscribeOnObserver<T> extends AtomicReference<Disposable> implements Observer<T>, Disposable {

    private static final long serialVersionUID = 8094547886072529208L;
    //保存创建的匿名的观察者对象
    final Observer<? super T> actual;

    final AtomicReference<Disposable> s;

    //构建函数的执行，保存原有的观察者对象
    SubscribeOnObserver(Observer<? super T> actual) {
        this.actual = actual;
        this.s = new AtomicReference<Disposable>();
    }
	....
}
所以这里 构建一个SubscribeOnObserver对象，本身实现了Observer接口，为一个观察者对象,同时保存了真正的观察者对象，也即是我们创建的匿名的观察者对象

之后执行 s.onSubscribe(parent);这里的s为我们创建的匿名观察者对象，所以会回调执行到对应的函数
@Override
public void onSubscribe(Disposable d)
{
   System.out.println("Disposable");
}

之后执行  parent.setDisposable(scheduler.scheduleDirect(new SubscribeTask(parent)));
首先分析 new SubscribeTask(parent)

final class SubscribeTask implements Runnable {
	//保存SubscribeOnObserver对象
	private final SubscribeOnObserver<T> parent;

    SubscribeTask(SubscribeOnObserver<T> parent) {
        this.parent = parent;
    }

    @Override
    public void run() {
         source.subscribe(parent);
    }
}

所以可以知道 构建一个Task对象 本身实现了Runnable接口,同时保存了SubscribeOnObserver 观察者对象

之后执行 scheduler.scheduleDirect(new SubscribeTask(parent)
public Disposable scheduleDirect(@NonNull Runnable run) {//run 为 SubscribeTask对象
    return scheduleDirect(run, 0L, TimeUnit.NANOSECONDS);
}

public Disposable scheduleDirect(@NonNull Runnable run, long delay, @NonNull TimeUnit unit) {//run 为 SubscribeTask对象
    //最终返回的对象为EventLoopWorker 
    final Worker w = createWorker();

    final Runnable decoratedRun = RxJavaPlugins.onSchedule(run);

    DisposeTask task = new DisposeTask(decoratedRun, w);

    w.schedule(task, delay, unit);

    return task;
}

final Worker w = createWorker();
createWorker()函数为一个抽象函数  public abstract Worker createWorker(); 具体的实现交给子类，由于当前我们的Scheduler为IoScheduler，所以找到对应的函数实现
public Worker createWorker() {
    return new EventLoopWorker(pool.get());
}

static final class EventLoopWorker extends Scheduler.Worker {
	....
    EventLoopWorker(CachedWorkerPool pool) {
        this.pool = pool;
        this.tasks = new CompositeDisposable();
        this.threadWorker = pool.get();
    }
	...
}

所以当执行  final Worker w = createWorker(); w对象为EventLoopWorker 

执行 RxJavaPlugins.onSchedule(run)；
public static Runnable onSchedule(@NonNull Runnable run) {
    //检查run不能为空
    ObjectHelper.requireNonNull(run, "run is null");
    //默认的onScheduleHandler为空，所以f为空，
    Function<? super Runnable, ? extends Runnable> f = onScheduleHandler;
    if (f == null) {
        return run;
    }
    return apply(f, run);
}

所以执行 final Runnable decoratedRun = RxJavaPlugins.onSchedule(run); decoratedRun即为我们传递进来的run也即为 SubscribeTask对象

执行  w.schedule(task, delay, unit);w对象为EventLoopWorker对象，所以会执行对应的函数
public Disposable schedule(@NonNull Runnable action, long delayTime, @NonNull TimeUnit unit) {
    if (tasks.isDisposed()) {
        // don't schedule, we are unsubscribed
        return EmptyDisposable.INSTANCE;
    }

    return threadWorker.scheduleActual(action, delayTime, unit, tasks);
}
执行 threadWorker.scheduleActual(action, delayTime, unit, tasks);

public ScheduledRunnable scheduleActual(final Runnable run, long delayTime, @NonNull TimeUnit unit, @Nullable DisposableContainer parent) {
    //前面已经分析过，这里最终得到还是传递进去的run对象,这里的run对象即为SubscribeTask对象
    Runnable decoratedRun = RxJavaPlugins.onSchedule(run);

    ScheduledRunnable sr = new ScheduledRunnable(decoratedRun, parent);

    if (parent != null) {
        if (!parent.add(sr)) {
            return sr;
        }
    }

    Future<?> f;
    try {
        if (delayTime <= 0) {
            f = executor.submit((Callable<Object>)sr);
        } else {
            f = executor.schedule((Callable<Object>)sr, delayTime, unit);
        }
        sr.setFuture(f);
    } catch (RejectedExecutionException ex) {
        if (parent != null) {
            parent.remove(sr);
        }
        RxJavaPlugins.onError(ex);
    }
    return sr;
}

构建一个 new ScheduledRunnable(decoratedRun, parent);对象
public final class ScheduledRunnable extends AtomicReferenceArray<Object> implements Runnable, Callable<Object>, Disposable {
	public ScheduledRunnable(Runnable actual, DisposableContainer parent) {
        super(3);
        //存储SubscribeTask对象
        this.actual = actual;
        this.lazySet(0, parent);
    }
	...
}
可以知道构建一个ScheduledRunnable对象，本身实现了Runnable接口，同时保存了run对象，这里为SubscribeTask对象
之后执行 
if (delayTime <= 0) {
    f = executor.submit((Callable<Object>)sr);
} else {
    f = executor.schedule((Callable<Object>)sr, delayTime, unit);
}
由于我们没有设置delayTime，所以会执行 executor.submit((Callable<Object>)sr); executord的定义为   private final ScheduledExecutorService executor;
赋值操作为  executor = SchedulerPoolFactory.create(threadFactory);

所以当执行 scheduleDirect的时候，就是将创建的SubscribeTask对象 封装成一个 ScheduledRunnable对象，之后交给线程池执行，从而达到切换线程的目的：
当任务执行的时候，就会回调执行提交任务的run方法也即是ScheduledRunnable对象 中的run方法

public void run() {
	...
    actual.run();
	....
}
执行 actual.run(); 这里的actual对象为我们传递进来的SubscribeTask对象,所以执行对应的方法
final class SubscribeTask implements Runnable {
    private final SubscribeOnObserver<T> parent;
    SubscribeTask(SubscribeOnObserver<T> parent) {
        this.parent = parent;
    }

    @Override
    public void run() {
        source.subscribe(parent);
    }
}

之后执行到了  source.subscribe(parent);这里要注意这个方法目前已经在子线程里面执行了,在这个方法之后的所有的操作都是在子线程中完成的，也即完成了线程的切换了
这里的source为创建ObservableSubscribeOn被观察者的时候，传递进来的上一个操作符的被观察者对象 也即是observableCreate对象，parent为 SubscribeOnObserver对象，
又因为subscribe为抽像函数，所以最终会执行到对应类的subscribeActual函数

protected void subscribeActual(Observer<? super T> observer) {//这里传递进来的observer对象为 SubscribeOnObserver对象

    //将观察者对象封装成一个CreateEmitter 对象,这个类为发射器类，用来发射事件的,至于为什么要封装成一个CreateEmitter对象，是因为
    //这个对象具有中断事件的能力，具备原子性的操作
    CreateEmitter<T> parent = new CreateEmitter<T>(observer);

    //observer为SubscribeOnObserver对象，这里也即是回调执行对应的方法 onSubscribe同时将发射器对象传递过来，
    observer.onSubscribe(parent);
    try {
        //source为我们创建的匿名内部类ObservableOnSubscribe，所以回调执行对应的函数subscribe,同时将这个发射器对象传递过去，这样就能执行事件的发送
        source.subscribe(parent);
    } catch (Throwable ex) {
        Exceptions.throwIfFatal(ex);
        parent.onError(ex);
    }
}

当执行 observer.onSubscribe(parent);，也即是会执行到SubscribeOnObserver对应的函数

@Override
public void onSubscribe(Disposable s) {//这里的s为CreateEmitter对象本身实现了Disposable接口，
    DisposableHelper.setOnce(this.s, s);
}

这里是通过原子操作，设置this.s的值为CreateEmitter对象，保存为同一个对象，这样当我们通过CreateEmitter设置取消的时候，这个观察者也知道取消的操作

source.subscribe(parent);source为我们创建的匿名内部类ObservableOnSubscribe，所以回调执行对应的函数subscribe
new ObservableOnSubscribe<String>()
{
    @Override
    public void subscribe(ObservableEmitter<String> emitter) throws Exception
    {
        emitter.onNext("1");
        emitter.onNext("2");
        emitter.onNext("3");
	}
}

当执行emitter.onNext("1");，emitter对象为 CreateEmitter，所以会执行对应的函数,也即是：
@Override
public void onNext(T t) {
    //注意这里不允许发射空事件
    if (t == null) {
        onError(new NullPointerException("onNext called with null. Null values are generally not allowed in 2.x operators and sources."));
        return;
    }
    //如果当前事件没有被中断
    if (!isDisposed()) {
        //最终回调执行 我们创建的观察者对象 ，所以观察者对象的OnNext函数就会被执行，这就达到了类似观察者，被观察者的效果
        observer.onNext(t);
    }
}

执行 observer.onNext(t); 这里的observer 对象为 SubscribeOnObserver对象，所以执行对应的函数
@Override
public void onNext(T t) {
    actual.onNext(t);
}

执行 actual.onNext(t); 这里的actual对象为我们创建的匿名的观察者对象，所以就会回调执行到
@Override
public void onNext(String s)
{
    System.out.println(s);
}

至此事件从发送到接受都已经完成


总结：线程的切换是通过线程池的方式来做到的，通过提交一个任务，让这个任务回调执行run方法的时候，才可以执行订阅的操作，通过RxJava的层层相扣的特性，找到最始的那个源，完成订阅，对于
事件的发送也是一样，最终找到最开始的那个观察者对象，这估计就是层层相扣的原因
```

流程分析：
![结果显示](/uploads/RxJava简单使用/subscribeOn.png)