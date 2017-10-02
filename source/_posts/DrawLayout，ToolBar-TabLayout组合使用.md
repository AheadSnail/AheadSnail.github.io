---
title: 'DrawLayout，ToolBar,TabLayout组合使用'
date: 2017-10-02 16:56:31
tags: [Android,DrawLayout，ToolBar,TabLayout]
description: DrawLayout，ToolBar,TabLayout组合使用
---

DrawLayout，ToolBar,TabLayout组合使用
<!--more-->

**运行结果**
===
![结果显示](/uploads/DrawlayoutDemo.gif)

**ToolBar使用流程**
===
 - 第一步  引入v7包
 - 第二步   修改Them主题  如果不修改，则会报异常 android:theme="@style/Theme.AppCompat.NoActionBar"
 - 异常 This Activity already has an action bar supplied by the window decor
 - 第三步 在布局中写入ToolBar
 - 第三步  把Activity替换成AppCampactActivity	 
 - 第四步   替换ToolBar setSupportActionBar(mToolbar);

**ToolBar 常用的API介绍**
```java
1. item的icon和title属性顾名思义，而app:showAsAction属性则是用来指定这个动作是放置在操作栏上，还是溢出菜单中。当它的值设置为”ifRomom”时，如果操作栏有空间放置，
则放置在操作栏上，否则放置到溢出菜单中。当它的值设置为”always”，这个动作将总是放置在操作栏上。当它的值设置为”never”，这个动作将总是放置在溢出菜单中。
app:showAsAction若为ifRoom则，作为app bar的按钮，若为never则为overflow menu中的选项，若app bar中的空间不足，则会将超出的部分全部放入overflow menu中

下面的内容为我们项目工程里面的style的默认的三个颜色值
设置ToolBar 颜色
<item name="colorPrimary">#2e8abb</item> <!--浅蓝色-->
设置状态蓝颜色
<item name="colorPrimaryDark">#3A5FCD</item> <!--深蓝色-->
设置副标题字体颜色
<item name="android:textColorSecondary">#00FFFF</item>

ToolBar  API
collapseActionView()
折叠当前展开了行动视图。

showOverflowMenu()
从显示相关的菜单溢出项目。

dismissPopupMenus()
关闭所有当前显示弹出式菜单，包括溢出或子菜单。

isOverflowMenuShowing()
检查溢出菜单是否正在显示。

inflateMenu(int resId)
膨胀的菜单资源到这个工具栏。

hideOverflowMenu()
隐藏关联菜单溢出项目。

setContentInsetEndWithActions(int insetEndWithActions)
设置开始的内容插入时操作按钮都存在使用。

setContentInsetStartWithNavigation(int insetStartWithNavigation)
设置启动内容插入时，导航按钮存在使用。

setContentInsetsRelative(int contentInsetStart, int contentInsetEnd)
设置此相对布局方向工具栏的内容插图。

setLogo(Drawable drawable)
设置一个Log图片。

setLogoDescription(int resId)
设置Log的说明。

setNavigationContentDescription(CharSequence description)
如果存在设置导航按钮的内容。

setNavigationOnClickListener(View.OnClickListener listener)
设置一个侦听器来导航事件

setOverflowIcon(Drawable icon)
设置图标使用的溢出按钮。

setSubtitle(CharSequence subtitle)
设置此工具栏的字幕。

setSubtitleTextAppearance(Context context, int resId)
设置文本颜色，大小，样式，颜色提示，并突出显示颜色从指定TextAppearance资源。

setTitleMargin(int start, int top, int end, int bottom)
设置标题边距。

setTitleTextAppearance(Context context, int resId)
设置文本颜色，大小，样式，颜色提示，并突出显示颜色从指定TextAppearance资源。

setTitleTextColor(int color)
设置标题的文本颜色，如果存在的话
```

**TabLayout的基本使用**
===
```java
abLayout的基本使用
1 在应用的build.gradle中添加support:design支持库
2 创建activity_weibo_timeline.xml文件，在布局文件中添加TabLayout及ViewPager：
3.Tab的模式
数据很多的时候我们应该怎么办呢，简书中的第二个Tab就是可以滑动的：
我们先多加几个tab：
超过屏幕之外了
然后设置属性为：
app:tabMode="scrollable"（默认是fixed：固定的，标签很多时候会被挤压，不能滑动。）

1.改变选中字体的颜色	（觉得选中的颜色不好看	）
app:tabSelectedTextColor="@android:color/holo_orange_light"

改变未选中字体的颜色
app:tabTextColor="@color/colorPrimary"

改变指示器下标的颜色
app:tabIndicatorColor="@android:color/holo_orange_light"

改变整个TabLayout的颜色
app:tabBackground="color"

改变TabLayout内部字体大小app:tabTextAppearance="@android:style/TextAppearance.Holo.Large"//设置文字的外貌

改变指示器下标的高度
app:tabIndicatorHeight="4dp"

有时候Tab只有文字感觉有点单调了
tabLayout.addTab(tabLayout.newTab().setText("Tab 1").setIcon(R.mipmap.ic_launcher));	

8.加入Padding
设置Tab内部的子控件的Padding：

app:tabPadding="xxdp"
app:tabPaddingTop="xxdp"
app:tabPaddingStart="xxdp"
app:tabPaddingEnd="xxdp"
app:tabPaddingBottom="xxdp"

设置整个TabLayout的Padding：
app:paddingEnd="xxdp"
app:paddingStart="xxdp"
9.内容的显示模式
app:tabGravity="center"//居中，如果是fill，则是充满

10.Tab的宽度限制
设置最大的tab宽度：
app:tabMaxWidth="xxdp"	
设置最小的tab宽度：
app:tabMinWidth="xxdp"
11.Tab的“Margin”
TabLayout开始位置的偏移量：
app:tabContentStart="100dp"	
```
**Palette的简单实用**
===
```java
Palette是一个类似调色板的工具类，根据传入的bitmap，提取出主体颜色，使得图片和颜色更加搭配，界面更协调。
Palette 可以从一张图片中提取颜色，我们可以把提取的颜色融入到App UI中，可以使UI风格更加美观融洽。比如，我们可以从图片中提取颜色设置给ActionBar做背景颜色，
这样ActionBar的颜色就会随着显示图片的变化而变化。

使用
 1compile 'com.android.support:palette-v7:23.4.0'
     Palette可以提取的颜色如下:
  ● Vibrant （有活力的）
  ● Vibrant dark（有活力的 暗色）
  ● Vibrant light（有活力的 亮色）
  ● Muted （柔和的）
  ● Muted dark（柔和的 暗色）
  ● Muted light（柔和的 亮色）
第一步  compile 'com.android.support:palette-v7:23.4.0'

第一步，我们需要通过一个Bitmap对象来生成一个对应的Palette对象。 Palette 提供了四个静态方法用来生成对象。
  ● Palette generate(Bitmap bitmap)
  ● Palette generate(Bitmap bitmap, int numColors)
  ● generateAsync(Bitmap bitmap, PaletteAsyncListener listener)
  ● generateAsync(Bitmap bitmap, int numColors, final PaletteAsyncListener listener)
不难看出，生成方法分为generate(同步)和generateAsync(异步)两种，如果图片过大使用generate方法，可能会阻塞主线程，我们更倾向于使用generateAsync的方法，
其实内部就是创建了一个AsyncTask。generateAsync方法需要一个PaletteAsyncListener对象用于监听生成完毕的回调。除了必须的Bitmap参数外，还可以传入一个numColors参数指定颜色数，默认是 16。
第二步，得到Palette对象后，就可以拿到提取到的颜色值
  ● Palette.getVibrantSwatch()
  ● Palette.getDarkVibrantSwatch()
  ● Palette.getLightVibrantSwatch()
  ● Palette.getMutedSwatch()
  ● Palette.getDarkMutedSwatch()
  ● Palette.getLightMutedSwatch()
第三步，使用颜色，上面get方法中返回的是一个 Swatch 样本对象，这个样本对象是Palette的一个内部类，它提供了一些获取最终颜色的方法。
  ● getPopulation(): 样本中的像素数量
  ● getRgb(): 颜色的RBG值
  ● getHsl(): 颜色的HSL值
  ● getBodyTextColor(): 主体文字的颜色值
  ● getTitleTextColor(): 标题文字的颜色值
通过 getRgb() 可以得到最终的颜色值并应用到UI中。getBodyTextColor() 和 getTitleTextColor() 可以得到此颜色下文字适合的颜色，这样很方便我们设置文字的颜色，使文字看起来更加舒服。
```
**源码实现**
===
```java
public class MainActivity extends AppCompatActivity
{
    private DrawerLayout mDrawLayout;

    private TabLayout mTabLayout;

    private ViewPager mViewPager;

    private Toolbar mToolbar;

    private final String[] titels = {"分类", "主页", "热门推荐", "热门收藏", "本月热榜", "今日热榜", "专栏", "随机"};
    private int[] drawbles = {R.drawable.a, R.drawable.b, R.drawable.c, R.drawable.d, R.drawable.e,
                              R.drawable.f, R.drawable.g, R.drawable.h};

    @Override
    protected void onCreate(Bundle savedInstanceState)
    {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        mToolbar = (Toolbar) findViewById(R.id.toolbar);
        //注意如果要想修改Toolbar的内容跟主题等要在setSupportActionBar内容之前修改，如果放在了后面修改是不可以的
        mToolbar.setTitle("热门推荐");
        //actionBarDrawerToggle 的添加要放在setSupportActionBar的后面
        setSupportActionBar(mToolbar);

        mDrawLayout = (DrawerLayout) findViewById(R.id.drawLayout);
        mTabLayout = (TabLayout) findViewById(R.id.tabLayout);
        //当TabLayout的内容过多的时候，可以设置这个值，然后TabLayout就可以滑动了
        mTabLayout.setTabMode(TabLayout.MODE_SCROLLABLE);
        //设置TabLayout下划线的高度，如果设置为0，就不会显示这个下划线

        //设置参数，后面的俩个参数是Accessibilty中辅助参数使用的, 第二个参数为DrawLayout,第三个参数为Toolbar，所以他们是同套使用的
        ActionBarDrawerToggle actionBarDrawerToggle = new ActionBarDrawerToggle(this, mDrawLayout, mToolbar, R.string.open, R.string.close);
        //设置Drawlayout监听
        mDrawLayout.addDrawerListener(actionBarDrawerToggle);
        //只有调用了这个方法，才会同步显示出来
        actionBarDrawerToggle.syncState();

        mViewPager = (ViewPager) findViewById(R.id.viewPager);
        //给TabLayout 跟ViewPager设置关联,里面的原理也是互相设置监听
        mTabLayout.setupWithViewPager(mViewPager);
        mViewPager.setAdapter(new MyViewPagerAdapter(getSupportFragmentManager()));

        //给Viewpager设置滑动的监听
        mViewPager.addOnPageChangeListener(new ViewPager.OnPageChangeListener()
        {
            @Override
            public void onPageScrolled(int position, float positionOffset, int positionOffsetPixels)
            {

            }

            @Override
            public void onPageSelected(int position)
            {
                changeThemae(position);
            }

            @Override
            public void onPageScrollStateChanged(int state)
            {

            }
        });

        //一上来就要改变主题
        changeThemae(0);
    }

    /**
     * 改变主题背景等
     *
     * @param position
     */
    private void changeThemae(int position)
    {
        Bitmap bitmap = BitmapFactory.decodeResource(getResources(), drawbles[position]);

        //Palette的作用是从一个图片里面分析出来6种颜色，占用比较多的颜色，其中的分析是耗时的
        Palette.from(bitmap)
               .generate(new Palette.PaletteAsyncListener()
               {
                   @Override
                   public void onGenerated(Palette palette)
                   {
                       //Might be null.
                       Palette.Swatch lightSwath = palette.getLightVibrantSwatch();
                       //如果获取到的swatch 为空，就遍历值到获取一个不为空的，而且要注意，不可能全部都为空的
                       if (lightSwath == null)
                       {
                           for (Palette.Swatch swatch : palette.getSwatches())
                           {
                               lightSwath = swatch;
                               break;
                           }
                       }

                       //颜色加深处理
                       mViewPager.setBackgroundColor(lightSwath.getRgb());
                       //设置下滑线的颜色
                       mTabLayout.setSelectedTabIndicatorColor(lightSwath.getRgb());
                       mTabLayout.setBackgroundColor(lightSwath.getRgb());
                       //设置ToolBar的背景
                       mToolbar.setBackgroundColor(lightSwath.getRgb());
                       if (Build.VERSION.SDK_INT > 21)
                       {
                           Window window = getWindow();
                           //状态栏 颜色加深,设置沉浸式的颜色
                           window.setStatusBarColor(colorBurn(lightSwath.getRgb()));
                           //设置虚拟按键的颜色
                           window.setNavigationBarColor(lightSwath.getRgb());
                       }
                   }
               });
    }

    /**
     * 颜色加深处理，如果不加深的话，状态栏就会显示不清楚
     *
     * @param rgb
     * @return
     */
    private int colorBurn(int rgb)
    {
        int red = Color.red(rgb);
        int green = Color.green(rgb);
        int blue = Color.blue(rgb);

        //要注意对于单通道来说，值变小代表颜色越深
        red = (int) (red * 0.8f);
        green = (int) (green * 0.8f);
        blue = (int) (blue * 0.8f);

        return Color.rgb(red, green, blue);
    }


    private class MyViewPagerAdapter extends FragmentPagerAdapter
    {

        public MyViewPagerAdapter(FragmentManager fm)
        {
            super(fm);
        }

        @Override
        public Fragment getItem(int position)
        {
            Fragment fragment = new ImageFrament();
            Bundle bundle = new Bundle();
            bundle.putInt("id", drawbles[position]);
            //传递参数
            fragment.setArguments(bundle);
            return fragment;
        }

        @Override
        public int getCount()
        {
            return titels.length;
        }

        //重写这个方法可以设置TabLayout中的内容，
        @Override
        public CharSequence getPageTitle(int position)
        {
            return titels[position];
        }
    }

    @Override
    public boolean onCreateOptionsMenu(Menu menu)
    {
        getMenuInflater().inflate(R.menu.main, menu);
        return super.onCreateOptionsMenu(menu);
    }
}
```	
菜单设置

```java
<?xml version="1.0" encoding="utf-8"?>
<menu xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto">

    <!--orderInCategory 越小，就越会优先显示-->
    <!--actionViewClass 可以设置控件的名字-->
    <!--showAsAction ifRoom 表示如果有空间的话，就会显示出来，always标识一定要显示出来，nerver标识会隐藏起来，设置在memnu菜单里面-->

    <item
        android:id="@+id/ab_search"
        android:orderInCategory="80"
        android:title="输入名字"
        app:actionViewClass="android.support.v7.widget.SearchView"
        app:showAsAction="ifRoom"/>
    <item
        android:id="@+id/action_share"
        android:orderInCategory="90"
        android:title="分享"
        app:actionProviderClass="android.support.v7.widget.ShareActionProvider"
        app:showAsAction="ifRoom"/>
    <item
        android:id="@+id/action_settings"
        android:orderInCategory="100"
        android:title="设置"
        app:showAsAction="never"/>
</menu>
```

主布局设置

```java
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    tools:context="com.example.administrator.drawlayoutexample.MainActivity">

    <include layout="@layout/tooblayout"/>

    <android.support.v4.widget.DrawerLayout
        android:id="@+id/drawLayout"
        android:layout_width="match_parent"
        android:layout_height="match_parent">

        <!-- DrawLayout使用中，里面只能包含俩个子View,一个是侧滑区域，一个是内容区域
        使用了android:layout_gravity="start" 的标识的为侧滑区域，注意这个要自己写完整
        并不会有提示
        -->
        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:layout_gravity="start"
            android:background="@drawable/drawer">
        </LinearLayout>

        <!--内容区域，因为里面的内容控件有俩个所以外面包裹了一层，整体作为一个-->
        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:orientation="vertical">

            <!--TabLayout在布局中设置的item如果跟ViewPager一起使用的时候，不会生效-->
            <android.support.design.widget.TabLayout
                android:id="@+id/tabLayout"
                android:layout_width="match_parent"
                android:layout_height="wrap_content">

                <!-- <android.support.design.widget.TabItem
                     android:layout_width="wrap_content"
                     android:layout_height="wrap_content"
                     android:text="分类"/>

                 <android.support.design.widget.TabItem
                     android:layout_width="wrap_content"
                     android:layout_height="wrap_content"
                     android:text="分类"/>

                 <android.support.design.widget.TabItem
                     android:layout_width="wrap_content"
                     android:layout_height="wrap_content"
                     android:text="分类"/>-->
            </android.support.design.widget.TabLayout>

            <android.support.v4.view.ViewPager
                android:id="@+id/viewPager"
                android:layout_width="match_parent"
                android:layout_height="match_parent">
            </android.support.v4.view.ViewPager>
        </LinearLayout>
    </android.support.v4.widget.DrawerLayout>
</LinearLayout>
```