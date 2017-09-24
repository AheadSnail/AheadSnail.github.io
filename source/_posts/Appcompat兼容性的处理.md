---
title: Appcompat兼容性的处理
date: 2017-09-17 17:04:16
tags: [Android,AppcompatActivity]
description: Appcompat兼容性的处理
---

了解了下AppcompatActivity的中的兼容性的处理
<!--more-->

**AppCompat兼容性**
===
**控件的兼容性 v4 v7 v13 数字代表的是支持的api 的version
AppCompatActivity 是如何做到兼容性的
原理： 继承了AppCompatActivity 的Activity 都会在解析XML 的时候，将xml里面所有的系统控件转换为 appCompatButton。它的源码是怎样的呢？ 所谓的兼容，就是一个着色问题！！！
AppCompatDelegate 的工作就是涂色。 替换：widget着色是通过这个widget 的layout 在inflation 的时候，被AppCompatDelegate 拦截下来，然后根据 控件的名字，强制被系统转换成为 
以AppCompat 开头的控件。
**

**1.首先查看继承的关系**
===
```java
public class AppCompatActivity extends FragmentActivity implements AppCompatCallback,
        TaskStackBuilder.SupportParentable, ActionBarDrawerToggle.DelegateProvider {
public class FragmentActivity extends BaseFragmentActivityJB implements
        ActivityCompat.OnRequestPermissionsResultCallback,
        ActivityCompatApi23.RequestPermissionsRequestCodeValidator {
abstract class BaseFragmentActivityJB extends BaseFragmentActivityHoneycomb {
abstract class BaseFragmentActivityHoneycomb extends BaseFragmentActivityGingerbread {
abstract class BaseFragmentActivityGingerbread extends SupportActivity {
public class SupportActivity extends Activity {
```
**2.AppCompatActivity 中代理的实现**
===
```java
  @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
	    //获取到代理的实例对象
        final AppCompatDelegate delegate = getDelegate();
        delegate.installViewFactory();
        delegate.onCreate(savedInstanceState);
        if (delegate.applyDayNight() && mThemeId != 0) {
            // If DayNight has been applied, we need to re-apply the theme for
            // the changes to take effect. On API 23+, we should bypass
            // setTheme(), which will no-op if the theme ID is identical to the
            // current theme ID.
            if (Build.VERSION.SDK_INT >= 23) {
                onApplyThemeResource(getTheme(), mThemeId, false);
            } else {
                setTheme(mThemeId);
            }
        }
        super.onCreate(savedInstanceState);
    }
     /**
     * @return The {@link AppCompatDelegate} being used by this Activity.
     */
    @NonNull
    public AppCompatDelegate getDelegate() {
        if (mDelegate == null) {
            mDelegate = AppCompatDelegate.create(this, this);
        }
        return mDelegate;
    }
    private static AppCompatDelegate create(Context context, Window window,
            AppCompatCallback callback) {
    //这边会根据不同的系统的版本号得到不同的代理对象，但是其他的本质都是直接，或者间接的继承AppCompatDelegateImplV9的s
        final int sdk = Build.VERSION.SDK_INT;
        if (BuildCompat.isAtLeastN()) {
            return new AppCompatDelegateImplN(context, window, callback);
        } else if (sdk >= 23) {
            return new AppCompatDelegateImplV23(context, window, callback);
        } else if (sdk >= 14) {
            return new AppCompatDelegateImplV14(context, window, callback);
        } else if (sdk >= 11) {
            return new AppCompatDelegateImplV11(context, window, callback);
        } else {
            return new AppCompatDelegateImplV9(context, window, callback);
        }
    }
    //比如AppCompatDelegateImplN  中的继承关系为
    @RequiresApi(24)
	@TargetApi(24)
	class AppCompatDelegateImplN extends AppCompatDelegateImplV23
	@RequiresApi(23)
	@TargetApi(23)
	class AppCompatDelegateImplV23 extends AppCompatDelegateImplV14 
	@RequiresApi(14)
	@TargetApi(14)
	class AppCompatDelegateImplV14 extends AppCompatDelegateImplV11
	@RequiresApi(11)
	@TargetApi(11)
	//当我们在我们的代码中执行了下面的代码的时候
	 @Override
    protected void onCreate(Bundle savedInstanceState)
    {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
    }
    //执行  setContentView(R.layout.activity_main); 的时候
    @Override
    public void setContentView(@LayoutRes int layoutResID) {
        getDelegate().setContentView(layoutResID);
    }
    //执行getDelegate().setContentView(layoutResID);的时候，就会回调到AppCompatDelegateImplV9中的
    @Override
    public void setContentView(int resId) {
        ensureSubDecor();
        ViewGroup contentParent = (ViewGroup) mSubDecor.findViewById(android.R.id.content);
        contentParent.removeAllViews();
	    //这边会采用布局加载器来加载布局，这是因为下面为  LayoutInflaterCompat.setFactory(layoutInflater, this);
        LayoutInflater.from(mContext).inflate(resId, contentParent);
        mOriginalWindowCallback.onContentChanged();
    }
	class AppCompatDelegateImplV11 extends AppCompatDelegateImplV9
	//AppCompatDelegateImplV9 中部分的代码，这个类为public abstract class AppCompatDelegat中的抽象方法的实现
	@Override
    public void installViewFactory() {
        LayoutInflater layoutInflater = LayoutInflater.from(mContext);
        if (layoutInflater.getFactory() == null) {
	        //设置了LayoutInflaterFactory 的回调this
            LayoutInflaterCompat.setFactory(layoutInflater, this);
        } else {
            if (!(LayoutInflaterCompat.getFactory(layoutInflater)
                    instanceof AppCompatDelegateImplV9)) {
                Log.i(TAG, "The Activity's LayoutInflater already has a Factory installed"
                        + " so we can not install AppCompat's");
            }
        }
    }
    //这个方法的回调onCreateView 为public interface LayoutInflaterFactory接口的实现
     /**
     * From {@link android.support.v4.view.LayoutInflaterFactory}
     */
    @Override
    public final View onCreateView(View parent, String name, Context context, AttributeSet attrs) {
        // First let the Activity's Factory try and inflate the view
        final View view = callActivityOnCreateView(parent, name, context, attrs);
        if (view != null) {
            return view;
        }
		
        // If the Factory didn't handle it, let our createView() method try
        return createView(parent, name, context, attrs);
    }
	//当执行到createView(parent, name, context, attrs);的时候
	@Override
    public View createView(View parent, final String name, @NonNull Context context,
            @NonNull AttributeSet attrs) {
        if (mAppCompatViewInflater == null) {
            mAppCompatViewInflater = new AppCompatViewInflater();
        }

        boolean inheritContext = false;
        if (IS_PRE_LOLLIPOP) {
            inheritContext = (attrs instanceof XmlPullParser)
                    // If we have a XmlPullParser, we can detect where we are in the layout
                    ? ((XmlPullParser) attrs).getDepth() > 1
                    // Otherwise we have to use the old heuristic
                    : shouldInheritContext((ViewParent) parent);
        }

        return mAppCompatViewInflater.createView(parent, name, context, attrs, inheritContext,
                IS_PRE_LOLLIPOP, /* Only read android:theme pre-L (L+ handles this anyway) */
                true, /* Read read app:theme as a fallback at all times for legacy reasons */
                VectorEnabledTintResources.shouldBeUsed() /* Only tint wrap the context if enabled */
        );
    }
    //当执行到 mAppCompatViewInflater.createView(parent, name, context, attrs, inheritContext的时候
	public final View createView(View parent, final String name, @NonNull Context context,
            @NonNull AttributeSet attrs, boolean inheritContext,
            boolean readAndroidTheme, boolean readAppTheme, boolean wrapContext) {
        final Context originalContext = context;

        // We can emulate Lollipop's android:theme attribute propagating down the view hierarchy
        // by using the parent's context
        if (inheritContext && parent != null) {
            context = parent.getContext();
        }
        if (readAndroidTheme || readAppTheme) {
            // We then apply the theme on the context, if specified
            context = themifyContext(context, attrs, readAndroidTheme, readAppTheme);
        }
        if (wrapContext) {
            context = TintContextWrapper.wrap(context);
        }
		//name为在布局里面设置的TextView,ImageView,这边就会根据这些名字对应的生成Appcompat对应的控件，这里就是适配的实现，可以在不同的版本下，内容的体现是一样
        View view = null;
        // We need to 'inject' our tint aware Views in place of the standard framework versions
        switch (name) {
            case "TextView":
                view = new AppCompatTextView(context, attrs);
                break;
            case "ImageView":
                view = new AppCompatImageView(context, attrs);
                break;
            case "Button":
                view = new AppCompatButton(context, attrs);
                break;
            case "EditText":
                view = new AppCompatEditText(context, attrs);
                break;
            case "Spinner":
                view = new AppCompatSpinner(context, attrs);
                break;
            case "ImageButton":
                view = new AppCompatImageButton(context, attrs);
                break;
            case "CheckBox":
                view = new AppCompatCheckBox(context, attrs);
                break;
            case "RadioButton":
                view = new AppCompatRadioButton(context, attrs);
                break;
            case "CheckedTextView":
                view = new AppCompatCheckedTextView(context, attrs);
                break;
            case "AutoCompleteTextView":
                view = new AppCompatAutoCompleteTextView(context, attrs);
                break;
            case "MultiAutoCompleteTextView":
                view = new AppCompatMultiAutoCompleteTextView(context, attrs);
                break;
            case "RatingBar":
                view = new AppCompatRatingBar(context, attrs);
                break;
            case "SeekBar":
                view = new AppCompatSeekBar(context, attrs);
                break;
        }

        if (view == null && originalContext != context) {
            // If the original context does not equal our themed context, then we need to manually
            // inflate it using the name so that android:theme takes effect.
            view = createViewFromTag(context, name, attrs);
        }

        if (view != null) {
            // If we have created a view, check it's android:onClick
            checkOnClickListener(view, attrs);
        }

        return view;
    }
```

**LinearLayoutCompat**
===
 - 这个类是干什么的，是控件还是组件，看他的初始化在干什么
 - 找到入口，一般是构造器
 - 找关键方法：onMeasure  onLayout   onDraw
 - onMeasure 是 计算子控件的大小，同时计算自己的大小。自己的宽高是由子空间的宽高决定的，根据摆放的方向不同而算法不同
 - onLayout 是将子控件的上下左右位置进行确定。然后布局到layout上面
 - onDraw    只用来画layout 自己的分割线，其他的内容由具体的item来绘制
```java
 @Override
 protected void onDraw(Canvas canvas) {
        if (mDivider == null) {
            return;
        }

        if (mOrientation == VERTICAL) {
            drawDividersVertical(canvas);
        } else {
            drawDividersHorizontal(canvas);
        }
    }
    
  void drawDividersVertical(Canvas canvas) {
        final int count = getVirtualChildCount();
        for (int i = 0; i < count; i++) {
            final View child = getVirtualChildAt(i);

            if (child != null && child.getVisibility() != GONE) {
                if (hasDividerBeforeChildAt(i)) {
                    final LayoutParams lp = (LayoutParams) child.getLayoutParams();
                    final int top = child.getTop() - lp.topMargin - mDividerHeight;
                    drawHorizontalDivider(canvas, top);
                }
            }
        }

        if (hasDividerBeforeChildAt(count)) {
            final View child = getVirtualChildAt(count - 1);
            int bottom = 0;
            if (child == null) {
                bottom = getHeight() - getPaddingBottom() - mDividerHeight;
            } else {
                final LayoutParams lp = (LayoutParams) child.getLayoutParams();
                bottom = child.getBottom() + lp.bottomMargin;
            }
            drawHorizontalDivider(canvas, bottom);
        }
    }
    void drawHorizontalDivider(Canvas canvas, int top) {
        mDivider.setBounds(getPaddingLeft() + mDividerPadding, top,
                getWidth() - getPaddingRight() - mDividerPadding, top + mDividerHeight);
        mDivider.draw(canvas);
    }

    void drawVerticalDivider(Canvas canvas, int left) {
        mDivider.setBounds(left, getPaddingTop() + mDividerPadding,
                left + mDividerWidth, getHeight() - getPaddingBottom() - mDividerPadding);
        mDivider.draw(canvas);
    }
```