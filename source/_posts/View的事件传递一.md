---
title: View的事件传递一
date: 2017-09-18 23:51:56
tags: [Android,View,TouchEvent]
description: 属性动画源码解析
---

View的事件传递的源码解析，也对一些疑问做了下解答
<!--more-->



# View的事件传递的源码解析（一）
#### 1.一个view里面如果同时设置了点击的事件，跟onTouch事件，如果onTouch事件返回的为true的话，为什么点击的事件会没有响应？

```
在View的dispatchTouchEvent中有下面的代码
public boolean dispatchTouchEvent(MotionEvent event) {
    boolean result = false; 
.....
    if (onFilterTouchEventForSecurity(event)) {
        if ((mViewFlags & ENABLED_MASK) == ENABLED && handleScrollBarDragging(event)) {
            result = true;
        }
        //noinspection SimplifiableIfStatement
        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnTouchListener != null
                && (mViewFlags & ENABLED_MASK) == ENABLED
                && li.mOnTouchListener.onTouch(this, event)) {这个就为你在代码中设置的onTouch事件，获取到返回值，如果返回的为true的话，result就为true
            result = true;
        }
  下面的是一个&&操作符符号，如果第一个为false，后面的就不会调用
        if (!result && onTouchEvent(event)) {
            result = true;
        }
    }
......
    return result;
```

}
####在onTouchEvent中有这样的代码
public boolean onTouchEvent(MotionEvent event)#  {

```
    final float x = event.getX();
    final float y = event.getY();
    final int viewFlags = mViewFlags;
    final int action = event.getAction();
  ......

    if (((viewFlags & CLICKABLE) == CLICKABLE ||
            (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE) ||
            (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE) {
        switch (action) {
            case MotionEvent.ACTION_UP:
                boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
                if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
                    // take focus if we don't have it already and we should in
                    // touch mode.
                    boolean focusTaken = false;
                    if (isFocusable() && isFocusableInTouchMode() && !isFocused()) {
                        focusTaken = requestFocus();
                    }

                    if (prepressed) {
                        // The button is being released before we actually
                        // showed it as pressed.  Make it show the pressed
                        // state now (before scheduling the click) to ensure
                        // the user sees it.
                        setPressed(true, x, y);
                   }

                    if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
                        // This is a tap, so remove the longpress check
                        removeLongPressCallback();

                        // Only perform take click actions if we were in the pressed state
                        if (!focusTaken) {
                            // Use a Runnable and post this rather than calling
                            // performClick directly. This lets other visual state
                            // of the view update before click actions start.
							也就是不管你怎么弄，下面的performClick都会被调用到
                            if (mPerformClick == null) {
                                mPerformClick = new PerformClick();
                            }
                            if (!post(mPerformClick)) {
                                performClick();
                            }
                        }
                    }
}
```
#### 当执行到 performClick();的时候
```
public boolean performClick() {
    final boolean result;
    final ListenerInfo li = mListenerInfo;
    if (li != null && li.mOnClickListener != null) {
        playSoundEffect(SoundEffectConstants.CLICK);
        li.mOnClickListener.onClick(this);这边就会回调回去执行到在代码中设置的onClick回调
        result = true;
    } else {
        result = false;
    }

    sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_CLICKED);
    return result;
}
```
