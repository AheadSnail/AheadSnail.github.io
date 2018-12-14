---
layout: pager
title: Android Aop 理解
date: 2018-06-23 09:42:56
tags: [Android,Aop]
description:  Android Aop 理解
---

### 概述

> Android Aop 理解

<!--more-->


### 什么是AOP？
> 大家都知道OOP，即ObjectOriented Programming，面向对象编程。而本文要介绍的是AOP。AOP是Aspect Oriented Programming的缩写，中译文为面向切向编程。OOP和AOP是什么关系呢？
首先:OOP和AOP都是方法论。我记得在刚学习C++的时候，最难学的并不是C++的语法，而是C++所代表的那种看问题的方法，即OOP。同样，今天在AOP中，我发现其难度并不在利用AOP干活，而是从AOP的角度来看待问题，设计解决方法。这就是为什么我特意强调AOP是一种方法论的原因！

在OOP的世界中，问题或者功能都被划分到一个一个的模块里边。每个模块专心干自己的事情，模块之间通过设计好的接口交互。从图示来看，OOP世界中，最常见的表示比如：

![结果显示](/uploads/AOP/oop.png)

> 图1中所示为AndroidFramework中的模块。OOP世界中，大家画的模块图基本上是这样的，每个功能都放在一个模块里。非常好理解，而且确实简化了我们所处理问题的难度。
OOP的精髓是把功能或问题模块化，每个模块处理自己的家务事。但在现实世界中，并不是所有问题都能完美得划分到模块中。举个最简单而又常见的例子：现在想为每个模块加上日志功能，
要求模块运行时候能输出日志。在不知道AOP的情况下，一般的处理都是：先设计一个日志输出模块，这个模块提供日志输出API，比如Android中的Log类。然后，
其他模块需要输出日志的时候调用Log类的几个函数，比如e(TAG,...)，w(TAG,...)，d(TAG,...)，i(TAG,...)等。

在没有接触AOP之前，包括我在内，想到的解决方案就是上面这样的。但是，从OOP角度看，除了日志模块本身，其他模块的家务事绝大部分情况下应该都不会包含日志输出功能。什么意思？
以ActivityManagerService为例，你能说它的家务事里包含日志输出吗？显然，ActivityManagerService的功能点中不包含输出日志这一项。但实际上，软件中的众多模块确实又需要打印日志。
这个日志输出功能，从整体来看，都是一个面上的。而这个面的范围，就不局限在单个模块里了，而是横跨多个模块。

在没有AOP之前，各个模块要打印日志，就是自己处理。反正日志模块的那几个API都已经写好了，你在其他模块的任何地方，任何时候都可以调用。功能是得到了满足，但是好像没有Oriented的感觉了。
是的，随意加日志输出功能，使得其他模块的代码和日志模块耦合非常紧密。而且，将来要是日志模块修改了API，则使用它们的地方都得改。这种搞法，一点也不酷。

AOP的目标就是解决上面提到的不cool的问题。在AOP中：
> 第一，我们要认识到OOP世界中，有些功能是横跨并嵌入众多模块里的，比如打印日志，比如统计某个模块中某些函数的执行时间等。这些功能在各个模块里分散得很厉害，可能到处都能见到。
第二，AOP的目标是把这些功能集中起来，放到一个统一的地方来控制和管理。如果说，OOP如果是把问题划分到单个模块的话，那么AOP就是把涉及到众多模块的某一类问题进行统一管理。
比如我们可以设计两个Aspects，一个是管理某个软件中所有模块的日志输出的功能，另外一个是管理该软件中一些特殊函数调用的权限检查。


### AOP基础语法
> AOP虽然是方法论，但就好像OOP中的Java一样，一些先行者也开发了一套语言来支持AOP。目前用得比较火的就是AspectJ了，它是一种几乎和Java完全一样的语言，
而且完全兼容Java（AspectJ应该就是一种扩展Java，但它不是像Groovy[1]那样的拓展。）。当然，除了使用AspectJ特殊的语言外，AspectJ还支持原生的Java，只要加上对应的AspectJ注解就好。所以，使用AspectJ有两种方法：

1.完全使用AspectJ的语言。这语言一点也不难，和Java几乎一样，也能在AspectJ中调用Java的任何类库。AspectJ只是多了一些关键词罢了。
2.或者使用纯Java语言开发，然后使用AspectJ注解，简称@AspectJ。

#### Join Points介绍
Join Points（以后简称JPoints）是AspectJ中最关键的一个概念。什么是JPoints呢？JPoints就是程序运行时的一些执行点。那么，一个程序中，哪些执行点是JPoints呢？比如：
> l. 一个函数的调用可以是一个JPoint。比如Log.e()这个函数。e的执行可以是一个JPoint，而调用e的函数也可以认为是一个JPoint。
2.设置一个变量，或者读取一个变量，也可以是一个JPoint。比如Demo类中有一个debug的boolean变量。设置它的地方或者读取它的地方都可以看做是JPoints。
3.for循环可以看做是JPoint。
理论上说，一个程序中很多地方都可以被看做是JPoint，但是AspectJ中，只有如表1所示的几种执行点被认为是JPoint

![结果显示](/uploads/AOP/JoinPoint.png)

#### Pointcuts介绍
> pointcuts这个单词不好翻译，此处直接用英文好了。那么，Pointcuts是什么呢？前面介绍的内容可知，一个程序会有很多的JPoints，即使是同一个函数（比如testMethod这个函数），
还分为call类型和execution类型的JPoint。显然，不是所有的JPoint，也不是所有类型的JPoint都是我们关注的。再次以AopDemo为例，我们只要求在Activity的几个生命周期函数中打印日志，
只有这几个生命周期函数才是我们业务需要的JPoint，而其他的什么JPoint我不需要关注。
怎么从一堆一堆的JPoints中选择自己想要的JPoints呢？恩，这就是Pointcuts的功能。一句话，Pointcuts的目标是提供一种方法使得开发者能够选择自己感兴趣的JoinPoints。

直接来看一个例子，现在我想把图2中的示例代码中，那些调用println的地方找到，该怎么弄？代码该这么写：
```java
public pointcut  testAll(): call(public  *  *.println(..)) && !within(TestAspect) ;  
```
注意，aspectj的语法和Java一样，只不过多了一些关键词

我们来看看上述代码
> 第一个public：表示这个pointcut是public访问。这主要和aspect的继承关系有关，属于AspectJ的高级玩法，本文不考虑。
pointcut：关键词，表示这里定义的是一个pointcut。pointcut定义有点像函数定义。总之，在AspectJ中，你得定义一个pointcut。
testAll()：pointcut的名字。在AspectJ中，定义Pointcut可分为有名和匿名两种办法。个人建议使用named方法。因为在后面，我们要使用一个pointcut的话，就可以直接使用它的名字就好。
testAll后面有一个冒号，这是pointcut定义名字后，必须加上。冒号后面是这个pointcut怎么选择Joinpoint的条件。
本例中，call(public  *  *.println(..))是一种选择条件。call表示我们选择的Joinpoint类型为call类型。

public  **.println(..)：这小行代码使用了通配符。
> 由于我们这里选择的JoinPoint类型为call类型，它对应的目标JPoint一定是某个函数。所以我们要找到这个/些函数。public  
表示目标JPoint的访问类型（public/private/protect）。第一个*表示返回值的类型是任意类型。第二个*用来指明包名。此处不限定包名。紧接其后的println是函数名。
这表明我们选择的函数是任何包中定义的名字叫println的函数。当然，唯一确定一个函数除了包名外，还有它的参数。在(..)中，就指明了目标函数的参数应该是什么样子的。
比如这里使用了通配符..，代表任意个数的参数，任意类型的参数。

再来看call后面的&&：
> AspectJ可以把几个条件组合起来，目前支持 &&，||，以及！这三个条件。这三个条件的意思不用我说了吧？和Java中的是一样的。
来看最后一个!within(TestAspectJ)：前面的!表示不满足某个条件。within是另外一种类型选择方法，特别注意，这种类型和前面讲到的joinpoint的那几种类型不同。within的类型是数据类型，
而joinpoint的类型更像是动态的，执行时的类型。

上例中的pointcut合起来就是：
> l 选择那些调用println（而且不考虑println函数的参数是什么）的Joinpoint。
2 另外，调用者的类型不要是TestAspect的。

#### 直接针对JoinPoint的选择
pointcuts中最常用的选择条件和Joinpoint的类型密切相关，比如图
![结果显示](/uploads/AOP/pointcutsAndJoinPoints.png)

以上图为例，如果我们想选择类型为methodexecution的JPoint，那么pointcuts的写法就得包括execution(XXX)来限定。
除了指定JPoint类型外，我们还要更进一步选择目标函数
> 选择的根据就是上图列出的什么MethodSignature，ConstructorSignature，TypeSinature，FieldSignature等。名字听起来陌生得很，
其实就是指定JPoint对应的函数（包括构造函数），Static block的信息。比如图4中的那个println例子，首先它的JPoint类型是call，所以它的查询条件是根据MethodSignature来表达。

一个Method Signature的完整表达式为：
> @注解 访问权限 返回值的类型 包名.函数名(参数)  
@注解和访问权限（public/private/protect，以及static/final）属于可选项。如果不设置它们，则默认都会选择。以访问权限为例，如果没有设置访问权限作为条件，那么public，private，protect及static、final的函数都会进行搜索。  
返回值类型就是普通的函数的返回值类型。如果不限定类型的话，就用*通配符表示  
包名.函数名用于查找匹配的函数。可以使用通配符，包括*和..以及+号。其中*号用于匹配除.号之外的任意字符，而..则表示任意子package，+号表示子类。  

比如:
> java.*.Date：可以表示java.sql.Date，也可以表示java.util.Date  
Test*：可以表示TestBase，也可以表示TestDervied  
java..*：表示java任意子类  
java..*Model+：表示Java任意package中名字以Model结尾的子类，比如TabelModel，TreeModel  等  
最后来看函数的参数。参数匹配比较简单，主要是参数类型，比如：  
(int, char)：表示参数只有两个，并且第一个参数类型是int，第二个参数类型是char  
(String, ..)：表示至少有一个参数。并且第一个参数类型是String，后面参数类型不限。在参数匹配中， ..代表任意参数个数和类型  
(Object ...)：表示不定个数的参数，且类型都是Object，这里的...不是通配符，而是Java中代表不定参数的意思 
	
Constructorsignature和Method Signature类似，只不过构造函数没有返回值，而且函数名必须叫new。比如： 
>  public *..TestDerived.new(..)：  
public：选择public访问权限  
*..代表任意包名  
TestDerived.new：代表TestDerived的构造函数  
(..)：代表参数个数和类型都是任意  

再来看Field Signature和Type Signature，用它们的地方见图5。下面直接上几个例子：  
> Field Signature标准格式：  
@注解 访问权限 类型 类名.成员变量名  
其中，@注解和访问权限是可选的  
类型：成员变量类型，*代表任意类型  
类名.成员变量名：成员变量名可以是*，代表任意成员变量  

比如，
> set(inttest..TestBase.base)：表示设置TestBase.base变量时的JPoint  
Type Signature：直接上例子  
staticinitialization(test..TestBase)：表示TestBase类的static block  
handler(NullPointerException)：表示catch到NullPointerException的JPoint。注意，图2的源码第23行截获的其实是Exception，其真实类型是NullPointerException。
但是由于JPointer的查询匹配是静态的，即编译过程中进行的匹配，所以handler(NullPointerException)在运行时并不能真正被截获。只有改成handler(Exception)，
或者把源码第23行改成NullPointerException才行。  	

#### 间接针对JPoint的选择

除了根据前面提到的Signature信息来匹配JPoint外，AspectJ还提供其他一些选择方法来选择JPoint。比如某个类中的所有JPoint，每一个函数执行流程中所包含的JPoint。
特别强调，不论什么选择方法，最终都是为了找到目标的JPoint。

下面列出了一些常用的非JPoint选择方法：

![结果显示](/uploads/AOP/简介选择JPoint.png)

#### advice和aspect介绍

恭喜，看到这个地方来，AspectJ的核心部分就掌握一大部分了。现在，我们知道如何通过pointcuts来选择合适的JPoint。那么，下一步工作就很明确了，选择这些JPoint后，我们肯定是需要干一些事情的。比如前面例子中的输出都有before，after之类的。这其实JPoint在执行前，执行后，都执行了一些我们设置的代码。在AspectJ中，这段代码叫advice。简单点说，advice就是一种Hook。

ASpectJ中有好几个Hook，主要是根据JPoint执行时机的不同而不同，比如下面的：
```java
before():testAll(){  
  System.out.println("before calling: " + thisJoinPoint);//打印这个JPoint的信息  
  System.out.println("      at:" + thisJoinPoint.getSourceLocation());//打印这个JPoint对应的源代码位置  
}  
```

testAll()是前面定义的pointcuts，而before()定义了在这个pointcuts选中的JPoint执行前我们要干的事情。下图列出了AspectJ所支持的Advice的类型：
![结果显示](/uploads/AOP/advice.png)

注意，after和before没有返回值，但是around的目标是替代原JPoint的，所以它一般会有返回值，而且返回值的类型需要匹配被选中的JPoint。我们来看个例子，见图7。

![结果显示](/uploads/AOP/around返回值.png)
> 第一个红框是修改后的testMethod，在这个testMethod中，肯定会抛出一个空指针异常。
第二个红框是我们配置的advice，除了before以外，还加了一个around。我们重点来看around，它的返回值是Object。虽然匹配的JPoint是testMethod，其定义的返回值是void。
但是AspectJ考虑的很周到。在around里，可以设置返回值类型为Object来表示返回任意类型的返回值。AspectJ在真正返回参数的时候，会自动进行转换。比如，假设inttestMethod定义了int作为返回值类型，
我们在around里可以返回一个Integer，AspectJ会自动转换成int作为返回值。

再看around中的//proceed()这句话。这代表调用真正的JPoint函数，即testMethod。由于这里我们屏蔽了proceed，所以testMethod真正的内容并未执行，故运行的时候空指针异常就不会抛出来。
也就是说，我们完全截获了testMethod的运行，甚至可以任意修改它，让它执行别的函数都没有问题。。

#### 参数传递和JPoint信息

到此，AspectJ最基本的东西其实讲差不多了，但是在实际使用AspectJ的时候，你会发现前面的内容还欠缺一点，尤其是advice的地方：

> 前面介绍的advice都是没有参数信息的，而JPoint肯定是或多或少有参数的。而且advice既然是对JPoint的截获或者hook也好，肯定需要利用传入给JPoint的参数干点什么事情。
比方所around advice，我可以对传入的参数进行检查，如果参数不合法，我就直接返回，根本就不需要调用proceed做处理。
往advice传参数比较简单，就是利用前面提到的this(),target(),args()等方法。另外，整个pointcuts和advice编写的语法也有一些区别。具体方法如下：

先在pointcuts定义时候指定参数类型和名字
```java
pointcut testAll(Test.TestDerived derived,int x):call(*Test.TestDerived.testMethod(..))  && target(derived)&& args(x)  
```
注意上述pointcuts的写法，首先在testAll中定义参数类型和参数名。这一点和定义一个函数完全一样,接着看target和args。此处的target和args括号中用得是参数名。而参数名则是在前面pointcuts中定义好的。这属于target和args的另外一种用法。

> 注意，增加参数并不会影响pointcuts对JPoint的匹配
上面的pointcuts选择和 pointcut testAll():call(*Test.TestDerived.testMethod(..)) && target(Test.TestDerived) &&args(int)是一样的  只不过我们需要把参数传入advice，才需要改造

接下来是修改advice：
```java
Object around(Test.TestDerived derived,int x):testAll(derived,x){  
    System.out.println("     arg1=" + derived);  
    System.out.println("     arg2=" + x);  
    return proceed(derived,x); //注意，proceed就必须把所有参数传进去。  
}  
```
advice的定义现在也和函数定义一样，把参数类型和参数名传进来。接着把参数名传给pointcuts，此处是testAll。注意，advice必须和使用的pointcuts在参数类型和名字上保持一致。然后在advice的代码中，你就可以引用参数了，比如derived和x，都可以打印出来。

总结，参数传递其实并不复杂，关键是得记住语法：
> pointcuts修改：像定义函数一样定义pointcuts，然后在this,target或args中绑定参数名（注意，不再是参数类型，而是参数名）。
advice修改：也像定义函数一样定义advice，然后在冒号后面的pointcuts中绑定参数名（注意是参数名）
在advice的代码中使用参数名。

### 使用AOP做权限申请
```java
不使用AOP做权限处理的时候

public class AopDemoActivity extends Activity {  
   private static final String TAG = "AopDemoActivity";  
 
   protected void onCreate(Bundle savedInstanceState) {  
       super.onCreate(savedInstanceState);  
       setContentView(R.layout.layout_main);  
       Log.e(TAG,"onCreate");  
    }  
   
    protectedvoid onResume() {  
       super.onResume();  
       Log.e(TAG, "onResume");  
       //checkPhoneState会检查app是否申明了android.permission.READ_PHONE_STATE权限
       checkPhoneState();  
    }  
 
   private boolean checkPermission(String permissionName){  
       try{  
           PackageManager pm = getPackageManager();  
           //调用PackageMangaer的checkPermission函数，检查自己是否申明使用某权限  
           int nret = pm.checkPermission(permissionName,getPackageName());  
           return nret == PackageManager.PERMISSION_GRANTED;  
        }......  
    }  
}  
```
我们需要在每一个使用的地方都要添加类似的代码,就算是抽离出一个模块，使用的时候也是有很大的耦合，比如当我要修改权限模块的时候，每一个使用的地方都要修改,使用AOP的处理：
```gradle
1.首先在项目主的build.gradle中添加配置
dependencies {
    classpath 'com.android.tools.build:gradle:3.0.1'
    classpath 'com.hujiang.aspectjx:gradle-android-plugin-aspectjx:1.1.0'(要添加的内容)
    classpath 'com.github.dcendents:android-maven-gradle-plugin:1.5'(要添加的内容)
}

2.新建一个android Library 然后在build.gradle 文件中添加配置

apply plugin: 'com.android.library'
apply plugin: 'com.github.dcendents.android-maven'(要添加的内容)

dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])
    implementation 'com.android.support:appcompat-v7:26.1.0'
    testImplementation 'junit:junit:4.12'
    androidTestImplementation 'com.android.support.test:runner:1.0.2'
    androidTestImplementation 'com.android.support.test.espresso:espresso-core:3.0.2'
	
    compile 'org.aspectj:aspectjrt:1.8.9'(要添加的内容)
}

3.然后在主工程的build.gradle 文件中添加配置

apply plugin: 'com.android.application'
apply plugin: 'android-aspectjx'(要添加的内容)

dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])
    implementation 'com.android.support:appcompat-v7:26.1.0'
    implementation 'com.android.support.constraint:constraint-layout:1.1.2'
    testImplementation 'junit:junit:4.12'
    androidTestImplementation 'com.android.support.test:runner:1.0.2'
    androidTestImplementation 'com.android.support.test.espresso:espresso-core:3.0.2'
	
    implementation project(":lib-permission")(要添加的内容，这个的意思是这个主工程要使用library库)
}
```

大致讲下思路：我们会新建一个透明的activity用来专门的处理权限的请求，因为这个透明的，所以用户是无感知的，而且这个Activity的调用时交由AspetJ来调用触发，最后将请求的权限的结果
通过反射的方式回调回去

libray代码的编写：
```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Permission {
    String[] value();//要请求的权限，这里支持传递一个权限的数组
    int requestCode() default PermissionUtils.DEFAULT_REQUEST_CODE; 
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface PermissionCanceled {
    int requestCode() default PermissionUtils.DEFAULT_REQUEST_CODE;
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface PermissionDenied {
    int requestCode() default PermissionUtils.DEFAULT_REQUEST_CODE;
}

//权限处理的回调
public interface IPermission {

    /**
     * 已经授权
     */
    void ganted();

    /**
     * 取消授权
     */
    void cancled();

    /**
     *被拒绝 点击了不再提示
     */
    void denied();

}

/**
 * 专门用来处理权限请求的用户无感知的 activity
 */
public class PermissionActivity extends Activity{

    private static final String PARAM_PERMISSION = "param_permission";
    private static final String PARAM_REQUEST_CODE = "param_request_code";

    private String[] mPermissions;
    private int mRequestCode;
    private static IPermission permissionListener;

    /**
     * 请求权限,由AspetJ来调用
     */
    public static void requestPermission(Context context, String[] permissions, int requestCode, IPermission iPermission) {
        permissionListener = iPermission;
		
        //启动当前的PermissionActivity ，传递要请求的权限
        Intent intent = new Intent(context, PermissionActivity.class);
        intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK | Intent.FLAG_ACTIVITY_CLEAR_TOP);
        Bundle bundle = new Bundle();
        bundle.putStringArray(PARAM_PERMISSION, permissions);
        bundle.putInt(PARAM_REQUEST_CODE, requestCode);
        intent.putExtras(bundle);
        context.startActivity(intent);
		
        //因为要做一个无感知的activity，所以要禁止掉界面跳转的动画
        if (context instanceof Activity) {
            ((Activity) context).overridePendingTransition(0, 0);
        }
    }

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        // 权限申请
        setContentView(R.layout.tim_permission_layout);
		
        this.mPermissions = getIntent().getStringArrayExtra(PARAM_PERMISSION);
        this.mRequestCode = getIntent().getIntExtra(PARAM_REQUEST_CODE, -1);

        if (mPermissions == null || mRequestCode < 0 || permissionListener == null) {
            this.finish();
            return;
        }

        //检查是否已授权
        if (PermissionUtils.hasPermission(this, mPermissions)) {
            permissionListener.ganted();
            finish();
            return;
        }
        ActivityCompat.requestPermissions(this, this.mPermissions, this.mRequestCode);

    }

    /**
     * grantResults对应于申请的结果，这里的数组对应于申请时的第二个权限字符串数组。
     * 如果你同时申请两个权限，那么grantResults的length就为2，分别记录你两个权限的申请结果
     */
    @Override
    public void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions, @NonNull int[] grantResults) {
        super.onRequestPermissionsResult(requestCode, permissions, grantResults);

        //请求权限成功
        if (PermissionUtils.verifyPermission(this, grantResults)) {
            permissionListener.ganted();
            finish();
            return;
        }

        //用户点击了不再显示
        if (!PermissionUtils.shouldShowRequestPermissionRationale(this, permissions)) {
            permissionListener.denied();
            finish();
            return;
        }

        //用户取消
        permissionListener.cancled();
        finish();
    }

    @Override
    public void finish() {
        super.finish();
        //因为要做一个无感知的activity，所以要禁止掉界面跳转的动画
        overridePendingTransition(0,0);
    }
}
```

因为要做一个透明的Activity，所以要设置透明的主题
```xml
<application>
    <activity android:name=".PermissionActivity" android:theme="@style/translucent"></activity>
</application>

布局也是做成一个透明的
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="@android:color/transparent">
</LinearLayout>
```
权限处理的工具类
```java
public class PermissionUtils {

    public static final int DEFAULT_REQUEST_CODE = 0xABC1994;

    private static SimpleArrayMap<String, Integer> MIN_SDK_PERMISSIONS;

    static {
        MIN_SDK_PERMISSIONS = new SimpleArrayMap<>(8);
        MIN_SDK_PERMISSIONS.put("com.android.voicemail.permission.ADD_VOICEMAIL", 14);
        MIN_SDK_PERMISSIONS.put("android.permission.BODY_SENSORS", 20);
        MIN_SDK_PERMISSIONS.put("android.permission.READ_CALL_LOG", 16);
        MIN_SDK_PERMISSIONS.put("android.permission.READ_EXTERNAL_STORAGE", 16);
        MIN_SDK_PERMISSIONS.put("android.permission.USE_SIP", 9);
        MIN_SDK_PERMISSIONS.put("android.permission.WRITE_CALL_LOG", 16);
        MIN_SDK_PERMISSIONS.put("android.permission.SYSTEM_ALERT_WINDOW", 23);
        MIN_SDK_PERMISSIONS.put("android.permission.WRITE_SETTINGS", 23);
    }


    /**
     * 检查是否需要请求权限
     * @param context
     * @param permissions
     * @return
     * false --- 需要  true ---不需要
     */
    public static boolean hasPermission(Context context, String ...permissions) {

        for (String permission : permissions) {
            if (permissionExists(permission) && !hasSelfPermission(context, permission)) {
                return false;
            }
        }
        return true;
    }

    /**
     * @Description 检测某个权限是否已经授权；如果已授权则返回true，如果未授权则返回false
     * @version
     */
    private static boolean hasSelfPermission(Context context, String permission) {
        try {
            // ContextCompat.checkSelfPermission，主要用于检测某个权限是否已经被授予。
            // 方法返回值为PackageManager.PERMISSION_DENIED或者PackageManager.PERMISSION_GRANTED
            // 当返回DENIED就需要进行申请授权了。
            return ContextCompat.checkSelfPermission(context, permission) == PackageManager.PERMISSION_GRANTED;
        } catch (RuntimeException t) {
            return false;
        }
    }

    /**
     * @Description 如果在这个SDK版本存在的权限，则返回true
     * @version
     */
    private static boolean permissionExists(String permission) {
        Integer minVersion = MIN_SDK_PERMISSIONS.get(permission);
        return minVersion == null || Build.VERSION.SDK_INT >= minVersion;
    }

	/**
     * @Description 检验返回的权限处理是否成功
     * 
     */
    public static boolean verifyPermission(Context context, int ... gantedResults) {

        if (gantedResults == null || gantedResults.length == 0 ) {
            return false;
        }

        for (int ganted : gantedResults) {
            if (ganted != PackageManager.PERMISSION_GRANTED) {
                return false;
            }
        }
        return true;
    }

    /**
     * @Description 检查需要给予的权限是否需要显示理由
     * @version
     */
    public static boolean shouldShowRequestPermissionRationale(Activity activity, String... permissions) {
        for (String permission : permissions) {
            // 这个API主要用于给用户一个申请权限的解释，该方法只有在用户在上一次已经拒绝过你的这个权限申请。
            // 也就是说，用户已经拒绝一次了，你又弹个授权框，你需要给用户一个解释，为什么要授权，则使用该方法。
            if (ActivityCompat.shouldShowRequestPermissionRationale(activity, permission)) {
                return true;
            }
        }
        return false;
    }


	/**
     * @Description 反射调用annotationClass 对应的方法
     * 
     */
    public static void invokAnnotation(Object object, Class annotationClass) {
        //获取切面上下文的类型
        Class<?> clz = object.getClass();
        //获取类型中的方法
        Method[] methods = clz.getDeclaredMethods();
        if (methods == null) {
            return;
        }
        for (Method method : methods) {
            //获取该方法是否有PermissionCanceled注解
            boolean isHasAnnotation = method.isAnnotationPresent(annotationClass);
            if (isHasAnnotation) {
                method.setAccessible(true);
                try {
                    method.invoke(object);
                } catch (IllegalAccessException e) {
                    e.printStackTrace();
                } catch (InvocationTargetException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}

@Aspect  //必须使用@AspectJ标注，这样class PermissionAspect就等同于 aspect PermissionAspect了  
public class PermissionAspect {

    private static final String TAG = "TimPermissionAspect";

    //寻找Jpoint
    //来看这个Pointcut，首先，它在选择Jpoint的时候，把@com.dn.tim.lib_permission.annotation.Permission使用上了，这表明所有那些public的，并且携带有这个注解的API都是目标JPoint 
    //接着，由于我们希望在函数中获取注解的信息，所有这里的poincut函数有一个参数，参数类型是 Permission，参数名为permission
    //这个参数我们需要在后面的advice里用上，所以pointcut还使用了@annotation(ann)这种方法来告诉 AspectJ，这个permission是一个注解 
    @Pointcut("execution(@com.dn.tim.lib_permission.annotation.Permission * *(..)) && @annotation(permission)")
    public void requestPermission(Permission permission) {

    }

    //接下来是advice，advice的真正功能由aroundJointPoint函数来实现，这个aroundJointPoint函数第二个参数就是我们想要 
    //的注解。在实际运行过程中，AspectJ会把这个信息从JPoint中提出出来并传递给aroundJointPoint函数。 
    @Around("requestPermission(permission)")
    public void aroundJointPoint(final ProceedingJoinPoint joinPoint, Permission permission) throws Throwable{

        //初始化context
        Context context = null;
        //获取到这个JointPoint 代表的this对象
        final Object object = joinPoint.getThis();
        if (joinPoint.getThis() instanceof Context) {
            context = (Context) object;
        } else if (joinPoint.getThis() instanceof android.support.v4.app.Fragment) {
            context = ((android.support.v4.app.Fragment) object).getActivity();
        } else if (joinPoint.getThis() instanceof android.app.Fragment) {
            context = ((android.app.Fragment) object).getActivity();
        } else {
        }

        if (context == null || permission == null) {
            Log.d(TAG, "aroundJonitPoint error ");
            return;
        }

        //调用启动透明的权限请求类
        final Context finalContext = context;
        TimPermissionActivity.requestPermission(context, permission.value(), permission.requestCode(), new IPermission() {
            @Override
            public void ganted() {
                try {
                    //around是替代了原JPoint，如果要执行原JPoint的话，需要调用proceed，也即是执行含有Permission注解的方法
                    joinPoint.proceed();
                } catch (Throwable throwable) {
                    throwable.printStackTrace();
                }
            }

            @Override
            public void cancled() {
                //反射调用含有PermissionCanceled 注解的方法
                PermissionUtils.invokAnnotation(object, PermissionCanceled.class);
            }

            @Override
            public void denied() {
                //反射调用含有PermissionDenied 注解的方法
                PermissionUtils.invokAnnotation(object, PermissionDenied.class);
                PermissionUtils.goToMenu(finalContext);
            }
        });
    }
}



@Permission({Manifest.permission.WRITE_EXTERNAL_STORAGE,Manifest.permission.CAMERA})
private void requestTwoPermission() {
    Toast.makeText(this, "请求两个权限成功（写和相机）", Toast.LENGTH_SHORT).show();
}
	
@PermissionCanceled()
private void cancel() {
    Log.i(TAG, "writeCancel: " );
    Toast.makeText(this, "cancel", Toast.LENGTH_SHORT).show();
}

@PermissionDenied()
private void deny() {
    Log.i(TAG, "writeDeny:");
    Toast.makeText(this, "deny", Toast.LENGTH_SHORT).show();
}
```


