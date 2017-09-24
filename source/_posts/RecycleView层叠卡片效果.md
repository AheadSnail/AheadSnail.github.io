---
title: RecycleView层叠卡片效果
date: 2017-09-24 20:45:48
tags: [Android,RecycleView,Scroll]
description: RecycleView层叠卡片效果
---

RecycleView层叠卡片效果
<!--more-->

**运行结果**
===
![结果显示](/uploads/1.png)

**源码实现为**
===

```java
mRecycleView = (RecyclerView) findViewById(R.id.rv);
mDatas = SwipeCardBean.initDatas();
自定义的LayoutManager
mLayoutManager = new SwipCardLayoutManager();
mRecycleView.setLayoutManager(mLayoutManager);

//添加ItemTouchCallBack 用来处理滑动的动画效果
mCallBack = new SwipCardCallBack(0,0,mAdapter,mDatas,mRecycleView);
ItemTouchHelper helper = new ItemTouchHelper(mCallBack);
helper.attachToRecyclerView(mRecycleView);
 ```
SwipCardLayoutManagerh实现为
```java
public class SwipCardLayoutManager extends RecyclerView.LayoutManager
{
    @Override
    public RecyclerView.LayoutParams generateDefaultLayoutParams()
    {
        return new RecyclerView.LayoutParams(ViewGroup.LayoutParams.WRAP_CONTENT,
                                             ViewGroup.LayoutParams.WRAP_CONTENT);
    }

    //父类的layoutManager是不会负责item的布置的，这个要由具体的对象来实现
    @Override
    public void onLayoutChildren(RecyclerView.Recycler recycler, RecyclerView.State state)
    {
        //h缓存的核心机制之一
        detachAndScrapAttachedViews(recycler);
        // return a != null ? a.getItemCount() : 0;获取到的是适配器中的数据的个数
        int itemCount = getItemCount();
        //防止越界，这里保证只会去测量，摆放，显示四个childItem
        int initCount = 0;
        if (itemCount < CardConfig.MAX_SHOW_COUNT)
        {
            initCount = 0;
        }
        else
        {
            initCount = itemCount - CardConfig.MAX_SHOW_COUNT;
        }

        for (int position = initCount; position < itemCount; position++)
        {
            //从recycleView中获取到对应的childItem
            View childView = recycler.getViewForPosition(position);
            //添加进当前要显示的view
            addView(childView);
            //执行测量
            childView.measure(0, 0);

            //获取到除了当前显示的这个item外，剩余的空间大小
            int widthSpec = getWidth() - getDecoratedMeasuredWidth(childView);
            //获取到除了当前显示的这个ite外，剩余的空间大小
            int heightSpec = getHeight() - getDecoratedMeasuredHeight(childView);
            //摆放当前的item
            layoutDecorated(childView, widthSpec / 2, heightSpec / 2, getWidth() - widthSpec / 2,getHeight() - heightSpec / 2);

            //执行下面摆放的错杂 ,这个的值的变化为3 2 1 0
            int level = itemCount - position - 1;
            if (level > 0)
            {
                childView.setScaleX(1 - CardConfig.SCALE_GAP * level);
                childView.setScaleY(1 - CardConfig.SCALE_GAP * level);
                if (level < CardConfig.MAX_SHOW_COUNT - 1)
                {
                    childView.setTranslationY(CardConfig.TRANS_Y_GAP * level);
                }
                else
                {
                    //最后一个的y的位置要跟前面的一个保持一样，重叠在一起就好了
                    childView.setTranslationY(CardConfig.TRANS_Y_GAP * (level - 1));
                }
            }
            super.onLayoutChildren(recycler, state);
        }
    }
}
 ```
 
 SwipCardCallBack实现为
 ```java
 public class SwipCardCallBack extends ItemTouchHelper.SimpleCallback
{

    private RecyclerView mRv;
    private RecyclerView.Adapter mAdapter;
    private List mDatas;

    public SwipCardCallBack(int dragDirs, int swipeDirs, RecyclerView.Adapter adapter, List datas,
                            RecyclerView recyclerView)
    {
        //第一个参数代表滑动的方向，后面的为swip的时候方向，这里要上下左右都有这个动画
        super(0,
              ItemTouchHelper.DOWN | ItemTouchHelper.UP | ItemTouchHelper.RIGHT | ItemTouchHelper.LEFT);
        mRv = recyclerView;
        mAdapter = adapter;
        mDatas = datas;
    }

    //滑动的时候会触发这个方法
    @Override
    public boolean onMove(RecyclerView recyclerView, RecyclerView.ViewHolder viewHolder,
                          RecyclerView.ViewHolder target)
    {
        return false;
    }

    // 在swipe 运动动画结束的时候调用 ，这里用来交换适配器中的数据
    @Override
    public void onSwiped(RecyclerView.ViewHolder viewHolder, int direction)
    {
        //这里做的操作就是当这个item滑动出去的时候，就添加到第一个的位置，这样造成循环
        Object bean = mDatas.remove(viewHolder.getLayoutPosition());
        //添加到第一个的位置
        mDatas.add(0, bean);
        //通知数据发生了变化，这里就使用移动的改变
        mAdapter.notifyItemMoved(viewHolder.getLayoutPosition(), 0);
    }

    //用来执行child的渲染,这里在滑动的时候给出一种弹跳的动画
    @Override
    public void onChildDraw(Canvas c, RecyclerView recyclerView, RecyclerView.ViewHolder viewHolder,
                            float dX, float dY, int actionState, boolean isCurrentlyActive)
    {

        double maxDistance = recyclerView.getWidth() * 0.5f;
        double distance = Math.sqrt(dX * dX + dY * dY);
        //求得滑动的比例
        double fraction = distance / maxDistance;
        if(fraction > 1)
        {
            fraction = 1;
        }
        // Returns the number of children in the group. 要注意这里返回的是当前recycleView 中的孩子的个数，因为我们只添加了四个
        //所以这里返回的就为4
        int childCount = recyclerView.getChildCount();
        for (int i = 0; i < childCount; i++)
        {
            View view = recyclerView.getChildAt(i);
            //变化的范围为 3 2 1 0
            int level = childCount - i - 1;

            if (level >= 0)
            {
                //前面的三个要执行变化，这样看起来就有了动画
                if (level < CardConfig.MAX_SHOW_COUNT - 1)
                {
                    view.setTranslationY((float) (CardConfig.TRANS_Y_GAP * level - fraction * CardConfig.TRANS_Y_GAP));
                    view.setScaleX((float) (1 - CardConfig.SCALE_GAP * level + fraction * CardConfig.SCALE_GAP));
                    view.setScaleY((float) (1 - CardConfig.SCALE_GAP * level + fraction * CardConfig.SCALE_GAP));
                    //view.setAlpha((float) (1 - 0.1 * level + fraction *CardConfig.SCALE_GAP));
                }
                //最后一个item要跟前面的一个重叠在一起
                else if (level == CardConfig.MAX_SHOW_COUNT - 1)
                { // 控制的是最下层的Item
                    view.setTranslationY((float) (CardConfig.TRANS_Y_GAP * (level - 1)));
                    view.setScaleX((float) (1 - CardConfig.SCALE_GAP * (level - 1)));
                    view.setScaleY((float) (1 - CardConfig.SCALE_GAP * (level - 1)));
                }
            }
        }


        super.onChildDraw(c, recyclerView, viewHolder, dX, dY, actionState, isCurrentlyActive);
    }
}
```
 