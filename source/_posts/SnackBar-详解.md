---
title: SnackBar 详解
date: 2017-09-26 23:57:09
tags: [Android,design,SnackBar]
description: SnackBar 使用详解，及代码分析
---

SnackBar 使用详解，及代码分析
<!--more-->

**SnackBar使用的好处**
===
 - dialog  笨拙  体验大打折扣，会阻断用户的连段性 ，交互性太强   
 - Toast   没有交互性  （只能交互 ）
 - SnackBar可以自动消失，也可以手动取消, 而Toast虽然可以手动取消, 但是无法一直显示
 - Snackbar类似dialog, 生命周期跟随当前Activity,  FrameLayout.addView()
 - 显示在最上层  没用控件可以覆盖它

 
**SnackBar的简单的使用**
===
```java
 //第一个参数要传递一个View，第二个参数为要显示的文字，第三个参数为显示的时间
 //第三个参数为要显示的样式，如果为LENGTH_LONG,LENGTH_SHOT会自动的消失，如果为LENGTH_INDEFINITE不会自动的消失(不点击的时候)
 Snackbar.make(mBtn, "厦门下雨了", Snackbar.LENGTH_INDEFINITE)
                        //可以设置点击的时候，动作的执行
                        .setAction("确定", new View.OnClickListener()
                        {
                            @Override
                            public void onClick(View view)
                            {
                                Toast.makeText(MainActivity.this,"确定了",Toast.LENGTH_SHORT).show();
                            }
                        })
                        //可以监听显示，消失的动作
                        .setCallback(new Snackbar.Callback()
                        {
                            @Override
                            public void onDismissed(Snackbar transientBottomBar, int event)
                            {
                                time=System.currentTimeMillis()-time;
                                Toast.makeText(MainActivity.this,"  时间" +time,Toast.LENGTH_SHORT).show();
                                super.onDismissed(transientBottomBar, event);
                            }

                            @Override
                            public void onShown(Snackbar sb)
                            {
                                time=System.currentTimeMillis();
                                super.onShown(sb);
                            }
                        })
                        .setActionTextColor(Color.BLUE)//设置action的颜色
                        .show();//最终一定要调用这个方法，要不然是不会显示的
```
**运行结果**
===
![结果显示](/uploads/snackBar.png)

当SnackBar的显示方式设置为LENGTH_INDEFINITE不会自动的消失,而设置的时间为LENGTH_LONG,LENGTH_SHOT会自动的消失源码体现:
```java
首先我们调用show的时候,这里假设都为第一次使用
 public void show(int duration, Callback callback) {
        synchronized (mLock) {
            if (isCurrentSnackbarLocked(callback)) {
                // Means that the callback is already in the queue. We'll just update the duration
                mCurrentSnackbar.duration = duration;

                // If this is the Snackbar currently being shown, call re-schedule it's
                // timeout
                mHandler.removeCallbacksAndMessages(mCurrentSnackbar);
                scheduleTimeoutLocked(mCurrentSnackbar);
                return;
            } else if (isNextSnackbarLocked(callback)) {
                // We'll just update the duration
                mNextSnackbar.duration = duration;
            } else {
                // Else, we need to create a new record and queue it
				因为我们是第一次进来，所以我们会走到这里
                mNextSnackbar = new SnackbarRecord(duration, callback);
            }

            if (mCurrentSnackbar != null && cancelSnackbarLocked(mCurrentSnackbar,
                    Snackbar.Callback.DISMISS_EVENT_CONSECUTIVE)) {
                // If we currently have a Snackbar, try and cancel it and wait in line
                return;
            } else {
                // Clear out the current snackbar
                mCurrentSnackbar = null;
                // Otherwise, just show it now
				然后走到这里
                showNextSnackbarLocked();
            }
        }
    }
	
	isCurrentSnackbarLocked(callback) 源码实现为：这里是通过判断当前的callback跟mCurrentSnackbar中设置的callback是否一样，如果是一样就表示为当前的callback
	private boolean isCurrentSnackbarLocked(Callback callback) {
        return mCurrentSnackbar != null && mCurrentSnackbar.isSnackbar(callback);
    }
	isNextSnackbarLocked(callback) 源码实现为 这里是通过判断当前的callback跟mNextSnackbar中设置的callback是否一样，如果是一样就表示为下一个
    private boolean isNextSnackbarLocked(Callback callback) {
        return mNextSnackbar != null && mNextSnackbar.isSnackbar(callback);
    }

	showNextSnackbarLocked();
	private void showNextSnackbarLocked() {
        if (mNextSnackbar != null) {
            mCurrentSnackbar = mNextSnackbar;
            mNextSnackbar = null;

            final Callback callback = mCurrentSnackbar.callback.get();
            if (callback != null) {
                callback.show();
            } else {
                // The callback doesn't exist any more, clear out the Snackbar
                mCurrentSnackbar = null;
            }
        }
    }
	然后通过callback的回答，通过handler的接受显示的消息，然后执行showview的方法
	final void showView() {
		  ....
		  mTargetParent.addView(mView);将当前的SnackBar的内容，添加到跟布局里面
		  判断是否允许执行动画，也就是默认的从上往下显示SnackBar的时候动画
		  if (ViewCompat.isLaidOut(mView)) {
            if (shouldAnimate()) {
                // If animations are enabled, animate it in
                animateViewIn();
            } else {
                // Else if anims are disabled just call back now
                onViewShown();
            }
	}
	void animateViewIn() {
        if (Build.VERSION.SDK_INT >= 12) {
            final int viewHeight = mView.getHeight();
            if (USE_OFFSET_API) {
                ViewCompat.offsetTopAndBottom(mView, viewHeight);
            } else {
                mView.setTranslationY(viewHeight);
            }
            final ValueAnimator animator = new ValueAnimator();
            animator.setIntValues(viewHeight, 0);
            animator.setInterpolator(FAST_OUT_SLOW_IN_INTERPOLATOR);
            animator.setDuration(ANIMATION_DURATION);
            animator.addListener(new AnimatorListenerAdapter() {
                @Override
                public void onAnimationStart(Animator animator) {
                    mContentViewCallback.animateContentIn(
                            ANIMATION_DURATION - ANIMATION_FADE_DURATION,
                            ANIMATION_FADE_DURATION);
                }

                @Override
                public void onAnimationEnd(Animator animator) {
                    onViewShown(); 当动画结束的时候
                }
            });
	}
	void onViewShown() {
    SnackbarManager.getInstance().onShown(mManagerCallback);
		if (mCallbacks != null) {
            // Notify the callbacks. Do that from the end of the list so that if a callback
            // removes itself as the result of being called, it won't mess up with our iteration
            int callbackCount = mCallbacks.size();
            for (int i = callbackCount - 1; i >= 0; i--) {
                mCallbacks.get(i).onShown((B) this);
            }
        }
    }
	/**
     * Should be called when a Snackbar is being shown. This is after any entrance animation has
     * finished.
     */
    public void onShown(Callback callback) {
        synchronized (mLock) {
            if (isCurrentSnackbarLocked(callback)) {
                scheduleTimeoutLocked(mCurrentSnackbar);
            }
        }
    }
	下面的这里就是为什么会消失和不会消失的源码体现了
	private void scheduleTimeoutLocked(SnackbarRecord r) {
        if (r.duration == Snackbar.LENGTH_INDEFINITE) { 如果方式设置的为Snackbar.LENGTH_INDEFINITE就直接return，所以会一直显示
            // If we're set to indefinite, we don't want to set a timeout
            return;
        }
		
		如果方式不为Snackbar.LENGTH_INDEFINITE,通过handler,setDeleyedMessage来做到隔多久消失
        int durationMs = LONG_DURATION_MS;
        if (r.duration > 0) {
            durationMs = r.duration;
        } else if (r.duration == Snackbar.LENGTH_SHORT) {
            durationMs = SHORT_DURATION_MS;
        }
        mHandler.removeCallbacksAndMessages(r);
        mHandler.sendMessageDelayed(Message.obtain(mHandler, MSG_TIMEOUT, r), durationMs);
    }
```

当SnackBar的显示方式设置为LENGTH_INDEFINITE不会自动的消失(不点击的时候),或者再创建一个SnackBar的时候，之前的那个就会消失掉
.setAction("确定", new View.OnClickListener()源码的实现方式为
```java
 /**
     * Set the action to be displayed in this {@link BaseTransientBottomBar}.
     *
     * @param text     Text to display for the action
     * @param listener callback to be invoked when the action is clicked
     */
    @NonNull
    public Snackbar setAction(CharSequence text, final View.OnClickListener listener) {
        final SnackbarContentLayout contentLayout = (SnackbarContentLayout) mView.getChildAt(0);
        final TextView tv = contentLayout.getActionView();

        if (TextUtils.isEmpty(text) || listener == null) {
            tv.setVisibility(View.GONE);
            tv.setOnClickListener(null);
        } else {
			我们的listener不为空
            tv.setVisibility(View.VISIBLE);
            tv.setText(text);
            tv.setOnClickListener(new View.OnClickListener() {
                @Override
                public void onClick(View view) {
                    listener.onClick(view);
                    // Now dismiss the Snackbar
                    dispatchDismiss(BaseCallback.DISMISS_EVENT_ACTION);
                }
            });
        }
        return this;
    }

	dispatchDismiss(BaseCallback.DISMISS_EVENT_ACTION);源码实现
	void dispatchDismiss(@BaseCallback.DismissEvent int event) {
        SnackbarManager.getInstance().dismiss(mManagerCallback, event);
    }
	SnackbarManager.getInstance().dismiss(mManagerCallback, event);源码实现
	 public void dismiss(Callback callback, int event) {
        synchronized (mLock) {
            if (isCurrentSnackbarLocked(callback)) { //isCurrentSnackbarLocked（）会比较当前显示的SnackBar设置的callback是否跟传递的callback是否一样,当前因为只有一个snackBar
                cancelSnackbarLocked(mCurrentSnackbar, event);
            } else if (isNextSnackbarLocked(callback)) {
                cancelSnackbarLocked(mNextSnackbar, event);
            }
        }
    }
	 cancelSnackbarLocked(mCurrentSnackbar, event);源码实现为
	 private boolean cancelSnackbarLocked(SnackbarRecord record, int event) {
		取出当前的callba
        final Callback callback = record.callback.get();
        if (callback != null) {
            // Make sure we remove any timeouts for the SnackbarRecord
            mHandler.removeCallbacksAndMessages(record);
            callback.dismiss(event);
            return true;
        }
        return false;
    }
	 callback.dismiss(event);源码实现
	   @Override
        public void dismiss(int event) {
            sHandler.sendMessage(sHandler.obtainMessage(MSG_DISMISS, event, 0,
                    BaseTransientBottomBar.this));
        }
		handler接受到消息
		se MSG_DISMISS:
          ((BaseTransientBottomBar) message.obj).hideView(message.arg1);
       return true;
	   最终执行了下面的关键代码，将当前的mView从根布局里面移除出去
	   if (Build.VERSION.SDK_INT < 11) {
            // We need to hide the Snackbar on pre-v11 since it uses an old style Animation.
            // ViewGroup has special handling in removeView() when getAnimation() != null in
            // that it waits. This then means that the calculated insets are wrong and the
            // any dodging views do not return. We workaround it by setting the view to gone while
            // ViewGroup actually gets around to removing it.
            mView.setVisibility(View.GONE);
        }
        // Lastly, hide and remove the view from the parent (if attached)
        final ViewParent parent = mView.getParent();
        if (parent instanceof ViewGroup) {
            ((ViewGroup) parent).removeView(mView);
        }
```

```
要想改变SnackBar里面左边显示的大小，跟颜色，可以通过下面的方式来改变
View view1=snackbar.getView();
TextView textView= (TextView) view1.findViewById(R.id.snackbar_text);
默认的情况下使用SnackBar的弹出样式只会在底部弹出,(如果包含了CoordinatorLayout就可以做到弹出位置的改变,具体看下面)
```

**SnackBar源码分析**
===
```java
当我们最简单的调用Snackbar.make(mBtn, "厦门下雨了", Snackbar.LENGTH_SHORT).show();的时候，先看make的实现
 @NonNull
 public static Snackbar make(@NonNull View view, @NonNull CharSequence text,
            @Duration int duration) {
		关键的方法，找到可以适用的Parent
        final ViewGroup parent = findSuitableParent(view);
		如果没有找到，抛出异常
        if (parent == null) {
            throw new IllegalArgumentException("No suitable parent found from the given view. "
                    + "Please provide a valid view.");
        }

		加载布局,当作一个content，添加到上面已经找的Parent里面
        final LayoutInflater inflater = LayoutInflater.from(parent.getContext());
        final SnackbarContentLayout content =
                (SnackbarContentLayout) inflater.inflate(
                        R.layout.design_layout_snackbar_include, parent, false);
						
        final Snackbar snackbar = new Snackbar(parent, content, content);
        snackbar.setText(text);
        snackbar.setDuration(duration);
        return snackbar;
}

 final ViewGroup parent = findSuitableParent(view);源码的实现
 private static ViewGroup findSuitableParent(View view) {
        ViewGroup fallback = null;
        do {
		    如果父布局中有CoordinatorLayout，就会找到这个布局做为parent，所以SnackBar就会添加到这里面，所以这个可以改变SnackBar弹出位置的关键实现
            if (view instanceof CoordinatorLayout) { 
                // We've found a CoordinatorLayout, use it
                return (ViewGroup) view;
            } else if (view instanceof FrameLayout) { 
				默认的情况下，我们没有使用CoordinatorLayout的话，就会找到我们的跟布局也就是DecorView，DecorView本身是继承了FrameLayout,android.R.id.content 也就为FrameLayout跟布局
                if (view.getId() == android.R.id.content) {
                    // If we've hit the decor content view, then we didn't find a CoL in the
                    // hierarchy, so use it.
                    return (ViewGroup) view; 找到跟布局将当前的view返回
                } else {
                    // It's not the content view but we'll use it as our fallback
                    fallback = (ViewGroup) view;
                }
            }

            if (view != null) {
                // Else, we will loop and crawl up the view hierarchy and try to find a parent
				这边遍历查找，如果是view的话，就找到parent，
                final ViewParent parent = view.getParent();  
                view = parent instanceof View ? (View) parent : null;
            }
        } while (view != null);

        // If we reach here then we didn't find a CoL or a suitable content view so we'll fallback
        return fallback;
}
```
**findSuitableParent 原理为**
![SnackBar原理](/uploads/SnackBar原理.png)
	
R.layout.design_layout_snackbar_include布局为。可以看到就是一个SnackbarContentLayout，下面的俩个就是显示一个在左边，一个在右边的Action控件
```java
<view
    xmlns:android="http://schemas.android.com/apk/res/android"
    class="android.support.design.internal.SnackbarContentLayout"
    android:theme="@style/ThemeOverlay.AppCompat.Dark"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:layout_gravity="bottom">

    <TextView
        android:id="@+id/snackbar_text"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_weight="1"
        android:paddingTop="@dimen/design_snackbar_padding_vertical"
        android:paddingBottom="@dimen/design_snackbar_padding_vertical"
        android:paddingLeft="@dimen/design_snackbar_padding_horizontal"
        android:paddingRight="@dimen/design_snackbar_padding_horizontal"
        android:textAppearance="@style/TextAppearance.Design.Snackbar.Message"
        android:maxLines="@integer/design_snackbar_text_max_lines"
        android:layout_gravity="center_vertical|left|start"
        android:ellipsize="end"
        android:textAlignment="viewStart"/>

    <Button
        android:id="@+id/snackbar_action"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginLeft="@dimen/design_snackbar_extra_spacing_horizontal"
        android:layout_marginStart="@dimen/design_snackbar_extra_spacing_horizontal"
        android:layout_gravity="center_vertical|right|end"
        android:minWidth="48dp"
        android:visibility="gone"
        android:textColor="?attr/colorAccent"
        style="?attr/borderlessButtonStyle"/>

</view>
```
	
当执行到这里的时候，传递Parnent,content

```java
final Snackbar snackbar = new Snackbar(parent, content, content);
private Snackbar(ViewGroup parent, View content, ContentViewCallback contentViewCallback) {
        super(parent, content, contentViewCallback);
    }
Snackbar的父类为BaseTransientBottomBar
protected BaseTransientBottomBar(@NonNull ViewGroup parent, @NonNull View content,
            @NonNull ContentViewCallback contentViewCallback) {
        if (parent == null) {
            throw new IllegalArgumentException("Transient bottom bar must have non-null parent");
        }
        if (content == null) {
            throw new IllegalArgumentException("Transient bottom bar must have non-null content");
        }
        if (contentViewCallback == null) {
            throw new IllegalArgumentException("Transient bottom bar must have non-null callback");
        }
		
		赋值操作
        mTargetParent = parent;
        mContentViewCallback = contentViewCallback;
        mContext = parent.getContext();

        ThemeUtils.checkAppCompatTheme(mContext);

        LayoutInflater inflater = LayoutInflater.from(mContext);
        // Note that for backwards compatibility reasons we inflate a layout that is defined
        // in the extending Snackbar class. This is to prevent breakage of apps that have custom
        // coordinator layout behaviors that depend on that layout.
		再次的加载一个布局
        mView = (SnackbarBaseLayout) inflater.inflate(
                R.layout.design_layout_snackbar, mTargetParent, false);
		再将内容添加到了mView里面
        mView.addView(content);
		.......
}
```

R.layout.design_layout_snackbar 布局，其实就是一个SnackbarLayout，控件
```java
<view xmlns:android="http://schemas.android.com/apk/res/android"
      class="android.support.design.widget.Snackbar$SnackbarLayout"
      android:layout_width="wrap_content"
      android:layout_height="wrap_content"
      android:layout_gravity="bottom|center_horizontal"
      android:theme="@style/ThemeOverlay.AppCompat.Dark"
      style="@style/Widget.Design.Snackbar" />

并且初始化了俩个全剧变量，在Snackbar的父类为BaseTransientBottomBar
final SnackbarManager.Callback mManagerCallback = new SnackbarManager.Callback() {
        @Override
        public void show() {
            sHandler.sendMessage(sHandler.obtainMessage(MSG_SHOW, BaseTransientBottomBar.this));
        }

        @Override
        public void dismiss(int event) {
            sHandler.sendMessage(sHandler.obtainMessage(MSG_DISMISS, event, 0,
                    BaseTransientBottomBar.this));
        }
 };
 static {
        sHandler = new Handler(Looper.getMainLooper(), new Handler.Callback() {
            @Override
            public boolean handleMessage(Message message) {
                switch (message.what) {
                    case MSG_SHOW:
                        ((BaseTransientBottomBar) message.obj).showView();
                        return true;
                    case MSG_DISMISS:
                        ((BaseTransientBottomBar) message.obj).hideView(message.arg1);
                        return true;
                }
                return false;
            }
        });
    }
```

当我们最简单的调用show();的时候，就会调用到BaseTransientBottomBar中的对应的方法 ,mManagerCallback为make的时候创建的全局变量

```java
/**
* Show the {@link BaseTransientBottomBar}.
*/
  public void show() {
     SnackbarManager.getInstance().show(mDuration, mManagerCallback);
  }
	
	public void show(int duration, Callback callback) {
        synchronized (mLock) {
            if (isCurrentSnackbarLocked(callback)) {
                // Means that the callback is already in the queue. We'll just update the duration
                mCurrentSnackbar.duration = duration;

                // If this is the Snackbar currently being shown, call re-schedule it's
                // timeout
                mHandler.removeCallbacksAndMessages(mCurrentSnackbar);
                scheduleTimeoutLocked(mCurrentSnackbar);
                return;
            } else if (isNextSnackbarLocked(callback)) {
                // We'll just update the duration
                mNextSnackbar.duration = duration;
            } else {
                // Else, we need to create a new record and queue it
				第一次执行的时候会先执行到这里,创建一个Snack的记录，并且给mNextSnackbar变量赋值
                mNextSnackbar = new SnackbarRecord(duration, callback);
            }

            if (mCurrentSnackbar != null && cancelSnackbarLocked(mCurrentSnackbar,
                    Snackbar.Callback.DISMISS_EVENT_CONSECUTIVE)) {
                // If we currently have a Snackbar, try and cancel it and wait in line
                return;
            } else {
				首先将mCurrentSnackbar = null;
                // Clear out the current snackbar
                mCurrentSnackbar = null;
                // Otherwise, just show it now
                showNextSnackbarLocked();
            }
        }
    }
```
	
mNextSnackbar = new SnackbarRecord(duration, callback);中实现为

```java
private static class SnackbarRecord {
    final WeakReference<Callback> callback;
    int duration;
    boolean paused;

    SnackbarRecord(int duration, Callback callback) {
            this.callback = new WeakReference<>(callback);
            this.duration = duration;
    }

    boolean isSnackbar(Callback callback) {
        return callback != null && this.callback.get() == callback;
    }
}
```
	
showNextSnackbarLocked();源码的实现为
```java
private void showNextSnackbarLocked() {
	//上面有给这个变量赋值 mNextSnackbar = new SnackbarRecord(duration, callback);
    if (mNextSnackbar != null) {
        mCurrentSnackbar = mNextSnackbar;
        mNextSnackbar = null;
			
		这边取出final WeakReference<Callback> callback;里面的内容，也就是mManagerCallback
        final Callback callback = mCurrentSnackbar.callback.get();
        if (callback != null) {
		    最终执行show的回调
            callback.show();
        } else {
            // The callback doesn't exist any more, clear out the Snackbar
             mCurrentSnackbar = null;
        }
    }
}
```

callback.show(); 实现也就是调用到了 mManagerCallback中的show回调
```java
	@Override
    public void show() {
	    通过handler来发送消息
        sHandler.sendMessage(sHandler.obtainMessage(MSG_SHOW, BaseTransientBottomBar.this));
    }
		
		case MSG_SHOW:
              ((BaseTransientBottomBar) message.obj).showView();
        return true;
		
	 ((BaseTransientBottomBar) message.obj).showView(); 中关键的代码	
      final void showView() {
		这样就将创建的view，添加到了跟布局里面，这样就可以显示在所有的布局的上面
		mTargetParent.addView(mView);
	}
```

**上面的流程图为**
===
![SnackBar时序图](/uploads/SnackBar时序图.png)

**总结**
===
我们可以使用这种方式实现全局的悬浮按钮。。。



