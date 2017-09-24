---
title: ListView 复用机制
date: 2017-09-14 22:08:23
tags: [Android,ListView,TouchEvent]
description: ListView复用源码解析
---

大致的讲解了下ListView的复用的原理，了解手势触摸的时候大致做了什么操作
<!--more-->

1.首先查看onTouchEvent，实现是在AbsListView里面
```java
  case MotionEvent.ACTION_MOVE:
  {
        onTouchMove(ev, vtev);
        break;
  }
```

2.onTouchMove方法中，实现的过程,其中的mTouchMode 在滑动的时候取值为TOUCH_MODE_SCROLL
```java
private void onTouchMove(MotionEvent ev, MotionEvent vtev) 
{
	case TOUCH_MODE_SCROLL:
    case TOUCH_MODE_OVERSCROLL:
            scrollIfNeeded((int) ev.getX(pointerIndex), y, vtev);
		    break;
}
```

3.scrollIfNeeded方法的实现为
```java
 private void scrollIfNeeded(int x, int y, MotionEvent vtev) 
 {
	 。。。
	 final int deltaY = rawDeltaY;
     int incrementalDeltaY = mLastY != Integer.MIN_VALUE ? y - mLastY + scrollConsumedCorrection : deltaY;
	 int overScrollDistance = -incrementalDeltaY;
	 trackMotionScroll(incrementalDeltaY, incrementalDeltaY);
	 。。。
 }
```

4.trackMotionScroll方法的实现为这里传递的为滑动的距离
```java
 boolean trackMotionScroll(int deltaY, int incrementalDeltaY)
  {
	    //如果incrementalDeltay小于0，代表向下滑动，如果大于0，代表向上滑动
		final boolean down = incrementalDeltaY < 0; 
		final int headerViewsCount = getHeaderViewsCount();
        final int footerViewsStart = mItemCount - getFooterViewsCount();
        int start = 0;
        int count = 0;
        
        if (down) {
	        //向上滑动，界面会往上移动，注意第一个item消失的时机点，top这里为正数，代表要滑动的距离，如果top > child.getBottom代表当前的item就要从屏幕上消失了,就可以添加到缓存机制中
            int top = -incrementalDeltaY;
            if ((mGroupFlags & CLIP_TO_PADDING_MASK) == CLIP_TO_PADDING_MASK) {
                top += listPadding.top;
            }
            for (int i = 0; i < childCount; i++) {
                final View child = getChildAt(i);
                //如果第一个childViewd的都不满足消失的点，就没有必要继续往下遍历了，优化。。。。
                if (child.getBottom() >= top) {   
                    break;
                } else {
                    //child.getBottom < top 代表当前的item要从屏幕上消失了
                    count++;
                    int position = firstPosition + i;
                    if (position >= headerViewsCount && position < footerViewsStart) {
                        // The view will be rebound to new data, clear any
                        // system-managed transient state.
                        child.clearAccessibilityFocus();
                        //将当前要移除的item，添加到缓存机制中
                        mRecycler.addScrapView(child, position);
                    }
                }
            }
        } else {
	        //如果是往下滑动，最后的一个item将会从屏幕上消失，这里要缓存起来,这里的incrementalDeltay为正数，这里的bottom代表最后一个item的bottom值
            int bottom = getHeight() - incrementalDeltaY;
            if ((mGroupFlags & CLIP_TO_PADDING_MASK) == CLIP_TO_PADDING_MASK) {
                bottom -= listPadding.bottom;
            }
	        //这里是从下往上遍历，也是一种优化
            for (int i = childCount - 1; i >= 0; i--) {
                final View child = getChildAt(i);
                if (child.getTop() <= bottom) { //最后一个item都不满足，就没有必要再次遍历了
                    break;
                } else {
	                // child.getTop() > bottom 最后一个item要消失了，缓存
                    start = i;
                    count++;
                    int position = firstPosition + i;
                    if (position >= headerViewsCount && position < footerViewsStart) {
                        // The view will be rebound to new data, clear any
                        // system-managed transient state.
                        child.clearAccessibilityFocus();
                        mRecycler.addScrapView(child, position);
                    }
                }
            }
            
        if (count > 0) {
	        //將当前要移除屏幕的item，从父布局中移除
            detachViewsFromParent(start, count);
            mRecycler.removeSkippedScrap();
        }

        // invalidate before moving the children to avoid unnecessary invalidate
        // calls to bubble up from the children all the way to the top
        if (!awakenScrollBars()) {
           invalidate();
        }

        offsetChildrenTopAndBottom(incrementalDeltaY);

        if (down) {
            mFirstPosition += count;
        }

		//取的是滑动的距离，这里是绝对值
        final int absIncrementalDeltaY = Math.abs(incrementalDeltaY);
        if (spaceAbove < absIncrementalDeltaY || spaceBelow < absIncrementalDeltaY) {
            fillGap(down);
	        }
        }
    }
```

5.fillGap(down) 函数 在AbsListView中为 abstract void fillGap(boolean down); 是一个抽象的方法，所以实现类在具体的子类里面，这里为ListView，传递的参数down代表是否是从下往上的填充布局
```java
    /**
     * {@inheritDoc}
     */
    @Override
    void fillGap(boolean down) {
        final int count = getChildCount();
        if (down) {
            int paddingTop = 0;
            if ((mGroupFlags & CLIP_TO_PADDING_MASK) == CLIP_TO_PADDING_MASK) {
                paddingTop = getListPaddingTop();
            }
            final int startOffset = count > 0 ? getChildAt(count - 1).getBottom() + mDividerHeight :
                    paddingTop;
            fillDown(mFirstPosition + count, startOffset);
            correctTooHigh(getChildCount());
        } else {
            int paddingBottom = 0;
            if ((mGroupFlags & CLIP_TO_PADDING_MASK) == CLIP_TO_PADDING_MASK) {
                paddingBottom = getListPaddingBottom();
            }
            final int startOffset = count > 0 ? getChildAt(0).getTop() - mDividerHeight :
                    getHeight() - paddingBottom;
            fillUp(mFirstPosition - 1, startOffset);
            correctTooLow(getChildCount());
        }
    }
```

6.这里的函数有俩个主要的函数   fillDown(mFirstPosition + count, startOffset);或者 fillUp(mFirstPosition - 1, startOffset);我们查看那一个即可
```java
 private View fillDown(int pos, int nextTop) {
        View selectedView = null;

        int end = (mBottom - mTop);
        if ((mGroupFlags & CLIP_TO_PADDING_MASK) == CLIP_TO_PADDING_MASK) {
            end -= mListPadding.bottom;
        }
		//这个可以保证childView是有限个的，只是一个屏幕的范围
        while (nextTop < end && pos < mItemCount) {
            // is this the selected item?
            boolean selected = pos == mSelectedPosition;
            //得到一个view，会先去从缓存中获取到，如果获取的为空话，适配器自己去创建一个View
            View child = makeAndAddView(pos, nextTop, true, mListPadding.left, selected);

            nextTop = child.getBottom() + mDividerHeight;
            if (selected) {
                selectedView = child;
            }
            pos++;
        }

        setVisibleRangeHint(mFirstPosition, mFirstPosition + getChildCount() - 1);
        return selectedView;
    }
```

7.makeAndAddView(pos, nextTop, true, mListPadding.left, selected);函数的实现为
```java
 private View makeAndAddView(int position, int y, boolean flow, int childrenLeft,
            boolean selected) {
        if (!mDataChanged) {
            // Try to use an existing view for this position.
            final View activeView = mRecycler.getActiveView(position);
            if (activeView != null) {
                // Found it. We're reusing an existing child, so it just needs
                // to be positioned like a scrap view.
                setupChild(activeView, position, y, flow, childrenLeft, selected, true);
                return activeView;
            }
        }

        // Make a new view for this position, or convert an unused view if
        // possible.
	    //得到一个itemView
        final View child = obtainView(position, mIsScrap);

        // This needs to be positioned and measured.
        setupChild(child, position, y, flow, childrenLeft, selected, mIsScrap[0]);

        return child;
    }
```

8.关注函数的 final View child = obtainView(position, mIsScrap);的实现为：
```java
 View obtainView(int position, boolean[] outMetadata) {
		.......
		 //这里是从我们的缓存池中获取回收的view，有可能为空，所以我们在适配器中的getView中要对scrapView中要判断是否为空，当为空的时候，就自己创建一个view,这就是listView的复用机制
		 final View scrapView = mRecycler.getScrapView(position);
         final View child = mAdapter.getView(position, scrapView, this);
         ......
 }
```