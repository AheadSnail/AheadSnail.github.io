---
title: View的事件传递二
date: 2017-09-18 23:45:55
tags: [Android,View,TouchEvent]
description: 属性动画源码解析
---
### 简介
View的事件传递的源码解析
我们就从Activity的中的下面的来触发，方法都为public的，说明是给别人调用的，我们就从这里出发
```java
public boolean dispatchTouchEvent(MotionEvent ev) {
    if (ev.getAction() == MotionEvent.ACTION_DOWN) {
        onUserInteraction();
    }
    if (getWindow().superDispatchTouchEvent(ev)) {
        return true;
    }
    return onTouchEvent(ev); 所以如果getWindow().superDispatchTouchEvent(ev)中有返回true的话，就不会执行到Activity中的onTouch事件
}
```

**要注意这边的getWindow()获取到的是phoneWindow的实例，只有一个孩子就是phoneWindow，所以就会调用到里面的superDispatchTouchEvent(ev)**
===

```java
@Override
public boolean superDispatchTouchEvent(MotionEvent event) {
    return mDecor.superDispatchTouchEvent(event); 这里的mDecor为 DecorView，所以就会执行到对应的方法里面
}
DecorView不做任何的处理，他是继承了FrameLayout的，FameLayout也不做处理，就到了ViewGroup里面了
public boolean superDispatchTrackballEvent(MotionEvent event) {
    return super.dispatchTrackballEvent(event);
}
ViewGroup中的实现为
@Override
public boolean dispatchTouchEvent(MotionEvent ev) {
	判断是否有其他的输入设备，比如手写笔
    if (mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onTouchEvent(ev, 1);
    }
    // If the event targets the accessibility focused view and this is it, start
    // normal event dispatch. Maybe a descendant is what will handle the click.
	判断是否开启了Accessibility，残疾人服务
    if (ev.isTargetAccessibilityFocus() && isAccessibilityFocusedViewOrHost()) {
        ev.setTargetAccessibilityFocus(false);
    }
   /** 第一步:对于ACTION_DOWN进行处理(Handle an initial down)
       因为ACTION_DOWN是一系列事件的开端,当是ACTION_DOWN时进行一些初始化操作.
       从源码的注释也可以看出来:清除以往的Touch状态(state)开始新的手势(gesture)
       cancelAndClearTouchTargets(ev)中有一个非常重要的操作:
       将mFirstTouchTarget设置为null!!!!   mFirstTouchTarget = null;
       随后在resetTouchState()中重置Touch状态标识
    * */
    boolean handled = false;
    if (onFilterTouchEventForSecurity(ev)) { 检查手势的安全，一般都为true
        final int action = ev.getAction();
        final int actionMasked = action & MotionEvent.ACTION_MASK;

        // Handle an initial down.
        if (actionMasked == MotionEvent.ACTION_DOWN) {
            // Throw away all previous state when starting a new touch gesture.
            // The framework may have dropped the up or cancel event for the previous gesture
            // due to an app switch, ANR, or some other state change.
            cancelAndClearTouchTargets(ev);   mFirstTouchTarget = null;
            resetTouchState();
        }
```
**Check for interception. 是否拦截**
===
```java
* 第二步:检查是否要拦截(Check for interception)
* 在dispatchTouchEvent(MotionEventev)这段代码中
   * 使用变量intercepted来标记ViewGroup是否拦截Touch事件的传递.
* 该变量在后续代码中起着很重要的作用.
* 拦截    intercepted =true
*/
        // Check for interception.
        final boolean intercepted;
        if (actionMasked == MotionEvent.ACTION_DOW || mFirstTouchTarget != null) {
		下面的这个设置是我们在代码中getParent().requestDisallowInterceptTouchEvent(true);中有这样的实现
		if (disallowIntercept) {mGroupFlags |= FLAG_DISALLOW_INTERCEPT;} else { mGroupFlags &= ~FLAG_DISALLOW_INTERCEPT;}，所以下面的就是用来检查是否有设置拦截的
            final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
            如果不设置的话，默认都为false的   
		if (!disallowIntercept) {
                intercepted = onInterceptTouchEvent(ev); 就会回调执行到decorView中的这个方法，默认为false的
				当然如果这个为true的话，代表拦截
				ev.setAction(action); // restore action in case it was changed
            } else {
                intercepted = false;
            }
        } else {
            // There are no touch targets and this action is not an initial down
            // so this view group continues to intercept touches.
            intercepted = true;
        }

        // If intercepted, start normal event dispatch. Also if there is already
        // a view that is handling the gesture, do normal event dispatch.
        if (intercepted || mFirstTouchTarget != null) { 如果当前是拦截了或者mFirstTouchTarget != null
            ev.setTargetAccessibilityFocus(false);
        }

  	    /**
         * 第三步:检查cancel(Check for cancelation)
        *
        */
        // Check for cancelation. 得到这个事件是否被取消了
        final boolean canceled = resetCancelNextUpFlag(this) || actionMasked == MotionEvent.ACTION_CANCEL;

        // Update list of touch targets for pointer down, if needed.
        final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
        TouchTarget newTouchTarget = null;
        boolean alreadyDispatchedToNewTouchTarget = false;
		如果当前的事件没有被取消，并且没有被拦截的话
 		/**
		  * 第四步:事件分发(Update list of touch targets for pointer down, if needed)
		*/
        不是ACTION_CANCEL并且ViewGroup的拦截标志位intercepted为false(不拦截)
        if (!canceled && !intercepted) {
            // If the event is targeting accessiiblity focus we give it to the
            // view that has accessibility focus and if it does not handle it
            // we clear the flag and dispatch the event to all children as usual.
            // We are looking up the accessibility focused host to avoid keeping
            // state since these events are very rare.
            View childWithAccessibilityFocus = ev.isTargetAccessibilityFocus()
                    ? findChildWithAccessibilityFocus() : null;

			第二次 move  触发时  压根就不会遍历子控件
            if (actionMasked == MotionEvent.ACTION_DOWN
				|| (split && actionMasked == 	MotionEvent.ACTION_POINTER_DOWN)
                    || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {

                final int actionIndex = ev.getActionIndex(); // always 0 for down
                final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
                        : TouchTarget.ALL_POINTER_IDS;

                // Clean up earlier touch targets for this pointer id in case they
                // have become out of sync.
                removePointersFromTouchTargets(idBitsToAssign);
				得到当前孩子的个数
                final int childrenCount = mChildrenCount;
				如果当前没有的newTouchTarget 为空，并且 孩子的个数不为0，默认开始的时候，newTouchTarger为空
                if (newTouchTarget == null && childrenCount != 0) {
                    获取到按下的坐标点
					final float x = ev.getX(actionIndex);
                    final float y = ev.getY(actionIndex);
                    // Find a child that can receive the event.
                    // Scan children from front to back.
					重排序 根据绘制的先后顺序重新的排序，后面绘制的一般排在前面
                    final ArrayList<View> preorderedList = buildTouchDispatchChildList();
					是否是自己排序，默认为false
                    final boolean customOrder = preorderedList == null && isChildrenDrawingOrderEnabled();
                    final View[] children = mChildren;
					事件分发down遍历子控件的过程
                    for (int i = childrenCount - 1; i >= 0; i--) {
					从上面排序后的集合里面得到索引
                        final int childIndex = getAndVerifyPreorderedIndex(
                                childrenCount, i, customOrder);
						在从上面排序后的集合里面得到孩子的view
                        final View child = getAndVerifyPreorderedView(
                                preorderedList, children, childIndex);

                        // If there is a view that has accessibility focus we want it
                        // to get the event first and if not handled we will perform a
                        // normal dispatch. We may do a double iteration but this is
                        // safer given the timeframe.
                        if (childWithAccessibilityFocus != null) {
                            if (childWithAccessibilityFocus != child) {
                                continue;
                            }
                            childWithAccessibilityFocus = null;
                            i = childrenCount - 1;
                        }

/**
                          *判断不能被接受的View    clickable   Invisiable   点击事件不在  view范围内
                         * 正在动画
                         *
                         * child   2
                         *
                         * child   1
                         */
                        if (!canViewReceivePointerEvents(child)
                                || !isTransformedTouchPointInView(x, y, child, null)) {
							跳过当前的子view
                            ev.setTargetAccessibilityFocus(false);
                            continue;
                        }
					     如果没有往里面赋值的话，得到的都为null，因为一开始还没有初始话
                        newTouchTarget = getTouchTarget(child);//for (TouchTarget target = mFirstTouchTarget; target != null; target = target.next)）
                        if (newTouchTarget != null) {
                            // Child is already receiving touch within its bounds.
                            // Give it the new pointer in addition to the ones it is handling.
                            newTouchTarget.pointerIdBits |= idBitsToAssign;
                            break;
                        }

                        resetCancelNextUpFlag(child);
						如果当前的子View已经消耗掉了当前的事件，就会执行里面的方法
                        if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                            // Child wants to receive touch within its bounds.
                            mLastTouchDownTime = ev.getDownTime();
                            if (preorderedList != null) {
                                // childIndex points into presorted list, find original index
                                for (int j = 0; j < childrenCount; j++) {
                                    if (children[childIndex] == mChildren[j]) {
                                        mLastTouchDownIndex = j;
                                        break;
                                    }
                                }
                            } else {
                                mLastTouchDownIndex = childIndex;
                            }
                            mLastTouchDownX = ev.getX();
                            mLastTouchDownY = ev.getY();
							能进里面来说明找到了能消耗当前事件的child View，就会给mFirstTarget赋值 mFirstTouchTarget = target;
                            newTouchTarget = addTouchTarget(child, idBitsToAssign);
                            alreadyDispatchedToNewTouchTarget = true;
							找到一个之后就没有必要再往下找了
                            break;
                        }
                        // The accessibility focus didn't handle the event, so clear
                        // the flag and do a normal dispatch to all children.
                        ev.setTargetAccessibilityFocus(false);
                    }
                    if (preorderedList != null) preorderedList.clear();
                }
				如果mFirstTouchTarget != null,代表当前已经找到了可以消耗这个事件的childView
                if (newTouchTarget == null && mFirstTouchTarget != null) {
                    // Did not find a child to receive the event.
                    // Assign the pointer to the least recently added target.
                    newTouchTarget = mFirstTouchTarget;
                    while (newTouchTarget.next != null) {
                        newTouchTarget = newTouchTarget.next;
                    }
                    newTouchTarget.pointerIdBits |= idBitsToAssign;
                }
            }
        }
		 第二次move事件触发的时候，上面的变量子控件，获取到可以消耗事件的代码根本不会执行，直接的将当前的事件传递给有需要的子View
        // Dispatch to touch targets.
        if (mFirstTouchTarget == null) { 如果上面的拦截事件返回了true，这个mFirstTouchTarget，默认开始的时候都是为null的，所以就会执行进来
            // No touch targets so treat this as an ordinary view.
            handled = dispatchTransformedTouchEvent(ev, canceled, null,TouchTarget.ALL_POINTER_IDS);
        } else {
            // Dispatch to touch targets, excluding the new touch target if we already
            // dispatched to it.  Cancel touch targets if necessary.
            TouchTarget predecessor = null;
            TouchTarget target = mFirstTouchTarget; 所以如果没有找到可以消耗的childView的时候，这个就为空
			，同时alreadyDispatchedToNewTouchTarget 为true，所以下面的只有在找到的时候执行
            while (target != null) {
                final TouchTarget next = target.next;
                if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                    handled = true;
                } else {
                    final boolean cancelChild = resetCancelNextUpFlag(target.child)
                            || intercepted;
					touchMove事件调用的时候，因为在TouchDown的时候就已经可以确定了可以消耗当前事件的View，所以这边直接传递执行下面的方法
					如果里面的孩子在执行获取到这个事件的时候，如果有消耗掉的话，就会返回true
                    if (dispatchTransformedTouchEvent(ev, cancelChild,target.child, target.pointerIdBits)) {
                        handled = true;
                    }
                    if (cancelChild) {
                        if (predecessor == null) {
                            mFirstTouchTarget = next;
                        } else {
                            predecessor.next = next;
                        }
                        target.recycle();
                        target = next;
                        continue;
                    }
                }
                predecessor = target;
                target = next;
            }
        }

        // Update list of touch targets for pointer up or cancel, if needed.
        if (canceled
                || actionMasked == MotionEvent.ACTION_UP
                || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
            resetTouchState();
        } else if (split && actionMasked == MotionEvent.ACTION_POINTER_UP) {
            final int actionIndex = ev.getActionIndex();
            final int idBitsToRemove = 1 << ev.getPointerId(actionIndex);
            removePointersFromTouchTargets(idBitsToRemove);
        }
    }

    if (!handled && mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onUnhandledEvent(ev, 1);
    }
    return handled; 最终返回的是有没有消耗掉这个事件
}
```
** 事件拦截处理 ，第三个参数如果为null的话，代表这个事件是给自己使用，被拦截了，如果不为null，要传递给子类**
===

```java
/**
 * Transforms a motion event into the coordinate space of a particular child view,
 * filters out irrelevant pointer ids, and overrides its action if necessary.
 * If child is null, assumes the MotionEvent will be sent to this ViewGroup instead.
 */
private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,View child, int desiredPointerIdBits) {
    final boolean handled;
    // Canceling motions is a special case.  We don't need to perform any transformations
    // or filtering.  The important part is the action, not the contents.
    final int oldAction = event.getAction();
    if (cancel || oldAction == MotionEvent.ACTION_CANCEL) { 判断当前的事件是否被取消了
  event.setAction(MotionEvent.ACTION_CANCEL);
        if (child == null) {
            handled = super.dispatchTouchEvent(event);
        } else {
            handled = child.dispatchTouchEvent(event);
        }
        event.setAction(oldAction);
        return handled;
    }

    // Calculate the number of pointers to deliver.
    final int oldPointerIdBits = event.getPointerIdBits();
    final int newPointerIdBits = oldPointerIdBits & desiredPointerIdBits;

    // If for some reason we ended up in an inconsistent state where it looks like we
    // might produce a motion event with no pointers in it, then drop the event.
    if (newPointerIdBits == 0) {
        return false;
    }

    // If the number of pointers is the same and we don't need to perform any fancy
    // irreversible transformations, then we can reuse the motion event for this
    // dispatch as long as we are careful to revert any changes we make.
    // Otherwise we need to make a copy.
    final MotionEvent transformedEvent;
    if (newPointerIdBits == oldPointerIdBits)  判断是否有新手指
 {
        if (child == null || child.hasIdentityMatrix()) {
            if (child == null) {
                handled = super.dispatchTouchEvent(event);
            } else {
                final float offsetX = mScrollX - child.mLeft;
                final float offsetY = mScrollY - child.mTop;
                event.offsetLocation(offsetX, offsetY);

                handled = child.dispatchTouchEvent(event);

                event.offsetLocation(-offsetX, -offsetY);
            }
            return handled;
        }
        transformedEvent = MotionEvent.obtain(event);
    } else {
        transformedEvent = event.split(newPointerIdBits);
    }

    // Perform any necessary transformations and dispatch.
    if (child == null) { 如果事件拦截返回的为true，传递的child就为null，这里的super为View，所以就会执行到对应的方法
        handled = super.dispatchTouchEvent(transformedEvent);
    } else {
        final float offsetX = mScrollX - child.mLeft;
        final float offsetY = mScrollY - child.mTop;
			transformedEvent.offsetLocation(offsetX, offsetY);
        if (! child.hasIdentityMatrix()) {
            transformedEvent.transform(child.getInverseMatrix());
        }
		当传递的child不为空的时候，就调用到了child的方法，当然如果当前的child就为一个简单的view，就调用到View中对应的这个方法，
		去判断是否有给当前的这个child设置onTouchListener，是否是返回true，如果返回true的时候，就不能执行到onTouchEvent方法，点击事件就会失灵等，
		如果当前的child为ViewGroup的话，就会跳到对应的方法，比如ViewGroup,然后ViewGroup中循环遍历起来
        handled = child.dispatchTouchEvent(transformedEvent);
    }

    // Done.
    transformedEvent.recycle();
    return handled; 根据上面返回的结果，true代表被消耗了，false代表没有被消耗
}
```

** 下面的方法为View中的方法在 handled = super.dispatchTouchEvent(transformedEvent);触发**
===

```java
/*
 * Pass the touch screen motion event down to the target view, or this
 * view if it is the target.
 *
 * @param event The motion event to be dispatched.
 * @return True if the event was handled by the view, false otherwise.
 */
public boolean dispatchTouchEvent(MotionEvent event) {
    // If the event should be handled by accessibility focus first.
    if (event.isTargetAccessibilityFocus()) { 检查是否开启了残疾辅助功能
        // We don't have focus or no virtual descendant has it, do not handle the event.
        if (!isAccessibilityFocusedViewOrHost()) {
            return false;
        }
        // We have focus and got the event, then use normal event dispatch.
        event.setTargetAccessibilityFocus(false);
    }

    boolean result = false;
	检查是否有其他的输入设备，比如手写笔
    if (mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onTouchEvent(event, 0);
    }

    final int actionMasked = event.getActionMasked();
    if (actionMasked == MotionEvent.ACTION_DOWN) {
        // Defensive cleanup for new gesture
        stopNestedScroll();
    }

    if (onFilterTouchEventForSecurity(event)) {
        if ((mViewFlags & ENABLED_MASK) == ENABLED && handleScrollBarDragging(event)) {
            result = true;
        }
        //noinspection SimplifiableIfStatement
        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnTouchListener != null
                && (mViewFlags & ENABLED_MASK) == ENABLED
                && li.mOnTouchListener.onTouch(this, event)) { 如果当前的view有设置onTouch事件，并且返回了tue的话，自己的onTouchEvent（）就不会被执行到
            result = true;
        }

        if (!result && onTouchEvent(event)) {如果上面没有设置touchListener的话，设置了没有返回true的话，就会执行到自己的onTouchEvent方法
            result = true;
        }
    }

    if (!result && mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onUnhandledEvent(event, 0);
    }

    // Clean up after nested scrolls if this is the end of a gesture;
    // also cancel it if we tried an ACTION_DOWN but we didn't want the rest
    // of the gesture.
    if (actionMasked == MotionEvent.ACTION_UP ||
            actionMasked == MotionEvent.ACTION_CANCEL ||
            (actionMasked == MotionEvent.ACTION_DOWN && !result)) {
        stopNestedScroll();
    }

    return result;
}

boolean isAccessibilityFocusedViewOrHost() {
    return isAccessibilityFocused() || (getViewRootImpl() != null && getViewRootImpl()
            .getAccessibilityFocusedHost() == this);
}

/**
 * Returns true if a child view contains the specified point when transformed
 * into its coordinate space.
 * Child must not be null.
 * @hide
 */
protected boolean isTransformedTouchPointInView(float x, float y, View child,
        PointF outLocalPoint) {
    final float[] point = getTempPoint();
    point[0] = x;
    point[1] = y;
    transformPointToViewLocal(point, child);
    final boolean isInView = child.pointInView(point[0], point[1]);
    if (isInView && outLocalPoint != null) {
        outLocalPoint.set(point[0], point[1]);
    }
    return isInView;
}
/**
 * @hide
 */
public void transformPointToViewLocal(float[] point, View child) {
 这个就是为什么我们通过view.getTop（）等获取到的是相对于父view的距离
    point[0] += mScrollX - child.mLeft;
    point[1] += mScrollY - child.mTop;

    if (!child.hasIdentityMatrix()) {
        child.getInverseMatrix().mapPoints(point);
    }
}
```





