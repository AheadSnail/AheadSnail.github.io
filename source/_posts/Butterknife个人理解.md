---
layout: pager
title: Butterknife个人理解
date: 2018-06-03 20:52:18
tags: [Android,Butterknife,Apt]
description:  Butterknife个人理解
---

Butterknife个人理解
<!--more-->

****什么是APT？****
===
```
APT(Annotation Processing Tool)是一种处理注释的工具,它对源代码文件进行检测找出其中的Annotation，使用Annotation进行额外的处理。
Annotation处理器在处理Annotation时可以根据源文件中的Annotation生成额外的源文件和其它的文件(文件具体内容由Annotation处理器的编写者决定),
APT还会编译生成的源文件和原来的源文件，将它们一起生成class文件,简而言之就是说apt可以帮我们搜索我们需要处理的注解，这个时机点发生在编译阶段，也就是由java文件，编译成class阶段
注意这个时间点，apt处理的时候，对应的class文件是没有生成的
```

****APT使用场景****
===
```
Android注解越来越引领潮流，比如 Dagger2, ButterKnife, EventBus3 等，他们都是注解类型，而且他们都有个共同点就是编译时生成代码，而不是运行时利用反射，这样大大优化了性能；
而这些框架都用到了同一个工具就是：APT(Annotation Processing Tool )，可以在代码编译期解析注解，并且生成新的 Java 文件，减少手动的代码输入
```

****Butterknife简单使用****
===
```java
1.在工程的build.gradle中导入butterknife插件 

buildscript {
    repositories {
        jcenter()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:2.3.0'
        //加入下面这段代码
        classpath 'com.jakewharton:butterknife-gradle-plugin:8.5.1'
    }
}

allprojects {
    repositories {
        jcenter()
    }
}

2.在项目的build.gradle中添加butterknife的插件，即是app中的builde.gradle 

apply plugin: 'com.android.application'
//加入下面这段代码
apply plugin: 'com.jakewharton.butterknife'

3.在项目的build.gradle中添加依赖，然后同步项目，即可下载butterknife库至项目中

dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])

    compile 'com.android.support:appcompat-v7:25.2.0'

    //加入下面这两行代码
    compile 'com.jakewharton:butterknife:8.5.1'
    annotationProcessor 'com.jakewharton:butterknife-compiler:8.5.1'

}

执行绑定: ButterKnife.bind(this);注意这个绑定的操作要放在setContentView()的后面,因为本质还是调用了findViewById的操作，所以要放在后面
@Override 
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.simple_activity);
    ButterKnife.bind(this);
}

//使用BindView来初始化控件
@BindView(R.id.title) TextView title;
```


****Butterknife源码分析****
===
```java
首先看BindView这个注解的声明：

@Retention(CLASS)
//标识作用在字段上面
@Target(FIELD)
public @interface BindView {
  //代表当前要绑定的资源id
  @IdRes int value();
}

之后好像就执行了一句ButterKnife.bind(this);，分析具体的实现：

@NonNull @UiThread
public static Unbinder bind(@NonNull Activity target) {
    //获取到根view
    View sourceView = target.getWindow().getDecorView();
    return createBinding(target, sourceView);
}

createBinding(target, sourceView);函数实现：

private static Unbinder createBinding(@NonNull Object target, @NonNull View source) {
    Class<?> targetClass = target.getClass();
    if (debug) Log.d(TAG, "Looking up binding for " + targetClass.getName());
	
    //找到targetClass对应的构造器
    Constructor<? extends Unbinder> constructor = findBindingConstructorForClass(targetClass);

    if (constructor == null) {
      return Unbinder.EMPTY;
    }

    //noinspection TryWithIdenticalCatches Resolves to API 19+ only type.
    try {
    //找到了构造器，就直接的new一个对象,同时传递了target，跟source进去
      return constructor.newInstance(target, source);
    } catch (IllegalAccessException e) {
      throw new RuntimeException("Unable to invoke " + constructor, e);
    } catch (InstantiationException e) {
      throw new RuntimeException("Unable to invoke " + constructor, e);
    } catch (InvocationTargetException e) {
      Throwable cause = e.getCause();
      if (cause instanceof RuntimeException) {
        throw (RuntimeException) cause;
      }
      if (cause instanceof Error) {
        throw (Error) cause;
      }
      throw new RuntimeException("Unable to create binding instance.", cause);
    }
}
  
首先分析 findBindingConstructorForClass(targetClass); 函数的实现为：  
private static Constructor<? extends Unbinder> findBindingConstructorForClass(Class<?> cls) {
    //首先从缓存的集合查找是否已经存在
    Constructor<? extends Unbinder> bindingCtor = BINDINGS.get(cls);
    //如果存在就直接返回
    if (bindingCtor != null) {
      if (debug) Log.d(TAG, "HIT: Cached in binding map.");
      return bindingCtor;
    }
    //获取到目标clas的类名
    String clsName = cls.getName();
	
    //过滤掉如果是以android. 或者java.开头的类
    if (clsName.startsWith("android.") || clsName.startsWith("java.")) {
      if (debug) Log.d(TAG, "MISS: Reached framework class. Abandoning search.");
      return null;
    }
    try {
      //加载目标类，目标类的形式为：clsName + "_ViewBinding" 如果当前是SimpleActivity 那么要加载的目标类就为SimpleActivity_ViewBinding
      Class<?> bindingClass = cls.getClassLoader().loadClass(clsName + "_ViewBinding");
      //得到目标类的结构体
      bindingCtor = (Constructor<? extends Unbinder>) bindingClass.getConstructor(cls, View.class);
      if (debug) Log.d(TAG, "HIT: Loaded binding class and constructor.");
    } catch (ClassNotFoundException e) {
      if (debug) Log.d(TAG, "Not found. Trying superclass " + cls.getSuperclass().getName());
      bindingCtor = findBindingConstructorForClass(cls.getSuperclass());
    } catch (NoSuchMethodException e) {
      throw new RuntimeException("Unable to find binding constructor for " + clsName, e);
    }
    //然后将找到的存储到集合中，方便下次使用的时候，直接的获取到,而不用再次的查找
    BINDINGS.put(cls, bindingCtor);
    return bindingCtor;
}

看了上面的代码可以发现执行 ButterKnife.bind(this);的操作就是加载一个以当前this对象，得到一个对应的目标类，目标类的形式为:this类名+ "_ViewBinding"
将这个类加载进来，然后得到对应的构造函数，然后执行这个对象的构造，同时传递了当前的this对象，当前界面的根view进去,可能有人会好奇，我们的当前的这个项目里面
根本没有这个类的存在，那么他在执行加载这个类的时候，是怎么找到的，原来这个类是由ButterKnife帮我们生成的，在编译的时候，生成这个文件，对应的目录为build/apt/目录下
```
生成目标类对应存在的地方，已经构造函数的声明
![结果显示](/uploads/Apt/目标类.jpg)

```java
经过上图可以知道，果然ButterKnife帮我们生成了一个目标类，而我们要加载的这个类就是他，经过上面我们知道我们还会调用这个类的构造函数，传递了目标对象，已经跟布局进去
对应的代码为：

@UiThread
public SimpleActivity_ViewBinding(final SimpleActivity target, View source) {
    this.target = target;

    View view;他
    //给title变量初始化
    target.title = Utils.findRequiredViewAsType(source, R.id.title, "field 'title'", TextView.class);
	.....
}

这里的target.title就为我们在使用的时候使用注解 @BindView(R.id.title) TextView title; 声明的变量，看来他的初始化操作是后面代码实现的
Utils.findRequiredViewAsType(source, R.id.title, "field 'title'", TextView.class); 函数的实现为：

public static <T> T findRequiredViewAsType(View source, @IdRes int id, String who,
      Class<T> cls) {
    //得到资源id对应的view对象
    View view = findRequiredView(source, id, who);
    //强制为对应的类型
    return castView(view, id, who, cls);
}

public static View findRequiredView(View source, @IdRes int id, String who) {
    //最终还是调用了findViewById实现的
    View view = source.findViewById(id);
    if (view != null) {
      return view;
    }
    String name = getResourceEntryName(source, id);
    throw new IllegalStateException("Required view '"
        + name
        + "' with ID "
        + id
        + " for "
        + who
        + " was not found. If this view is optional add '@Nullable' (fields) or '@Optional'"
        + " (methods) annotation.");
}

castView(view, id, who, cls);函数的实现：
public static <T> T castView(View view, @IdRes int id, String who, Class<T> cls) {
    try {
      //强转类型
      return cls.cast(view);
    } catch (ClassCastException e) {
      String name = getResourceEntryName(view, id);
      throw new IllegalStateException("View '"
          + name
          + "' with ID "
          + id
          + " for "
          + who
          + " was of the wrong type. See cause for more info.", e);
    }
}

经过上面分析我们可以知道，我们最终给这个使用了注解的变量赋值的操作，最终还是通过findviewById的操作来完成的，然后执行类型的转换，最后给这个变量赋值,所以我们最后的
疑点就是这个类是怎么样生成的，这个就是使用了apt技术,在Butterknife的源码中有一个butterknife-compiler java library，其中有一个这样的文件 ButterKnifeProcessor,这就是
产生这个文件的入口

//声明java编译器要处理的注解处理器为当前的这个类 ButterKnifeProcessor,要不然编译器不知道你是使用哪个注解处理器来处理,所以要指定一个
@AutoService(Processor.class)
public final class ButterKnifeProcessor extends AbstractProcessor {
  ....
  //初始化的操作,可以得到一些有用的工具类
  @Override public synchronized void init(ProcessingEnvironment env) {
    super.init(env);
	.....
    typeUtils = env.getTypeUtils();
    filer = env.getFiler();
    try {
      trees = Trees.instance(processingEnv);
    } catch (IllegalArgumentException ignored) {
    }
  }

  //要指定 支持的java sdk的版本号
  @Override public Set<String> getSupportedOptions() {
    return ImmutableSet.of(OPTION_SDK_INT, OPTION_DEBUGGABLE);
  }

  //标识当前的处理器要处理的注解有
  @Override public Set<String> getSupportedAnnotationTypes() {
    Set<String> types = new LinkedHashSet<>();
    for (Class<? extends Annotation> annotation : getSupportedAnnotations()) {
      types.add(annotation.getCanonicalName());
    }
    return types;
  }

  //当前的这个注解处理器，要负责的注解类型
  private Set<Class<? extends Annotation>> getSupportedAnnotations() {
    Set<Class<? extends Annotation>> annotations = new LinkedHashSet<>();
    annotations.add(BindAnim.class);
    annotations.add(BindArray.class);
    annotations.add(BindBitmap.class);
    annotations.add(BindBool.class);
    annotations.add(BindColor.class);
    annotations.add(BindDimen.class);
    annotations.add(BindDrawable.class);
    annotations.add(BindFloat.class);
    annotations.add(BindFont.class);
    annotations.add(BindInt.class);
    annotations.add(BindString.class);
    annotations.add(BindView.class);
    annotations.add(BindViews.class);
    annotations.addAll(LISTENERS);
    return annotations;
  }
  ....
  
  //这个方法是这个注解处理器，帮我们查找分析我们的代码之后，找到我们需要的注解，返回的结果
  @Override public boolean process(Set<? extends TypeElement> elements, RoundEnvironment env) {

    //找到必要的信息
    Map<TypeElement, BindingSet> bindingMap = findAndParseTargets(env);
    // 循环输出生成的文件
    for (Map.Entry<TypeElement, BindingSet> entry : bindingMap.entrySet()) {

       //获取到类的Element
      TypeElement typeElement = entry.getKey();
      //获取到对应的BindingSet对象
      BindingSet binding = entry.getValue();

      //生成对应的java文件
      JavaFile javaFile = binding.brewJava(sdk, debuggable);
      try {
        //写java文件
        javaFile.writeTo(filer);
      } catch (IOException e) {
        error(typeElement, "Unable to write binding for type %s: %s", typeElement, e.getMessage());
      }
    }
    return false;
  }  
}

所以首先分析 findAndParseTargets(env);实现
private Map<TypeElement, BindingSet> findAndParseTargets(RoundEnvironment env) {
    Map<TypeElement, BindingSet.Builder> builderMap = new LinkedHashMap<>();
    Set<TypeElement> erasedTargetNames = new LinkedHashSet<>();
	
	....
	//处理BindView注解的集合 env.getElementsAnnotatedWith(BindView.class)能得到当前的代码中使用了BindView注解的Element对象集合
    for (Element element : env.getElementsAnnotatedWith(BindView.class)) {
      try {
        //解析BindView注解,存储必要的信息
        parseBindView(element, builderMap, erasedTargetNames);
      } catch (Exception e) {
        logParsingError(element, BindView.class, e);
      }
    }
	....
}

env.getElementsAnnotatedWith(BindView.class) 能得到当前的代码中使用了BindView注解的Element对象集合,这里介绍下Element

Element元素，源代码中的每一个部分都是一个特定的元素类型，分别代表了包，类，方法等，下面是一个列子

package com.example;//PackageElement

public class Foo()//TypeElement
{
	private int a;//VariableElement
	private Foo other;
	
	public Foo(){};//ExecuteableElement
	
	public void setA(int a)
	{
		this.a = a;
	}
}

这些Element元素，相当于XML中d中的DOM树，可以通过一个元素去访问它的父元素，或者子元素
element.getEnclosingElement();//获取父元素
element.getEnclosedElement();//获取子元素

上面就是Element的大概了

for (Element element : env.getElementsAnnotatedWith(BindView.class)) {
    try {
        //解析BindView注解,存储必要的信息
        parseBindView(element, builderMap, erasedTargetNames);
    } catch (Exception e) {
        logParsingError(element, BindView.class, e);
    }
}

parseBindView(element, builderMap, erasedTargetNames);函数实现

//解析BindView
private void parseBindView(Element element, Map<TypeElement, BindingSet.Builder> builderMap, Set<TypeElement> erasedTargetNames) {

    //获取到父类的Element，因为BindView只能做用在成员变量上面，所以本身的type即为ValibaElement类型，所以父类即为TypeElement，代表一个类
    TypeElement enclosingElement = (TypeElement) element.getEnclosingElement();

    // Start by verifying common generated code restrictions.
    //检查是否合格，这里检查成员属性不能为private,static,类不能不是一个class类型，不能含有private,检查是否绑定的合法，不能绑定在android.或者java.开头的
    boolean hasError = isInaccessibleViaGeneratedCode(BindView.class, "fields", element) || isBindingInWrongPackage(BindView.class, element);

    // Verify that the target type extends from View.
    //获取到当前属性的类型 比如我们是这样使用 @BindView(R.id.title) TextView title; 那么这个type就为TextView
    TypeMirror elementType = element.asType();
    if (elementType.getKind() == TypeKind.TYPEVAR) {
      TypeVariable typeVariable = (TypeVariable) elementType;
      elementType = typeVariable.getUpperBound();
    }

    //获取到对应类的类名
    Name qualifiedName = enclosingElement.getQualifiedName();
    //获取到属性名   比如我们是这样使用 @BindView(R.id.title) TextView title; 那么这个simpleName就为 title
    Name simpleName = element.getSimpleName();

    // 检查注解元素的类型是否View或者是接口
    if (!isSubtypeOfType(elementType, VIEW_TYPE) && !isInterface(elementType)) {
      if (elementType.getKind() == TypeKind.ERROR) {
        note(element, "@%s field with unresolved type (%s) "
                + "must elsewhere be generated as a View or interface. (%s.%s)",
            BindView.class.getSimpleName(), elementType, qualifiedName, simpleName);
      } else {
        error(element, "@%s fields must extend from View or be an interface. (%s.%s)",
            BindView.class.getSimpleName(), qualifiedName, simpleName);
        hasError = true;
      }
    }

    //如果上面有错误就返回了
    if (hasError) {
       return;
    }

    // Assemble information on the field.
    //获取到注解BindView中设置的值,也即是传递的id 比如我们是这样使用@BindView(R.id.title) TextView title 那么这个id就为R.id.title
    int id = element.getAnnotation(BindView.class).value();

    //以当前注解的类为key从builderMap中获取到对应的 BindingSet.Builder
    BindingSet.Builder builder = builderMap.get(enclosingElement);
    //将获取到的id注解的值，转为一个对象Id
    Id resourceId = elementToId(element, BindView.class, id);

    //如果获取到的builder不为空,也即是当前类中的第二个成员属性要添加进来
    if (builder != null) {
      //是否之前已经存储过,当前再次的添加，不允许
      String existingBindingName = builder.findExistingBindingName(resourceId);
      if (existingBindingName != null) {
        error(element, "Attempt to use @%s for an already bound ID %d on '%s'. (%s.%s)",
            BindView.class.getSimpleName(), id, existingBindingName,
            enclosingElement.getQualifiedName(), element.getSimpleName());
        return;
      }
    } else {
      //如果为空，则创建一个BindingSet对象
      builder = getOrCreateBindingBuilder(builderMap, enclosingElement);
    }

    //代表属性名
    String name = simpleName.toString();
    //当前属性的类型
    TypeName type = TypeName.get(elementType);
    //判断当前属性是否含有@Nullable注解,如果没有就返回true,有就返回fasle
    boolean required = isFieldRequired(element);

    //首先构建一个FieldViewBinding对象,然后添加到builder中，以resourceId
    builder.addField(resourceId, new FieldViewBinding(name, type, required));

    // Add the type-erased version to the valid binding targets set.
    erasedTargetNames.add(enclosingElement);
}

boolean hasError = isInaccessibleViaGeneratedCode(BindView.class, "fields", element) || isBindingInWrongPackage(BindView.class, element);错误的检查，对应的函数实现

private boolean isInaccessibleViaGeneratedCode(Class<? extends Annotation> annotationClass,
      String targetThing, Element element) {
    boolean hasError = false;

    //获取到父类的Element的类型,由于当前的Element为ValibaElement，所以父类即为TypeElement，用来代表一个类
    TypeElement enclosingElement = (TypeElement) element.getEnclosingElement();

    // Verify field or method modifiers.
    //成员变量不能是私有的，而且不能是static类型的
    Set<Modifier> modifiers = element.getModifiers();
    if (modifiers.contains(PRIVATE) || modifiers.contains(STATIC)) {
      error(element, "@%s %s must not be private or static. (%s.%s)",
          annotationClass.getSimpleName(), targetThing, enclosingElement.getQualifiedName(),
          element.getSimpleName());
      hasError = true;
    }

    // Verify containing type.
    //如果父类不是一个class类型
    if (enclosingElement.getKind() != CLASS) {
      error(enclosingElement, "@%s %s may only be contained in classes. (%s.%s)",
          annotationClass.getSimpleName(), targetThing, enclosingElement.getQualifiedName(),
          element.getSimpleName());
      hasError = true;
    }

    // Verify containing class visibility is not private.
    //父类class的修饰符不能有private
    if (enclosingElement.getModifiers().contains(PRIVATE)) {
      error(enclosingElement, "@%s %s may not be contained in private classes. (%s.%s)",
          annotationClass.getSimpleName(), targetThing, enclosingElement.getQualifiedName(),
          element.getSimpleName());
      hasError = true;
    }
    return hasError;
}

//检查要绑定的这个类是否合法，比如不能绑定在以android.开头的或者以java.开头的
private boolean isBindingInWrongPackage(Class<? extends Annotation> annotationClass,
      Element element) {

    //获取到父类的Element，代表一个类的类型
    TypeElement enclosingElement = (TypeElement) element.getEnclosingElement();
    //获取到类名
    String qualifiedName = enclosingElement.getQualifiedName().toString();

    //类名中不能是以android.开头的
    if (qualifiedName.startsWith("android.")) {
      error(element, "@%s-annotated class incorrectly in Android framework package. (%s)",
          annotationClass.getSimpleName(), qualifiedName);
      return true;
    }

    //类名中也不能是以java.开头的
    if (qualifiedName.startsWith("java.")) {
      error(element, "@%s-annotated class incorrectly in Java framework package. (%s)",
          annotationClass.getSimpleName(), qualifiedName);
      return true;
    }

    return false;
} 

之后执行 builder = getOrCreateBindingBuilder(builderMap, enclosingElement); 细节实现为：

//得到或者创建一个BindingSet对象，并且添加到builderMap中
private BindingSet.Builder getOrCreateBindingBuilder(Map<TypeElement, BindingSet.Builder> builderMap, TypeElement enclosingElement) {
    BindingSet.Builder builder = builderMap.get(enclosingElement);
    if (builder == null) {
      //真正的构建一个BindingSet对象
      builder = BindingSet.newBuilder(enclosingElement);
      //将创建的BindingSet对象，添加到builderMap中
      builderMap.put(enclosingElement, builder);
    }
    //返回这个builder对象
    return builder;
}

builder = BindingSet.newBuilder(enclosingElement);函数的实现为:

//创建一个Builder对象,传递的为能代表当前类的Element对象
static Builder newBuilder(TypeElement enclosingElement) {

    //获取到当前类的类型
    TypeMirror typeMirror = enclosingElement.asType();

    //判断是否为view,判断的原理也只是简单的判断typeMirror.toString字符串是否跟android.view.View相等
    boolean isView = isSubtypeOfType(typeMirror, VIEW_TYPE);
    //判断是否为Activity类型
    boolean isActivity = isSubtypeOfType(typeMirror, ACTIVITY_TYPE);
    //判断是否为Dialog类型
    boolean isDialog = isSubtypeOfType(typeMirror, DIALOG_TYPE);

    //获取到对应的类型
    TypeName targetType = TypeName.get(typeMirror);
    //如果当前的类型为泛型
    if (targetType instanceof ParameterizedTypeName) {
      targetType = ((ParameterizedTypeName) targetType).rawType;
    }

    //获取到当前类的包名,根据当前属于TypeElement，再上一级也即是到了 PackageElement，代表包的Element，这个肯定能获取到值
    String packageName = getPackage(enclosingElement).getQualifiedName().toString();
    //获取到全类名，这里是有包括包名的
    String className = enclosingElement.getQualifiedName().toString().substring(packageName.length() + 1).replace('.', '$');
    //获取到要创建目标类的全类名,类的简称为 className + "_ViewBinding"
    ClassName bindingClassName = ClassName.get(packageName, className + "_ViewBinding");

    //判断是否为final类型
    boolean isFinal = enclosingElement.getModifiers().contains(Modifier.FINAL);
    //然后构建一个Builder对象
    return new Builder(targetType, bindingClassName, isFinal, isView, isActivity, isDialog);
}

之后执行 builder.addField(resourceId, new FieldViewBinding(name, type, required));首先是构造一个FieldViewBinding对象

final class FieldViewBinding implements MemberViewBinding {
  //代表当前属性名
  private final String name;
  //代表当前的属性的类型
  private final TypeName type;
  private final boolean required;

  FieldViewBinding(String name, TypeName type, boolean required) {
    this.name = name;
    this.type = type;
    this.required = required;
  }
  ....
}

分析builder.addField()函数的实现：

//添加属性字段
void addField(Id id, FieldViewBinding binding) {
   getOrCreateViewBindings(id).setFieldBinding(binding);
}

//得到或者创建ViewBindBinder
private ViewBinding.Builder getOrCreateViewBindings(Id id) {
    //以id 为key 从viewIdMap中获取到对应的ViewBinding.Builder ,如果为空，则构建一个ViewBinding.Builder对象，然后添加到集合中
    ViewBinding.Builder viewId = viewIdMap.get(id);
    if (viewId == null) {
        viewId = new ViewBinding.Builder(id);
        viewIdMap.put(id, viewId);
    }
    return viewId;
}

viewIdMap 的声明为：
//用来存储当前类，对应的资源id为key，value为ViewBinding.Builder，这个对象封装了属性的基本信息
private final Map<Id, ViewBinding.Builder> viewIdMap = new LinkedHashMap<>();

函数 new ViewBinding.Builder(id);的实现为：

public static final class Builder {
    //保存对应的资源Id对象
    private final Id id;

    private final Map<ListenerClass, Map<ListenerMethod, Set<MethodViewBinding>>> methodBindings =
        new LinkedHashMap<>();

    //保存FieldViewBinding 对象
    FieldViewBinding fieldBinding;

    Builder(Id id) {
      this.id = id;
    }
	
    //设置FieldViewBinding 对象
    public void setFieldBinding(FieldViewBinding fieldBinding) {
      if (this.fieldBinding != null) {
        throw new AssertionError();
      }
      this.fieldBinding = fieldBinding;
    }
	....
}

所以builder.addField(resourceId, new FieldViewBinding(name, type, required));执行操作就是将属性的name，type等，封装成一个对象FieldViewBinding,然后以
当前属性的resourceId，保存在 ViewBinding.Builder 对象中的private final Map<Id, ViewBinding.Builder> viewIdMap = new LinkedHashMap<>()中

parseBindView()函数执行完毕之后，findAndParseTargets继续执行

//将builderMap中的 builder对象，相应的构建成一个BindSet对象,然后存储到bindingMap中，然后返回这个对象
Deque<Map.Entry<TypeElement, BindingSet.Builder>> entries = new ArrayDeque<>(builderMap.entrySet());
Map<TypeElement, BindingSet> bindingMap = new LinkedHashMap<>();
while (!entries.isEmpty()) {
    Map.Entry<TypeElement, BindingSet.Builder> entry = entries.removeFirst();

    TypeElement type = entry.getKey();
    BindingSet.Builder builder = entry.getValue();

    TypeElement parentType = findParentType(type, erasedTargetNames);
    if (parentType == null) {
        //将builder对象，构建成一个BindingSet对象
        bindingMap.put(type, builder.build());
    } else {
        BindingSet parentBinding = bindingMap.get(parentType);
        if (parentBinding != null) {
          builder.setParent(parentBinding);
          bindingMap.put(type, builder.build());
        } else {
          // Has a superclass binding but we haven't built it yet. Re-enqueue for later.
          entries.addLast(entry);
        }
    }
}

由于我们上面得到都是一个Build对象，所以要转成对应的对象，所以执行  bindingMap.put(type, builder.build());

//将Binder对象，构建成一个BindSet对象
BindingSet build() {
    //ImmutableList.builder() 实现为 return new Builder<E>();
    ImmutableList.Builder<ViewBinding> viewBindings = ImmutableList.builder();
	
    // Map<Id, ViewBinding.Builder> viewIdMap ，所以先将viewBinding.Builder调用build()方法得到一个ViewBinding对象，然后添加到集合中
    for (ViewBinding.Builder builder : viewIdMap.values()) {
        viewBindings.add(builder.build());
    }
    //构建一个BindingSet对象
    return new BindingSet(targetTypeName, bindingClassName, isFinal, isView, isActivity, isDialog,
        viewBindings.build(), collectionBindings.build(), resourceBindings.build(),
        parentBinding);
}

viewBindings.add(builder.build());实现为先将ViewBinding.Builder对象转成一个ViewBinding对象，然后添加到viewBindings 集合中
//构建成一个ViewBinding对象
public ViewBinding build() {
    return new ViewBinding(id, methodBindings, fieldBinding);
}

所以最终构建返回一个BindSet对象
final class BindingSet {
  ....
  //viewBinding 存储了当前类中所包含的成员属性的对象
  private final ImmutableList<ViewBinding> viewBindings;
  private final ImmutableList<FieldCollectionViewBinding> collectionBindings;
  private final ImmutableList<ResourceBinding> resourceBindings;
  private final BindingSet parentBinding;

  private BindingSet(TypeName targetTypeName, ClassName bindingClassName, boolean isFinal,
      boolean isView, boolean isActivity, boolean isDialog, ImmutableList<ViewBinding> viewBindings,
      ImmutableList<FieldCollectionViewBinding> collectionBindings,
      ImmutableList<ResourceBinding> resourceBindings, BindingSet parentBinding) {
    this.isFinal = isFinal;
    this.targetTypeName = targetTypeName;
    this.bindingClassName = bindingClassName;
    this.isView = isView;
    this.isActivity = isActivity;
    this.isDialog = isDialog;
    this.viewBindings = viewBindings;
    this.collectionBindings = collectionBindings;
    this.resourceBindings = resourceBindings;
    this.parentBinding = parentBinding;
  }
  ...
}

findAndParseTargets函数执行完毕

// 循环输出生成的文件
for (Map.Entry<TypeElement, BindingSet> entry : bindingMap.entrySet()) {

    //获取到类的Element
    TypeElement typeElement = entry.getKey();
    //获取到对应的BindingSet对象
    BindingSet binding = entry.getValue();

    //生成对应的java文件
    JavaFile javaFile = binding.brewJava(sdk, debuggable);
    try {
        //写java文件
        javaFile.writeTo(filer);
    } catch (IOException e) {
        error(typeElement, "Unable to write binding for type %s: %s", typeElement, e.getMessage());
    }
}

执行 binding.brewJava(sdk, debuggable);

JavaFile brewJava(int sdk, boolean debuggable) {
    return JavaFile.builder(bindingClassName.packageName(), createType(sdk, debuggable))
        .addFileComment("Generated code from Butter Knife. Do not modify!")
        .build();
}

createType(sdk, debuggable)函数的实现为：下面使用到的api是由javapoat的内容，具体网上有实现，主要完成的工作就是用来拼接类的内容，可以生成一个类

这个类是他帮我们动态生成的，这里先看一下，因为接下来要分析怎么拼接的
public class SimpleActivity_ViewBinding implements Unbinder {
  private SimpleActivity target;

  @UiThread
  public SimpleActivity_ViewBinding(SimpleActivity target) {
    this(target, target.getWindow().getDecorView());
  }

  @UiThread
  public SimpleActivity_ViewBinding(final SimpleActivity target, View source) {
    this.target = target;

    View view;
    target.title = Utils.findRequiredViewAsType(source, R.id.title, "field 'title'", TextView.class);
    }
}

private TypeSpec createType(int sdk, boolean debuggable) {
    //添加类名 还有public ,结果也即为 public class SimpleActivity_ViewBinding
    TypeSpec.Builder result = TypeSpec.classBuilder(bindingClassName.simpleName()).addModifiers(PUBLIC);
    //判断是否是final类型
    if (isFinal) {
      result.addModifiers(FINAL);
    }


    if (parentBinding != null) {
      result.superclass(parentBinding.bindingClassName);
    } else {
      //给这个类添加UNBINDER接口，结果即为  public class SimpleActivity_ViewBinding implements Unbinder
      result.addSuperinterface(UNBINDER);
    }

    //是否含有target属性
    if (hasTargetField()) {
      //有就添加一个target属性 结果即为: private SimpleActivity target;
      result.addField(targetTypeName, "target", PRIVATE);
    }

    //添加对应的构造函数
    if (isView) {
      result.addMethod(createBindingConstructorForView());
    } else if (isActivity) {
      //由于我们当前为Activity所以会进去这里，主要完成一个参数的构造函数的生成
      result.addMethod(createBindingConstructorForActivity());
    } else if (isDialog) {
      result.addMethod(createBindingConstructorForDialog());
    }

    if (!constructorNeedsView()) {
      // Add a delegating constructor with a target type + view signature for reflective use.
      result.addMethod(createBindingViewDelegateConstructor());
    }

    //添加俩个参数的构造函数的生成
    result.addMethod(createBindingConstructor(sdk, debuggable));


    if (hasViewBindings() || parentBinding == null) {
       result.addMethod(createBindingUnbindMethod(result));
    }

    return result.build();
}

createBindingConstructorForActivity()函数实现的细节为:
//给Activity添加一个构造函数,这里是添加一个参数 target的构造函数 ,对应的生成结果为：
//  @UiThread
//  public SimpleActivity_ViewBinding(SimpleActivity target) {
//    this(target, target.getWindow().getDecorView());
//  }
private MethodSpec createBindingConstructorForActivity() {
    MethodSpec.Builder builder = MethodSpec.constructorBuilder()
        .addAnnotation(UI_THREAD)//构造函数上面添加一个UiThread注解
        .addModifiers(PUBLIC)//添加public
        .addParameter(targetTypeName, "target");//添加一个参数为 target
    if (constructorNeedsView()) {
      builder.addStatement("this(target, target.getWindow().getDecorView())");//添加 this(target, target.getWindow().getDecorView());语句
    } else {
      builder.addStatement("this(target, target)");
    }
    return builder.build();
}

函数 result.addMethod(createBindingViewDelegateConstructor()); 实现细节
private MethodSpec createBindingConstructor(int sdk, boolean debuggable) {
    //创建构造函数
    MethodSpec.Builder constructor = MethodSpec.constructorBuilder()
        .addAnnotation(UI_THREAD)//添加UiThread注解
        .addModifiers(PUBLIC);

    if (hasMethodBindings()) {
      constructor.addParameter(targetTypeName, "target", FINAL);//对应的生成结果为给构造函数添加了一个参数 final SimpleActivity target
    } else {
      constructor.addParameter(targetTypeName, "target");
    }

    if (constructorNeedsView()) {
      constructor.addParameter(VIEW, "source");//对应的生成结果为给构造函数添加了一个参数 View source
    } else {
      constructor.addParameter(CONTEXT, "context");
    }

	....
    if (hasTargetField()) {
      constructor.addStatement("this.target = target");//对应的生成代码即为 this.target = target;
      constructor.addCode("\n");
    }

    if (hasViewBindings()) {
      if (hasViewLocal()) {
        // Local variable in which all views will be temporarily stored.
        constructor.addStatement("$T view", VIEW); //对应的生成代码即为 View view;
    }

    //解析属性字段的集合
    for (ViewBinding binding : viewBindings) {
       addViewBinding(constructor, binding, debuggable);
    }

	....
    return constructor.build();
}

//添加属性字段对应的代码 ，对应生成的代码，生成的结果要为target.title = Utils.findRequiredViewAsType(source, R.id.title, "field 'title'", TextView.class);
private void addViewBinding(MethodSpec.Builder result, ViewBinding binding, boolean debuggable) {
    if (binding.isSingleFieldBinding()) {
      // Optimize the common case where there's a single binding directly to a field.
      FieldViewBinding fieldBinding = binding.getFieldBinding();

      CodeBlock.Builder builder = CodeBlock.builder()
          .add("target.$L = ", fieldBinding.getName());//对应的生成结果为 target.title =

      boolean requiresCast = requiresCast(fieldBinding.getType());
      if (!debuggable || (!requiresCast && !fieldBinding.isRequired())) {//非调试模式，不会进去
        if (requiresCast) {
          builder.add("($T) ", fieldBinding.getType());
        }
        builder.add("source.findViewById($L)", binding.getId().code);
      } else {
        builder.add("$T.find", UTILS);//对应的生成代码为 Utils.find
        builder.add(fieldBinding.isRequired() ? "RequiredView" : "OptionalView");//对应的生成代码为Utils.findRequiredView
        if (requiresCast) {
          builder.add("AsType");//对应的生成代码Utils.findRequiredViewAsType
        }
        builder.add("(source, $L", binding.getId().code);//对应的生成代码为 Utils.findRequiredViewAsType(source, R.id.title
        if (fieldBinding.isRequired() || requiresCast) {
          builder.add(", $S", asHumanDescription(singletonList(fieldBinding)));//生成的结果为 Utils.findRequiredViewAsType(source, R.id.title, "field 'title'",
        }
        if (requiresCast) {
          //生成的结果为Utils.findRequiredViewAsType(source, R.id.title, "field 'title'", TextView.class
          builder.add(", $T.class", fieldBinding.getRawType());
        }
        builder.add(")");
      }
      result.addStatement("$L", builder.build());
      //直接返回
      return;
    }
   ....
}

上面就是Butterknife怎么帮我们动态的生成一个这样类的过程
```

****实现BindView简单版****
===
```java
1.首先在根目录的build.gradle 中添加下面的代码
buildscript {
    
    repositories {
        google()
        jcenter()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:2.3.0'
        //要添加的内容
        classpath 'com.neenbedankt.gradle.plugins:android-apt:1.8'


        // NOTE: Do not place your application dependencies here; they belong
        // in the individual module build.gradle files
    }
}

2.在app目录的build.gradle中添加

apply plugin: 'com.android.application'
apply plugin: 'com.neenbedankt.android-apt' //添加这行

之后创建俩个java library,app 引用这俩个model
compile project(':butterknife-annotion')

//指明这个可以用来处理app目录的注解
apt project(':butterkniffe-complier')

这个是在app代码中
public class MainActivity extends AppCompatActivity
{
    @BindView(R.id.tv1)
    public TextView mTv1;

    @Override
    protected void onCreate(Bundle savedInstanceState)
    {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        ButterKnife.bind(this);

        Toast.makeText(this,"--->"+mTv1.toString(),Toast.LENGTH_LONG).show();
    }
}


public interface ViewBinder<T>
{
    void bind(T target);
}

public class ButterKnife
{
    //Api调用的方法提供，绑定
    public static void bind(Activity activity)
    {
        //要加载的目标类
        String loadActivityName = activity.getClass().getSimpleName() + "$ViewBinder";
        try
        {
            ViewBinder instance = (ViewBinder) Class.forName(loadActivityName).newInstance();
            instance.bind(activity);
        }
        catch (Exception e)
        {
            e.printStackTrace();
        }
    }

}


//这个类是在 butterknife-annotion java library
@Retention(RetentionPolicy.RUNTIME)
//标识这个注解作用在成员变量上
@Target(ElementType.FIELD)
public @interface BindView
{
    //标识这个注解一定要传递参数
    int value();
}


这个是我们需要动态生成的类形式：,所以接下来就是拼接这个类的过程
package com.example.com.mybutterknifedemo;
import com.example.com.mybutterknifedemo.ViewBinder;

public class MainActivity$ViewBinder implements  ViewBinder<com.example.com.mybutterknifedemo.MainActivity> {
    public void bind( com.example.com.mybutterknifedemo.MainActivity target) {
	target.mTv1=(android.widget.TextView)target.findViewById(2131427445);
    }
}

ButterKnifeAnnotaionProcess 注解处理器，用来动态的生成目标类 这个是在butterkniffe-complier java library 中
这个java library module 要添加一个AutoService依赖,所以要在build.gradle 文件中，添加

dependencies {
    implementation fileTree(include: ['*.jar'], dir: 'libs')
    compile project(':butterknife-annotion')
    compile 'com.google.auto.service:auto-service:1.0-rc2'
}

tasks.withType(JavaCompile) {
    options.encoding = "UTF-8"
}



//标识编译的时候使用这个注解处理器来处理我们的注解，因为可能有多个这样的类，但是起作用的就是以这个注解为标识的
@AutoService(Processor.class)
public class ButterKnifeAnnotaionProcess extends AbstractProcessor
{
    private Types mTypeUtils;
    private Elements mElementUtils;
    private Filer mFiler;
    private Messager mMessager;

    @Override
    public synchronized void init(ProcessingEnvironment processingEnvironment)
    {
        super.init(processingEnvironment);
        //初始化我们需要的基础工具
        mTypeUtils = processingEnv.getTypeUtils();
        mElementUtils = processingEnv.getElementUtils();
        mFiler = processingEnv.getFiler();
        //我们可以通过这个工具来打印一些消息
        mMessager = processingEnv.getMessager();
    }

    //这个方法标识我们的这个注解处理器，需要处理什么样的注解,注意这是一个集合
    @Override
    public Set<String> getSupportedAnnotationTypes()
    {
        //这里我们标识我们这个注解处理器，能处理的注解为BindView注解
        Set<String> annotaionSets = new HashSet<>();
        annotaionSets.add(BindView.class.getCanonicalName());
        return annotaionSets;
    }

    //标识我们的注解处理器中java的版本号，注意这个注解处理器运行在java的环境中，并不是在android的环境，所以要指定java的环境,这里我们指定为java_7
    @Override
    public SourceVersion getSupportedSourceVersion()
    {
        return SourceVersion.RELEASE_8;
    }

    //之后，编译器就会将符合我们上面特点，收集他们的信息内容
    @Override
    public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment)
    {
        mMessager.printMessage(Diagnostic.Kind.NOTE, "sdfsdfsdfsdf");

        //这里的set集合代表我们要收集的注解的集合，因为这里我们只指定了一个注解BindView，所以这里只打印一个BindView
        Iterator<? extends TypeElement> iterator = set.iterator();
        while(iterator.hasNext())
        {
            TypeElement typeElement = iterator.next();
            System.out.println(typeElement.getQualifiedName());
        }

        //获取到含有BindView注解的Element集合，也即是当前BindView作用的元素所对应的Element集合,注意，这里是收集了所有的
        //含有这个注解的集合
        Set<? extends Element> annotatedWith = roundEnvironment.getElementsAnnotatedWith(BindView.class);

        //这个集合用来将所采集的全部的Element集合，用来分隔开，key为全类名，value为当前这个类所包含的这个Element的集合
        Map<String,List<VariableElement>> cacheMap = new HashMap<>();

        Iterator<? extends Element> annotaionIterator = annotatedWith.iterator();
        while (annotaionIterator.hasNext())
        {
            //因为我们的注解只是用在成员变量上面，所有这里返回的type
            VariableElement variableElement = (VariableElement) annotaionIterator.next();
            //获取到当前成员变量对应的全类名
            String activityName = getActivityName(variableElement);
            mMessager.printMessage(Diagnostic.Kind.NOTE, "activityName ==" + activityName);

            //获取分组，如果为空构建一个,然后添加到对应的分组中
            List<VariableElement> variableElements = cacheMap.get(activityName);
            if(variableElements == null)
            {
                variableElements = new ArrayList<>();
                cacheMap.put(activityName,variableElements);
            }
            variableElements.add(variableElement);
        }

        //到了这里分组完成,我们要根据分组的情况对应的生成类

        Iterator<String> cacheMapSetIterator = cacheMap.keySet().iterator();
        while(cacheMapSetIterator.hasNext())
        {
            //对应的activity的全类名,这个要用在泛型上面，所以一定要用这个值
            String activityName = cacheMapSetIterator.next();
            //当前activity一共有多少个成员变量的元素集合
            List<VariableElement> variableElements = cacheMap.get(activityName);
            //要动态生成的新类,注意这个是一定要包含类的包名的情况
            String newActivityBinder = activityName + "$ViewBinder";

            //生成类
            try
            {
                //创建一个类
                JavaFileObject sourceFile = mFiler.createSourceFile(newActivityBinder);
                Writer writer = sourceFile.openWriter();
                //获取到包名
                TypeElement typeElement = (TypeElement) variableElements.get(0).getEnclosingElement();
                String packageName = mElementUtils.getPackageOf(typeElement).getQualifiedName().toString();

                //注意这个是不包含包名的的名称
                String activitySimpleName = variableElements.get(0).getEnclosingElement().getSimpleName().toString() + "$ViewBinder";
                //类头
                writeHeader(writer, packageName, activityName, activitySimpleName);

                mMessager.printMessage(Diagnostic.Kind.NOTE,"size == "+ variableElements.size());

                //中间部分
                for (VariableElement variableElement : variableElements) {
                    //获取到成员变量上面的BindView注解，获取到对应的资源id
                    BindView bindView = variableElement.getAnnotation(BindView.class);
                    int id = bindView.value();
                    //获得当前成员变量的名称
                    String fieldName = variableElement.getSimpleName().toString();
                    //获取到类型,比如TextView，或者ImageView等
                    TypeMirror typeMirror=variableElement.asType();
                    //TextView
                    writer.write("target." + fieldName + "=(" + typeMirror.toString() + ")target.findViewById(" + id + ");");
                    writer.write("\n");
                }

                //结尾部分
                writer.write("\n");
                writer.write("}");
                writer.write("\n");
                writer.write("}");
                writer.close();
            }
            catch (IOException e)
            {
                e.printStackTrace();
            }
        }
        return false;
    }

    /**
     * 写类的头部分
     * @param writer
     * @param packageName
     * @param activityName       泛型中要填充类的全类名，这里一定要传递为全类名，要不然会出现问题
     * @param activitySimpleName 要生成的类的名字
     */
    private void writeHeader(Writer writer, String packageName, String activityName, String activitySimpleName) {
        try {
            writer.write("package "+packageName+";");
            writer.write("\n");
            //这个要自己导入，因为这个类是在主工程里面，我们生成的代码也是在主工程里面，所以这样写不会有问题，可以访问到
            writer.write("import com.example.com.mybutterknifedemo.ViewBinder;");
            writer.write("\n");
            writer.write("public class " + activitySimpleName + " implements  ViewBinder<" + activityName + "> {");
            writer.write("\n");
            writer.write(" public void bind( "+activityName+" target) {");
            writer.write("\n");
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    /**
     * 获取到当前成员变量对应的VariableElemnt对应的全类名
     * @param variableElement
     * @return
     */
    private String getActivityName(VariableElement variableElement)
    {
        //这个是获取到父类的节点，因为当前为VariableElement，所以父类也即是代表了TypeElement，这个是用来代表类的
        TypeElement typeElement = (TypeElement) variableElement.getEnclosingElement();
        //获取到当前元素对应的包名
        String packageName = mElementUtils.getPackageOf(typeElement).getQualifiedName().toString();
        //打印packageName
        mMessager.printMessage(Diagnostic.Kind.NOTE, "packageName ==" + packageName);
        String activityName = typeElement.getSimpleName().toString();
        return packageName+"."+activityName;
    }
}
```