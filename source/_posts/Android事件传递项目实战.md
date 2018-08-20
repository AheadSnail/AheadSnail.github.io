---
layout: pager
title: Android事件传递项目实战
date: 2018-08-17 06:58:14
tags: [Android,TouchEvent]
description:  Android事件传递项目实战
---

Android事件传递项目实战
<!--more-->

****简介****
===
```
Android事件传递,好像大家都懂，无非就是事件从最外层传递进来，然后判断你是否需要拦截事件对应的也就是(onInterceptTouchEvent 事件)，如果这个事件返回了true就代表当前层要
拦截事件，那么就会将事件传递给当前控件的TouchEvent，如果在TouchEvent事件中返回了true，那么事件也就被消耗了，如果没有控件想要消耗事件，那么就会将事件一层一层的传递
给子View，如果一直都没有控件想要消耗事件，那么事件又会一层一层的传递回来，相应的触发对应的OnTouchEvent方法,

大概的事件传递过程就是这样，但是如果在项目中可能就没有那么简单了，下面就是我在帮我同学解决事件传递的过程, 首先来看需求是怎么样的
```
![结果显示](/uploads/Andorid事件传递项目实战/原型图.jpg)
```java
首先先来解释下，首先有一个刷新控件，也即是SwipeRefreshLayout,之后下面有一个对话框之类的，其实就是一个背景图，右下角有一个分享的按钮，这个按钮是要支持点击的
这个内容是相对固定的 里面的内容就是RecycleView的子条目了，当然不一定是在这个对话框的里面，如果内容多的话，下面也是要有条目的，


1.首先想到的是，能不能使用一个ReleativeLayout，里面有俩个child，一个是那个对话框的视图，另一个就是下拉刷新的控件，而且由于对话框中的分析按钮是要支持点击的。
而且从界面来看，对话框背景应该要放在下层，这样上层ListView的条目才能显示，不会被遮盖，那界面的布局如下：

<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <RelativeLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content">

        <Button
            android:id="@+id/but12"
            android:layout_width="match_parent"
            android:layout_height="80dp"
            android:text="点击"/>
    </RelativeLayout>

    <vide.m4399.com.myswiperefreshlayout.MySwipeRefreshLayout
        android:id="@+id/mySwipRefreshLayout"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:layout_centerHorizontal="true">
            <ListView
                android:id="@+id/listView"
                android:layout_width="match_parent"
                android:layout_height="match_parent"
                android:visibility="visible">
            </ListView>
    </vide.m4399.com.myswiperefreshlayout.MySwipeRefreshLayout>

</RelativeLayout>

点击按钮，并没有任何的显示 一开始怀疑是上层的ListView条目导致的，后来把ListView直接设置gone，发现还是无法点击，下面是对应的布局文件

<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <RelativeLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content">

        <Button
            android:id="@+id/but12"
            android:layout_width="match_parent"
            android:layout_height="80dp"
            android:text="点击"/>
    </RelativeLayout>

    <vide.m4399.com.myswiperefreshlayout.MySwipeRefreshLayout
        android:id="@+id/mySwipRefreshLayout"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:layout_centerHorizontal="true">
            <ListView
                android:id="@+id/listView"
                android:layout_width="match_parent"
                android:layout_height="match_parent"
                android:visibility="gone">
            </ListView>
    </vide.m4399.com.myswiperefreshlayout.MySwipeRefreshLayout>
</RelativeLayout>

那这样就有点不解了，为什么按钮接受不到事件的传递，带着这个问题去查找源码,我们知道事件的传递是从外层调用的，这里就简单的从Activity 的 dispatchTouchEvent传递，
之后传递给PhoneWindow dispatchTouchEvent，再一层一层的往里面传递，所以我们可以直接在ViewGroup 的dispatchTouchEvent 中查看而且本身PhoneWindow也是一个ViewGroup
由于这个方法的内容比较多，这里只会看关键的实现
```
![结果显示](/uploads/Andorid事件传递项目实战/viewGroup事件传递一.png)
```java
mFirstTouchTarget变量默认是为null的,那么就会往下执行 final boolean disallowIntercept =(mGroupFlags & FLAG_DISALLOW_INTERCEPT)!=0; 
这个变量的值我们可以通过设置requestDisallowInterceptTouchEvent(true);将他设置为true，这个方法代表是否需要拦截父类的事件,默认不设置的话为false，那么就会执行 
intercepted = onInterceptTouchEvent(ev); 判断当前的view是否需要拦截事件,也即是触发当前View的onInterceptTouchEvent函数，而且同时将返回值赋值给intercepted
这里先假设最外层的view在onInterceptTouchEvent 中返回了true，或者通过设置requestDisallowInterceptTouchEvent(true),将intercepted变为了true，会执行什么逻辑
```
![结果显示](/uploads/Andorid事件传递项目实战/View事件传递6.png)
```java
因为mFirstTouchTarget变量默认是为null的，所以会执行dispatchTransformedTouchEvent函数，下面是该函数的关键部分
```
![结果显示](/uploads/Andorid事件传递项目实战/View事件传递五.png)
```java
因为这里的child为空，所以会执行  handled = super.dispatchTouchEvent(transformedEvent); 这里执行的是View中对应的方法,下面是该方法关键代码
```
![结果显示](/uploads/Andorid事件传递项目实战/View事件传递7.png)
```java
通过上面可以知道，如果将intercept设置为true，就会执行自己的onTouchEvent方法，同时返回这个事件的返回值 
如果intercepted为false，代表事件不拦截,回到开头的部分，就会有下面的逻辑
```
![结果显示](/uploads/Andorid事件传递项目实战/View事件传递二.png)
```java
这里判断是否事件取消，是否拦截，如果当前事件没有取消，没有被拦截，而且这里要注意的是if的判断，这里只有事件为MotionEvent.ACTION_DOWN的时候才会往下执行
其实这里是在down事件的时候查找到对应的事件消耗对象,如果在down事件结束之后都没有找到事件的消耗者，就执行自己的onTouchEvent方法
```
![结果显示](/uploads/Andorid事件传递项目实战/View事件传递三.png)
```java
注意这里的for循环，这里是从后面的孩子索引开始遍历的
```
![结果显示](/uploads/Andorid事件传递项目实战/判断是否能接受事件.png)
```java
检查当前view是否具备了接收这个事件的前提，比如是否visibial，是否点击的点 在当前view的范围之内等，如果这些都满足，才会执行事件的传递
```
![结果显示](/uploads/Andorid事件传递项目实战/View事件传递四.png)
```java
这里会根据上面排序的孩子的情况，遍历执行 dispatchTransformedTouchEvent函数，这个函数是用来传递事件的，下面是这个函数的关键代码
```
![结果显示](/uploads/Andorid事件传递项目实战/View事件传递五.png)
```java
因为上面是遍历当前view所拥有的孩子情况，那么child肯定不为空，那么就会执行child.dispatchTouchEvent(transformedEvent);，这样就实现了传递事件,然后子View又会继续的传递
给他的子view,如果有其中的一个孩子需要接收这个事件，也即是对应的执行onTouchEvent，同时返回了true，这个dispatchTransformedTouchEvent就会返回true，那么就会给
mFirstTouchTarget变量赋值代表找到了事件的消耗者了,注意这里是针对上一层才会有赋值
```
![结果显示](/uploads/Andorid事件传递项目实战/View事件传递8.png)
![结果显示](/uploads/Andorid事件传递项目实战/View事件传递9.png)
![结果显示](/uploads/Andorid事件传递项目实战/View事件传递11.png)
```java
这里就会根据mFirstTouchTarget变量是否有完成赋值，如果没有赋值，那么就会执行自己的OnTouchEvent方法判断是否需要拦截事件，如果mFirstTouchTarget已经完成了赋值，就会执行
下面的while循环，这里为什么要有一个这样的while循环，是这样的，事件是一层一层传递下去的，如果找到事件的消耗者了，又会一层一层的往上返回，相应的
dispatchTransformedTouchEvent函数都会返回true，而我们的mFirstTouchTarget是在当前返回true的时候，赋值的，也即是说mFirstTouchTarget 代表的view不是真实的事件消耗者
这是循环调用的关系，比如有三层A B C 假设C层是一个View，而且消耗了事件，那么他的mFirstTouchTarget根本没有机会赋值，因为他没有孩子了,但是上层B的mFirstTouchTarget
就准确的记录了C接收到了这个事件，然后再往上返回，那么A的mFirstTouchTarge自然值就为B，所以当找到了事件的时候，要执行while循环，找到真正的下消耗事件的view,而且也是通过
调用dispatchTransformedTouchEvent函数 来找,当然如果这个对象就是事件消耗的对象，那么就直接返回true了，
```
如果事件不为Action_Down事件又会执行什么?
![结果显示](/uploads/Andorid事件传递项目实战/viewGroup事件传递一.png)
```java
当事件不为Action_Down事件，并且mFirstTarget为空的时候，前面已经分析过，会最终执行到OnTouchEvent函数，而如果事件不为Action_Down，但是mFirstTarget不为空的时候，就会判断
当前view是否需要拦截事件，也即是触发对应的onInterceptTouchEvent,如果不需要拦截事件，也即是intercepted 返回了false,那么此时就不会再次的触发遍历查找子view，下面是
对应的代码体现
```
![结果显示](/uploads/Andorid事件传递项目实战/View事件传递二.png)
```java
可知，对应事件的Action_Move，并不会再次的进去查找事件的消耗者，而是会执行前面所说的，根据当前mFirstTarget所代表对象，找到真正的事件消耗者
```
![结果显示](/uploads/Andorid事件传递项目实战/View事件传递总结.png)
```java
事件机制总结 这里根据mFirstTarget的情况来分别说明：

如果mFirstTarget为空
首先事件是从外层一层一层的传递的，会根据当前层是否需要拦截事件的返回值(onInterceptTouchEvent)，如果返回了true，代表需要拦截，就执行当前view的OnTouchEvent
方法，同时返回值就为当前onTouchEvent的返回值，如果onInterceptTouchEvent 返回了false，代表不需要拦截，那么就要判断是否需要遍历查找子View是否需要事件，
前提是只有在TouchEvent_Down的时候才会触发。之后会根据当前view的孩子情况，从后面开始遍历，分别执行事件传递 ,如果当前child View为 ViewGroup 
又会将事件传递下去(onInterceptTouchEvent),这里要注意的是传递事件的时候还有一个重要的前提是当前view是否具备了接收这个事件的前提，比如是否visibial，是否点击的点
在当前view的范围之内等，如果这些都满足，才会执行事件的传递,如果是View就会检查(onTouchEvent)的返回值,如果有一个孩子onTouchEvent返回了true，代表需要消耗这个事件，
那么上一层的view mFirstTouchTarget变量就赋值为当前这个需要事件的子view，之后一层一层的返回，对应的mFirstTouchTarget也相应的赋值，只是都是对应的子类而已
当找到对应的事件消耗者之后，就中断查找了,如果都没有找到，就会触发对应层的onTouchEvent方法，判断是否需要消耗这个事件

如果mFirstTarget不为空
那么代表找到了事件的消耗者了，事件也是从外层一层一层传递的，首先会判断当前层是否需要拦截事件也就是触发对应的(onInterceptTouchEvent),如果返回了true,代表需要拦截，
就执行当前view的OnTouchEvent方法，同时返回值就为当前onTouchEvent的返回值,如果onInterceptTouchEvent 返回了false，代表不需要拦截，就会根据当前mFirstTouchTarget
对象所指向的对象，找到具体的准确的需要事件的子类，这里是通过while循环实现的

```
那么我们就查找上面的问题
```java
其实前面已经提到了一点了，那就是孩子的遍历事件的时候，是从后面开始的，对应我们的布局来说，因为我们的RelativeLayout是放在前头的，那么我们接收事件的时候，就应该是
MySwipeRefreshLayout优先遍历,
```
![结果显示](/uploads/Andorid事件传递项目实战/事件接收的先后顺序.png)
```java
又因为他也是一个MySwipeRefreshLayout，又为一个ViewGroup，而且判断一个控件是否能接收事件，也是有要求的，比如界面可见，点击范围等，因为当前的listView为gone，所以
会跳过这次的事件传递，那么对于当前层也即是MySwipeRefreshLayout来说就找不到事件消耗者了，根据前面分析可只，他这个时候，就会触发自己的onTouchEvent方法
```
![结果显示](/uploads/Andorid事件传递项目实战/swipeLayoutOnTouch.png)
```java
通过上面可知，MySwipeRefreshLayout中重写了onTouchEvent方法，而且返回了true，导致事件在这一层销毁了，那么对应我们的背景那层，根本还没有机会传递，就已经终止了，所以
导致他点击没有效果
```
那如果將背景放在上面一层
```java
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <RelativeLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content">

        <Button
            android:id="@+id/but12"
            android:layout_width="match_parent"
            android:layout_height="80dp"
            android:text="点击"/>
    </RelativeLayout>

    <vide.m4399.com.myswiperefreshlayout.MySwipeRefreshLayout
        android:id="@+id/mySwipRefreshLayout"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:layout_centerHorizontal="true">
            <ListView
                android:id="@+id/listView"
                android:layout_width="match_parent"
                android:layout_height="match_parent"
                android:visibility="gone">
            </ListView>
    </vide.m4399.com.myswiperefreshlayout.MySwipeRefreshLayout>
</RelativeLayout>

这样是可以点击，但是下拉刷新的时候，下拉刷新控件，会被那个背景框挡住，因为背景跟内容不是同一层所以会这样，也就是最开始的图效果
```
尝试二 ： 那么既然不能放在同一层，那么就尝试的放在 下拉刷新的里面
```java

<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <vide.m4399.com.myswiperefreshlayout.MySwipeRefreshLayout
        android:id="@+id/mySwipRefreshLayout"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:layout_centerHorizontal="true">

        <RelativeLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content">

            <Button
                android:id="@+id/but"
                android:layout_width="match_parent"
                android:layout_height="80dp"
                android:clickable="false"
                android:text="点击"/>
        </RelativeLayout>

        <ListView
            android:id="@+id/listView"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:visibility="visible">
        </ListView>

    </vide.m4399.com.myswiperefreshlayout.MySwipeRefreshLayout>

</RelativeLayout>

```
![结果显示](/uploads/Andorid事件传递项目实战/swipeLayout只能有一个子类.jpg)
```java
可以看得出来，上面的布局只显示了一个控件，listView并没有显示出来，为什么会出现这样的问题呢，带着问题就去看MySwipeRefreshLayout的源码，这里要注意,这个类其实就是
Android提供的源码，这里只是拷贝出来，方便写点注释什么的,首先来分析onLayout，因为这个方法是负责给子view摆放位置的，既然没有显示出来，可能根本就没有摆放

//布置子view,onLayout的触发，在子布局中，当有一个成员的有发生变化的时候，就会触发这个方法，重新的布置,比如由gone 变成visible 再变成gone 都会触发这个方法
@Override
protected void onLayout(boolean changed, int left, int top, int right, int bottom)
{
    final int width = getMeasuredWidth();
    final int height = getMeasuredHeight();
    if (getChildCount() == 0)
    {
        return;
    }
    if (mTarget == null)
    {
        ensureTarget();
    }
    if (mTarget == null)
    {
        return;
    }
    final View child = mTarget;
    final int childLeft = getPaddingLeft();
    final int childTop = getPaddingTop();
    final int childWidth = width - getPaddingLeft() - getPaddingRight();
    final int childHeight = height - getPaddingTop() - getPaddingBottom();
    child.layout(childLeft, childTop, childLeft + childWidth, childTop + childHeight);
    int circleWidth = mCircleView.getMeasuredWidth();
    int circleHeight = mCircleView.getMeasuredHeight();

    //父类在布局子类的时候，就有设置了要放置的范围
    mCircleView.layout((width / 2 - circleWidth / 2), mCurrentTargetOffsetTop, (width / 2 + circleWidth / 2), mCurrentTargetOffsetTop + circleHeight);
}

可以看到上面只涉及到了俩个View的摆放，一个是mTarget代表的空间，一个是下拉刷新圆圈,那么看下mTarget 代表的是什么 从代码可知 是通过 ensureTarget();来完成赋值的

private void ensureTarget()
{
    // Don't bother getting the parent height if the parent hasn't been laid
    // out yet.
    //不要去打扰获取父类的高度，如果父类还没有布局完成的话
    if (mTarget == null)
    {
        for (int i = 0; i < getChildCount(); i++)
        {
            View child = getChildAt(i);
            if (!child.equals(mCircleView))
            {
                mTarget = child;
                break;
            }
        }
    }
}
可以看到这里通过遍历当前控件的所有子View，如果这个子View不是mCircleView ,那么就中断循环了，也就是mTarget 的值其实就是第一个child的控件，而我们上面的布局第一个
控件刚好就是那个背景框，第二个控件为ListView为第二个控件,那既然是这样，那就再包裹一层就好了

<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <vide.m4399.com.myswiperefreshlayout.MySwipeRefreshLayout
        android:id="@+id/mySwipRefreshLayout"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:layout_centerHorizontal="true">

        <RelativeLayout
            android:layout_width="match_parent"
            android:layout_height="match_parent">

            <RelativeLayout
                android:layout_width="match_parent"
                android:layout_height="wrap_content">

                <Button
                    android:id="@+id/but"
                    android:layout_width="match_parent"
                    android:layout_height="80dp"
                    android:clickable="false"
                    android:text="点击"/>
            </RelativeLayout>

            <ListView
                android:id="@+id/listView"
                android:layout_width="match_parent"
                android:layout_height="match_parent"
                android:visibility="visible">
            </ListView>

        </RelativeLayout>

    </vide.m4399.com.myswiperefreshlayout.MySwipeRefreshLayout>

</RelativeLayout>
```
![结果显示](/uploads/Andorid事件传递项目实战/swipeLayout包裹.jpg)
```java
可以看出来，通过包裹一层解决了上面的显示问题,而且点击的时候，是有反应的，但是当listView的条目内容，超过一个屏幕的时候，先滑动到底部，然后往上滑动的时候会出现
无法滑动listView，而且出现了SwipeLayout的下拉刷新的显示
```
![结果显示](/uploads/Andorid事件传递项目实战/SwipeLayout包裹出现的问题.jpg)
```java
出现这个问题，大致就是，在listView往上滑动的事件给swipeLayout消耗掉了，导致下一层的view ListView不能接受到事件，导致无法滑动,前面分析过事就件的传递流程
会从外层一层一层的传递进来，假设当前层就为MySwipeRefreshLayout ,那么根据事件传递的过程，首先会执行onInterceptTouchEvent 决定是否需要拦截事件,如果返回true
就会转而执行自己的onTouchEvent方法，这样子类将不会有机会得到这个事件，下面是这个函数的实现


//拦截触摸的事件，这样就会触发onTouchEvent事件
@Override
public boolean onInterceptTouchEvent(MotionEvent ev)
{
	ensureTarget();

    final int action = MotionEventCompat.getActionMasked(ev);

    int pointerIndex;


    if (mReturningToStart && action == MotionEvent.ACTION_DOWN)
    {
        mReturningToStart = false;
    }

    //检查是否可以消耗事件
    if (!isEnabled() || mReturningToStart || canChildScrollUp() || mRefreshing || mNestedScrollInProgress)
    {
        // Fail fast if we're not in a state where a swipe is possible
        return false;
    }

    switch (action)
    {
        //单点触摸动作 ,onInterceptTouchEvent 会接收到OnTouchDown事件，后面的OnTouchEvent不会接收到这个事件
        case MotionEvent.ACTION_DOWN:
            //手指按下的时候，将一开始从-80的位置，带到了0的位置

            setTargetOffsetTopAndBottom(mOriginalOffsetTop - mCircleView.getTop(), true);

            //获取触摸的时候，事件的手指id
            mActivePointerId = ev.getPointerId(0);
            //开始拖动的标识置为false
            mIsBeingDragged = false;

            //根据触摸的手指id，如果没有找到索引的id的话，就直接的返回
            pointerIndex = ev.findPointerIndex(mActivePointerId);
            if (pointerIndex < 0)
            {
                return false;
            }
            //根据索引的id找到手指按下的时候，y的距离
            mInitialDownY = ev.getY(pointerIndex);
        break;

        //触摸点移动动作 onInterceptTouchEvent的开始会接收到这个事件，后面都由OnTouchEvent中来接收
        case MotionEvent.ACTION_MOVE:
            //首先判断手指的id要是允许的
            if (mActivePointerId == INVALID_POINTER)
            {
                Log.e(LOG_TAG, "Got ACTION_MOVE event but don't have an active pointer id.");
                return false;
            }

            pointerIndex = ev.findPointerIndex(mActivePointerId);
            if (pointerIndex < 0)
            {
                return false;
            }
            //获取手指触摸的时候，y的距离
            final float y = ev.getY(pointerIndex);
            startDragging(y);
            break;

        //多点离开动作
        case MotionEventCompat.ACTION_POINTER_UP:
            onSecondaryPointerUp(ev);
            break;

        //单点触摸离开动作  触摸动作取消
        case MotionEvent.ACTION_UP:
        case MotionEvent.ACTION_CANCEL:
            //触摸完成，将一些值置为默认值
            mIsBeingDragged = false;
            mActivePointerId = INVALID_POINTER;
            break;
    }
    Log.d(LOG_TAG, " onInterceptTouchEvent mIsBeingDragged " + mIsBeingDragged);
    return mIsBeingDragged;
}

我们在这个函数的返回值的时候，加上了一句  Log.d(LOG_TAG, " onInterceptTouchEvent mIsBeingDragged " + mIsBeingDragged); ，当滑动到底部的时候，往上滑动
然后根据正常情况下跟当前情况的日志打印 查看是否是这个函数出现了问题

```
![结果显示](/uploads/Andorid事件传递项目实战/异常情况.png)
```java
上面的是我们当前布局情况，出现的结果，下面看看正常的情况，布局文件如下:

<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">


    <vide.m4399.com.myswiperefreshlayout.MySwipeRefreshLayout
        android:id="@+id/mySwipRefreshLayout"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:layout_centerHorizontal="true">


        <ListView
            android:id="@+id/listView"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:visibility="visible">
        </ListView>


    </vide.m4399.com.myswiperefreshlayout.MySwipeRefreshLayout>

</RelativeLayout>

```
![结果显示](/uploads/Andorid事件传递项目实战/wipeLayout正常的情况.png)
```java
可以看到后者在往上滑动的时候，并没有打印任何的日志，而前面的一种确打印了内容，所以可以知道是这个函数出现了问题,当然你也可以更多的日志，从而确定是哪里出现的问题
这里就直接说了，是因为onInterceptTouchEvent中的下面这段代码，导致的，本来要返回false的，导致没有返回，从而往下执行，而在move的事件的时候会根据滑动的距离决定
是否要显示下拉刷新的控件，

if (!isEnabled() || mReturningToStart || canChildScrollUp() || mRefreshing || mNestedScrollInProgress)
 {
    // Fail fast if we're not in a state where a swipe is possible
    return false;
 }

而且问题是出现在 canChildScrollUp() 函数,上面这段代码的意思是说，如果子类view也即是mTargetView如果可以往上滑动的时候，就应该返回false，不应该拦截,让子类去
消耗事件，下面看看这个函数的实现

//是否子view可以往下滑动
public boolean canChildScrollUp()
{
    if (mChildScrollUpCallback != null)
    {
        return mChildScrollUpCallback.canChildScrollUp(this, mTarget);
    }
    if (android.os.Build.VERSION.SDK_INT < 14)
    {
        if (mTarget instanceof AbsListView)
        {
            final AbsListView absListView = (AbsListView) mTarget;
            return absListView.getChildCount() > 0 && (absListView.getFirstVisiblePosition() > 0 || absListView.getChildAt(0).getTop() < absListView.getPaddingTop());
        }
        else
        {
            return ViewCompat.canScrollVertically(mTarget, -1) || mTarget.getScrollY() > 0;
        }
    }
    else
    {
        return ViewCompat.canScrollVertically(mTarget, -1);
    }
}
 
可以看出来会执行    return ViewCompat.canScrollVertically(mTarget, -1);下面是这个函数的实现
@Override
public boolean canScrollVertically(View v, int direction) {
    return (v instanceof ScrollingView) && canScrollingViewScrollVertically((ScrollingView) v, direction);
}

可以看出来这里会根据mTarget是否为ScrollingView,如果不是就直接返回false，那么就不会返回这个这个事件，导致在move的时候，出现了误判，所以我们可以简单的重写这个方法
将用这个mTarget判断的地方，用mTarget中子类ListView 来判断，就可以简单的解决这个问题了，

mSwipeRefreshLaout.setOnChildScrollUpCallback(new MySwipeRefreshLayout.OnChildScrollUpCallback()
{
    @Override
    public boolean canChildScrollUp(MySwipeRefreshLayout parent, @Nullable View mTarget)
    {
        if (android.os.Build.VERSION.SDK_INT < 14)
        {
            if (mTarget instanceof AbsListView)
            {
                final AbsListView absListView = (AbsListView) mTarget;
                return absListView.getChildCount() > 0 && (absListView.getFirstVisiblePosition() > 0 || absListView.getChildAt(0).getTop() < absListView.getPaddingTop());
            }
            else
            {
                return ViewCompat.canScrollVertically(mTarget, -1) || mTarget.getScrollY() > 0;
            }
        }
        else
        {
            if(mTarget instanceof RelativeLayout)
            {
                RelativeLayout relativeLayout = (RelativeLayout) mTarget;
                for(int i = 0;i<relativeLayout.getChildCount();i++)
                {
                    View childView = relativeLayout.getChildAt(i);
                    if(childView instanceof ListView)
                    {
                         return ViewCompat.canScrollVertically(childView, -1);
                    }
                }
            }
            return ViewCompat.canScrollVertically(mTarget, -1);
        }
    }
});

上面的这个监听setOnChildScrollUpCallback 是本身就提供的，触发的时机是在下面有体现，如果有设置监听的话会执行监听
//是否子view可以往下滑动
public boolean canChildScrollUp()
{
    if (mChildScrollUpCallback != null)
    {
        return mChildScrollUpCallback.canChildScrollUp(this, mTarget);
    }
	....
}
```
![结果显示](/uploads/Andorid事件传递项目实战/修改之后的内容.gif)