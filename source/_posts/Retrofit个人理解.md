---
layout: pager
title: Retrofit个人理解
date: 2018-05-26 09:46:40
tags: [Android,OkHttp,Retrofit]
description:  Hermes个人理解
---

Retrofit个人理解
<!--more-->

简介
```
Retrofit做为一个目前java网络请求框架最牛逼的一个项目，网上关于他的介绍有很多，大体来说就是，通过他来执行网络请求，可以少写很多的代码。。支持数据的转换功能等
Retrofit本质是对于OkHttp的再次封装，Retrofit的源码是比较少的，网络请求这一块还是由OkHttp来执行
```

****Retrofit简单的使用****
===
```java
Retrofit retrofit = new Retrofit.Builder().baseUrl("https://api.github.com")
                                                  .addConverterFactory(GsonConverterFactory.create())
                                                  .build();
GitHub github = retrofit.create(GitHub.class);
Call<List<Contributor>> call = github.contributors("square", "retrofit");
call.enqueue(new Callback<List<Contributor>>()
{
    @Override
    public void onResponse(Call<List<Contributor>> call, Response<List<Contributor>> response)
    {
        for (Contributor contributor:response.body())
        {
            System.out.println(contributor.login + "(" + contributor.contributions + ")");
        }
    }

    @Override
    public void onFailure(Call<List<Contributor>> call, Throwable t)
    {

    }
});

```

执行的结果为：
![结果显示](/uploads/retrofit执行结果.png)

****Retrofit源码分析****
===
```java

首先分析
Retrofit retrofit = new Retrofit.Builder().baseUrl("https://api.github.com")
                                                  .addConverterFactory(GsonConverterFactory.create())
                                                  .build();
												  
构建Retrofit是使用构建者模式，所以这里首先要构建一个Builder对象，来完成默认参数的配置,这也是构建者默认的主要作用
public Builder() {
    this(Platform.get());
}												  

Platform.get() 函数的实现

static Platform get()
{
    return PLATFORM;
}

private static final Platform PLATFORM = findPlatform();

//找到对应的平台
private static Platform findPlatform()
{
    try
    {
        //如果Android的，构建一个Android的平台
        Class.forName("android.os.Build");
        if (Build.VERSION.SDK_INT != 0)
        {
            return new Android();
        }
    }
    catch (ClassNotFoundException ignored)
    {
    }
    try
    {
        Class.forName("java.util.Optional");
        return new Java8();
    }
    catch (ClassNotFoundException ignored)
    {
    }
    return new Platform();
}
通过上面的代码可知，对应Android环境来说，这里我们获取到的是一个Android平台的对象,这个类里面主要复写了一下父类的方法实现，之后用到里面内容的时候会介绍

所以当获取到Platform对象之后，赋值给Builder中的 platform成员变量
Builder(Platform platform) {
    this.platform = platform;
}

之后执行 baseUrl("https://api.github.com") 函数实现
public Builder baseUrl(String baseUrl) { 
   ....
   HttpUrl httpUrl = HttpUrl.parse(baseUrl);
   ....
   this.baseUrl = baseUrl; //将baseUrl封装成一个HttpUrl然后保存到Builder的成员变量baseUrl中
}

addConverterFactory(GsonConverterFactory.create())函数实现：
首先看GsonConverterFactory.create()得到的是什么对象
public static GsonConverterFactory create() {
    return create(new Gson());
}
public static GsonConverterFactory create(Gson gson) {
    if (gson == null) throw new NullPointerException("gson == null");
    return new GsonConverterFactory(gson);
}
private GsonConverterFactory(Gson gson) {
    this.gson = gson;
}
创建了一个GsonConverterFactory对象，这个对象里面保存了一个Gson对象
      
//添加转换器工厂，
public Builder addConverterFactory(Converter.Factory factory) {
    //添加到转换器集合中
    converterFactories.add(checkNotNull(factory, "factory == null"));
    return this;
}
converterFactories定义为：
转换器 比如GsonConverter 转换器集合 相当于是数据转换器
private final List<Converter.Factory> converterFactories = new ArrayList<>();
所以converterFactories成员变量集合中就有了一个GsonConverterFactory 对象

最后执行.build();
public Retrofit build() {
	//baseUrl是必须要设置的,前面我们已经设置了
    if (baseUrl == null) {
        throw new IllegalStateException("Base URL required.");
    }

    //如果callFactory为空，则构建一个，其本质为OkHttpClient 对象，因为OkHttpClient实现了Call.Factory接口,本质也是用他来执行网络请求的
    okhttp3.Call.Factory callFactory = this.callFactory;
    if (callFactory == null) {
        callFactory = new OkHttpClient();
    }

    //如果没有设置callbackExecutor对象，则才能够platform中获取，这里为Android 所以这个值为 MainThreadExecutor 对象
    Executor callbackExecutor = this.callbackExecutor;
    if (callbackExecutor == null) {
        callbackExecutor = platform.defaultCallbackExecutor();
    }
    
    //将callAdapterFactories集合中的内容拷贝到callAdapterFactories集合中 
    List<CallAdapter.Factory> callAdapterFactories = new ArrayList<>(this.callAdapterFactories);
    //platform 为Android 所以调用对应的函数 callbackExecutor 为MainThreadExecutor  所以这里添加的为 ExecutorCallAdapterFactory 对象
    callAdapterFactories.add(platform.defaultCallAdapterFactory(callbackExecutor));

    //构建一个集合对象，用来存储Converter.Factory对象
    List<Converter.Factory> converterFactories = new ArrayList<>(1 + this.converterFactories.size());
    //添加一个默认的 BuiltInConverters 对象
    converterFactories.add(new BuiltInConverters());
    //再添加用户自定义的,这里也即是添加我们上面分析的 GsonConverterFactory对象
    converterFactories.addAll(this.converterFactories);

    //构建一个Retrofit对象
    return new Retrofit(callFactory, baseUrl, unmodifiableList(converterFactories),
            unmodifiableList(callAdapterFactories), callbackExecutor, validateEagerly);
}

执行 platform.defaultCallbackExecutor(); 经我们前面介绍知道platform 对象为Android，所以会执行对应的函数 
public Executor defaultCallbackExecutor()
{
    return new MainThreadExecutor();
}
//线程池
static class MainThreadExecutor implements Executor
{
    //主线程的Handler，主要是为了切换到主线程操作
    private final Handler handler = new Handler(Looper.getMainLooper());

    @Override
    public void execute(Runnable r)
    {
        //切换到主线程
        handler.post(r);
    }
}
所以 platform.defaultCallbackExecutor()会得到一个MainThreadExecutor 对象，实现了Executor接口

然后执行  callAdapterFactories.add(platform.defaultCallAdapterFactory(callbackExecutor)); 这里的platform为Android会,执行对应的函数实现,同时callbackExecutor为上面创建的MainThreadExecutor
CallAdapter.Factory defaultCallAdapterFactory(@Nullable Executor callbackExecutor)
{
    if (callbackExecutor == null)//callbackExecutor 为上面创建的MainThreadExecutor 所以不为空
    {
        throw new AssertionError();
    }
    return new ExecutorCallAdapterFactory(callbackExecutor);
}
构建一个ExecutorCallAdapterFactory对象,之后再来分析这个类的作用
final class ExecutorCallAdapterFactory extends CallAdapter.Factory
{
    //存储 MainThreadExecutor对象
    final Executor callbackExecutor;

    //传递进来的参数 默认为 MainThreadExecutor 对象
    ExecutorCallAdapterFactory(Executor callbackExecutor)
    {
        this.callbackExecutor = callbackExecutor;
    }
	....
}
所以最后当  callAdapterFactories.add(platform.defaultCallAdapterFactory(callbackExecutor));执行完毕之后，这个集合里面就会存在一个成员变量为ExecutorCallAdapterFactory对象

执行
converterFactories.add(new BuiltInConverters());
converterFactories.addAll(this.converterFactories);

final class BuiltInConverters extends Converter.Factory {
	....
}
构建一个BuiltInConverters 对象，添加到集合中,同时添加了我们上面创建的 GsonConverterFactory对象，所以这个集合有俩个成员变量
最后执行  return new Retrofit(callFactory, baseUrl, unmodifiableList(converterFactories), unmodifiableList(callAdapterFactories), callbackExecutor, validateEagerly);
//Builder模式
Retrofit(okhttp3.Call.Factory callFactory, HttpUrl baseUrl,
             List<Converter.Factory> converterFactories, List<CallAdapter.Factory>
                     callAdapterFactories,
             @Nullable Executor callbackExecutor, boolean validateEagerly) {

    this.callFactory = callFactory;
    this.baseUrl = baseUrl;
    this.converterFactories = converterFactories; // Copy+unmodifiable at call site.
    this.callAdapterFactories = callAdapterFactories; // Copy+unmodifiable at call site.
    this.callbackExecutor = callbackExecutor;
    this.validateEagerly = validateEagerly;
}
将Builder里面的参数，配置到Retrofit对象中,返回一个Retrofit对象

接着我们执行  GitHub github = retrofit.create(GitHub.class);
public <T> T create(final Class<T> service) {
    //必须是一个接口 而且当前的这个借口不存在其他接口的继承关系
    Utils.validateServiceInterface(service);
	....
    //采用动态代理的方式：这里采用动态代理是为了获取到接口上的注解的信息 使用动态代理，只是单纯的为了拿到这个method上所有的注解
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[]{service},
        new InvocationHandler() {

        //这里获取到的是Android 平台的实现
        private final Platform platform = Platform.get();

        @Override
        public Object invoke(Object proxy, Method method, @Nullable Object[] args) throws Throwable {
            // If the method is a method from Object then defer to normal invocation.
            //如果当前调用的方法是属于object的，就直接调用
            if (method.getDeclaringClass() == Object.class) {
                return method.invoke(this, args);
            }
            //这里的platform 为 Android 这个对象的对应方法 没有复写，采用的是默认的实现为false
            if (platform.isDefaultMethod(method)) {
                return platform.invokeDefaultMethod(method, service, proxy, args);
            }
			
            //将当前执行的Method交给 ServiceMethod来处理
            ServiceMethod<Object, Object> serviceMethod = (ServiceMethod<Object, Object>) loadServiceMethod(method);
            //装处理Okhttp的 Call-> RealCall args代表调用这个方法传递的参数
            OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
            //执行函数 内部会使用适配器转换OKhttp的Response
            return serviceMethod.adapt(okHttpCall);
        }
    });
}

//请求的接口
public interface GitHub
{
    @GET("/repos/{owner}/{repo}/contributors?sort=desc")
    Call<List<Contributor>> contributors(@Path("owner") String owner, @Path("repo") String repo);
}

首先分析下这里为什么要创建一个动态代理对象，这里创建动态代理对象，主要是为了获取这个接口方法上面的注解参数类型，然后解析到这些注解的值，最后拼接成一个完整的url，这就是动态代理的好处
所以当我们的代码这样调用的时候  Call<List<Contributor>> call = github.contributors("square", "retrofit"); 就会执行到动态代理对象的InvocationHandler 回调函数invoke里面

当执行到   ServiceMethod<Object, Object> serviceMethod = (ServiceMethod<Object, Object>) loadServiceMethod(method);

//将当前的mothod 封装成一个ServiceMethod对象,并返回
ServiceMethod<?, ?> loadServiceMethod(Method method) {
    //首先根据key 为method 从缓存中查找，如果查找到了，就直接返回 定义为 private final Map<Method, ServiceMethod<?, ?>> serviceMethodCache = new ConcurrentHashMap<>();
    ServiceMethod<?, ?> result = serviceMethodCache.get(method);
    if (result != null) return result;
    //如果到了这里，就说明没有查找到
    synchronized (serviceMethodCache) {
        //双重检测，防止并发的时候出现异常
        result = serviceMethodCache.get(method);
        if (result == null) {
            //配置ServiceMethod的成员
            result = new ServiceMethod.Builder<>(this, method).build();
            //将解析之后的结果保存到集合中，方便下次使用的时候，就可以直接的获取了
            serviceMethodCache.put(method, result);
        }
    }
    return result;
}

当执行到  result = new ServiceMethod.Builder<>(this, method).build(); 

首先构建一个ServiceMethod内部类中的Builder对象，下面是对应的函数实现:
Builder(Retrofit retrofit, Method method) {
this.retrofit = retrofit;
    this.method = method;
    //获取到方法上面的注解集合
    this.methodAnnotations = method.getAnnotations();
    //获取到方法上面的参数类型的集合
    this.parameterTypes = method.getGenericParameterTypes();
    //获取到方法参数枚举的集合 ，比如我们的参数注解为 @Path("owner") String owner, @Path("repo") String repo ，所以这个集合的内容，应该为 Path{"owner",fasle} Path{"repo",false} 注解含有默认的参数
    this.parameterAnnotationsArray = method.getParameterAnnotations();
}
之后执行.build()
//开始构建
public ServiceMethod build() {
    //1 获得对应的适配器 ,根据当前method的返回值类型，获取对应的数据转换器对象
    callAdapter = createCallAdapter();
	
    //返回类型如果是 Call<ResponseBody> 就是 ResponseBody ,由于我们的接口的返回值类型为 Call<List<Contributor>> 所以返回的类型为 List<Contributor>
    responseType = callAdapter.responseType();
    ...

    //3 根据返回的结果类型，获得对应的结果转换器 这里返回的为 GsonResponseBodyConverter对象
    responseConverter = createResponseConverter();

    //4 解析注解 获得请求的函数定义 比如  GET、POST 比如  @GET("/repos/{owner}/{repo}/contributors")
    for (Annotation annotation : methodAnnotations) {
        parseMethodAnnotation(annotation);
    }
	
    ...
    //5 解析 函数中 参数上的注解 比如  @Query @Body @Path等
    int parameterCount = parameterAnnotationsArray.length;
    //构建一个存储结果的集合
    parameterHandlers = new ParameterHandler<?>[parameterCount];
    for (int p = 0; p < parameterCount; p++) {
        //parameterTypes 为 参数类型的集合 由于我们的函数参数的为(@Path("owner") String owner, @Path("repo") String repo);，所以这里parameterTypes为俩个String.class
        Type parameterType = parameterTypes[p];
        if (Utils.hasUnresolvableType(parameterType)) {
            throw parameterError(p, "Parameter type must not include a type variable or " +"wildcard: %s",parameterType);
        }

        //遍历获取到参数上的注解,第一个为 @Path("owner") 第二个为 @Path("repo") 
        Annotation[] parameterAnnotations = parameterAnnotationsArray[p]; 
        if (parameterAnnotations == null) {
            throw parameterError(p, "No Retrofit annotation found.");
        }
        // 解析参数注解
        parameterHandlers[p] = parseParameter(p, parameterType, parameterAnnotations);
    }
    ....
    //构建一个ServiceMethod对象
    return new ServiceMethod<>(this);
}

首先分析  callAdapter = createCallAdapter(); 函数实现为：
private CallAdapter<T, R> createCallAdapter() {
    //实际调用函数的返回值类型 Type 这里值为 Call<List<Contributor>>
    Type returnType = method.getGenericReturnType();
    //这个返回类型是否无法处理,比如含有通配符的 extend 或者 super等
    ...
    //如果返回值类型为void 也不允许
    if (returnType == void.class) {
        throw methodError("Service methods cannot return void.");
    }

    //获取当前方法上面的所有的注解
    Annotation[] annotations = method.getAnnotations();
        try {
            //根据返回值的类型，已经方法上面的注解集合获得一个CallAdapter对象
            return (CallAdapter<T, R>) retrofit.callAdapter(returnType, annotations);
        } catch (RuntimeException e) { // Wide exception range because factories are user code.
                throw methodError(e, "Unable to create call adapter for %s", returnType);
    }
}

执行 retrofit.callAdapter(returnType, annotations);
public CallAdapter<?, ?> nextCallAdapter(@Nullable CallAdapter.Factory skipPast, Type returnType, Annotation[] annotations) {
	...
    //默认skipPast 为null ,callAdapterFactories.indexOf 如果找不到会返回-1 所以一开始的时候，start为0
    int start = callAdapterFactories.indexOf(skipPast) + 1;
    for (int i = start, count = callAdapterFactories.size(); i < count; i++) {
        //能够获得适配器 查看一个默认的实现 ExecutorCallAdapterFactory ，由于callAdapterFactories 中存在一个ExecutorCallAdapterFactory,所以调用对应的函数
        CallAdapter<?, ?> adapter = callAdapterFactories.get(i).get(returnType, annotations, this);
        //如果获取到了，就直接返回
        if (adapter != null) {
            return adapter;
        }
    }

    //如果到了就说明没有获取到,那就抛出一个异常
	...
    throw new IllegalArgumentException(builder.toString());
}

执行 callAdapterFactories.get(i).get(returnType, annotations, this); 由于我们在一开始创建的时候给这个集合赋值了一个ExecutorCallAdapterFactory所以会调用对应的函数
public CallAdapter<?, ?> get(Type returnType, Annotation[] annotations, Retrofit retrofit)
{
    //判断返回的类型,如果不是Call的类型，就直接返回null,这里的value为 Call<List<Contributor>>
    if (getRawType(returnType) != Call.class)
    {
        return null;
    }
    //获得原始的 返回值的类型 这里获取的值就为 List<Contributor> 类型
    final Type responseType = Utils.getCallResponseType(returnType);
    //返回一个匿名实现类回去
    return new CallAdapter<Object, Call<?>>()
    {
        @Override
        public Type responseType()
        {
            //返回当前的callAdapter解析返回的类型
            return responseType;
        }

        @Override
        public Call<Object> adapt(Call<Object> call) //这里的call为OkHttpCall对象
        {
            return new ExecutorCallbackCall<>(callbackExecutor, call);
        }
    };
}

所以执行 callAdapter = createCallAdapter();主要干的事情就是从集合里面找到对应的转换器，通过解析调用参数的返回值类型，得到真实的返回值类型，最后返回一个 匿名实现类回去

执行  responseType = callAdapter.responseType(); 这里的callAdapter为我们创建的匿名内部类，所以这里调用他的responseType()返回就会获取到他的responseType 这里值为 List<Contributor>

执行  responseConverter = createResponseConverter();
//创建返回结果的转换器
private Converter<ResponseBody, T> createResponseConverter() {
    //获得当前方法上的所有的注解
    Annotation[] annotations = method.getAnnotations();
    try {
        return retrofit.responseBodyConverter(responseType, annotations);
    } catch (RuntimeException e) { // Wide exception range because factories are user code.
        throw methodError(e, "Unable to create converter for %s", responseType);
    }
} 

public <T> Converter<ResponseBody, T> nextResponseBodyConverter(@Nullable Converter.Factory skipPast, Type type, Annotation[] annotations) {
	...
    //skipPast 为空，所以start 开始为 0, 因为converterFactories.indexOf如果找不到会返回-1
    int start = converterFactories.indexOf(skipPast) + 1;
    //然后从converterFactories 集合遍历每一个转换器 ,这里有俩个，第一个是BuiltInConverters ，第二个为 GsonConverterFactory ，所以这里第一个为 BuiltInConverters
    for (int i = start, count = converterFactories.size(); i < count; i++) {
        //第一个为BuiltInConverters ，所以调用对应的函数,由于  BuiltInConverters 能处理的返回值类型的为ResponseBody类型，或者是Void类型，对于其他的类型直接返回null
        //所以这里获取我们的第二个转换器  即为 GsonConverterFactory ，所以调用对应的函数
        Converter<ResponseBody, ?> converter = converterFactories.get(i).responseBodyConverter(type, annotations, this);
        //如果返回值类型不为空，则直接返回，这里返回的为 GsonResponseBodyConverter 对象
        if (converter != null) {
            //noinspection unchecked
            return (Converter<ResponseBody, T>) converter;
        }
    }
	...
    throw new IllegalArgumentException(builder.toString());
}

这里会遍历converterFactories集合中的元素，然后调用对应的responseBodyConverter函数，因为我们这个集合里面的元素有 BuiltInConverters，GsonConverterFactory，所以会调用对应的函数实现

这里先分析下BuiltInConverters的函数实现：
public Converter<ResponseBody, ?> responseBodyConverter(Type type, Annotation[] annotations, Retrofit retrofit) {
if (type == ResponseBody.class) {
    //TODO 如果有 Streaming 注解 对response的处理会不一样(以流的形式处理，如在下载大文件时候)
      return Utils.isAnnotationPresent(annotations, Streaming.class) ? StreamingResponseBodyConverter.INSTANCE : BufferingResponseBodyConverter.INSTANCE;
    }
	//如果返回值的类型为Void
    if (type == Void.class) {
      return VoidResponseBodyConverter.INSTANCE;
	}
	return null;
}
所以BuiltInConverters 能处理的返回值类型的为ResponseBody类型，或者是Void类型，对于其他的类型直接返回null，由于我们的结果中有这样的判断if(converter != null)所以会过滤掉
  
接着分析 GsonConverterFactory 对应的函数实现：
@Override
public Converter<ResponseBody, ?> responseBodyConverter(Type type, Annotation[] annotations, Retrofit retrofit) {
    //将响应ResponseBody转成其他类型(javabean)
    TypeAdapter<?> adapter = gson.getAdapter(TypeToken.get(type));
    return new GsonResponseBodyConverter<>(gson, adapter);
}
可以看出最终会返回GsonResponseBodyConverter对象,所以对于执行 responseConverter = createResponseConverter();最终 responseConverter为 GsonResponseBodyConverter

执行
for (Annotation annotation : methodAnnotations) {
    parseMethodAnnotation(annotation);
}

//解析方法上面的注解  比如  @GET("/repos/{owner}/{repo}/contributors?sort=desc") 根据不同的注解，使用不同的解析
private void parseMethodAnnotation(Annotation annotation) {
    if (annotation instanceof DELETE) {
        parseHttpMethodAndPath("DELETE", ((DELETE) annotation).value(), false);
    } else if (annotation instanceof GET) {
        //注解是GET ,((GET) annotation).value() 获取到对应注解的值
        parseHttpMethodAndPath("GET", ((GET) annotation).value(), false);
    } else if (annotation instanceof HEAD) {
        parseHttpMethodAndPath("HEAD", ((HEAD) annotation).value(), false);
        if (!Void.class.equals(responseType)) {
            throw methodError("HEAD method must use Void as response type.");
    }
    .....
}

因为我们的请求方法上面的注解为  @GET("/repos/{owner}/{repo}/contributors?sort=desc"),所以会进入到 parseHttpMethodAndPath("GET", ((GET) annotation).value(), false);

//解析注解的内容 对于Get解析调用为  parseHttpMethodAndPath("GET", ((GET) annotation).value(), false);
private void parseHttpMethodAndPath(String httpMethod, String value, boolean hasBody) {
	
    //给httpMethod赋值，代表当前解析方式，比如 Get,Post 等
    this.httpMethod = httpMethod;
    //是否有body内容
    his.hasBody = hasBody;

    //如果注解上面没有参数，就直接返回
    if (value.isEmpty()) {
        return;
    }

    //判断URL是否已经有 ?xxx=xxx 也即是当前的值中是否有参数设置了
    // Get the relative URL path and existing query string, if present.
    int question = value.indexOf('?');
    //如果question为-1，代表没有直接在注解上面传递参数
    if (question != -1 && question < value.length() - 1) {
        //如果进来就代表有直接在注解上面设置参数
        //获取到参数的内容
        String queryParams = value.substring(question + 1);
        //判断是否合法,如果不合法就直接抛出异常
        Matcher queryParamMatcher = PARAM_URL_REGEX.matcher(queryParams);
        if (queryParamMatcher.find()) {
            throw methodError("URL query string \"%s\" must not have replace block. "+ "For dynamic query parameters use @Query.", queryParams);
        }
    }
    //保存当前注解上的url  这里为 /repos/{owner}/{repo}/contributors?sort=desc
    this.relativeUrl = value;  
    //解析出地址中的参数 {xx} （?之前的）,也即是在注解上面的参数，后面用来拼接内容 如果当前的参数类型为 /repos/{owner}/{repo}/contributors ，这里的返回的内容为 owner,repo
    this.relativeUrlParamNames = parsePathParameters(value);
}

parsePathParameters(value);函数的实现为：
static Set<String> parsePathParameters(String path) {
//就是类似将这样的字符串 /repos/{owner}/{repo}/contributors 提取出  {  } 里面的内容，这里也即是返回 owner,repo集合
    Matcher m = PARAM_URL_REGEX.matcher(path);
    Set<String> patterns = new LinkedHashSet<>();
    while (m.find()) {
        patterns.add(m.group(1));
    }
    return patterns;
}
当  this.relativeUrlParamNames = parsePathParameters(value); 执行完之后，this.relativeUrlParamNames 集合中含有元素 owner,repo,这里也即是获取到接下来要替换的参数name
 
接着执行 解析参数的注解内容中有这样的代码  parseParameter(p, parameterType, parameterAnnotations);
//解析方法上面的参数的注解
private ParameterHandler<?> parseParameter(int p, Type parameterType, Annotation[] annotations) {
    ParameterHandler<?> result = null;
    //你可以在一个参数中定义多个注解 但是我不接受
    for (Annotation annotation : annotations) {
        //解析注解
        ParameterHandler<?> annotationAction = parseParameterAnnotation(p, parameterType, annotations, annotation);

        //如果为空，跳过这个参数注解解析
        if (annotationAction == null) {
            continue;
        }

        //默认为空
        if (result != null) {
            throw parameterError(p, "Multiple Retrofit annotations found, only one " +"allowed.");
        }

        //赋值给result
        result = annotationAction;
    }

    //如果为空抛出异常
    if (result == null) {
        throw parameterError(p, "No Retrofit annotation found.");
    }
    return result;
} 

执行  parseParameterAnnotation(p, parameterType, annotations, annotation);
//解析参数的注解,根据不同的注解的类型 比如 Call<List<Contributor>> contributors(@Path("owner") String owner, @Path("repo") String repo);
private ParameterHandler<?> parseParameterAnnotation(int p, Type type, Annotation[] annotations, Annotation annotation) {
    if (annotation instanceof Url) {
               
    } else if (annotation instanceof Path) {//如果参数上的注解为Path
        //标识当前的参数注解为path
        gotPath = true;

        //获取到参数注解的值 ,假设当前的注解为@Path("owner")
        Path path = (Path) annotation;
        //那么这个value获取到的就为owner
        String name = path.value();
        //检测name是否合法
        validatePathName(p, name);

        //这里得到的是默认的converter对象，为 BuiltInConverters.ToStringConverter.INSTANCE;
        Converter<?, String> converter = retrofit.stringConverter(type, annotations);
        //构建一个ParameterHandler.Path对象  Path注解中，还有一个默认的参数，encode 默认为false,所以这里也为false
        return new ParameterHandler.Path<>(name, converter, path.encoded());
	}
	....
}
执行 retrofit.stringConverter(type, annotations);
public <T> Converter<T, String> stringConverter(Type type, Annotation[] annotations) {
    ...
    //然后从converterFactories 集合遍历每一个转换器 ,这里有俩个，第一个是BuiltInConverters ，第二个为 GsonConverterFactory ，所以这里第一个为 BuiltInConverters
    for (int i = 0, count = converterFactories.size(); i < count; i++) {
        //这里默认的俩个都没有实现这个方法，由父类实现，父类默认为null
        Converter<?, String> converter = converterFactories.get(i).stringConverter(type, annotations, this);
        if (converter != null) {
           //noinspection unchecked
            return (Converter<T, String>) converter;
        }
    }
	
    //所以会进入到这里,返回一个默认的
    return (Converter<T, String>) BuiltInConverters.ToStringConverter.INSTANCE;
}

BuiltInConverters , GsonConverterFactory 对应的stringConverter函数实现为：
public @Nullable Converter<?, String> stringConverter(Type type, Annotation[] annotations,
        Retrofit retrofit) {
    return null;
}
所以会得到一个 Converter<T, String>) BuiltInConverters.ToStringConverter.INSTANCE;对象

static final class ToStringConverter implements Converter<Object, String> {
    static final ToStringConverter INSTANCE = new ToStringConverter();
    @Override public String convert(Object value) {
      return value.toString();
    }
}

最后构建一个 new ParameterHandler.Path<>(name, converter, path.encoded());对象，存储这些有用的信息
Path(String name, Converter<T, String> valueConverter, boolean encoded) {
    this.name = checkNotNull(name, "name == null");
    this.valueConverter = valueConverter;
    this.encoded = encoded;
}

最后执行 return new ServiceMethod<>(this);

//使用Builder构建一个ServiceMethod对象
ServiceMethod(Builder<R, T> builder) {
    this.callFactory = builder.retrofit.callFactory();
    this.callAdapter = builder.callAdapter;
    this.baseUrl = builder.retrofit.baseUrl();
    this.responseConverter = builder.responseConverter;
    this.httpMethod = builder.httpMethod;
    this.relativeUrl = builder.relativeUrl;
    this.headers = builder.headers;
    this.contentType = builder.contentType;
    this.hasBody = builder.hasBody;
    this.isFormEncoded = builder.isFormEncoded;
    this.isMultipart = builder.isMultipart;
    this.parameterHandlers = builder.parameterHandlers;
}

所以对于  ServiceMethod<Object, Object> serviceMethod = (ServiceMethod<Object, Object>) loadServiceMethod(method);所做的事情，就是解析调用函数的注解，获取到对应的内容，为后面
拼接完整的url做了基础

接着执行   OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
OkHttpCall(ServiceMethod<T, ?> serviceMethod, @Nullable Object[] args) {
    this.serviceMethod = serviceMethod;
    this.args = args;
}

执行 serviceMethod.adapt(okHttpCall);

T adapt(Call<R> call) { // call 为OkHttpCall 对象
    //这里的callAdapter对象为 ExecutorCallAdapterFactory 中调动get函数创建的匿名的内部类，所以会调用对应的函数 所以这边返回的对象为 ExecutorCallbackCall 对象
    return callAdapter.adapt(call);
}

因为这里的callAdapter对象为 ExecutorCallbackCall中返回的匿名内部类对象 ，之前是由于这句赋值的   callAdapter = createCallAdapter();
所以会调用到对应的函数实现： 
return new CallAdapter<Object, Call<?>>()
{
    @Override
    public Type responseType()
    {
        //返回当前的callAdapter解析返回的类型
        return responseType;
    }

    @Override
    public Call<Object> adapt(Call<Object> call) //这里的call为OkHttpCall对象
    {
        return new ExecutorCallbackCall<>(callbackExecutor, call);
    }
};

所以会执行 new ExecutorCallbackCall<>(callbackExecutor, call); 这里的 callbackExecutor 存储 MainThreadExecutor对象 call为 OkHttpCall对象
static final class ExecutorCallbackCall<T> implements Call<T>
{
    //MainThreadExecutor对象
    final Executor callbackExecutor;
    //这里的call为OkHttpCall对象
    final Call<T> delegate;

    ExecutorCallbackCall(Executor callbackExecutor, Call<T> delegate)
    {
        this.callbackExecutor = callbackExecutor;
        this.delegate = delegate;
    }
	...	
}

所以当我们这样调用的时候 Call<List<Contributor>> call = github.contributors("square", "retrofit"); 返回的call即为 ExecutorCallbackCall对象
然后执行  call.enqueue(new Callback<List<Contributor>>(callback)，就会执行对应的函数即为：
@Override
public void enqueue(final Callback<T> callback)
{
    checkNotNull(callback, "callback == null");

    //真正的任务执行还是OkHttpCall来执行
    delegate.enqueue(new Callback<T>()
    {
        @Override
        public void onResponse(Call<T> call, final Response<T> response)
        {
            //这里可以看出，callbackexecutor做的任务主要是用来切换线程的，因为当前的回调是在子线程里面的，而我们android是要在主线程的
            //这里的callbackExecutor 对象为 MainThreadExecutor ，所以执行对应的函数
            callbackExecutor.execute(new Runnable()
            {
                @Override
                public void run()
                {
                    //下面的逻辑就是在主线程里面了
                    if (delegate.isCanceled())
                    {
                        // Emulate OkHttp's behavior of throwing/delivering an IOException on cancellation.
                        callback.onFailure(ExecutorCallbackCall.this, new IOException("Canceled"));
                    }
                    else
                    {
                        callback.onResponse(ExecutorCallbackCall.this, response);
                    }
                 }
             });
        }

        @Override
        public void onFailure(Call<T> call, final Throwable t)
        {
            //这里可以看出，callbackexecutor做的任务主要是用来切换线程的，因为当前的回调是在子线程里面的，而我们android是要在主线程的
            callbackExecutor.execute(new Runnable()
            {
                //下面的逻辑就是在主线程里面了
                @Override
                public void run()
                {
                    callback.onFailure(ExecutorCallbackCall.this, t);
                }
            });
        }
});

执行 delegate.enqueue(new Callback<T>()),因为delegate 对象为 OkHttpClient,所以会执行对应的函数
//异步方法的调用
@Override public void enqueue(final Callback<T> callback) {
    okhttp3.Call call;

    Throwable failure;
    synchronized (this) {
      //如果当前已经执行过了，就抛出异常
      if (executed) throw new IllegalStateException("Already executed.");
      //标识当前已经执行过了
      executed = true;

      //默认rawCall为空
      call = rawCall;
      failure = creationFailure;
      if (call == null && failure == null) {
        try {
          //获得OkHttp的Call
          call = rawCall = createRawCall();
        } catch (Throwable t) {
          throwIfFatal(t);
          failure = creationFailure = t;
        }
      }
    }

    //如果creationFailure不为空，就要抛出异常
    if (failure != null) {
      callback.onFailure(this, failure);
      return;
    }

    //如果当前是取消状态，直接取消
    if (canceled) {
        call.cancel();
    }

    //异步任务的 执行
    call.enqueue(new okhttp3.Callback() {
      @Override public void onResponse(okhttp3.Call call, okhttp3.Response rawResponse) {
        //解析Response 然后响应
        Response<T> response;
        try {
          response = parseResponse(rawResponse);
        } catch (Throwable e) {
          throwIfFatal(e);
          callFailure(e);
          return;
        }

        //回调通知成功，这里的就会掉到ExecutorCallbackCall 中
        try {
          callback.onResponse(OkHttpCall.this, response);
        } catch (Throwable t) {
          t.printStackTrace();
        }
      }

      @Override public void onFailure(okhttp3.Call call, IOException e) {
        callFailure(e);
      }

      //回调通知失败
      private void callFailure(Throwable e) {
        try {
          callback.onFailure(OkHttpCall.this, e);
        } catch (Throwable t) {
          t.printStackTrace();
        }
      }
    });
}

首先要创建一个OkHttp的Call对象 ，一开始为空，所有已执行创建的代码 call = rawCall = createRawCall();
//创建一个okHttp的call对象
private okhttp3.Call createRawCall() throws IOException {
    //创建Okhttp3的Call对象
    okhttp3.Call call = serviceMethod.toCall(args);
    if (call == null) {
      throw new NullPointerException("Call.Factory returned null.");
    }
    return call;
}
    
//构建一个Http的请求 用给定的参数
okhttp3.Call toCall(@Nullable Object... args) throws IOException {
    // 构建OkHttp的Request RequestBuilder#build
    RequestBuilder requestBuilder = new RequestBuilder(httpMethod, baseUrl, relativeUrl,
        headers,contentType, hasBody, isFormEncoded, isMultipart);

    @SuppressWarnings("unchecked") // It is an error to invoke a method with the wrong arg types.
    //根据注解上面解析的    ParameterHandler 集合，用来拼接url
    ParameterHandler<Object>[] handlers = (ParameterHandler<Object>[]) parameterHandlers;

    //如果参数的类型跟注解上面要填写的长度不一样，抛出异常
    int argumentCount = args != null ? args.length : 0;
    if (argumentCount != handlers.length) {
        throw new IllegalArgumentException("Argument count (" + argumentCount
            + ") doesn't match expected count (" + handlers.length + ")");
    }

    for (int p = 0; p < argumentCount; p++) {
        //这里handlers类型为 ParameterHandler.Path ，所以会调用到对应的函数,完成参数的的替换
         handlers[p].apply(requestBuilder, args[p]);
    }
    //利用OkHttp 得到一个真正的Call对象，这里即为RealCall对象
    return callFactory.newCall(requestBuilder.build());
}

执行 handlers[p].apply(requestBuilder, args[p]);
//用来拼接参数
@Override void apply(RequestBuilder builder, @Nullable T value) throws IOException {
    if (value == null) {
         throw new IllegalArgumentException(
            "Path parameter \"" + name + "\" value must not be null.");
    }
    //这里的ValueConverter的类型为 BuiltInConverters.ToStringConverter.INSTANCE 所以调用对应的构造函数
    builder.addPathParam(name, valueConverter.convert(value), encoded);
}

valueConverter.convert(value)函数的实现为：可以看出来，本质也即是调用value的toString
static final class ToStringConverter implements Converter<Object, String> {
    static final ToStringConverter INSTANCE = new ToStringConverter();

    @Override public String convert(Object value) {
      return value.toString();
    }
}

执行  builder.addPathParam(name, valueConverter.convert(value), encoded); 将方法注解上面的url内容中的参数替换成value
void addPathParam(String name, String value, boolean encoded) {
    if (relativeUrl == null) {
      // The relative URL is cleared when the first query parameter is set.
      throw new AssertionError();
    }
    //将relateiveUrl的 { value }替换成value， 比如 {owner} 替换成 square {repo} 替换成 retrofit,
    //从这里可以看出 relateiveUrl中{name} 要跟 参数注解的name要一样，这里要注意了
    relativeUrl = relativeUrl.replace("{" + name + "}", canonicalizeForPath(value, encoded));
}

从替换参数可以知道，我们在函数上面写的参数name ，要跟注解上面的name一样 最终relativeUrl 变成  "/repos/square/retrofit/contributors?sort=desc"

最后执行  return callFactory.newCall(requestBuilder.build()); 这个就调用到了OkHttp中创建RealCall对象了，对于OkHttp这里就不分析了

得到OkHttp中的Call对象之后，继续执行
//异步任务的 执行
 call.enqueue(new okhttp3.Callback() {
	@Override public void onResponse(okhttp3.Call call, okhttp3.Response rawResponse) {
		//解析Response 然后响应
        Response<T> response;
        try {
          response = parseResponse(rawResponse);
        } catch (Throwable e) {
          throwIfFatal(e);
          callFailure(e);
          return;
        }

        //回调通知成功，这里的就会掉到ExecutorCallbackCall 中
        try {
          callback.onResponse(OkHttpCall.this, response);
        } catch (Throwable t) {
          t.printStackTrace();
        }
}

可以看出最终发起网络请求也是通过OkHttp来发起网络请求的，这里假设已经返回了结果了，那么就会执行到
response = parseResponse(rawResponse);

//解析请求之后，返回的内容
Response<T> parseResponse(okhttp3.Response rawResponse) throws IOException {
    //得到返回内容的body部分
      ResponseBody rawBody = rawResponse.body();

    // Remove the body's source (the only stateful object) so we can pass the response along.
    rawResponse = rawResponse.newBuilder()
        .body(new NoContentResponseBody(rawBody.contentType(), rawBody.contentLength()))
        .build();

    int code = rawResponse.code();
    if (code < 200 || code >= 300) {
      try {
        // Buffer the entire body to avoid future I/O.
        ResponseBody bufferedBody = Utils.buffer(rawBody);
        return Response.error(bufferedBody, rawResponse);
      } finally {
        rawBody.close();
      }
    }

    if (code == 204 || code == 205) {
      rawBody.close();
      return Response.success(null, rawResponse);
    }

    //执行response 转换器
    ExceptionCatchingRequestBody catchingBody = new ExceptionCatchingRequestBody(rawBody);
    try {
      //利用对应的结果转换器，转成对应的对象
      T body = serviceMethod.toResponse(catchingBody);
      return Response.success(body, rawResponse);
    } catch (RuntimeException e) {
      // If the underlying source threw an exception, propagate that rather than indicating it was
      // a runtime exception.
      catchingBody.throwIfCaught();
      throw e;
    }
}

执行 T body = serviceMethod.toResponse(catchingBody);
R toResponse(ResponseBody body) throws IOException {
    //这里的responseConverter对象为 GsonResponseBodyConverter对象，所以执行对应的函数
    return responseConverter.convert(body);
}

执行 responseConverter.convert(body); 因为 responseConverter对象为 GsonResponseBodyConverter,所以会执行对应的函数
@Override public T convert(ResponseBody value) throws IOException {
    //利用Gson将ResponseBody 内容解析成一个对象,并返回
    JsonReader jsonReader = gson.newJsonReader(value.charStream());
    try {
      T result = adapter.read(jsonReader);
      if (jsonReader.peek() != JsonToken.END_DOCUMENT) {
        throw new JsonIOException("JSON document was not fully consumed.");
      }
      return result;
    } finally {
      value.close();
    }
}

所以上面最终会通过Gson来解析返回的内容，封装成将要返回的对象

执行
return Response.success(body, rawResponse);
public static <T> Response<T> success(@Nullable T body, okhttp3.Response rawResponse) {
    checkNotNull(rawResponse, "rawResponse == null");
    if (!rawResponse.isSuccessful()) {
      throw new IllegalArgumentException("rawResponse must be successful response");
    }
    //构建一个Response对象
    return new Response<>(rawResponse, body, null);
}
将得到的对象，再封装成一个Response对象返回

在得到了要返回的Resonse对象之后，继续执行
//回调通知成功，这里的就会掉到ExecutorCallbackCall 中
try {
        callback.onResponse(OkHttpCall.this, response);
    } catch (Throwable t) 
    {
        t.printStackTrace();
    }

执行callback.onResponse(OkHttpCall.this, response); ，这里的callback对象为ExecutorCallbackCall 中的Callback对象，所以回调回去
	
delegate.enqueue(new Callback<T>()
{
    @Override
    public void onResponse(Call<T> call, final Response<T> response)
    {
        //这里可以看出，callbackexecutor做的任务主要是用来切换线程的，因为当前的回调是在子线程里面的，而我们android是要在主线程的
        //这里的callbackExecutor 对象为 MainThreadExecutor ，所以执行对应的函数
        callbackExecutor.execute(new Runnable()
        {
            @Override
            public void run()
            {
                //下面的逻辑就是在主线程里面了
                if (delegate.isCanceled())
                {
                    // Emulate OkHttp's behavior of throwing/delivering an IOException on cancellation.
                    callback.onFailure(ExecutorCallbackCall.this, new IOException("Canceled"));
                }
                else
                {
                    callback.onResponse(ExecutorCallbackCall.this, response);
                }
        }
    });
	
	.....
}

执行 callbackExecutor.execute(new Runnable())，这里的 callbackExecutor对象为MainThreadExecutor ，所以会执行对应的函数
//线程池
static class MainThreadExecutor implements Executor
{

    //主线程的Handler，主要是为了切换到主线程操作
    private final Handler handler = new Handler(Looper.getMainLooper());

    @Override
    public void execute(Runnable r)
    {
        //切换到主线程
        handler.post(r);
    }
}

所以会执行handler.post(r); 利用主线程的Handler Looper对象，成功的将我们的线程从子线程切换到了主线程,
之后执行 callback.onResponse(ExecutorCallbackCall.this, response); 也就回到了我们的代码里面

所以可以看出来这个 ExecutorCallAdapterFactory 这个工厂其实也没有做什么事情，真正的网络请求，也是交给OkHttpCall 来执行，这个类主要做的事情，就是用来做线程切换逻辑，而真正做线程切换
又交给了 MainThreadExecutor， MainThreadExecutor又利用主线程的Handler来完成切换
```
