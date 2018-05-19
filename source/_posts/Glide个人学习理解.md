---
layout: pager
title: Glide个人学习理解
date: 2018-05-18 13:55:17
tags: [Glide]
description:  Glide个人学习理解
---

Glide个人学习理解
<!--more-->

简介
```
Glide源码众多，自然没有办法做到每一个都去详细的了解。。这里只是大致的分析他的大致流程，这里的分析大致会分为下面的三个点来

1.Glide 内存缓存机制
2.Glide 生命周期机制
3.Glide 注册机的机制

```
 ****Glide内存缓存机制****
 ===
```java
Glide中的内存缓存分为活动缓存还有内存缓存，对应的类为ActiveResources,LruResourceCache
我们先看看他们在Glide中是怎么样使用的，在Engine中有这样的代码
	//构建一个key ，利用EngineKey来构建一个key，如果宽高不一样的化，这里会得到俩个不一样的key
    EngineKey key = keyFactory.buildKey(model, signature, width, height, transformations, resourceClass, transcodeClass, options);

    //首先从活动的缓存中获取
    EngineResource<?> active = loadFromActiveResources(key, isMemoryCacheable);
    if (active != null) {
      //如果从活动资源中获取到了这个资源，Juin直接通过资源准备好了，return   DataSource.MEMORY_CACHE 标识当前的这个资源是来自内存缓存中
      cb.onResourceReady(active, DataSource.MEMORY_CACHE);
      if (VERBOSE_IS_LOGGABLE) {
        logWithTimeAndKey("Loaded resource from active resources", startTime, key);
      }
      return null;
    }

    //如果到了这里，说明活动缓存中没有找到对应的key对应的缓存
    EngineResource<?> cached = loadFromCache(key, isMemoryCacheable);
    //如果从内存缓存中找到了，直接回调，填充内容
    if (cached != null) {
      cb.onResourceReady(cached, DataSource.MEMORY_CACHE);
      if (VERBOSE_IS_LOGGABLE) {
        logWithTimeAndKey("Loaded resource from cache", startTime, key);
      }
      return null;
    }
	
  //从活动缓存中获取  isMemoryCacheable 标识，是否允许内存缓存
  @Nullable
  private EngineResource<?> loadFromActiveResources(Key key, boolean isMemoryCacheable) {
    //如果不允许内存缓存，就直接返回null
    if (!isMemoryCacheable) {
      return null;
    }
    //如果从活动缓存中获取到了key对应的资源，那么引用技术加一,并返回这个资源
    EngineResource<?> active = activeResources.get(key);
    if (active != null) {
      active.acquire();
    }
    return active;
  }

  //从内存缓存中获取资源， isMemoryCacheable 标识，是否允许内存缓存
  private EngineResource<?> loadFromCache(Key key, boolean isMemoryCacheable) {
    //如果不允许内存缓存，就直接返回null
    if (!isMemoryCacheable) {
      return null;
    }

    //从内存缓存中移除key，判断返回值是否为空，构建EngineResouece
    EngineResource<?> cached = getEngineResourceFromCache(key);
    if (cached != null) {
      //如果从内存缓存中获取到了，标识这个资源的引用加一
      cached.acquire();
      //然后添加到活动缓存中，这就是为什么要从内存缓存中移除了
      activeResources.activate(key, cached);
    }
    return cached;
  }

  Glide 里面的Resource采用的是引用技术的方式，使用这个对象，就会让这个引用计数加一，如果不用了响应的就会让这个引用计数减一，如果引用计数达到了0，就代表这个资源要被释放了
  上面的大致流程为：首先从活动缓存中获取，如果获取到了，就返回这个资源的引用，同时让这个资源的引用计数加一处理，如果从活动缓存中没有获取到，就会尝试的从内存缓存中获取
  如果获取到了，就会将这个引用资源从内存缓存中移除，然后添加到了活动缓存中，同时资源的引用计数加一处理，至于为什么要从内存缓存中移除大致是因为Glide的内存缓存是采用Lru算法的
  是有一个最大的缓存大小的，而活动缓存是采用HashMap来存储的，如果不从内存缓存移除的化，就会导致活动缓存存在一个对象的引用，内存缓存也存在一个对象的引用，如果此时内存缓存因为
  Lru算法的机制导致这个对象刚好要被销毁，此时活动的缓存的这个对象，你刚好在屏幕上展示，这就会出现问题。。。而且为了防止Lru算法的问题 活动缓存他使用的是HashMap来存储活动缓存
  同时这个HashMap存储的是虚引用的形式 final Map<Key, ResourceWeakReference> activeEngineResources = new HashMap<>();
  
  ActivityResource中添加一个资源是调用下面的这种形式
   //资源引用
  void activate(Key key, EngineResource<?> resource) {
    //将这个资源构成一个WeakReference对象，注意这里传递了 getReferenceQueue() ，大体的作用就是当你这个虚引用被资源回收掉了，这个被回收的对象会添加到ReferenceQueue中
	//这样我们查看这个ReferenceQueue中有没有内容就可以知道哪个对象被回收了，这样我们可以将这个引用从我们的HashMap中移除，防止内存泄漏
    ResourceWeakReference toPut =
        new ResourceWeakReference(
            key,
            resource,
            getReferenceQueue(),
            isActiveResourceRetentionAllowed);

    //添加到map中
    ResourceWeakReference removed = activeEngineResources.put(key, toPut);
    //返回值为之前已经存在的，如果不为空，就重置这个
    if (removed != null) {
      removed.reset();
    }
  }
 
  //ReferenceQueue对象，可以用来监听虚引用被回收的过程，就是说当你的虚引用的对象被回收的时候，会添加到这个队列里面，所以你可以获取到这个被回收
  //掉的虚引用，然后从map里面清除这个对象
  private ReferenceQueue<EngineResource<?>> getReferenceQueue() {
    if (resourceReferenceQueue == null) {
      resourceReferenceQueue = new ReferenceQueue<>();
      //创建一个清理Weak引用的线程,用来回收那些被释放的
      cleanReferenceQueueThread = new Thread(new Runnable() {
        @SuppressWarnings("InfiniteLoopStatement")
        @Override
        public void run() {
          Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
		  //查找是否有对象被清理了
          cleanReferenceQueue();
        }
      }, "glide-active-resources");
	  //线程启动
      cleanReferenceQueueThread.start();
    }
    return resourceReferenceQueue;
  }
  
    //清除WeakReference引用对象
  @SuppressWarnings("WeakerAccess")
  @Synthetic void cleanReferenceQueue() {
    while (!isShutdown) {
      try {
          //获取到当前已经被释放到的虚引用,如果获取到的这个引用不为空，就代表这个对象被销毁了
        ResourceWeakReference ref = (ResourceWeakReference) resourceReferenceQueue.remove();
        //然后通过handler来发送消息，通知当前的虚引用已经被移除了，可以从map里面移除了
        mainHandler.obtainMessage(MSG_CLEAN_REF, ref).sendToTarget();

        // This section for testing only.
        DequeuedResourceCallback current = cb;
        if (current != null) {
          current.onResourceDequeued();
        }
        // End for testing only.
      } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
      }
    }
  }
  
   //用来接收被移除的通知，接收到之后从活动缓存的map中移除掉
  private final Handler mainHandler = new Handler(Looper.getMainLooper(), new Callback() {
    @Override
    public boolean handleMessage(Message msg) {
      if (msg.what == MSG_CLEAN_REF) {
	    //从活动缓存中移除这个对象引用，同时给这个资源设置监听，回调执行onResourceReleased
        cleanupActiveReference((ResourceWeakReference) msg.obj);
        return true;
      }
      return false;
    }
  });
  
  //清除活动缓存从map的集合里面
  @SuppressWarnings("WeakerAccess")
  @Synthetic void cleanupActiveReference(@NonNull ResourceWeakReference ref) {
    Util.assertMainThread();

    //首先从活动缓存的集合中移除掉
    activeEngineResources.remove(ref.key);
    if (!ref.isCacheable || ref.resource == null) {
      return;
    }

    //当从活动缓存中移除这个资源的时候，构建一个EngineResource对象
    EngineResource<?> newResource = new EngineResource<>(ref.resource, /*isCacheable=*/ true, /*isRecyclable=*/ false);
    //设置资源的监听
    newResource.setResourceListener(ref.key, listener);
    //回调通知资源被释放了从活动缓存里面，Engine接受到了这个回调
    listener.onResourceReleased(ref.key, newResource);
  }
  
  下面为Engine接受的回调，至于为什么能接受到这个回调，是因为构建的时候，传递了this进去，Engine本身实现了这个借口
  //活动缓存中移除虚引用之后的回调通知，通知这个资源要添加到内存缓存中了
  @Override
  public void onResourceReleased(Key cacheKey, EngineResource<?> resource) {
    Util.assertMainThread();
    activeResources.deactivate(cacheKey);
    if (resource.isCacheable()) {
      //如果这个资源可以被缓存的，添加到内存缓存中,就是说如果活动获取中不要这个资源了，可以先存储到内存缓存中
      cache.put(cacheKey, resource);
    } else {
      //如果是不可以被缓存的，直接回收掉这个资源
      resourceRecycler.recycle(resource);
    }
  }
    
  如果内存缓存由于Lru算法导致移除的化，会怎么样呢，这样	
  //当资源从 Lru内存缓存中移除的回调,当达到了最多的lru算法内存的时候，就会移除
  @Override
  protected void onItemEvicted(@NonNull Key key, @Nullable Resource<?> item) {
    if (listener != null && item != null) {
      //通过接口，通知资源被移除了，listener 也为Engine实现 Engine中有这样的代码 cache.setResourceRemovedListener(this);  所以Engine能接受到这个移除的回调
      listener.onResourceRemoved(item);
    }
  }
  
  //如果当前的这个资源从内存缓存中移除的化，就直接回收掉
  @Override
  public void onResourceRemoved(@NonNull final Resource<?> resource) {
    Util.assertMainThread();
    resourceRecycler.recycle(resource); //真正的回收资源
  }
  
  resourceRecycler中的函数实现
  //回收资源
  void recycle(Resource<?> resource) {
    Util.assertMainThread();
    //如果当前正在回收,那就利用handler，handler可以做到相当于是一个队列的形式存在
    if (isRecycling) {
      // If a resource has sub-resources, releasing a sub resource can cause it's parent to be
      // synchronously
      // evicted which leads to a recycle loop when the parent releases it's children. Posting
      // breaks this loop.
      handler.obtainMessage(ResourceRecyclerCallback.RECYCLE_RESOURCE, resource).sendToTarget();
    } else {
      //当前没有执行回收，直接执行资源的回收操作
      isRecycling = true;
      resource.recycle();
      isRecycling = false;
    }
  }
  
  resource.recycle();就会执行到EngineResource中对应的方法,可以发现要想回收这个资源，前提条件是这个资源的引用计数必须为0带可以回收
  //回收操作
  @Override
  public void recycle() {
    //如果当前的引用技术大于0，不允许回收
    if (acquired > 0) {
      throw new IllegalStateException("Cannot recycle a resource while it is still acquired");
    }
    //如果当前已经被回收过了，不允许回收
    if (isRecycled) {
      throw new IllegalStateException("Cannot recycle a resource that has already been recycled");
    }
    isRecycled = true;
    if (isRecyclable) {
      //资源的回收
      resource.recycle();
    }
  }
  
  接下来分析下EngineResouce的引用计数的实现
  当获取到一个资源的时候，会调用acquire函数,标记当前的引用计数加一
   void acquire() {
    //如果当前资源以及回收过了一次
    if (isRecycled) {
      throw new IllegalStateException("Cannot acquire a recycled resource");
    }
    //如果不是主线程的化
    if (!Looper.getMainLooper().equals(Looper.myLooper())) {
      throw new IllegalThreadStateException("Must call acquire on the main thread");
    }
    //引用计数加一的操作
    ++acquired;
  }
  
  //监听了EngineJobListener,这是任务完成的监听,
  @SuppressWarnings("unchecked")
  @Override
  public void onEngineJobComplete(EngineJob<?> engineJob, Key key, EngineResource<?> resource) {
    Util.assertMainThread();
    // A null resource indicates that the load failed, usually due to an exception.
    if (resource != null) {
      //资源加载成功，设置ResourceListener，用于监听资源的释放
      resource.setResourceListener(key, this);
      //如果资源是可以缓存的，添加到活动缓存中
      if (resource.isCacheable()) {
        activeResources.activate(key, resource);
      }
    }
    //然后从jobs任务里面移除这个engineJob
    jobs.removeIfCurrent(key, engineJob);
  }
  
  //因为这个request要被移除了，所以这个资源对应的引用要减一处理
  public void release(Resource<?> resource) {
    Util.assertMainThread();
    if (resource instanceof EngineResource) {
        //资源的引用要减一处理
      ((EngineResource<?>) resource).release();
    } else {
      throw new IllegalArgumentException("Cannot release anything but an EngineResource");
    }
  }
  
  //就会调用到EngineResource中对应的函数
  void release() {
    if (acquired <= 0) {
      throw new IllegalStateException("Cannot release a recycled or not yet acquired resource");
    }
    if (!Looper.getMainLooper().equals(Looper.myLooper())) {
      throw new IllegalThreadStateException("Must call release on the main thread");
    }
	
	//当引用计数达到了0的时候，执行回调函数，就会将这个资源添加到内存缓存中，或者回收掉
    if (--acquired == 0) {
      listener.onResourceReleased(key, this);
    }
  }
  
  //活动缓存中移除虚引用之后的回调通知，通知这个资源要添加到内存缓存中了
  @Override
  public void onResourceReleased(Key cacheKey, EngineResource<?> resource) {
    Util.assertMainThread();
    activeResources.deactivate(cacheKey);
    if (resource.isCacheable()) {
      //如果这个资源可以被缓存的，添加到内存缓存中
      cache.put(cacheKey, resource);
    } else {
      //如果是不可以被缓存的，直接回收掉这个资源
      resourceRecycler.recycle(resource);
    }
  }
  
  总结下，当Glide要加载资源的时候，首先会去活动缓存中查找，如果查找到了，就会直接返回，并让这个引用加一处理，如果没有查找到，就会去内存缓存中查找，如果查找到了要从内存缓存中
  移除这个对象，然后添加到活动缓存，同时这个资源的引用计数要加一处理，如果是新的资源就会去网络或者磁盘中获取到，当获取成功的时候，就会添加到活动缓存中，活动缓存是采用HashMap的实现
  存储的是虚引用，由于采用了ReferenceQueue所以可以知道哪些对象由于被回收，从而进行从HashMap中移除这个对象，然后回调执行onResourceReleased 回调，会将这个资源添加到内存缓存中，
  内存缓存使用了Lru算法，所以会有到达内存最大值的情况，此时会执行移除操作，回调执行onResourceRemoved函数，然后使用resourceRecycler 来执行回收，当然这个资源如果引用计数不为0的化
  是不会被回收
  
```
 ****Glide 生命周期机制****
 ===
```java
首先查看Glide中调用with的时候，干了什么事情
//获取到RequestManager对象
  @NonNull
  public RequestManager get(@NonNull Context context) {
    if (context == null) {
      throw new IllegalArgumentException("You cannot start a load on a null Context");
    } else if (Util.isOnMainThread() && !(context instanceof Application)) {//如果当前不是在主线程，或者传递进来的context不属于Application对象
      if (context instanceof FragmentActivity) {
        return get((FragmentActivity) context);
      } else if (context instanceof Activity) {
        return get((Activity) context);
      } else if (context instanceof ContextWrapper) {
        return get(((ContextWrapper) context).getBaseContext());
      }
    }

    //如果是context 对象是Application 或者不是在主线程中调用这个with 则构建一个ApplicationManager对象
    return getApplicationManager(context);
  }
  
  //获取一个ApplicationManager对象
  @NonNull
  private RequestManager getApplicationManager(@NonNull Context context) {
    // Either an application context or we're on a background thread.
    if (applicationManager == null) {
      synchronized (this) {
        if (applicationManager == null) {
          // Normally pause/resume is taken care of by the fragment we add to the fragment or
          // activity. However, in this case since the manager attached to the application will not
          // receive lifecycle events, we must force the manager to start resumed using
          // ApplicationLifecycle.

          // TODO(b/27524013): Factor out this Glide.get() call.
          Glide glide = Glide.get(context.getApplicationContext());
          //利用factory 来生产RequestManager对象
          applicationManager =
              factory.build(
                  glide,
                  new ApplicationLifecycle(),
                  new EmptyRequestManagerTreeNode(),
                  context.getApplicationContext());
        }
      }
    }
    return applicationManager;
  }
  
  对于传递进来的context 对象是Application 或者不是在主线程中调用这个with 则直接构建一个RequestManager对象，并返回。。并没有管理Fragment的操作，所以这样会导致他的生命周期
  为整个应用程序的生命周期。。所以不推荐这样使用
  
  如果是正常的使用就会进入里面
   if (context instanceof FragmentActivity) {
        return get((FragmentActivity) context);
      } else if (context instanceof Activity) {
        return get((Activity) context);
      } else if (context instanceof ContextWrapper) {
        return get(((ContextWrapper) context).getBaseContext());
      }
  这里分析一个既可，那俩个是类似的	分析 return get((Activity) context); 实现
 
  //如果绑定的是Activity对象
  @SuppressWarnings("deprecation")
  @NonNull
  public RequestManager get(@NonNull Activity activity) {
    if (Util.isOnBackgroundThread()) {
      return get(activity.getApplicationContext());
    } else {
      assertNotDestroyed(activity);
      //获取到Activity中的FragmentManager对象
      android.app.FragmentManager fm = activity.getFragmentManager();
      return fragmentGet(activity, fm, /*parentHint=*/ null, isActivityVisible(activity));
    }
  }
  
  private RequestManager fragmentGet(@NonNull Context context,
      @NonNull android.app.FragmentManager fm,
      @Nullable android.app.Fragment parentHint,
      boolean isParentVisible) {
    //获取到RequestManagerFragment对象
    RequestManagerFragment current = getRequestManagerFragment(fm, parentHint, isParentVisible);
    //获取到RequestManagerFragment中的RequestManager对象,
    RequestManager requestManager = current.getRequestManager();
    //第一次获取肯定为空
    if (requestManager == null) {
      // TODO(b/27524013): Factor out this Glide.get() call.
      Glide glide = Glide.get(context);
      //则利用工厂factory 构建一个RequestManager对象
      requestManager = factory.build(glide, current.getGlideLifecycle(), current.getRequestManagerTreeNode(), context);
      //给RequestManagerFragment设置RequestManager对象
      current.setRequestManager(requestManager);
    }
    //返回RequestManager对象
    return requestManager;
  }
  
  getRequestManagerFragment(fm, parentHint, isParentVisible);实现过程
  @NonNull
  private RequestManagerFragment getRequestManagerFragment( @NonNull final android.app.FragmentManager fm, @Nullable android.app.Fragment parentHint,boolean isParentVisible) {
    //如果当前的Activity 中的FragmentManager对象中查找是否已经添加了
    RequestManagerFragment current = (RequestManagerFragment) fm.findFragmentByTag(FRAGMENT_TAG);
    //如果没有添加
    if (current == null) {
      //再去临时的集合中查找
      current = pendingRequestManagerFragments.get(fm);
      //如果还是没有，就构建一个
      if (current == null) {
	    //构建一个新的RequestManagerFragment对象
        current = new RequestManagerFragment();
        current.setParentFragmentHint(parentHint);
        if (isParentVisible) {
          current.getGlideLifecycle().onStart();
        }
        //添加到临时的集合中
        pendingRequestManagerFragments.put(fm, current);
        //再利用FragmentManager对象，添加当前创建的Fragment，这样这个创建的Fragment就跟Activity绑定在一起了，所以这个Fragment可以收到生命周期的管理
        fm.beginTransaction().add(current, FRAGMENT_TAG).commitAllowingStateLoss();
        //这里为什么要使用一个handler来移除消息，是因为上面FragmentManager添加Fragment的时候内部是通过Handler来添加的，所以是异步的，所以这里再发送一个消息的化
        //就能保证这个消息接收到的时候，前面的Fragment肯定已经添加成功了，这样才可以移除临时集合的Fragment，注意要保证这个handler为主线程的，才能共享一个MessageQueue对象
        handler.obtainMessage(ID_REMOVE_FRAGMENT_MANAGER, fm).sendToTarget();
      }
    }
    //如果之前已经添加，就直接返回
    return current;
  }

Handler接收到了  ID_REMOVE_FRAGMENT_MANAGER消息的实现
case ID_REMOVE_FRAGMENT_MANAGER://如果收到了这个消息，说明创建的Fragment肯定已经添加到了FragmentManager中，所以这里可以从临时集合中移除了
    android.app.FragmentManager fm = (android.app.FragmentManager) message.obj;
    key = fm;
	如果执行到了这里，那么肯定是已经将创建的RequestManagerFragment对象添加到了FragmentManager中了
    removed = pendingRequestManagerFragments.remove(fm);
break;

    构建RequestManagerFragment对象的时候
    RequestManagerFragment current = new RequestManagerFragment();
	 public RequestManagerFragment() {
		this(new ActivityFragmentLifecycle());
	}
	
   对于ActivityFragmentLifecycle，他是实现了 class ActivityFragmentLifecycle implements Lifecycle ，而Lifecycle接口有这样的函数
   public interface Lifecycle {
    void addListener(@NonNull LifecycleListener listener);
    void removeListener(@NonNull LifecycleListener listener);
  }

	RequestManagerFragment(@NonNull ActivityFragmentLifecycle lifecycle) {
		this.lifecycle = lifecycle;
	}
	
	//构建一个RequestManager对象,同时将在RequestManagerFragment中创建的lifecycle对象传递过去
	requestManager = factory.build(glide, current.getGlideLifecycle(), current.getRequestManagerTreeNode(), context);
	
	RequestManager(
      Glide glide,
      Lifecycle lifecycle,
      RequestManagerTreeNode treeNode,
      RequestTracker requestTracker,
      ConnectivityMonitorFactory factory,
      Context context) {
    this.glide = glide;
    this.lifecycle = lifecycle;
    this.treeNode = treeNode;
    this.requestTracker = requestTracker;
    this.context = context;
	....
	
	//将RequestManager添加到lifecycle里面，这样RequestManager就能接受到生命周期的回调
    lifecycle.addListener(this);
	
	//将监听网络的类，加入生命周期的集合中,受到生命周期的回调
    lifecycle.addListener(connectivityMonitor);
}

对于 lifecycle.addListener()，就会执行到ActivityFragmentLifecycle 中对应的函数
  @Override
  public void addListener(@NonNull LifecycleListener listener) {
    lifecycleListeners.add(listener);
    if (isDestroyed) {
      listener.onDestroy();
    } else if (isStarted) {
      listener.onStart();
    } else {
      listener.onStop();
    }
  }

 其中的lifecycleListeners为一个集合，用来存储回调 定义为 
  //声明周期管理的接口集合
  private final Set<LifecycleListener> lifecycleListeners = Collections.newSetFromMap(new WeakHashMap<LifecycleListener, Boolean>());
  
  所以当我们创建的RequestManagerFragment接受到了onStart(),onStop()生命周期回调的时候
  @Override
  public void onStart() {
    super.onStart();
    //传递生命周期
    lifecycle.onStart();
  }

  @Override
  public void onStop() {
    super.onStop();
    //传递生命周期
    lifecycle.onStop();
  }  
  
   lifecycle.onStart(); 就会执行到ActivityLifeCycle中的onStart()函数
  //接受到了RequestManagerFragment的生命周期的回调
  void onStart() {
    //标识已经开始
    isStarted = true;
    for (LifecycleListener lifecycleListener : Util.getSnapshot(lifecycleListeners)) {
      lifecycleListener.onStart();
    }
  }

  lifecycle.onStop();  就会执行到ActivityLifeCycle中的onStop()函数
  //接受到了RequestManagerFragment的生命周期的回调
  void onStop() {
    isStarted = false;
    for (LifecycleListener lifecycleListener : Util.getSnapshot(lifecycleListeners)) {
      lifecycleListener.onStop();
    }
  }
  
  这样就会执行到RequestManager中对应的生命周期，用来执行请求的执行
  /**
   *
   * 因为RequestManager是有添加到创建的RequestManagerFragment中的ActivityLifecycle集合中的，所以RequestManager是可以响应到生命周期的回调的
   *
   */
  @Override
  public void onStart() {
    //恢复请求的执行
    resumeRequests();
    //然后转发给targetTracker ，相当于targetTracker也受到了RequestManagerFragment的生命周期的管理
    targetTracker.onStart();
  }
  
  /**
   *
   * 因为RequestManager是有添加到创建的RequestManagerFragment中的ActivityLifecycle集合中的，所以RequestManager是可以响应到生命周期的回调的
   */
  @Override
  public void onStop() {
    pauseRequests();
    //然后转发给targetTracker ，相当于targetTracker也受到了RequestManagerFragment的生命周期的管理
    targetTracker.onStop();
  }
  
  //网络监听器收到了RequestManagerFragment的生命周期的回调,动态注册网络监听
  @Override
  public void onStart() {
    register();
  }

  // //网络监听器收到了RequestManagerFragment的生命周期的回调,注销网络监听
  @Override
  public void onStop() {
    unregister();
  }
  
  我们在构建一个请求的时候会将构建的请求添加给RequestManager来管理,具体的代码实现为 
  //先从RequestManager中移除这个targer
  requestManager.clear(target);
  //给target设置Reuqest,实际是获取到Target中的View，将request作为一个tag设置到view上，绑定起来
  target.setRequest(request);
  //将当前的请求交给RequestManager来处理
  requestManager.track(target, request);

  就会调用到RequestManager中对应的track 函数
  void track(@NonNull Target<?> target, @NonNull Request request) {
    //将当前的target交给targetTracker， 的targets 集合中保存,也即是设置关联
    targetTracker.track(target);
    //将当亲的request交给RequestTracker管理
    requestTracker.runRequest(request);
  }
  
  这里有俩个成员变量   
  //构建一个TargetTracker对象，本身实现了LifecycleListener
  private final TargetTracker targetTracker = new TargetTracker();
  new RequestTracker(),//构建一个RequestTracker对象，由他管理Request的执行
  
  targetTracker.track(target)函数实现为
  //targets为一个集合用来存储Target 对象，而且也是受生命周期的监听的
  private final Set<Target<?>> targets = Collections.newSetFromMap(new WeakHashMap<Target<?>, Boolean>());
  //将当前的target添加到targets集合中，受到生命周期的回调
  public void track(@NonNull Target<?> target) {
    targets.add(target);
  }
  
  requestTracker.runRequest(request);函数实现为
  //运行一个Request
  public void runRequest(@NonNull Request request) {
    //首先将当前的request添加到集合中
    requests.add(request);
    //如果当前没有处于暂停状态，直接执行请求
    if (!isPaused) {
      request.begin();
    } else {
      if (Log.isLoggable(TAG, Log.VERBOSE)) {
        Log.v(TAG, "Paused, delaying request");
      }
      //否则添加到pendingRequests集合中
      pendingRequests.add(request);
    }
  }
  
  而pendingRequests集合的定义为
  //还没有开始的请求集合
  private final List<Request> pendingRequests = new ArrayList<>();
  
  先总结下创建Request的过程，当创建一个Request的时候，会添加到RequestManager中，RequestManager中又有俩个成员变量，一个是RequestTracker用来真正的管理Request，还有一个为TargetTracker
  用来存储当前的Target对象
  
  当我们的RequestManager接收到了生命周期的回调的时候
    public void onStart() {
    //恢复请求的执行
    resumeRequests();
    //然后转发给targetTracker ，相当于targetTracker也受到了RequestManagerFragment的生命周期的管理
    targetTracker.onStart();
  }
  
   public void resumeRequests() {
    Util.assertMainThread();
    //利用requestTracker来执行恢复所有的请求
    requestTracker.resumeRequests();
  }
  
  requestTracker.resumeRequests(); 函数的实现
  //开始执行目前所有没有完成或者失败的请求
  public void resumeRequests() {
    //标识当前没有暂停
    isPaused = false;
    for (Request request : Util.getSnapshot(requests)) {
       //执行Request 针对当前Request没有执行完成，并且 没有被取消，没有正在运行的请求
      if (!request.isComplete() && !request.isCancelled() && !request.isRunning()) {
        //请求开始执行
        request.begin();
      }
    }
    //清除等待的请求的队列
    pendingRequests.clear();
  }
  
  targetTracker.onStart(); 实现为
  //TargetTracker虽然也实现了LifecycleListener，但是他并不是也是添加到RequestManagerFragment中的ActivityLifecycle集合中，他是由RequestManager来触发的
  //也即是RequestManager实现了LifecycleListener，并添加到了ActivityLifecycle集合中,所以RequestManager能收到生命周期的回调，而RequestManager中又有一个TargetTracker的实例对象
  //所以他可以通过这个对象，将生命周期传递进来，所以TargetTracker能间接的受到RequestManagerFragment的生命周期的管理，对于他里面targets，原理也是一样，又在传递一层生命周期而已
  //所以也间接的受到了RequestManagerFragment的生命周期的管理,这样就能在（ImageViewTarget）onStart的时候执行动画，onStop的是停止动画
  @Override
  public void onStart() {
    for (Target<?> target : Util.getSnapshot(targets)) {
      target.onStart();
    }
  }
  
  而对于网络监听器,在构建DefaultConnectivityMonitor 的时候，传递进了一个RequestManagerConnectivityListener ，这个对象实现了ConnectivityListener接口，而且将管理请求的requestTracker对象
  传递进来
   RequestManagerConnectivityListener(@NonNull RequestTracker requestTracker) {
      this.requestTracker = requestTracker;
    }

    @Override
    public void onConnectivityChanged(boolean isConnected) {
      //当网络发生变化的时候
      if (isConnected) {
        //重新执行网络请求
        requestTracker.restartRequests();
      }
    }
  
  所以当DefaultConnectivityMonitor的网络状态发生改变的时候
  //网络变化的广播
  private final BroadcastReceiver connectivityReceiver = new BroadcastReceiver() {
    @Override
    public void onReceive(@NonNull Context context, Intent intent) {
      boolean wasConnected = isConnected;
      isConnected = isConnected(context);
      if (wasConnected != isConnected) {
        if (Log.isLoggable(TAG, Log.DEBUG)) {
          Log.d(TAG, "connectivity changed, isConnected: " + isConnected);
        }

        //回调网络改变的监听 也即是执行到了RequestManagerConnectivityListener 中对应的方法  requestTracker.restartRequests(); 最终做到重新执行请求
        listener.onConnectivityChanged(isConnected);
      }
    }
  };
  
  总结下Glide的生命周期的管理，一个Activity对应一个RequestMangerFragment，一个RequestMangerFragment对应一个RequestManager对象，对应一个ActivityLifeCycle对象,对应一个RequestTracker对象
  对应一个TargetTracker对象,RequestManger中实现了LifeCyecle接口，还有DefaultConnectivityMonitor 实现了LifeCycle接口,并且将自己添加到了ActivityLifeCycle中的集合中，当RequestMangerFragmenet
  接受到了onStart的生命周期的回调的时候，就会执行ActivityLifeCycle中的onStart函数，onStart函数中就会遍历执行集合中的每一个回调，依次传递过去，所以RequstManager收到了onStart的回调
  在onStart的函数中，会利用RequestTracker执行真正的请求的执行，会利用TargerTracker执行动画等,DefaultConnectivityMonitor 接受到了onStart回调，就会执行动态的注册网络的监听,
  对于RequetMangerFragment onStop的时候,流程是一样的，结果就是会利用RequestTracker在暂停网络的请求，利用TargetTracker执行动画的停止，移除DefaultConnectivityMonitor 的动态注册广播
```
 ****Glide注册机的机制****
 ===
```java
我们在使用Glide的时候，这样使用   Glide.with(this).load("https://ps.ssl.qhimg.com/sdmt/89_135_100/t01418930ed0ec37af3.jpg").into(iv);对应的函数声明为
 public RequestBuilder<Drawable> load(@Nullable String string) ，当然还有其他的形式，比如 public RequestBuilder<Drawable> load(@Nullable Bitmap bitmap) 
 public RequestBuilder<Drawable> load(@Nullable Uri uri),public RequestBuilder<Drawable> load(@Nullable File file)...也即是说我们可以在load的参数里面可以传很多的类型
 Glide存在一个注册机，他在这个注册机里面预先注册了很多的model 对应的解析器，所以当在使用的时候就会根据传递进来的参数，获取到对应的解析器，完成对应的处理，源码体现在
 Glide的初始化的时候，就有这样的代码
 
 //构建注册类里面含有多个注册的集合
 registry = new Registry();
 registry.register(new DefaultImageHeaderParser());
 
 Registry的构造函数实现为
 public Registry() {
    this.modelLoaderRegistry = new ModelLoaderRegistry(throwableListPool);
    this.encoderRegistry = new EncoderRegistry();
    this.decoderRegistry = new ResourceDecoderRegistry();
    this.resourceEncoderRegistry = new ResourceEncoderRegistry();
    this.dataRewinderRegistry = new DataRewinderRegistry();
    this.transcoderRegistry = new TranscoderRegistry();
    this.imageHeaderParserRegistry = new ImageHeaderParserRegistry();
    //往 ResourceDecoderRegistry 里面添加一些key
    setResourceDecoderBucketPriorityList(Arrays.asList(BUCKET_GIF, BUCKET_BITMAP, BUCKET_BITMAP_DRAWABLE));
  }
  可以看到这里有很多的注册类的对象，每一个注册类里面本质都是有一个集合用来存储对应的要注册的类，比如ModelLoaderRegistry构造函数为
  public ModelLoaderRegistry(@NonNull Pool<List<Throwable>> throwableListPool) {
    this(new MultiModelLoaderFactory(throwableListPool));
  }
  而MulteModelLoaderFacotry的构造函数为
  public class MultiModelLoaderFactory {
    ....
	//Entry集合对象
	private final List<Entry<?, ?>> entries = new ArrayList<>();
	...
	public MultiModelLoaderFactory(@NonNull Pool<List<Throwable>> throwableListPool) {
		this(throwableListPool, DEFAULT_FACTORY);
  }
  ...
  }
  再比如EncoderRegistry 构造函数为
  public class EncoderRegistry {
	private final List<Entry<?>> encoders = new ArrayList<>();
	...
  }
  ResourceDecoderRegistry 构造函数为
  public class ResourceDecoderRegistry {
	//解码的集合
	private final Map<String, List<Entry<?, ?>>> decoders = new HashMap<>();
	...
  }
  在Glide的构造函数中还有这样的代码,下面的是省略的代码，每一种都抽出一个来解析
  //注册
  registry
        .append(ByteBuffer.class, new ByteBufferEncoder())//ByteBuffer转换为File
		...
        /* Bitmaps */
        .append(Registry.BUCKET_BITMAP, ByteBuffer.class, Bitmap.class, byteBufferBitmapDecoder)//添加一个从ByteBuffer转成Bitmap的 byteBufferBitmapDecoder Entry对象
		...
		.register(new ByteBufferRewinder.Factory())
        .append(File.class, ByteBuffer.class, new ByteBufferFileLoader.Factory())
		...
		.register(Bitmap.class, byte[].class, bitmapBytesTranscoder)
		...
    
首先看 .append(ByteBuffer.class, new ByteBufferEncoder()) 源码实现为
  @NonNull
  public <Data> Registry append(@NonNull Class<Data> dataClass, @NonNull Encoder<Data> encoder) {
    encoderRegistry.append(dataClass, encoder);
    return this;
  }
  public synchronized <T> void append(@NonNull Class<T> dataClass, @NonNull Encoder<T> encoder) {
    encoders.add(new Entry<>(dataClass, encoder));
  }
  可以看到他会将传递进来的参数构建成一个Entry对象然后存储到encoders集合中
 
对于 .append(Registry.BUCKET_BITMAP, ByteBuffer.class, Bitmap.class, byteBufferBitmapDecoder) 函数实现为
public <Data, TResource> Registry append( String bucket,Class<Data> dataClass,Class<TResource> resourceClass,ResourceDecoder<Data, TResource> decoder) {
    decoderRegistry.append(bucket, decoder, dataClass, resourceClass);
    return this;
}
//添加entry对象，往decoders 里面
public synchronized <T, R> void append(@NonNull String bucket,
      @NonNull ResourceDecoder<T, R> decoder,
      @NonNull Class<T> dataClass, @NonNull Class<R> resourceClass) {
      //getOrAddEntryList(bucket) 可能会返回一个刚构建的集合，也可能返回之前已经存在的集合，添加一个entry对象
    getOrAddEntryList(bucket).add(new Entry<>(dataClass, resourceClass, decoder));
}
//如果根据制定的bucket的key获取到value，如果没有就构建一个然后添加到decoders 里面，否则就直接的获取到
@NonNull
private synchronized List<Entry<?, ?>> getOrAddEntryList(@NonNull String bucket) {
    //如果bucketPriorityList 没有包含这个bucket，就添加进里面，相当于是一个key的集合
    if (!bucketPriorityList.contains(bucket)) {
      // Add this unspecified bucket as a low priority bucket.
      bucketPriorityList.add(bucket);
    }
    //再从decoders集合中获取到对应的value,如果不存在就构建一个
    List<Entry<?, ?>> entries = decoders.get(bucket);
    if (entries == null) {
      entries = new ArrayList<>();
      decoders.put(bucket, entries);
    }
    return entries;
}
可以看到他会将传递的第一个参数bucket，也即是BUCKET_BITMAP(bitmap)作为key从decoders中获取到对应的List<<Entry>> ，然后将传递的其他的参数构建成一个entry对象，然后添加到这个集合中

对于.register(new ByteBufferRewinder.Factory())
@NonNull
  public Registry register(@NonNull DataRewinder.Factory<?> factory) {
    dataRewinderRegistry.register(factory);
    return this;
}
public synchronized void register(@NonNull DataRewinder.Factory<?> factory) {
    rewinders.put(factory.getDataClass(), factory);
}
而rewinders本质为   private final Map<Class<?>, DataRewinder.Factory<?>> rewinders = new HashMap<>();
所以也是存储到集合中

对于.append(File.class, ByteBuffer.class, new ByteBufferFileLoader.Factory())
public <Model, Data> Registry append(
      @NonNull Class<Model> modelClass, @NonNull Class<Data> dataClass,
      @NonNull ModelLoaderFactory<Model, Data> factory) {
    modelLoaderRegistry.append(modelClass, dataClass, factory);
    return this;
}
public synchronized <Model, Data> void append(
      @NonNull Class<Model> modelClass,
      @NonNull Class<Data> dataClass,
      @NonNull ModelLoaderFactory<? extends Model, ? extends Data> factory) {
  multiModelLoaderFactory.append(modelClass, dataClass, factory);
  cache.clear();
}

synchronized <Model, Data> void append(
      @NonNull Class<Model> modelClass,
      @NonNull Class<Data> dataClass,
      @NonNull ModelLoaderFactory<? extends Model, ? extends Data> factory) {
    add(modelClass, dataClass, factory, /*append=*/ true);
}
private <Model, Data> void add(
      @NonNull Class<Model> modelClass,
      @NonNull Class<Data> dataClass,
      @NonNull ModelLoaderFactory<? extends Model, ? extends Data> factory,
      boolean append) {
    Entry<Model, Data> entry = new Entry<>(modelClass, dataClass, factory);
    entries.add(append ? entries.size() : 0, entry);
}
entries 本质为 private final List<Entry<?, ?>> entries = new ArrayList<>();
这里所做的就是将传递进来的参数封装成一个entry对象，然后添加到entries集合中

对于 .register(Bitmap.class, byte[].class, bitmapBytesTranscoder)
public <TResource, Transcode> Registry register(
      @NonNull Class<TResource> resourceClass, @NonNull Class<Transcode> transcodeClass,
      @NonNull ResourceTranscoder<TResource, Transcode> transcoder) {
    transcoderRegistry.register(resourceClass, transcodeClass, transcoder);
    return this;
}
public synchronized <Z, R> void register(
      @NonNull Class<Z> decodedClass, @NonNull Class<R> transcodedClass,
      @NonNull ResourceTranscoder<Z, R> transcoder) {
    transcoders.add(new Entry<>(decodedClass, transcodedClass, transcoder));
}
transcoders 本质为   private final List<Entry<?, ?>> transcoders = new ArrayList<>();
这里所做的就是将传递进来的参数封装成一个entry对象，然后添加到transcoders集合中

总结，Glide在构造函数中初始化的时候，就往不同的注册机里面添加了很多的注册类

下面介绍怎么使用
当我们在使用load(String url);的时候执行了
@NonNull
private RequestBuilder<TranscodeType> loadGeneric(@Nullable Object model) {
    //标识当前RequestBuilder的model对象
    this.model = model;
    //标识已经设置了model
    isModelSet = true;
    return this;
}
会将当前传递的model，存储在成员变量中，同时将isModelSet变量置为true

在使用into(ImageView iv)的时候，执行了

//将传递进来的target 构建request对象 ,如果没有设置errorBuilder的化，返回的是一个SingleReuqest对象 
Request request = buildRequest(target, targetListener, options);

当Request请求开始执行的时候
会执行到  onSizeReady(overrideWidth, overrideHeight); 在这个函数中，会执行Engine的load函数，如果从活动缓存中还有内存缓存中都没有获取到的化，就会执行下面的代码
//添加资源请求的回调
engineJob.addCallback(cb);
//交给线程池执行，DecodeJob本身实现了Runnable接口 所以可以执行到decodeJob run方法
engineJob.start(decodeJob);

在DecoderJob中的run方法中会执行
runWrapped();
//执行
private void runWrapped() {
    switch (runReason) {
      case INITIALIZE:
        //初始化的时候状态为Stage.INITIALIZE ，所以这里会返回 Stage.RESOURCE_CACHE
        stage = getNextStage(Stage.INITIALIZE);
        //得到下一个生成器
        currentGenerator = getNextGenerator();
        //执行生成器
        runGenerators();
        break;
      case SWITCH_TO_SOURCE_SERVICE://在执行SOURCE中获取的时候，状态已经切换为了SWITCH_TO_SOURCE_SERVICE，所以会进入里面
        runGenerators();
        break;
      case DECODE_DATA:
        decodeFromRetrievedData();
        break;
      default:
        throw new IllegalStateException("Unrecognized run reason: " + runReason);
    }
}
因为一开始状态为INITIALIZE ，所以这里返回的currentGenerator对象为 ResourceCacheGenerator
 //执行生成器
  private void runGenerators() {
    currentThread = Thread.currentThread();
    startFetchTime = LogTime.getLogTime();

    //标识是否已经开始
    boolean isStarted = false;
    //currentGenerator.startNext() 会循环的进行查找对应的注册机，如果能找到则返回true，否则返回false
    while (!isCancelled && currentGenerator != null && !(isStarted = currentGenerator.startNext())) {
      //如果到了这里，说明还没有找到，那么说明在磁盘缓存中没有存在，那么获取下一个状态
      stage = getNextStage(stage);
      //继续根据下一个状态，找到下一个生成器,然后继续执行while循环,如果是SOURCE状态的化，currentGenerator也已经赋值为了SourceGenerator
      currentGenerator = getNextGenerator();

      //如果状态为SOURCE
      if (stage == Stage.SOURCE) {
         //任务的执行，转到线程池里面执行,
        reschedule();
        return;
      }
    }
}
当执行到了currentGenerator.startNext()的时候，就会先执行到ResourceCacheGenerator 中的对应的方法
public boolean startNext() {
    List<Key> sourceIds = helper.getCacheKeys();
    if (sourceIds.isEmpty()) {
      return false;
    }

    //如果获取到缓存的key集合不为空
    List<Class<?>> resourceClasses = helper.getRegisteredResourceClasses();
    if (resourceClasses.isEmpty()) {
      if (File.class.equals(helper.getTranscodeClass())) {
        return false;
      }
      // TODO(b/73882030): This case gets triggered when it shouldn't. With this assertion it causes
      // all loads to fail. Without this assertion it causes loads to miss the disk cache
      // unnecessarily
      // throw new IllegalStateException(
      //    "Failed to find any load path from " + helper.getModelClass() + " to "
      //        + helper.getTranscodeClass());
    }

    //如果获取到的resourceClass集合对象不为空
    while (modelLoaders == null || !hasNextModelLoader()) {
      resourceClassIndex++;
      if (resourceClassIndex >= resourceClasses.size()) {
        sourceIdIndex++;
        if (sourceIdIndex >= sourceIds.size()) {
          return false;
        }
        resourceClassIndex = 0;
      }

      Key sourceId = sourceIds.get(sourceIdIndex);
      Class<?> resourceClass = resourceClasses.get(resourceClassIndex);
      Transformation<?> transformation = helper.getTransformation(resourceClass);
      // PMD.AvoidInstantiatingObjectsInLoops Each iteration is comparatively expensive anyway,
      // we only run until the first one succeeds, the loop runs for only a limited
      // number of iterations on the order of 10-20 in the worst case.
      //构建一个ResourceCacheKey
      currentKey =
          new ResourceCacheKey(// NOPMD AvoidInstantiatingObjectsInLoops
              helper.getArrayPool(),
              sourceId,
              helper.getSignature(),
              helper.getWidth(),
              helper.getHeight(),
              transformation,
              resourceClass,
              helper.getOptions());

      //然后从磁盘缓存中获取
      cacheFile = helper.getDiskCache().get(currentKey);
      //如果可以从磁盘缓存中获取到
      if (cacheFile != null) {
        sourceKey = sourceId;
        //然后在获取到File对应的ModeLLoader集合列表
        modelLoaders = helper.getModelLoaders(cacheFile);
        modelLoaderIndex = 0;
      }
    }

    //然后再遍历获取的ModelLoader，分别构建对应的LoadData对象
    loadData = null;
    boolean started = false;
    while (!started && hasNextModelLoader()) {
      ModelLoader<File, ?> modelLoader = modelLoaders.get(modelLoaderIndex++);
      loadData = modelLoader.buildLoadData(cacheFile, helper.getWidth(), helper.getHeight(), helper.getOptions());
      if (loadData != null && helper.hasLoadPath(loadData.fetcher.getDataClass())) {
        //标识已经找到了，返回true,如果没有找到返回false
        started = true;
        //最后执行加载的是loaderData对象的fetcher对象来执行加载,同时传递了回调DataCallback
        loadData.fetcher.loadData(helper.getPriority(), this);
      }
    }
    return started;
  }

首先看 helper.getCacheKeys();的函数实现
//获取到缓存的Keys的集合
List<Key> getCacheKeys() {
    if (!isCacheKeysSet) {
      isCacheKeysSet = true;
      //先清除缓存的key集合
      cacheKeys.clear();

      //获取到当前model对应的LoadData集合对象
      List<LoadData<?>> loadData = getLoadData();
      //noinspection ForLoopReplaceableByForEach to improve perf
      for (int i = 0, size = loadData.size(); i < size; i++) {
        LoadData<?> data = loadData.get(i);
        if (!cacheKeys.contains(data.sourceKey)) {
          cacheKeys.add(data.sourceKey);
        }
        for (int j = 0; j < data.alternateKeys.size(); j++) {
          if (!cacheKeys.contains(data.alternateKeys.get(j))) {
            cacheKeys.add(data.alternateKeys.get(j));
          }
        }
      }
    }
    return cacheKeys;
}
getLoadData();函数的实现为

//获取到LoadData的集合
List<LoadData<?>> getLoadData() {
    if (!isLoadDataSet) {
      isLoadDataSet = true;
      //获取之前先清除集合的内容
      loadData.clear();
      //从注册机里面获取到对应的model对应的ModelLoader的集合
      List<ModelLoader<Object, ?>> modelLoaders = glideContext.getRegistry().getModelLoaders(model);
      //noinspection ForLoopReplaceableByForEach to improve perf
      //遍历构建LoadData对象，然年后添加到loadData集合中
      for (int i = 0, size = modelLoaders.size(); i < size; i++) {
        ModelLoader<Object, ?> modelLoader = modelLoaders.get(i);
        //构建LoadData对象
        LoadData<?> current = modelLoader.buildLoadData(model, width, height, options);
        //如果不为空，然后添加到集合 loadData中
        if (current != null) {
          loadData.add(current);
        }
      }
    }
    return loadData;
}

当执行到这里的时候，看到没有这里会根据当前传递进来的model从注册机里面获取到对应的值，函数的实现为
List<ModelLoader<Object, ?>> modelLoaders = glideContext.getRegistry().getModelLoaders(model);

//根据传递进来的model 获取到对应的可以解析这个model类型的集合List<ModelLoader>
@NonNull
public <Model> List<ModelLoader<Model, ?>> getModelLoaders(@NonNull Model model) {
    //返回当前可以处理这个model类型的ModelLoader集合
    List<ModelLoader<Model, ?>> result = modelLoaderRegistry.getModelLoaders(model);
    //如果返回的集合为空，则构建一个空的
    if (result.isEmpty()) {
      throw new NoModelLoaderAvailableException(model);
    }
    //返回集合
    return result;
}
  
modelLoaderRegistry.getModelLoaders(model);函数的实现为
//获取到model对应的ModelLoader的集合
@NonNull
public synchronized <A> List<ModelLoader<A, ?>> getModelLoaders(@NonNull A model) {
    //根据model获取到对应的List<ModelLoader>集合
    List<ModelLoader<A, ?>> modelLoaders = getModelLoadersForClass(getClass(model));
    int size = modelLoaders.size();
    //创建一个集合，用来存储当前那些是可以处理这个model
    List<ModelLoader<A, ?>> filteredLoaders = new ArrayList<>(size);
    //noinspection ForLoopReplaceableByForEach to improve perf
    //遍历判断那些是可以处理这个model的
    for (int i = 0; i < size; i++) {
      ModelLoader<A, ?> loader = modelLoaders.get(i);
      if (loader.handles(model)) {
        //如果可以处理，就添加到集合里面，然后返回这个集合
        filteredLoaders.add(loader);
      }
    }
    return filteredLoaders;
}

getModelLoadersForClass函数的实现为
@NonNull
private <A> List<ModelLoader<A, ?>> getModelLoadersForClass(@NonNull Class<A> modelClass) {
    //首先从缓存的cache 中获取到 modelClass对应的List<ModelLoader>集合
    List<ModelLoader<A, ?>> loaders = cache.get(modelClass);
    if (loaders == null) {
	  //如果是第一次的化，那么当然是没有缓存的了，所以会进入这里面，根据mutimodelLoaderFactory来构建一个 
      loaders = Collections.unmodifiableList(multiModelLoaderFactory.build(modelClass));
      cache.put(modelClass, loaders);
    }
    return loaders;
} 

multiModelLoaderFactory.build(modelClass)函数的实现为
@NonNull
synchronized <Model> List<ModelLoader<Model, ?>> build(@NonNull Class<Model> modelClass) {
    try {
	  //查找到的要返回的ModelLoader集合
      List<ModelLoader<Model, ?>> loaders = new ArrayList<>();
	  //遍历集合中的内容，查看哪个是合适的
      for (Entry<?, ?> entry : entries) {
        // Avoid stack overflow recursively creating model loaders by only creating loaders in
        // recursive requests if they haven't been created earlier in the chain. For example:
        // A Uri loader may translate to another model, which in turn may translate back to a Uri.
        // The original Uri loader won't be provided to the intermediate model loader, although
        // other Uri loaders will be.
        if (alreadyUsedEntries.contains(entry)) {
          continue;
        }
		//判断当前的这个entry是否能够处理这个modelClass，这里的判断为  return this.modelClass.isAssignableFrom(modelClass); 也即是简单的判断俩个class是否是一样的，或者是对应的子类
        if (entry.handles(modelClass)) {
          alreadyUsedEntries.add(entry);
		  //如果找到了进入里面，首先根据build函数根据entry获取到一个ModelLoader对象,然后添加到loaders集合中
          loaders.add(this.<Model, Object>build(entry));
          alreadyUsedEntries.remove(entry);
        }
      }
      return loaders;
    } catch (Throwable t) {
      alreadyUsedEntries.clear();
      throw t;
    }
}

this.<Model, Object>build(entry)函数的实现为
@NonNull
@SuppressWarnings("unchecked")
private <Model, Data> ModelLoader<Model, Data> build(@NonNull Entry<?, ?> entry) {
    return (ModelLoader<Model, Data>) Preconditions.checkNotNull(entry.factory.build(this));
}
可以看到他只是调用了  entry.factory.build(this)，这个factory是我们传递进来的存储的，这里举一个列子,比如在Glide中注册的时候，就有这样的形势
.append(File.class, InputStream.class, new FileLoader.StreamFactory())
那么这个factory即为new FileLoader.StreamFactory(),所以这里会执行对应的build函数，同时传递了this进来
public static class Factory<Data> implements ModelLoaderFactory<File, Data> {
    private final FileOpener<Data> opener;

    public Factory(FileOpener<Data> opener) {
      this.opener = opener;
    }

    @NonNull
    @Override
    public final ModelLoader<File, Data> build(@NonNull MultiModelLoaderFactory multiFactory) {
      return new FileLoader<>(opener);
    }

    @Override
    public final void teardown() {
      // Do nothing.
    }
  }
所以会构建一个  new FileLoader<>(opener); 对象,所以得到了一个ModelLoader对象
public class FileLoader<Data> implements ModelLoader<File, Data> {
  private static final String TAG = "FileLoader";

  private final FileOpener<Data> fileOpener;

  // Public API.
  @SuppressWarnings("WeakerAccess")
  public FileLoader(FileOpener<Data> fileOpener) {
    this.fileOpener = fileOpener;
  }
...  
}

可能会有人为什么还要传递this进来，也即是MultiModelLoaderFactory进来，这是因为可能一个model又可能对应有多个的ModelLoader对象，比如
 .append(String.class, InputStream.class, new DataUrlLoader.StreamFactory<String>())
 
当调用了  this.<Model, Object>build(entry)的时候，会执行
public static class StreamFactory implements ModelLoaderFactory<String, InputStream> {

    @NonNull
    @Override
    public ModelLoader<String, InputStream> build(MultiModelLoaderFactory multiFactory) {
      return new StringLoader<>(multiFactory.build(Uri.class, InputStream.class));
    }

    @Override
    public void teardown() {
      // Do nothing.
    }
  }  

当执行build的时候又会有这样的代码  ,
new StringLoader<>(multiFactory.build(Uri.class, InputStream.class));  

multiFactory.build(Uri.class, InputStream.class)函数的实现为
@NonNull
public synchronized <Model, Data> ModelLoader<Model, Data> build(@NonNull Class<Model> modelClass,
      @NonNull Class<Data> dataClass) {
    try {
	  //用来存储当前哪些是合法的
      List<ModelLoader<Model, Data>> loaders = new ArrayList<>();
      boolean ignoredAnyEntries = false;
      for (Entry<?, ?> entry : entries) {
        // Avoid stack overflow recursively creating model loaders by only creating loaders in
        // recursive requests if they haven't been created earlier in the chain. For example:
        // A Uri loader may translate to another model, which in turn may translate back to a Uri.
        // The original Uri loader won't be provided to the intermediate model loader, although
        // other Uri loaders will be.
        if (alreadyUsedEntries.contains(entry)) {
          ignoredAnyEntries = true;
          continue;
        }
		//判断当前的这个entry是否能够处理这个modelClass，这里的判断为  return handles(modelClass) && this.dataClass.isAssignableFrom(dataClass); 也只是简单的判断俩个类的类型都要符合
        if (entry.handles(modelClass, dataClass)) {
          alreadyUsedEntries.add(entry);
          loaders.add(this.<Model, Data>build(entry));
          alreadyUsedEntries.remove(entry);
        }
      }
	  
	  //这里还会根据得到的结果数量，构造不同的Loader,如果这里返回的loaders的数量大于1的化，则会执行factory.build(loaders, throwableListPool); 构建一个MultiModelLoader同时传递集合进去
      if (loaders.size() > 1) {
        return factory.build(loaders, throwableListPool);
      } else if (loaders.size() == 1) {
        return loaders.get(0);
      } else {
        // Avoid crashing if recursion results in no loaders available. The assertion is supposed to
        // catch completely unhandled types, recursion may mean a subtype isn't handled somewhere
        // down the stack, which is often ok.
        if (ignoredAnyEntries) {
          return emptyModelLoader();
        } else {
          throw new NoModelLoaderAvailableException(modelClass, dataClass);
        }
      }
    } catch (Throwable t) {
      alreadyUsedEntries.clear();
      throw t;
    }
  }
 }
 
factory.build(loaders, throwableListPool); 函数的实现 
static class Factory {
    @NonNull
    public <Model, Data> MultiModelLoader<Model, Data> build(
        @NonNull List<ModelLoader<Model, Data>> modelLoaders,
        @NonNull Pool<List<Throwable>> throwableListPool) {
      return new MultiModelLoader<>(modelLoaders, throwableListPool);
    }
}

所以这里传递的this只是为了转换的作用，比如当前我有String形式对应的一个处理对象，下次如果传递的是Url的时候，可以将这个URl转化成String的形势，这样就没有必要再创建一个Url对应的处理对象
这里是为复用


继续回到ResourceCacheGenetator 中的startNext函数
//然后再遍历获取的ModelLoader，分别构建对应的LoadData对象
    loadData = null;
    boolean started = false;
    while (!started && hasNextModelLoader()) {
      ModelLoader<File, ?> modelLoader = modelLoaders.get(modelLoaderIndex++);
      loadData = modelLoader.buildLoadData(cacheFile, helper.getWidth(), helper.getHeight(), helper.getOptions());
      if (loadData != null && helper.hasLoadPath(loadData.fetcher.getDataClass())) {
        //标识已经找到了，返回true,如果没有找到返回false
        started = true;
        //最后执行加载的是loaderData对象的fetcher对象来执行加载,同时传递了回调DataCallback
        loadData.fetcher.loadData(helper.getPriority(), this);
      }
    }
    return started;


 如果执行到了这里，说明找到了对应的类型来解析，这里假设	fetcher 为ByteBufferFetcher 对象，那么执行loadData即为
 loadData.fetcher.loadData(helper.getPriority(), this);	
 
private static final class ByteBufferFetcher implements DataFetcher<ByteBuffer> {

    private final File file;

    @Synthetic
    @SuppressWarnings("WeakerAccess")
    ByteBufferFetcher(File file) {
      this.file = file;
    }

    @Override
    public void loadData(@NonNull Priority priority,
        @NonNull DataCallback<? super ByteBuffer> callback) {
      ByteBuffer result;
      try {
	    //真正的从文件中获取到流
        result = ByteBufferUtil.fromFile(file);
      } catch (IOException e) {
        if (Log.isLoggable(TAG, Log.DEBUG)) {
          Log.d(TAG, "Failed to obtain ByteBuffer for file", e);
        }
        callback.onLoadFailed(e);
        return;
      }

      callback.onDataReady(result);
    }
...
}

@NonNull
  public static ByteBuffer fromFile(@NonNull File file) throws IOException {
    RandomAccessFile raf = null;
    FileChannel channel = null;
    try {
      long fileLength = file.length();
      // See #2240.
      if (fileLength > Integer.MAX_VALUE) {
        throw new IOException("File too large to map into memory");
      }
      // See b/67710449.
      if (fileLength == 0) {
        throw new IOException("File unsuitable for memory mapping");
      }

      raf = new RandomAccessFile(file, "r");
      channel = raf.getChannel();
      return channel.map(FileChannel.MapMode.READ_ONLY, 0, fileLength).load();
    } finally {
      if (channel != null) {
        try {
          channel.close();
        } catch (IOException e) {
          // Ignored.
        }
      }
      if (raf != null) {
        try {
          raf.close();
        } catch (IOException e) {
          // Ignored.
        }
      }
    }
  }
  
 至此知道了是怎么样来加载获取到数据的，其他的也是一样的

 总结，我们解析 获取一个资源的时候，会首先去内存缓存中查找，如果没有查找到则会构建一个EncoderJob，已经DecoderJob对象，在DecoderJob中会依次去ResourceCacheGenerator，
 DataCacheGenerator，中获取，对应的都为磁盘缓存，如果还是没有获取到，就会从SourceGenerator 获取，从网络上获取，代码体现为
 while (!isCancelled && currentGenerator != null && !(isStarted = currentGenerator.startNext())) {
    //如果到了这里，说明还没有找到，那么说明在磁盘缓存中没有存在，那么获取下一个状态
    stage = getNextStage(stage);
    //继续根据下一个状态，找到下一个生成器,然后继续执行while循环,如果是SOURCE状态的化，currentGenerator也已经赋值为了SourceGenerator
    currentGenerator = getNextGenerator();

    //如果状态为SOURCE
    if (stage == Stage.SOURCE) {
         //任务的执行，转到线程池里面执行,
        reschedule();
        return;
    }
  }  
```


