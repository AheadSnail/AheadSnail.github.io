---
layout: pager
title: traceView 分析UI卡顿
date: 2017-12-12 20:21:19
tags: [Android,traceView]
description: 使用traceView分析UI卡顿
---

使用traceView分析UI卡顿
<!--more-->

什么是卡顿现象
```
首先，我们要知道Android系统每隔16ms就重新绘制一次Activity，也就是说，我们的应用必须在16ms内完成屏幕刷新的全部逻辑操作，
这样才能达到每秒60帧，然而这个每秒帧数的参数由手机硬件所决定，现在大多数手机屏幕刷新率是60赫兹（赫兹是国际单位制中频率的单位，
它是每秒中的周期性变动重复次数的计量），也就是说我们有16ms（1000ms/60次=16.66ms）的时间去完成每帧的绘制逻辑操作，如果错过了，
比如说我们花费34ms才完成计算，那么就会出现我们称之为丢帧的情况。
```
![结果显示](/uploads/渲染卡顿.png)
```
安卓系统尝试在屏幕上绘制新的一帧，但是这一帧还没准备好，所以画面就不会刷新。如果用户盯着同一张图看了32ms而不是16ms，用户会很容易察觉出卡顿感，
哪怕仅仅出现一次掉帧，用户都会发现动画不是很顺畅，如果出现多次掉帧，用户就会开始抱怨卡顿，如果此时用户正在和系统进行交互操作，例如滑动列表或者输入数据，
那么卡顿感就会更加明显
```

**卡顿引起的原因**
===
1.内存抖动的问题。
```
要知道，堆内存都有一定的大小，能容纳的数据是有限制的，当Java堆的大小太大时，垃圾收集会启动停止堆中不再应用的对象，来释放内存。
现在，内存抖动这个术语可用于描述在极短时间内分配给对象的过程。例如，当你在循环语句中配置一系列临时对象，或者在绘图功能中配置大量对象时，这相当于内循环，
当屏幕需要重新绘制或出现动画时，你需要一帧帧使用这些功能，不过它会迅速增加你的堆的压力。这两种情况下，我们都制定了解决方案，可在短时间内创造大量的对象。根据创造的对象的量，
或者每个对象的大小，你可能很快就消耗掉所有剩余内存，导致垃圾收集强行开启。随着它们的开启运行，会消耗更多宝贵的帧时间，下面为图解
```

绘图过程中GC回收
![结果显示](/uploads/绘图过程中GC的回收.png)

GC回收时间过长导致卡顿
![结果显示](/uploads/GC时间过长导致卡顿.png)
![结果显示](/uploads/GC回收时间过程导致二.png)

下面的代码为演示内存抖动引起的卡顿效果
```java
public class MemoryChurnActivity extends Activity {
    public static final String LOG_TAG = "Ricky";

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_caching_exercise);

        Button theButtonThatDoesFibonacciStuff = (Button) findViewById(R.id.caching_do_fib_stuff);
        theButtonThatDoesFibonacciStuff.setText("走一个");

        theButtonThatDoesFibonacciStuff.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                imPrettySureSortingIsFree();
            }
        });
        WebView webView = (WebView) findViewById(R.id.webview);
        webView.getSettings().setUseWideViewPort(true);
        webView.getSettings().setLoadWithOverviewMode(true);
        webView.loadUrl("file:///android_asset/shiver_me_timbers.gif");
    }

    /**
     *　排序后打印二维数组，一行行打印
     */
    public void imPrettySureSortingIsFree() {
        int dimension = 300;
        int[][] lotsOfInts = new int[dimension][dimension];
        Random randomGenerator = new Random();
        for(int i = 0; i < lotsOfInts.length; i++) {
            for (int j = 0; j < lotsOfInts[i].length; j++) {
                lotsOfInts[i][j] = randomGenerator.nextInt();
            }
        }

        for(int i = 0; i < lotsOfInts.length; i++) {
            String rowAsStr = "";
            //排序
            int[] sorted = getSorted(lotsOfInts[i]);
            //拼接打印
            for (int j = 0; j < lotsOfInts[i].length; j++) {
                rowAsStr += sorted[j];
                if(j < (lotsOfInts[i].length - 1)){
                    rowAsStr += ", ";
                }
            }
            Log.i("ricky", "Row " + i + ": " + rowAsStr);
        }
```

内存抖动之前正常显示，也就是还没有按下走一个按钮的时候UI效果跟内存的情况
![结果显示](/uploads/内存抖动之前正常显示.gif)

内存抖动显示，也就是按下了走一个按钮的时候UI效果跟内存的情况
![结果显示](/uploads/内存抖动显示.gif)

Android Studio中有一个很方便的可以查看由于某一个操作引起的内存抖动的情况，就是Monitors中的 Allocation Tracker
![结果显示](/uploads/使用Allocation Tracker查看内存抖动.png)

Allocation Trace分析的情况,我们可以快速方便的分析是哪个类引起那么大的内存分配
![结果显示](/uploads/Allocation Trace分析的情况.png)


**TraceView简单的了解使用**
===

使用TraceView来分析
![结果显示](/uploads/traceView简单实用.png)

打开App操作你的应用后，再次点击的话就停止追踪并且自动打开traceview分析面板:
![结果显示](/uploads/traceView简单实用面板分析.png) 

traceview的面板分上下两个部分:
时间线面板以每个线程为一行，右边是该线程在整个过程中方法执行的情况
分析面板是以表格的形式展示所有线程的方法的各项指标

时间面板
![结果显示](/uploads/traceView时间线面板.png) 

左边是线程信息,main线程就是Android应用的主线程，这个线程是都会有的，其他的线程可能因操作不同而发生改变.每个线程的右边对应的是该线程中每个方法的执行信息，
左边为第一个方法执行开始，最右边为最后一个方法执行结束，其中的每一个小立柱就代表一次方法的调用，你可以把鼠标放到立柱上，就会显示该方法调用的详细信息: 
![结果显示](/uploads/traceView方法详细调用.png) 

你可以随意滑动你的鼠标，滑倒哪里，左上角就会显示该方法调用的信息。 
1.如果你想在分析面板中详细查看该方法，可以双击该立柱，分析面板自动跳转到该方法: 
![结果显示](/uploads/traceView矩形柱体方法的详细.png) 

2.放大某个区域 

刚打开的面板中，是我们采集信息的总览，但是一些局部的细节我们看不太清，没关系，该工具支持我们放大某个特殊的时间段,只要按下ctrl再移动鼠标的范围就可以放大被选中的区域
![结果显示](/uploads/traceView放大.png) 

如果想回到最初的状态，双击时间线就可以。 

可以看出来，每一个方法都是用一个凹型结构来表示，坐标的凸起部分表示方法的开始，右边的凸起部分表示方法的结束，中间的直线表示方法的持续.
![结果显示](/uploads/traceView柱体代笔的意思.png) 

```
分析面板每项代表的意义
名称	意义
Name	方法的详细信息，包括包名和参数信息
Incl Cpu Time	Cpu执行该方法该方法及其子方法所花费的时间
Incl Cpu Time %	Cpu执行该方法该方法及其子方法所花费占Cpu总执行时间的百分比
Excl Cpu Time	Cpu执行该方法所话费的时间
Excl Cpu Time %	Cpu执行该方法所话费的时间占Cpu总时间的百分比
Incl Real Time	该方法及其子方法执行所话费的实际时间，从执行该方法到结束一共花了多少时间
Incl Real Time %	上述时间占总的运行时间的百分比
Excl Real Time %	该方法自身的实际允许时间
Excl Real Time	上述时间占总的允许时间的百分比
Calls+Recur	调用次数+递归次数，只在方法中显示，在子展开后的父类和子类方法这一栏被下面的数据代替
Calls/Total	调用次数和总次数的占比
Cpu Time/Call	Cpu执行时间和调用次数的百分比，代表该函数消耗cpu的平均时间
Real Time/Call	实际时间于调用次数的百分比，该表该函数平均执行时间

你可以点击某个函数展开更详细的信息: 
![结果显示](/uploads/TraceView面板详细分析.png) 

展开后，大多数有以下两个类别:
Parents:调用该方法的父类方法
Children:该方法调用的子类方法
如果该方法含有递归调用，可能还会多出两个类别:
Parents while recursive:递归调用时所涉及的父类方法
Children while recursive:递归调用时所涉及的子类方法
```

![结果显示](/uploads/traceView数值.png)  

parent百分比为什么为100%
![结果显示](/uploads/traceViewParent百分比.png)
```
其中的Incl Cpu Time%变成了100%，因为在这个地方，总时间为当前方法的执行时间，这个时候的Incl Cpu Time%只是计算该方法调用的总时间中被各父类方法调用的时间占比，
比如Parents有2个父类方法，那就能看出每个父类方法调用该方法的时间分布。因为我们父类只有一个，所以肯定是100%，Incl Real Time一栏也是一样的
所以对于children中也是一样的原理，如果只有一个孩子那肯定也是100%，如果有多个的话就不一定了，所以如果有多个还要找到占比较大的那个
```
**TraceView分析内存抖动**
===

首先一看这个整体图，就知道是有问题的，一堆密密麻麻的
![结果显示](/uploads/traceView内存抖动整体图.png)

我们随便选择上面的一段，然后放大显示下面的，这里注意看parent为我们的包名package com.example.android.mobileperf.compute;,而根据上面的介绍traceView可知，parent代表当前的这个方法
是由谁来调用的,所以我们可以清楚的知道哪个方法有问题
![结果显示](/uploads/traceView内存抖动放大.png)

当然你可以点击parent选项，查看这个方法是由谁来调用的，这里显示为View的onClick，而我们的代码确实是由于onClick来触发的。。。至此找到了哪个方法有问题
![结果显示](/uploads/traceViewParent再次查找.png)

那我把那平缓分配内存的方法放到子线程执行结果是怎么样呢
```java
 public void imPrettySureSortingIsFree() {
        new Thread(new Runnable()
        {
            @Override
            public void run()
            {
                int dimension = 500;
                int[][] lotsOfInts = new int[dimension][dimension];
                Random randomGenerator = new Random();
                for(int i = 0; i < lotsOfInts.length; i++) {
                    for (int j = 0; j < lotsOfInts[i].length; j++) {
                        lotsOfInts[i][j] = randomGenerator.nextInt();
                    }
                }

                for(int i = 0; i < lotsOfInts.length; i++) {
                    String rowAsStr = "";
                    //排序
                    int[] sorted = getSorted(lotsOfInts[i]);
                    //拼接打印
                    for (int j = 0; j < lotsOfInts[i].length; j++) {
                        rowAsStr += sorted[j];
                        if(j < (lotsOfInts[i].length - 1)){
                            rowAsStr += ", ";
                        }
                    }
                    Log.i("ricky", "Row " + i + ": " + rowAsStr);
                }
            }
        }).start();
```

子线程执行的结果显示
![结果显示](/uploads/子线程执行traceView.gif)

```
那为什么我们把这些代码都放到了子线程了，还是会造成卡顿呢，这是因为，一个app运行的时候，分配的内存空间是有限的，当你的堆内存使用达到了一定程度的时候，就会启动gc来回收垃圾内存，
子线程的内存使用也是属于你app内存的使用的一部分，而且当一旦启动了gc操作，应用程序所有的资源都会极力的配合gc的操作，所以在子线程造成了内存的抖动，导致了gc的平缓发生，整个
应用程序都是会受到影响的,所以唯一的优化方案就是经历的减少内存的分配，尤其是在循环里面，由于垃圾回收机制不会很及时，所以内循环分配的内存空间很快就会增加，然后达到了gc的启动，就造成
了内存抖动
```

2.调用次数不多，但是每一次执行都很耗时

java代码是这样写的，在点击的时候计算斐波那契数列
```java
public class CachingActivity extends Activity {
    public static final String LOG_TAG = "Ricky";

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_caching_exercise);

        Button theButtonThatDoesFibonacciStuff = (Button) findViewById(R.id.caching_do_fib_stuff);
        theButtonThatDoesFibonacciStuff.setText("计算斐波那契数列");

        theButtonThatDoesFibonacciStuff.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Log.i(LOG_TAG, String.valueOf(computeFibonacci(40)));
            }
        });
        WebView webView = (WebView) findViewById(R.id.webview);
        webView.getSettings().setUseWideViewPort(true);
        webView.getSettings().setLoadWithOverviewMode(true);
        webView.loadUrl("file:///android_asset/shiver_me_timbers.gif");
    }

    public int computeFibonacci(int positionInFibSequence) {
        //0 1 1 2 3 5 8
        if (positionInFibSequence <= 2) {
            return 1;
        } else {
            return computeFibonacci(positionInFibSequence - 1)
                    + computeFibonacci(positionInFibSequence - 2);
        }
    }
}
```
耗时方法的执行造成困顿，可以看到，整个过程中内存的使用情况是很少的，而cpu的使用达到了25
![结果显示](/uploads/traceView执行耗时方法.gif)

使用TraceView来分析，可以看到整个一片都是灰色的，只有一种颜色说明只有一个方法被调用
![结果显示](/uploads/traceView耗时方法的执行.png)

我们随便放大一个范围来分析
![结果显示](/uploads/traceView耗时方法的放大分析.png)

```
根据我们上面介绍的，展开后
如果该方法含有递归调用，可能还会多出两个类别:
Parents while recursive:递归调用时所涉及的父类方法
Children while recursive:递归调用时所涉及的子类方法
所以这个方法涉及到了递归的调用

重点是Calls+RecurCalls/Total的值为1/31616 ，1表示这个方法被调用了一次，31616表示这个被递归调用的次数
而Cpu Time/call 表示cpu方法每次调用执行的时间，RealTime/call表示该方法执行一次被调用的真正时间
可以看出来一次方法的执行是不耗时的，至于Parent 跟 children中的cpu使用率达到了100%，是因为他只有
一个父类，还有只有一个子类，所以就100%
```

而且我们也可以非常名了的知道哪个方法出现了问题，还有是由谁来触发的
![结果显示](/uploads/traceView耗时操作分析结果.png)

像这种耗时的操作就完全可以放在子线程里面执行，因为没有涉及到内存的抖动
```java
private void threadExec(final int positionInFibSequence)
    {
        new Thread(new Runnable()
        {
            @Override
            public void run()
            {
                computeFibonacci(positionInFibSequence);
            }
        }).start();
    }

    public int computeFibonacci(int positionInFibSequence) {
        //0 1 1 2 3 5 8
        if (positionInFibSequence <= 2) {
            return 1;
        } else {
            return computeFibonacci(positionInFibSequence - 1)
                    + computeFibonacci(positionInFibSequence - 2);
        }
    }
```

放在子线程中执行的结果
![结果显示](/uploads/traceView子线程执行耗时方法.gif)
