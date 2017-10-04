---
title: Android沉浸式适配，CardView使用
date: 2017-10-03 08:48:14
tags: [Android,沉浸式适配,CardView]
description: Android沉浸式适配，CardView使用
---

Android沉浸式适配，CardView使用
<!--more-->

**沉浸式状态栏**
===

```
Google从Android kitkat(android 4.4)开始,给我们开发者提供了一套能透明的系统ui样式给状态栏和导航栏，
这样的话就不用向以前那样每天面对着黑乎乎的上下两条黑栏了，还可以调成跟Activity一样的样式，形成一个完整的主题,
IOS7.0以上系统一样了。

这样设计的好处就是用户可以沉浸到设计的App中，没有其他因素可以影响到用户，
更符合整体设计风格，用户所看到的一屏 包含App内容  状态栏   内容区，导航栏，
内容区为开发者所控制  导航栏与状态栏 为本身系统拥有，沉浸式设计就是为导航栏和状态栏展开

沉浸式设计的目的就是让App 的效果其能够与手机整体的状态融为一体。不会因为看到其他不和谐的因素而影响用户
```

**5.0以下  4.4以上   以下是另一套**
===

```java
android 5.0以上 提供了很简便的方法，可以直接的去修改状态栏跟导航栏的颜色,从而做到沉浸式的效果
if(Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP)
{
    getWindow().setNavigationBarColor(getResources().getColor(R.color.colorAccent));
    getWindow().setStatusBarColor(getResources().getColor(R.color.colorAccent));
}

对于4.4以上的可以通过这种方式做到状态栏跟导航栏透明，达到一种沉浸式的效果,androidStudio会有提示这些style应该放在value-19使用
将导航栏编程透明
<item name="android:windowTranslucentNavigation">true</item>
将状态栏设置成透明
<item name="android:windowTranslucentStatus">true</item>
```

```java
通过在style属性中使用
<item name="android:windowTranslucentNavigation">true</item>
<item name="android:windowTranslucentStatus">true</item>
android的沉浸式一般跟Toolbar一起使用
```
![结果显示](/uploads/4.4沉浸式状态栏异常.png)

上面的结果虽然可以看到沉浸式的效果，但是发现toolbal的显示有点问题，解决这个问题，我们可以在跟布局里面加上android:fitsSystemWindows="true"
![结果显示](/uploads/4.4FitSystem.png)

**fitSystemWindows属性**
===
Boolean internal attribute to adjust view layout based on system windows such as the status bar. If true, 
adjusts the padding of this view to leave space for the system windows. Will only take effect if this view is in a non-embedded activity. 
这个一个boolean值的内部属性，让view可以根据系统窗口(如status bar)来调整自己的布局，如果值为true,就会调整view的paingding属性来给system windows留出空间…. 

实际效果： 
当status bar为透明或半透明时(4.4以上),系统会设置view的paddingTop值为一个适合的值(status bar的高度)让view的内容不被上拉到状态栏，
当在不占据status bar的情况下(4.4以下)会设置paddingTop值为0(因为没有占据status bar所以不用留出空间)。

但是虽要在每一个的跟布局里面加入这个，而且如果布局里面有ScrollWView的时候，会有异常

**解决的方案是通过在代码中反射**
===

反射系统的R文件类，得到对应的高度,他们存放的路径为 E:\sdk\platforms\android-25\data\res\values,4.4的跟5.0的是有不一样的
![结果显示](/uploads/状态栏的属性值.png)

```java
public class BaseActivity extends AppCompatActivity{
    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        //setContentView之前  全屏
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {
            getWindow().addFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS);
            getWindow().addFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_NAVIGATION);
        }
    }

    /**
     * 5.0  4.4
      * @param toolbar
     * @param styleColor
     */
    public void setToolBarStyle(Toolbar toolbar, View bottomView, int styleColor) {
//        5.0  4.4
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT
                && Build.VERSION.SDK_INT < Build.VERSION_CODES.LOLLIPOP) {
            if (toolbar != null) {
//                ViewGroup.LayoutParams layoutParams=topView.getLayoutParams();
//                getResources().getDimension(android.R.dimen.s)
                int statusHeight=getStatusHeight();
                Log.i("tuch", "   statusHeight  " + statusHeight);
				
                方法一我们可以在toolbar的上面顶一个View，然后反射得到状态栏的高度，动态的设置这个顶上去view的高度，设置背景颜色，达到沉浸式的效果
                layoutParams.height=statusHeight;
                topView.setBackgroundColor(Color.GREEN);

                //第二种，就跟在代码中设置fitSystemWindow的原理一样，通过给toolbar设置paddingTop来解决
                toolbar.setPadding(0,toolbar.getPaddingTop()+statusHeight,0,0);
                //下面的导航栏
				先判断是否有导航栏
                if (haveNavgtion()) {
					原理跟状态栏一样，可以在下面放置一个View，然后动态的设置他们的高度，跟背景
                    ViewGroup.LayoutParams layoutParams=bottomView.getLayoutParams();
                    layoutParams.height+=getNavigationHeight();
                    Log.i("tuch", "getNavigationHeight  " + getNavigationHeight());
                    bottomView.setLayoutParams(layoutParams);
                    bottomView.setBackgroundColor(styleColor);

                }



            }

        }else if(Build.VERSION.SDK_INT >=Build.VERSION_CODES.LOLLIPOP) {	
             getWindow().setNavigationBarColor(getResources().getColor(R.color.colorAccent));
			getWindow().setStatusBarColor(getResources().getColor(R.color.colorAccent));
        }else {
            //没救了
        }



    }

	反射系统的R文件类，得到对应的高度
    private int getNavigationHeight() {
        int height=-1;
        try {
            Class<?> clazz=Class.forName("com.android.internal.R$dimen");
            Object  object=clazz.newInstance();
            String heightStr=clazz.getField("navigation_bar_height").get(object).toString();
            height = Integer.parseInt(heightStr);
            //dp--px
            height = getResources().getDimensionPixelSize(height);
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (InstantiationException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (NoSuchFieldException e) {
            e.printStackTrace();
        }


        return height;
    }

	判断是否有导航栏
    @RequiresApi(api = Build.VERSION_CODES.JELLY_BEAN_MR1)
    private boolean haveNavgtion() {
        //屏幕的高度  真实物理的屏幕
        Display display=getWindowManager().getDefaultDisplay();
        DisplayMetrics displayMetrics=new DisplayMetrics();
        display.getRealMetrics(displayMetrics);
       int heightDisplay=displayMetrics.heightPixels;
        //为了防止横屏
        int widthDisplay=displayMetrics.widthPixels;
        DisplayMetrics contentDisplaymetrics=new DisplayMetrics();
        display.getMetrics(contentDisplaymetrics);
        int contentDisplay=contentDisplaymetrics.heightPixels;
        int contentDisplayWidth=contentDisplaymetrics.widthPixels;
        //屏幕内容高度  显示内容的屏幕
        int w=widthDisplay-contentDisplayWidth;
        //哪一方大于0   就有导航栏
        int h=heightDisplay-contentDisplay;
        return w>0||h>0;
    }

	反射系统的R文件类，得到对应的高度
    private int getStatusHeight() {
        int height=-1;
        try {
            Class<?> clazz=Class.forName("com.android.internal.R$dimen");
            Object  object=clazz.newInstance();
            String heightStr=clazz.getField("status_bar_height").get(object).toString();
            height = Integer.parseInt(heightStr);
            //dp--px
            height = getResources().getDimensionPixelSize(height);
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (InstantiationException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (NoSuchFieldException e) {
            e.printStackTrace();
        }


        return height;
    }
}
```
**CardView**
```java
 CardView简介
  ● CardView继承自FrameLayout类。
  ● CardView是一种卡片视图，主要是以卡片形式显示内容。
CardView功能
  ● CardView实现在一个卡片布局中显示相同的内容，卡片布局可以设置圆角和阴影，还可以布局其他的View。
  ● CardView即可作为一般的布局使用，也可以作为ListView和RecyclerView的Item使用。
CardView何时使用
  ● 需要显示层次性的内容，可以考虑使用。
  ● 需要显示列表或网格时，可以考虑使用。
CardView位置
  ● 包名：android.support.v7.widget.CardView
  ● 文件地址有两个
      ○ android-sdk/extras/android/m2repository/com/android/support/cardview-v7
      ○ android-sdk/extras/android/support/v7/cardView
      ○ CardView引用

导入
Android Studio
dependencies {
  compile 'com.android.support:cardview-v7:23.1.1'
}

代码讲解    。。。。

兼容
  ● 对低版本SDK的兼容（低于Android L版本）
      ○ 针对Android L以下的版本，对CardView添加了一个Elevation的元素，即XML中的app:cardElevation和代码中的setCardElevation。
      ○ 对于在低版本中，设置了CardElevation后，CardView会自动留出空间供阴影显示，但对于Android L版本，就需要手动设置Margin边距来预留空间，这样的结果就是在Android 5.0以上的手机上可以正常显示，但对于Android 4.4.x的手机上就发现边距很大，导致浪费了屏幕空间。
      ○ 解决上面问题，需要我们做适配。可以在/res/value和/res/value-v21分别创建dimens.xml文件，在dimens.xml里，随意命名，对于Android 5.0以上的设置数值0dp，对于Android 5.0以下的设置数值（根据实际需求）。这样就解决低版本中边距过大或者视觉效果不好的问题了。
  ● 对低版本SDK的兼容（低于Android L版本）setElevation的问题
      ○ 由于缺少一些系统API（如 RenderThread），CardView中的Elevation兼容实现并不完美，和真正的实现方法还是有较大的差距（不是指效果），所以调用 setCardElevation也不能随心所欲地传入一个Float型，在低版本系统使用时应当处理一下传入的数值或加上try-catch（不推荐）。
      ○ 
xml布局使用==============================
  ● android:cardCornerRadius
      ○ 在xml文件中设置card圆角的大小
  ● CardView.setRadius
      ○ 在代码中设置card圆角的大小
  ● android:cardBackgroundColor
      ○ 在xml文件中设置card背景颜色
  ● android:elevation
      ○ 在xml文件中设置阴影的大小
  ● card_view:cardElevation
      ○ 在xml文件中设置阴影的大小
  ● card_view:cardMaxElevation
      ○ 在xml文件中设置阴影最大高度
  ● card_view:cardCornerRadius
      ○ 在xml文件中设置卡片的圆角大小
  ● card_view:contentPadding
      ○ 在xml文件中设置卡片内容于边距的间隔
  ● card_view:contentPaddingBottom
      ○ 在xml文件中设置卡片内容于下边距的间隔
  ● card_view:contentPaddingTop
      ○ 在xml文件中设置卡片内容于上边距的间隔
  ● card_view:contentPaddingLeft
      ○ 在xml文件中设置卡片内容于左边距的间隔
  ● card_view:contentPaddingRight
      ○ 在xml文件中设置卡片内容于右边距的间隔
  ● card_view:contentPaddingStart
      ○ 在xml文件中设置卡片内容于边距的间隔起始
  ● card_view:contentPaddingEnd
      ○ 在xml文件中设置卡片内容于边距的间隔终止
  ● card_view:cardUseCompatPadding
      ○ 在xml文件中设置内边距，V21+的版本和之前的版本仍旧具有一样的计算方式
  ● card_view:cardPreventConrerOverlap
      ○ 在xml文件中设置内边距，在V20和之前的版本中添加内边距，这个属性为了防止内容和边角的重叠
波纹点击效果
  ● 默认情况，CardView是不可点击的，并且没有任何的触摸反馈效果。
  ● 触摸反馈动画在用户点击CardView时可以给用户以视觉上的反馈。
  ● 实现这种行为，你必须提供一下属性:android:clickable和android:foreground。
<android.support.v7.widget.CardView
  xmlns:android="http://schemas.android.com/apk/res/android"
  xmlns:card_view="http://schemas.android.com/apk/res-auto"
  ...
  android:clickable="true"
  android:foreground="?android:attr/selectableItemBackground">
```
