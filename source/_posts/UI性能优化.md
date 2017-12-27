---
layout: pager
title: UI性能优化
date: 2017-12-26 22:53:13
tags: [Android,UI,性能优化]
description:  UI性能优化
---

 Android UI性能优化
<!--more-->

**渲染的基本认识**
===

卡顿现象
```
渲染功能是应用程序最普遍的功能，开发任何应用程序都是这样，一方面，设计师要求为用户展现可用性最高的超然体验，
另一方面，那些华丽的图片和动画，并不是在所有的设备上都能刘畅地运行。我们来了解一下什么是渲染性能。

首先，我们要知道Android系统每隔16ms就重新绘制一次Activity，也就是说，我们的应用必须在16ms内完成屏幕刷新的全部逻辑操作，这样才能达到每秒60帧，
然而这个每秒帧数的参数由手机硬件所决定，现在大多数手机屏幕刷新率是60赫兹（赫兹是国际单位制中频率的单位，它是每秒中的周期性变动重复次数的计量），
也就是说我们有16ms（1000ms/60次=16.66ms）的时间去完成每帧的绘制逻辑操作，如果错过了，比如说我们花费34ms才完成计算，那么就会出现我们称之为丢帧的情况。
```

![结果显示](/uploads/UI性能优化/UI性能卡顿原理.png)
```
安卓系统尝试在屏幕上绘制新的一帧，但是这一帧还没准备好，所以画面就不会刷新。如果用户盯着同一张图看了32ms而不是16ms，比如上图中前面一帧是正常渲染的，但是紧接着渲染下面
一帧的时候，发现下一帧图片还没有准备好，所以这一帧就会直接的被跳过，如果上一帧图片结束的时候，下一帧一定要准备好，如果没有准备
好就只有等下一帧的到来，这就像我们等地铁一样，这一趟没有赶上，只能等下一趟的道理一样,这样用户就相当于盯着同一张图片看来32ms而不是16ms,用户会很容易察觉出卡顿感，
哪怕仅仅出现一次掉帧，用户都会发现动画不是很顺畅，如果出现多次掉帧，用户就会开始抱怨卡顿，如果此时用户正在和系统进行交互操作，例如滑动列表或者输入数据,
那么卡顿感就会更加明显，用户会毫不留情地对我们的应用进行吐槽，现在我们对绘制每帧花费的时间有了更清晰的了解，再来看看是什么原因导致了卡顿，如何去解决应用中的这些问题
```

渲染管线
```
Android系统的渲染管线分为两个关键组件：CPU和GPU，它们共同工作，在屏幕上绘制图片，每个组件都有自身定义的特定流程。我们必须遵守这些特定的操作规则才能达到效果。
```
![结果显示](/uploads/UI性能优化/性能优化渲染管线.png)
```
在CPU方面，最常见的性能问题是不必要的布局和失效，这些内容必须在视图层次结构中进行测量、清除并重新创建，
引发这种问题通常有两个原因：一是重建显示列表的次数太多，二是花费太多时间作废视图层次并进行不必要的重绘，这两个原因在更新显示列表或者其他缓存GPU资源时导致CPU工作过度。

在GPU方面，最常见的问题是我们所说的过度绘制（overdraw），通常是在像素着色过程中，通过其他工具进行后期着色时浪费了GPU处理时间。
接下来我们将讲解更多关于失效布局和重绘的内容，以及如何使用SDK中的工具找出拖累应用性能的原因
```
CPU和GPU认识
```
想要开发一款性能优越的应用，我们必须了解底层是如何运行的。有一个主要问题就是，Activity是如何绘制到屏幕上的？那些复杂的XML布局文件和标记语言，是如何转化成用户能看懂的图像的？
实际上，这是由格栅化操作来完成的，格栅化就是将例如字符串、按钮、路径或者形状的一些高级对象，拆分到不同的像素上在屏幕上进行显示，格栅化是一个非常费时的操作。
我们所有人的手机里面都有一块特殊硬件，它就是图像处理器（GPU显卡的处理器），目的就是加快格栅化的操作，GPU在上个世纪90年代被引入用来帮助加快格栅化操作
```
![结果显示](/uploads/UI性能优化/性能优化GPU.png)
```
GPU使用一些指定的基础指令集，主要是多边形和纹理，也就是图片，CPU在屏幕上绘制图像前会向GPU输入这些指令，这一过程通常使用的API就是Android的OpenGL ES，
这就是说，在屏幕上绘制UI对象时无论是按钮、路径或者复选框，都需要在CPU中首先转换为多边形或者纹理，然后再传递给GPU进行格栅化。(重点)
下图显示了CPU，GPU工作处理的事情，CPU只是用来计算的，将我们UI，XmL上的按钮，路径或者复选框，都需要先在CPU中计算为多边形或者纹理，然后传递个GPU进行栅格化
```
![结果显示](/uploads/UI性能优化/性能优化CPU工作.png)
```
我们要知道，一个UI对象转换为一系列多边形和纹理的过程肯定相当耗时，从CPU上传处理数据到GPU同样也很耗时。所以很明显，我们需要尽量减少对象转换的次数，以及上传数据的次数，
幸亏，OpenGL ES API允许数据上传到GPU后可以对数据进行保存，当我们下次绘制一个按钮时，只需要在GPU存储器里引用它，然后告诉OpenGL如何绘制就可以了，
一条经验之谈：渲染性能的优化就是尽可能地上传数据到GPU，然后尽可能长地在不修改的情况下保存数据，因为每次上传资源到GPU时，我们都会浪费宝贵的处理时间，
Android系统的Honeycomb版本发布之后，整个UI渲染系统就在GPU中运行，之后各个版本都在渲染系统性能方面有更多改进。
```
![结果显示](/uploads/UI性能优化/性能优化CPUGPU工作性质.png)
```
上图介绍了CPU，GPU工作的主要内容，可以发现CPU主要进行的是计算，转换工作，比如布局文件的测量，布局，等，GPU主要进行栅格化的操作
```

**GPU的主要问题 -过度绘制（overdraw）**
===

```
如果我们曾经粉刷过房子，我们应该知道，给墙壁粉刷工作量非常大，如果我们需要重新粉刷，第一次的粉刷就白干了。同样的道理，我们的应用程序会因为过度绘制，从而导致性能问题，
如果我们想兼顾高性能和完美的设计，往往会碰到一种性能问题，即过度绘制。过度绘制是一个术语，指的是屏幕上的某个像素点在同一帧的时间内被绘制了多次。
假如我们有一堆重叠的UI卡片，最接近用户的卡片在最上面，其余卡片都藏在下面，也就是说我们花大力气绘制的那些下面的卡片基本都是不可见的。
```
![结果显示](/uploads/UI性能优化/过度绘制的图片.png)
```
问题就在于此，因为每次像素经过渲染后，并不是用户最后看到的部分，这就是在浪费GPU的时间。目前流行的一些布局是一把双刃剑，带给我们漂亮视觉感受的同时，也造成过度绘制的问题，
为了最大限度地提高应用程序的性能，我们必须尽量减少过度绘制。幸运的是，Android手机提供了查看过度绘制情况的工具，在开发者选项中打开“Show GPU overdraw”选项，
手机屏幕显示会出现一些异常不用过于惊慌，Android在屏幕上使用不同颜色，标记过度绘制的区域，如果某个像素点只渲染了一次，我们看到的是它原来的颜色，随着过度绘制的增多，
标记颜色也会逐渐加深，例如1倍过度绘制会被标记为蓝色，2倍、3倍、4倍过度绘制遵循同样的模式。所以当我们调试应用程序的用户界面时，目标就是尽可能的减少过度绘制，
将红色区块转变成蓝色区块，为了完成目标有两种清楚过度绘制的方法，首先要从视图中清楚那些，不必要的背景和图片，他们不会在最终渲染图像中显示，
记住，这些都会影响性能。其次，对视图中重叠的屏幕区域进行定义，从而降低CPU和GPU的消耗，接下来我们深入了解过度绘制
```
![结果显示](/uploads/UI性能优化/过度绘制的级别.png)

案例讲解，当我们打开设置中gpu的选择看下面的这个页面
![结果显示](/uploads/UI性能优化/UI性能优化案例一.jpg)
```
上图中可以看到整个页面都存在过度绘制的情况，注意看颜色的区别，如果是浅绿色的话，就代表是多了一重绘制,如果是浅红色的话，就代表是多了俩重绘制，
如果是红的话，就代表是多了三重绘制，所以上面的整个页面都是多重绘制的情况,正常的情况应该是浅蓝色的页面，这样的页面才是正常的页面，下面来分析代码
```
源码分析
```java
首先看整个页面的布局文件，这里为R.layout.activity_chatum_latinum
public class ChatumLatinumActivity extends ActionBarActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_chatum_latinum);

        if (savedInstanceState == null) {
            getSupportFragmentManager().beginTransaction()
                    .add(R.id.activity_chatum_latinum_container, new ChatsFragment())
                    .commit();
        }
    }
}
R.layout.activity_chatum_latinum 的布局细节为

<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/activity_chatum_latinum_container"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="@android:color/white"
    tools:context=".MainActivity"
    tools:ignore="MergeRootFrame" />
	
可以看到我们这里设置了一层背景，会不会是因为这层背景的原因呢，这里我们去掉试试
```

![结果显示](/uploads/UI性能优化/UI性能优化背景优化.png)
```java
通过上图可以看出来，好像真的是因为多了背景的原因，因为去掉了背景之后，颜色由之前的浅绿色，变成了浅蓝色，浅蓝色代表了正常的绘制，不存在多重绘制，但是如果我们的需求就是
要设置背景的时候，这个背景我们是不可以去掉的，这里有一个知识点，就是如果我们用到了MaterialDesign的主题会默认给一个背景。这样我们就存在了俩重背景，那么这个MaterialDesign的背景
就是多余的，我们的背景就是需要的，所以解决的办法：将主题添加的背景去掉

public class ChatumLatinumActivity extends ActionBarActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_chatum_latinum);

        //我们可以通过这种办法去掉materDesign默认添加的背景颜色
        getWindow().setBackgroundDrawable(null);

        if (savedInstanceState == null) {
            getSupportFragmentManager().beginTransaction()
                    .add(R.id.activity_chatum_latinum_container, new ChatsFragment())
                    .commit();
        }
    }
}
```
![结果显示](/uploads/UI性能优化/UI性能优化背景优化.png)
```
可以看到效果跟我们去掉的背景是一样的，这里我们继续查找其他的问题点
下面的布局是里面的fragmetnd的布局文件，这里可以看到，这里的设置的背景也是没有用的
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:paddingLeft="@dimen/activity_horizontal_margin"
    android:paddingRight="@dimen/activity_horizontal_margin"
    android:paddingTop="@dimen/activity_vertical_margin"
    android:paddingBottom="@dimen/activity_vertical_margin"
    android:orientation="vertical"
    android:background="@android:color/white"> (去掉多余的背景颜色)

    <TextView
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:padding="@dimen/narrow_space"
        android:textSize="@dimen/large_text_size"
        android:layout_marginBottom="@dimen/wide_space"
        android:text="@string/header_text" />

    <ListView
        android:id="@+id/listview_chats"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:divider="@android:color/transparent"
        android:dividerHeight="@dimen/divider_height" />
</LinearLayout>

下面是listView每一个item的布局文件，这里也存在多余的背景
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="horizontal"
    android:paddingBottom="@dimen/chat_padding_bottom">

    <ImageView
        android:id="@+id/chat_author_avatar"
        android:layout_width="@dimen/avatar_dimen"
        android:layout_height="@dimen/avatar_dimen"
        android:layout_margin="@dimen/avatar_layout_margin" />

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:background="@android:color/darker_gray"  多余的背景，去掉
        android:orientation="vertical">

        <RelativeLayout
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:textColor="#78A"
            android:background="@android:color/white"     多余的背景，去掉
            android:orientation="horizontal">

            <TextView xmlns:android="http://schemas.android.com/apk/res/android"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_alignParentLeft="true"
                android:padding="@dimen/narrow_space"
                android:gravity="bottom"
                android:id="@+id/chat_author_name" />

            <TextView xmlns:android="http://schemas.android.com/apk/res/android"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_alignParentRight="true"
                android:textStyle="italic"
                android:padding="@dimen/narrow_space"
                android:id="@+id/chat_datetime" />
        </RelativeLayout>

        <TextView xmlns:android="http://schemas.android.com/apk/res/android"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:padding="@dimen/narrow_space"
            android:background="@android:color/white"   多余的背景，去掉
            android:id="@+id/chat_text" />
    </LinearLayout>
</LinearLayout>
```
去掉上面多余的背景之后，显示的结果，可以看到整个页面都不存在多重绘制的情况
![结果显示](/uploads/UI性能优化/UI性能优化最终结果.png)

自定义控件剪裁作用
```
值得指出的是，Android系统知道过度绘制是个麻烦，Android会设法避免绘制，那些在最终图片中不显示的UI组件，这种优化类型，称作剪辑，它对UI性能非常重要。
如果我们能确定某个对象会被完全阻挡，那就完全没有必要绘制它，事实上，这是最重要的性能优化方法之一，而且是有Android系统执行的，但是不幸的是，这一技术无法应对复杂的自定义的View，
系统无法检测onDraw具体会执行什么操作。这些情况下，底层系统无法识别如何去绘制对象，系统很难将覆盖的View，从渲染管道中清除。例如，这叠牌只有最上面的牌是完全可见的，
其他牌都被挡住了，这就意味着绘制那些重叠的像素就是浪费时间。
```
![结果显示](/uploads/UI性能优化/图片剪裁多重绘制.png)

案例分析
![结果显示](/uploads/UI性能优化/重叠性能优化.png)
```java
我们的代码是这样实现的，只是简单的将所有的bitmap，叠加起来，利用canvas.drawBitmap(c.bitmap,c.x,0f,paint)，利用left ,top实现间隔，所以这里就造成了重叠部分的多重绘制
@Override
protected void onDraw(Canvas canvas) {
    super.onDraw(canvas);
    for (DroidCard c : mDroidCards){
        drawDroidCard(canvas, c);
   }
}

/**
* 绘制DroidCard
* @param canvas
* @param c
*/
private void drawDroidCard(Canvas canvas, DroidCard c) {
    canvas.drawBitmap(c.bitmap,c.x,0f,paint);
}
```
我们可以这样的优化代码
```java
 @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        //最后一个我们单独的绘制
        for(int i= 0;i<mDroidCards.size() -1;i++)
        {
            DroidCard c = mDroidCards.get(i);
            drawDroidCard(canvas, c,i);
        }
        //最后一张图片直接绘制即可
        canvas.drawBitmap(mDroidCards.get(mDroidCards.size()-1).bitmap,mDroidCards.get(mDroidCards.size()-1).x,0f,paint);
    }

    /**
     * 绘制DroidCard
     * @param canvas
     * @param c
     */
    private void drawDroidCard(Canvas canvas, DroidCard c,int index) {
        //先保存canvas
        canvas.save();
        //裁剪canvas，裁剪的大小为要显示的大小
        canvas.clipRect(c.x,0,mDroidCards.get(index+1).x,c.height);
        //再在上面绘制内容
        canvas.drawBitmap(c.bitmap,c.x,0f,paint);
        //再恢复回来
        canvas.restore();
    }
```
可以看出，优化之后，不存在多重绘制的部分
![结果显示](/uploads/UI性能优化/重叠部分优化的效果.png)

**CPU的优化**
===

![结果显示](/uploads/UI性能优化/性能优化CPUGPU工作性质.png)

```
通过上图可以知道cpu主要的操作就是计算，比如我们布局文件的measure,layout等，系统的measure,layout都是从跟布局，一层一层的测量，布局，绘制的
所以如果层级关系越深，测量，布局，绘制等用的时间也就越多，cpu的压力也就越大，所以我们通过减轻布局文件的层次来达到优化的效果
```
Hierarchy Viewer工具
```
Hierarchy Viewer将帮助我们快速可视化整个UI结构，另外，它还提供一个更好的方法，让我们理解这个结构内的独特视图的相对渲染性能。它看起来是这样的，让我们来进行设置。
具体的位置在Android Studio中的Android Device Monitor中，找到Hierarchy View工具
HierarchyView工具的使用，如果发现自己使用的时候跟我的不一样，比如当选中一个布局的时候，第四个步骤不可以被点击，ViewProperites里面没有内容
如果你当前有打开gpu的显示过度绘制的范围选项，就要关闭它，最后就是先将开发者选项关闭掉，然后再打开，恢复默认的状态
```
![结果显示](/uploads/UI性能优化/HierarchyView工具的使用.png)

当我们随便选择一个布局文件，然后点上面的三角形，就出现了下面这样的情况，有很多红，绿，黄的小圆点
![结果显示](/uploads/UI性能优化/ViewTree显示的多重颜色.png)

```
三个圆点分别代表：测量、布局、绘制三个阶段的性能表现。
1）绿色：渲染的管道阶段，这个视图的渲染速度快于至少一半的其他的视图。
2）黄色：渲染速度比较慢的50%。
3）红色：渲染速度非常慢。
```

当我们点击一个布局的时候，会显示下面的内容，有测量，布局，绘制显示的时间，右边有此时当前view的属性，比如位置等
![结果显示](/uploads/UI性能优化/单个布局详情的内容.jpg)

我们这里就要注意，那些显示黄色，还有红色的小圆点，看有没有办法优化他们
优化思想:查看自己的布局，层次是否很深以及渲染比较耗时，然后想办法能否减少层级以及优化每一个View的渲染时间。

![结果显示](/uploads/UI性能优化/listView每一个item的情况.png)
上面显示的是我们listView中item的布局,我们可以发现有一些黄色，还有红色的点，我们可以优化他们，系统的就没有办法了

```java
下面为我们每一个item的布局文件，可以发现，这里有很多重嵌套，我们完全可以去掉他们
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="horizontal"
    android:paddingBottom="@dimen/chat_padding_bottom">

    <ImageView
        android:id="@+id/chat_author_avatar"
        android:layout_width="@dimen/avatar_dimen"
        android:layout_height="@dimen/avatar_dimen"
        android:layout_margin="@dimen/avatar_layout_margin" />

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical">

        <RelativeLayout
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:textColor="#78A"
            android:orientation="horizontal">

            <TextView xmlns:android="http://schemas.android.com/apk/res/android"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_alignParentLeft="true"
                android:padding="@dimen/narrow_space"
                android:gravity="bottom"
                android:id="@+id/chat_author_name" />

            <TextView xmlns:android="http://schemas.android.com/apk/res/android"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_alignParentRight="true"
                android:textStyle="italic"
                android:padding="@dimen/narrow_space"
                android:id="@+id/chat_datetime" />
        </RelativeLayout>

        <TextView xmlns:android="http://schemas.android.com/apk/res/android"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:padding="@dimen/narrow_space"
            android:id="@+id/chat_text" />
    </LinearLayout>
</LinearLayout>
```

```java
优化后的代码为
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
                android:layout_width="match_parent"
                android:layout_height="match_parent">

    <ImageView
        android:id="@+id/chat_author_avatar"
        android:layout_width="@dimen/avatar_dimen"
        android:layout_height="@dimen/avatar_dimen"
        android:src="@drawable/alex"/>

    <TextView
        android:id="@+id/chat_author_name"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_toRightOf="@id/chat_author_avatar"
        android:paddingLeft="@dimen/narrow_space"
        android:text="XXX"/>

    <TextView
        android:id="@+id/chat_datetime"
        android:layout_alignParentRight="true"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:paddingRight="@dimen/narrow_space"
        android:textStyle="italic"
        android:text="AAA"/>

    <TextView
        android:id="@+id/chat_text"
        android:layout_toRightOf="@id/chat_author_name"
        android:layout_below="@id/chat_datetime"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:paddingLeft="@dimen/narrow_space"
        android:paddingBottom="@dimen/chat_padding_bottom"
        android:text="BBB"/>
</RelativeLayout>
```

运行的结果为
![结果显示](/uploads/UI性能优化/减少布局层次优化的结果.png)

**总结**
===
```
要想减少gpu的压力，我们可以利用gpu的过度范围查看器，看哪些地方是有被多重绘制的，哪些地方可以被优化的，较少不必要的多重绘制
较少cpu的压力的话，我们可以利用Hierarchy Viewer工具 ，可以帮我们分析我们当前界面的层级关系，通过减少不必要的层级，来较少cpu的不必要的measure，layout，层级
越多cpu计算他们的时间也就越多，我们可以在开发中使用merge 还有 ViewStub 优化我们的层级关系等，
还有就是就算当前界面没有存在多重绘制，并不代表我们的UI的层级就是少的，上面的分析就是在gpu没有多重绘制的情况下出现的层级深的关系，这俩者没有必要的联系

```

