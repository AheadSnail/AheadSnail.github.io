---
layout: pager
title: Android IBinder机制理解
date: 2017-12-21 21:14:25
tags: [Android,IBinder,IPC]
description:  Android IBinder机制理解
---
什么是IPC机制
```
（1）IPC是Inter-Process Communication的缩写，含义为进程间通信或者跨进程通信，是指两个进程之间进行数据交换的过程。
（2）ANR是Application Not Responding的缩写，即应用无响应。主线程执行大量的耗时操作容易导致ANR现象发生。
（3）在Android中最有特色的进程间通信方式就是Binder了，通过Binder可以轻松地实现进程间通信。
（4）Android还支持Socket，通过Socket也可以实现任意两个终端或者两个进程之间的通信。
```

什么是Binder
```
（1）Binder实现了IBinder接口。
（2）从IPC角度来说，Binder是Android中的一种跨进程通信方式。Binder还可以理解为一种虚拟的物理设备，它的设备驱动是/dev/binder，这种通信方式在Linux中没有。
（3）从Android Framework角度来说，Binder是ServiceManager连接各种Manager（ActivityManager、WindowManager，等等）和相应ManagerService的桥梁。
（4）从Android应用层来说，Binder是客户端和服务端进行通信的媒介，当bindService的时候，服务端会返回一个包含了服务端业务调用的Binder对象，
通过这个对象，客户端就可以获取服务端提供的服务或者数据，这里的服务包括普通服务和基于AIDL的服务。
（5）AIDL即Android interface definition Language，即Android接口定义语言。
```

2、在分析Binder的工作原理之前，我们先补充一下Android设计模式之Proxy模式
```
Proxy代理模式简介：
代理模式是对象的结构模式。代理模式给某一个对象提供一个代理对象，并由代理对象控制对原对象的引用。
模式的使用场景：
就是一个人或者机构代表另一个人或者机构采取行动。在一些情况下，一个客户不想或者不能够直接引用一个对象，而代理对象可以在客户端和目标对象之间起到中介的作用。
```

![结果显示](/uploads/代理模式原理图.png)
```
抽象对象角色AbstarctObject：声明了目标对象和代理对象的共同接口，这样一来在任何可以使用目标对象的地方都可以使用代理对象。
目标对象角色RealObject：定义了代理对象所代表的目标对象。
代理对象角色ProxyObject：代理对象内部含有目标对象的引用，从而可以在任何时候操作目标对象；代理对象提供一个与目标对象相同的接口，
以便可以在任何时候替代目标对象。代理对象通常在客户端调用传递给目标对象之前或之后，执行某个操作，而不是单纯地将调用传递给目标对象。
```

Proxy代理模式的简单实现
```java
抽象对象角色
public abstract class AbstractObject {  
    //操作  
    public abstract void operation();  
}  

目标对象角色
public class RealObject extends AbstractObject {  
    @Override  
    public void operation() {  
        //一些操作  
        System.out.println("一些操作");  
    }  
}  

代理对象角色
public class ProxyObject extends AbstractObject{  
    RealObject realObject = new RealObject();//目标对象角色  
    @Override  
    public void operation() {  
        //调用目标对象之前可以做相关操作  
        System.out.println("before");          
        realObject.operation();        //目标对象角色的操作函数  
        //调用目标对象之后可以做相关操作  
        System.out.println("after");  
    }  
}  

客户端
public class Client {  
    public static void main(String[] args) {  
        AbstractObject obj = new ProxyObject();  
        obj.operation();  
    }  
}  
```

写Aidl要注意的地方
```
其实AIDL这门语言非常的简单，基本上它的语法和 Java 是一样的，只是在一些细微处有些许差别——毕竟它只是被创造出来简化Android程序员工作的
太复杂不好——所以在这里我就着重的说一下它和 Java 不一样的地方。主要有下面这些点：

文件类型：用AIDL书写的文件的后缀是 .aidl，而不是 .java。
数据类型：AIDL默认支持一些数据类型，在使用这些数据类型的时候是不需要导包的，但是除了这些类型之外的数据类型，在使用之前必须导包，就算目标文件与当前正在编写的 .aidl 文件在同一个包下——在 Java 中，
这种情况是不需要导包的。比如，现在我们编写了两个文件，一个叫做 Book.java ，另一个叫做 BookManager.aidl，它们都在 com.lypeer.aidldemo 包下 ，现在我们需要在 .aidl 文件里使用 Book 对象，
那么我们就必须在 .aidl 文件里面写上 import com.lypeer.aidldemo.Book; 哪怕 .java 文件和 .aidl 文件就在一个包下。 

默认支持的数据类型包括： 
Java中的八种基本数据类型，包括 byte，short，int，long，float，double，boolean，char。
String 类型。
CharSequence类型。
List类型：List中的所有元素必须是AIDL支持的类型之一，或者是一个其他AIDL生成的接口，或者是定义的parcelable（下文关于这个会有详解）。List可以使用泛型。
Map类型：Map中的所有元素必须是AIDL支持的类型之一，或者是一个其他AIDL生成的接口，或者是定义的parcelable。Map是不支持泛型的。

定向tag：这是一个极易被忽略的点——这里的“被忽略”指的不是大家都不知道，而是很少人会正确的使用它。在我的理解里，定向 tag 是这样的：
AIDL中的定向 tag 表示了在跨进程通信中数据的流向，其中 in 表示数据只能由客户端流向服务端， out 表示数据只能由服务端流向客户端，
而 inout 则表示数据可在服务端与客户端之间双向流通。其中，数据流向是针对在客户端中的那个传入方法的对象而言的。in 为定向 tag 的话表现为服务端将会接收到一个那个对象的完整数据，
但是客户端的那个对象不会因为服务端对传参的修改而发生变动；out 的话表现为服务端将会接收到那个对象的的空对象，但是在服务端对接收到的空对象有任何修改之后客户端将会同步变动
inout 为定向 tag 的情况下，服务端将会接收到客户端传来对象的完整信息，并且客户端将会同步服务端对该对象的任何变动。 
另外，Java 中的基本类型和 String ，CharSequence 的定向 tag 默认且只能是 in 。还有，请注意，请不要滥用定向 tag ，而是要根据需要选取合适的——要是不管三七二十一，
全都一上来就用 inout ，等工程大了系统的开销就会大很多——因为排列整理参数的开销是很昂贵的。两种AIDL文件：在我的理解里，所有的AIDL文件大致可以分为两类。
一类是用来定义parcelable对象，以供其他AIDL文件使用AIDL中非默认支持的数据类型的。一类是用来定义方法接口，以供系统使用来完成跨进程通信的。可以看到，两类文件都是在“定义”些什么，
而不涉及具体的实现，这就是为什么它叫做“Android接口定义语言”。 注：所有的非默认支持数据类型必须通过第一类AIDL文件定义才能被使用。


于不同的进程有着不同的内存区域，并且它们只能访问自己的那一块内存区域，所以我们不能像平时那样，传一个句柄过去就完事了——句柄指向的是一个内存区域，
现在目标进程根本不能访问源进程的内存，那把它传过去又有什么用呢？所以我们必须将要传输的数据转化为能够在内存之间流通的形式。这个转化的过程就叫做序列化与反序列化。
简单来说是这样的：比如现在我们要将一个对象的数据从客户端传到服务端去，我们就可以在客户端对这个对象进行序列化的操作，将其中包含的数据转化为序列化流，
然后将这个序列化流传输到服务端的内存中去，再在服务端对这个数据流进行反序列化的操作，从而还原其中包含的数据——通过这种方式，我们就达到了在一个进程中访问另一个进程的数据的目的。
而通常，在我们通过AIDL进行跨进程通信的时候，选择的序列化方式是实现 Parcelable 接口。关于实现 Parcelable 接口之后里面具体有那些方法啦，每个方法是干嘛的啦，
这些我就不展开来讲了，那並非这篇文章的重点，我下面主要讲一下如何快速的生成一个合格的可序列化的类（以Book.java为例）。
注：若AIDL文件中涉及到的所有数据类型均为默认支持的数据类型，则无此步骤。因为默认支持的那些数据类型都是可序列化的。

但是请注意，这里有一个坑：默认生成的模板类的对象只支持为 in 的定向 tag 。为什么呢？因为默认生成的类里面只有 writeToParcel() 方法，而如果要支持为 out 或者 inout 的定向 tag 的话，
还需要实现 readFromParcel() 方法——而这个方法其实并没有在 Parcelable 接口里面，所以需要我们从头写，读取的顺序要跟写的顺序保持一致

注意：这里又有一个坑！大家可能注意到了，在 Book.aidl 文件中，我一直在强调：Book.aidl与Book.java的包名应当是一样的。这似乎理所当然的意味着这两个文件应当是在同一个包里面的——事实上，
很多比较老的文章里就是这样说的，他们说最好都在 aidl 包里同一个包下，方便移植——然而在 Android Studio 里并不是这样。如果这样做的话，系统根本就找不到 Book.java 文件，
从而在其他的AIDL文件里面使用 Book 对象的时候会报 Symbol not found 的错误。为什么会这样呢？因为 Gradle 。大家都知道，Android Studio 是默认使用 Gradle 来构建 Android 项目的，
而 Gradle 在构建项目的时候会通过 sourceSets 来配置不同文件的访问路径，从而加快查找速度——问题就出在这里。Gradle 默认是将 java 代码的访问路径设置在 java 包下的，这样一来，
如果 java 文件是放在 aidl 包下的话那么理所当然系统是找不到这个 java 文件的。那应该怎么办呢？

又要 java文件和 aidl 文件的包名是一样的，又要能找到这个 java 文件——那么仔细想一下的话，其实解决方法是很显而易见的。首先我们可以把问题转化成：如何在保证两个文件包名一样的情况下，
让系统能够找到我们的 java 文件？这样一来思路就很明确了：要么让系统来 aidl 包里面来找 java 文件，要么把 java 文件放到系统能找到的地方去，也即放到 java 包里面去。
接下来我详细的讲一下这两种方式具体应该怎么做：

修改 build.gradle 文件：在 android{} 中间加上下面的内容：
sourceSets {
    main {
        java.srcDirs = ['src/main/java', 'src/main/aidl']
    }
}

也就是把 java 代码的访问路径设置成了 java 包和 aidl 包，这样一来系统就会到 aidl 包里面去查找 java 文件，也就达到了我们的目的。
只是有一点，这样设置后 Android Studio 中的项目目录会有一些改变，我感觉改得挺难看的。

把 java 文件放到 java 包下去：把 Book.java 放到 java 包里任意一个包下，保持其包名不变，与 Book.aidl 一致。只要它的包名不变，Book.aidl 就能找到 Book.java ，
而只要 Book.java 在 java 包下，那么系统也是能找到它的。但是这样做的话也有一个问题，就是在移植相关 .aidl 文件和 .java 文件的时候没那么方便，不能直接把整个 aidl 文件夹拿过去完事儿了，
还要单独将 .java 文件放到 java 文件夹里去。

我们可以用上面两个方法之一来解决找不到 .java 文件的坑，具体用哪个就看大家怎么选了，反正都挺简单的。
到这里我们就已经将AIDL文件新建并且书写完毕了，clean 一下项目，如果没有报错，这一块就算是大功告成了。
```
![结果显示](/uploads/IBinder机制原理图.png)
```
Binder一个很重要的作用是：将客户端的请求参数通过Parcel包装后传到远程服务端，远程服务端解析数据并执行对应的操作，
同时客户端线程挂起，当服务端方法执行完毕后，再将返回结果写入到另外一个Parcel中并将其通过Binder传回到客户端，
客户端接收到返回数据的Parcel后，Binder会解析数据包中的内容并将原始结果返回给客户端，至此，整个Binder的工作过程就完成了。
由此可见，Binder更像一个数据通道，Parcel对象就在这个通道中跨进程传输，至于双方如何通信，这并不负责，只需要双方按照约定好的规范去打包和解包数据即可。
```

AIDL案例
```java
调用者调用bindService来绑定服务
 this.bindService(new Intent(),new ServiceConnection() {
			
			@Override
			public void onServiceDisconnected(ComponentName name) {
				// TODO Auto-generated method stub
				
			}
			
			@Override
			public void onServiceConnected(ComponentName name, IBinder service) {
				//当服务连接的时候，会返回远程IBinder引用
				IBinderTest asInterface = IBinderTest.Stub.asInterface(service);
				try {
					asInterface.testBinder();
				} catch (RemoteException e) {
					// TODO Auto-generated catch block
					e.printStackTrace();
				}
			}
		},Context.BIND_AUTO_CREATE);

下面这个类是远程提供的服务
public class MyService extends Service{

	private MyBinder binder;
	
	@Override
	public IBinder onBind(Intent intent) {
		// TODO Auto-generated method stub
		return binder;
	}
	
	@Override
	public void onCreate() {
		// TODO Auto-generated method stub
		binder = new MyBinder();
		super.onCreate();
	}

	
	class MyBinder extends IBinderTest.Stub
	{

		@Override
		public void testBinder() throws RemoteException {
			// TODO Auto-generated method stub
			
		}

		@Override
		public String getBinderString() throws RemoteException {
			// TODO Auto-generated method stub
			return null;
		}
		
	}
}

下面这个是远程提供的aidl文件
package com.dn.inter;
interface IBinderTest{
	void testBinder();
	String getBinderString();
}
		
下面的这个类是远程提供的aidl自动生成的类，注意在客户端使用远程文件的aidl的时候，一定要保证俩者的包名路径是一样的，这样aidl文件才会自动的生成下面的这个类文件
public interface IBinderTest extends android.os.IInterface {
	/** Local-side IPC implementation stub class. */
	public static abstract class Stub extends android.os.Binder implements com.dn.inter.IBinderTest {
		private static final java.lang.String DESCRIPTOR = "com.dn.inter.IBinderTest";

		/** Construct the stub at attach it to the interface. */
		public Stub() {
			this.attachInterface(this, DESCRIPTOR);
		}

		/**
		 * Cast an IBinder object into an com.dn.inter.IBinderTest interface,
		 * generating a proxy if needed.
		 */
		public static com.dn.inter.IBinderTest asInterface(android.os.IBinder obj) {
			if ((obj == null)) {
				return null;
			}
			android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
			if (((iin != null) && (iin instanceof com.dn.inter.IBinderTest))) {
				return ((com.dn.inter.IBinderTest) iin);
			}
			return new com.dn.inter.IBinderTest.Stub.Proxy(obj);
		}

		@Override
		public android.os.IBinder asBinder() {
			return this;
		}

		@Override
		public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags)
				throws android.os.RemoteException {
			switch (code) {
			case INTERFACE_TRANSACTION: {
				reply.writeString(DESCRIPTOR);
				return true;
			}
			case TRANSACTION_testBinder: {
				data.enforceInterface(DESCRIPTOR);
				this.testBinder();
				reply.writeNoException();
				return true;
			}
			case TRANSACTION_getBinderString: {
				data.enforceInterface(DESCRIPTOR);
				java.lang.String _result = this.getBinderString();
				reply.writeNoException();
				reply.writeString(_result);
				return true;
			}
			}
			return super.onTransact(code, data, reply, flags);
		}

		private static class Proxy implements com.dn.inter.IBinderTest {
			private android.os.IBinder mRemote;

			Proxy(android.os.IBinder remote) {
				mRemote = remote;
			}

			@Override
			public android.os.IBinder asBinder() {
				return mRemote;
			}

			public java.lang.String getInterfaceDescriptor() {
				return DESCRIPTOR;
			}

			@Override
			public void testBinder() throws android.os.RemoteException {
				android.os.Parcel _data = android.os.Parcel.obtain();
				android.os.Parcel _reply = android.os.Parcel.obtain();
				try {
					_data.writeInterfaceToken(DESCRIPTOR);
					mRemote.transact(Stub.TRANSACTION_testBinder, _data, _reply, 0);
					_reply.readException();
				} finally {
					_reply.recycle();
					_data.recycle();
				}
			}

			@Override
			public java.lang.String getBinderString() throws android.os.RemoteException {
				android.os.Parcel _data = android.os.Parcel.obtain();
				android.os.Parcel _reply = android.os.Parcel.obtain();
				java.lang.String _result;
				try {
					_data.writeInterfaceToken(DESCRIPTOR);
					mRemote.transact(Stub.TRANSACTION_getBinderString, _data, _reply, 0);
					_reply.readException();
					_result = _reply.readString();
				} finally {
					_reply.recycle();
					_data.recycle();
				}
				return _result;
			}
		}

		static final int TRANSACTION_testBinder = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
		static final int TRANSACTION_getBinderString = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);
	}

	public void testBinder() throws android.os.RemoteException;

	public java.lang.String getBinderString() throws android.os.RemoteException;
}
```

AIDL源码解析
```java
当客户端调用了 this.bindService(new Intent(),new ServiceConnection());,这里的this为context的实现类，其实就是ContextImpl
 @Override
 public boolean bindService(Intent service, ServiceConnection conn,
            int flags) {
        warnIfCallingFromSystemProcess();
        return bindServiceCommon(service, conn, flags, Process.myUserHandle());
    }
当调用到bindServiceCommon(service, conn, flags, Process.myUserHandle());
private boolean bindServiceCommon(Intent service, ServiceConnection conn, int flags,
            UserHandle user) {
        ...... 中间省略代码
        try {
            int res = ActivityManagerNative.getDefault().bindService(
                mMainThread.getApplicationThread(), getActivityToken(),
                service, service.resolveTypeIfNeeded(getContentResolver()),
                sd, flags, user.getIdentifier());
            if (res < 0) {
                throw new SecurityException(
                        "Not allowed to bind to service " + service);
            }
            return res != 0;
        } catch (RemoteException e) {
            return false;
        }
 }
 这里的ActivityManagerNative.getDefault()为
   /**
     * Retrieve the system's default/global activity manager.
     */
 static public IActivityManager getDefault() {
        return gDefault.get();
 }
 调用gDefault.get()就会执行到下面的crate函数
 private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
        protected IActivityManager create() {
            IBinder b = ServiceManager.getService("activity");
            if (false) {
                Log.v("ActivityManager", "default service binder = " + b);
            }
            IActivityManager am = asInterface(b);
            if (false) {
                Log.v("ActivityManager", "default service = " + am);
            }
            return am;
        }
    };
这里要注意一下public abstract class ActivityManagerNative extends Binder implements IActivityManager
而且 IActivityManager继承于， public interface IActivityManager extends IInterface 
ActivityManagerNative就跟我们使用aidl自动生成的文件是一样的
都是继承与Binder,实现了IInterface接口，其实他本质就相当于是一个aidl文件

通过调用ActivityManagerNative.getDefault()得到一个IActivityManager对象，其本身是一个接口，如果要启动的服务不是在当前的进程的话，就得到一个ActivityManagerProxy对象
里面的mRemote保存了远程服务端的IBinder对象
所以当执行到bindService的时候，执行的为ActivityManagerProxy中的bindService方法
 public int bindService(IApplicationThread caller, IBinder token,
            Intent service, String resolvedType, IServiceConnection connection,
            int flags, int userId) throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        data.writeStrongBinder(caller != null ? caller.asBinder() : null);
        data.writeStrongBinder(token);
        service.writeToParcel(data, 0);
        data.writeString(resolvedType);
        data.writeStrongBinder(connection.asBinder());
        data.writeInt(flags);
        data.writeInt(userId);
        mRemote.transact(BIND_SERVICE_TRANSACTION, data, reply, 0);
        reply.readException();
        int res = reply.readInt();
        data.recycle();
        reply.recycle();
        return res;
}
当执行到  mRemote.transact(BIND_SERVICE_TRANSACTION, data, reply, 0);的时候，客户端就会往IBinder驱动里面写数据，客户端会挂起，一直等待服务端的执行结果
同时会调用服务端的onTransact方法，同时将传递的数据回调回来服务端这边解析数据，调用服务端对应的方法，同时将结果写回IBinder启动中，客户端继续往下面执行
case BIND_SERVICE_TRANSACTION: {
            data.enforceInterface(IActivityManager.descriptor);
            IBinder b = data.readStrongBinder();
            IApplicationThread app = ApplicationThreadNative.asInterface(b);
            IBinder token = data.readStrongBinder();
            Intent service = Intent.CREATOR.createFromParcel(data);
            String resolvedType = data.readString();
            b = data.readStrongBinder();
            int fl = data.readInt();
            int userId = data.readInt();
            IServiceConnection conn = IServiceConnection.Stub.asInterface(b);
            int res = bindService(app, token, service, resolvedType, conn, fl, userId);
            reply.writeNoException();
            reply.writeInt(res);
            return true;
}

当执行到 int res = bindService(app, token, service, resolvedType, conn, fl, userId);的时候，就会执行下面的方法
public int bindService(IApplicationThread caller, IBinder token,
            Intent service, String resolvedType,
            IServiceConnection connection, int flags, int userId) {
        enforceNotIsolatedCaller("bindService");

        // Refuse possible leaked file descriptors
        if (service != null && service.hasFileDescriptors() == true) {
            throw new IllegalArgumentException("File descriptors passed in Intent");
        }

        synchronized(this) {
            return mServices.bindServiceLocked(caller, token, service, resolvedType,
                    connection, flags, userId);
        }
    }
这里的  final ActiveServices mServices，当执行到bindServiceLocked
 int bindServiceLocked(IApplicationThread caller, IBinder token,
            Intent service, String resolvedType,
            IServiceConnection connection, int flags, int userId) {

			.....前面省略很多代码，前面的代码有检查如果在同一个进程的话，就不会去启动其他的进程
			//这里假设远程的不是同一个进程，而且服务没有启动过，就会执行下面的代码
			// If this is the first app connected back to this binding,
            // and the service had previously asked to be told when
            // rebound, then do so.
            if (b.intent.apps.size() == 1 && b.intent.doRebind) {
                requestServiceBindingLocked(s, b.intent, callerFg, true);
            }
}
当执行到requestserviceBindingLocked函数
 private final boolean requestServiceBindingLocked(ServiceRecord r,
            IntentBindRecord i, boolean execInFg, boolean rebind) {
        ......省略代码
        r.app.thread.scheduleBindService(r, i.intent.getIntent(), rebind,r.app.repProcState);
        return true;
}
这里的r.app.thread为 IApplicationThread thread;本质是一个接口，这里的实现类为ApplicationThread,所以就会执行到
  public final void scheduleBindService(IBinder token, Intent intent,
                boolean rebind, int processState) {
            .....省略代码
            sendMessage(H.BIND_SERVICE, s);
}
这里handler接收者为
case BIND_SERVICE:
handleBindService((BindServiceData)msg.obj)；               
break;

所以执行handleBindService就会执行到
private void handleBindService(BindServiceData data) {
		这里获取到我们远程调用的service,data.token为我们的类名
        Service s = mServices.get(data.token);
        if (DEBUG_SERVICE)
            Slog.v(TAG, "handleBindService s=" + s + " rebind=" + data.rebind);
        if (s != null) {
            try {
                data.intent.setExtrasClassLoader(s.getClassLoader());
                data.intent.prepareToEnterProcess();
                try {
                    if (!data.rebind) {
					    //这边调用远程service.onBind()函数，并且返回IBinder对象，对应于我们的代码也就是 返回了一个 class MyBinder extends IBinderTest.Stub这样的实例对象
                        IBinder binder = s.onBind(data.intent);
						//最终将的当前的service 公开
                        ActivityManagerNative.getDefault().publishService(
                                data.token, data.intent, binder);
                    } else {
                        s.onRebind(data.intent);
                        ActivityManagerNative.getDefault().serviceDoneExecuting(
                                data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
                    }
                    ensureJitEnabled();
                } catch (RemoteException ex) {
                }
            } catch (Exception e) {
                if (!mInstrumentation.onException(s, e)) {
                    throw new RuntimeException(
                            "Unable to bind to service " + s
                            + " with " + data.intent + ": " + e.toString(), e);
                }
            }
        }
}

mServices.get(data.token)中获取service的时候，如果要获取的service不存在的话，就会执行下面的方法
 private void handleCreateService(CreateServiceData data) {
	 LoadedApk packageInfo = getPackageInfoNoCheck(
                data.info.applicationInfo, data.compatInfo);
        Service service = null;
        try {
			//根据我们要加载的class，获取到我们的service实例,这里远程的service对象被创建了
            java.lang.ClassLoader cl = packageInfo.getClassLoader();
            service = (Service) cl.loadClass(data.info.name).newInstance();
        } catch (Exception e) {
            if (!mInstrumentation.onException(service, e)) {
                throw new RuntimeException(
                    "Unable to instantiate service " + data.info.name
                    + ": " + e.toString(), e);
            }
        }

        try {
            if (localLOGV) Slog.v(TAG, "Creating service " + data.info.name);

            ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
            context.setOuterContext(service);

            Application app = packageInfo.makeApplication(false, mInstrumentation);
            service.attach(context, this, data.info.name, data.token, app,
                    ActivityManagerNative.getDefault());
			
			//这里执行了远程service的onCreate回调函数
            service.onCreate();
			//然后将当前创建的service，添加到mServices中，之后有的话，就不会再次的创建service了,这也就是为什么重新bindService的时候，不会重新的创建service
            mServices.put(data.token, service);
            try {
                ActivityManagerNative.getDefault().serviceDoneExecuting(
                        data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
            } catch (RemoteException e) {
                // nothing to do.
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(service, e)) {
                throw new RuntimeException(
                    "Unable to create service " + data.info.name
                    + ": " + e.toString(), e);
            }
        }
 }
 
 当执行到ActivityManagerNative.getDefault().publishService(data.token, data.intent, binder);
 这里的ActivityManagerNative.getDefault()为public interface IActivityManager extends IInterface 是一个接口，这里的实现类为ActivityManagerService
  public void publishService(IBinder token, Intent intent, IBinder service) {
        // Refuse possible leaked file descriptors
        if (intent != null && intent.hasFileDescriptors() == true) {
            throw new IllegalArgumentException("File descriptors passed in Intent");
        }

        synchronized(this) {
            if (!(token instanceof ServiceRecord)) {
                throw new IllegalArgumentException("Invalid service token");
            }
            mServices.publishServiceLocked((ServiceRecord)token, intent, service);
        }
}
 void publishServiceLocked(ServiceRecord r, Intent intent, IBinder service) {
		.....省略代码
        try {
			  //这边最终利用我们传递的conn回调回去,也即为我们的代码 public void onServiceConnected(ComponentName name, IBinder service) 至此将远程的IBinder传递回来了
              c.conn.connected(r.name, service);
            } catch (Exception e) {
                log.w(TAG, "Failure sending service " + r.name +
                    " to connection " + c.conn.asBinder() +
                    " (in " + c.binding.client.processName + ")", e);
                 }
               }
            }
        }    
    }
	
@Override
	public void onServiceConnected(ComponentName name, IBinder service) {
		IBinderTest asInterface = IBinderTest.Stub.asInterface(service);
		try {
			asInterface.testBinder();
		} catch (RemoteException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
}
当执行到这里的时候IBinderTest asInterface = IBinderTest.Stub.asInterface(service);就会执行到远端AIDL自动生成的类文件中的
/* 
* 用于将服务端的Binder对象转换成客户端所需的AIDL接口类型的对象， 
* 这种转换过程是区分进程的， 
* 如果客户端和服务端位于同一进程，那么此方法返回的就是服务端的Stub对象本身， 
* 否则返回的是系统封装后的Stub.proxy代理对象。 
* */  
public static com.dn.inter.IBinderTest asInterface(android.os.IBinder obj) {
	if ((obj == null)) {
		return null;
	}
	android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
	if (((iin != null) && (iin instanceof com.dn.inter.IBinderTest))) {
		return ((com.dn.inter.IBinderTest) iin);
	}
	return new com.dn.inter.IBinderTest.Stub.Proxy(obj);这里为远程调用会执行到这里
}

//这个类为远程代理类，我们不能直接的操作远程的内容，只能通过代理的机制来访问,通过代理来通知我们要干什么事情
private static class Proxy implements com.dn.inter.IBinderTest {
	//创建一个引用来保存远程的IBInder对象
	private android.os.IBinder mRemote;

	Proxy(android.os.IBinder remote) {
		mRemote = remote;
	}
}

当我们在代码中这样调用asInterface.testBinder();的时候，就会调用到proxy中的对应的函数，也就是我们客户端通过操作代理类，代理类来告知真正的目标对象
/* 
* 这个方法运行在客户端， 
* 因为当客户端和服务端不在同一进程时，服务端返回代理类Proxy，所以客户端会通过Proxy调用到代理类的getBookList方法， 
* 当客户端远程调用此方法时，它的内部实现是这样的： 
* 首先创建该方法所需要的输入型Parcel对象_data、输出型Parcel对象_reply和返回值对象List， 
* 然后把该方法的参数信息写入_data中， 
* 接着调用transact方法来发起RPC（远程过程调用）请求，同时当前线程挂起， 
* 然后服务端的onTransact方法会被调用，直到RPC过程返回后，当前线程继续执行， 
* 并从_reply中取出RPC过程的返回结果。 
* 最后返回_reply中的数据。 
* */  
@Override
public void testBinder() throws android.os.RemoteException {
	//得到俩个Parcel对象，用来进程将传递内容,持久化操作
	android.os.Parcel _data = android.os.Parcel.obtain();
	android.os.Parcel _reply = android.os.Parcel.obtain();
	try {
			_data.writeInterfaceToken(DESCRIPTOR);
			mRemote.transact(Stub.TRANSACTION_testBinder, _data, _reply, 0);
			_reply.readException();
		} finally {
			_reply.recycle();
			_data.recycle();
		}
}

上面所说的通过调用transact方法来发起远程调用，其实本质是首先将内容写到IBinder中，IBinder本质为是设备驱动/dev/binder，这种通信方式在Linux中没有。是通过NDK的方法来调用c层代码的
比如当调用 	mRemote.transact(Stub.TRANSACTION_testBinder, _data, _reply, 0);就会调用到IBinder中的实现类为BinderProxy ，这个就跟proxy是一样的，是代理，Binder就相当于是服务端

final class BinderProxy implements IBinder {
    public native boolean pingBinder();
    public native boolean isBinderAlive();

    public IInterface queryLocalInterface(String descriptor) {
        return null;
    }

    public boolean transact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
        Binder.checkParcel(this, code, data, "Unreasonably large binder buffer");
        return transactNative(code, data, reply, flags);
    }
}
所以就会执行transactNative(code, data, reply, flags);本质为native方法，
public native boolean transactNative(int code, Parcel data, Parcel reply,int flags) throws RemoteException;
当数据传递进去了之后，就会回调服务端的onTransact方法会被调用，也就是IBinderTest.Stub服务端的对象中的
@Override
public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags)
	throws android.os.RemoteException {
	switch (code) {
	case INTERFACE_TRANSACTION: {
		reply.writeString(DESCRIPTOR);
		return true;
	}
	case TRANSACTION_testBinder: {
		data.enforceInterface(DESCRIPTOR);
		this.testBinder();//这边就会执行到服务端中的testBinder();这个函数，至此服务端的那个真正的方法被调用了
		reply.writeNoException();
		return true;
	}
	case TRANSACTION_getBinderString: {
		data.enforceInterface(DESCRIPTOR);
		java.lang.String _result = this.getBinderString();
		reply.writeNoException();
		reply.writeString(_result);
		return true;
		}
	}
	return super.onTransact(code, data, reply, flags);
}

case TRANSACTION_testBinder: {
		data.enforceInterface(DESCRIPTOR);
		this.testBinder();//这边就会执行到服务端中的testBinder();这个函数，至此服务端的那个真正的方法被调用了
		reply.writeNoException();
		return true;
}
当访问的代码被调用完成了之后，客户端从_reply中取出RPC过程的返回结果。最后返回_reply中的数据。 
```

Binder调用流程图,这里的IActivityManager 就是一个充当中间人，代理的作用，由他来告知，真正执行者，执行特定的动作
![结果显示](/uploads/Binder调用流程图.png)

创建的IBinder对象是怎么样注入IBinder驱动中的
```java
@Override
	public void onCreate() {
		// TODO Auto-generated method stub
		binder = new MyBinder();
		super.onCreate();
	}
	class MyBinder extends IBinderTest.Stub
	
	当创建binder = new MyBinder();的时候，就会执行父类的Binder的构造函数
	  * Default constructor initializes the object.
     */
    public Binder() {
        init();

        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Binder> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Binder class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }
    }
	构造函数中的 init(); 为private native final void init(); ，就是这里完成注册的,所以IBinder驱动会持有对应的对象，所以能回调回服务端，也能传递内容
```

ServiceManager，ActivityManager跟IPC进程间的联系
```java
 还记得有这样的代码
 int res = ActivityManagerNative.getDefault().bindService(
                mMainThread.getApplicationThread(), getActivityToken(),
                service, service.resolveTypeIfNeeded(getContentResolver()),
                sd, flags, user.getIdentifier());
				
其中 ActivityManagerNative.getDefault()为
 static public IActivityManager getDefault() {
        return gDefault.get();
 }
 调用gDefault.get()就会执行到下面的crate函数
 private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
        protected IActivityManager create() {
            IBinder b = ServiceManager.getService("activity");
            if (false) {
                Log.v("ActivityManager", "default service binder = " + b);
            }
            IActivityManager am = asInterface(b);
            if (false) {
                Log.v("ActivityManager", "default service = " + am);
            }
            return am;
        }
 };
 ```java
 
 中有这样的代码  IBinder b = ServiceManager.getService("activity");下面是ServiceManager的源码，以及注释
 大致就是说返回一个服务的引用，通知特定的名字来查找，如果要查找的服务不存在的话，就会得到一个null的对象
 
 ```java
 /**
     * Returns a reference to a service with the given name.
     *
     * @param name the name of the service to get
     * @return a reference to the service, or <code>null</code> if the service doesn't exist
     */
    public static IBinder getService(String name) {
        return null;
 
 而且可以在ActivityManagerService中可以这样的代码，添加了服务
 public void setSystemProcess() {
        try {
            ServiceManager.addService(Context.ACTIVITY_SERVICE, this, true);
            ServiceManager.addService(ProcessStats.SERVICE_NAME, mProcessStats);
            ServiceManager.addService("meminfo", new MemBinder(this));
            ServiceManager.addService("gfxinfo", new GraphicsBinder(this));
            ServiceManager.addService("dbinfo", new DbBinder(this));
            if (MONITOR_CPU_USAGE) {
                ServiceManager.addService("cpuinfo", new CpuBinder(this));
            }
            ServiceManager.addService("permission", new PermissionController(this));
}
```
下面为ServiceManager的作用,保存了系统服务
![结果显示](/uploads/ServiceManager的作用.png)
当执行 IActivityManager am = asInterface(b);的时候
```java
/**
 * Cast a Binder object into an activity manager interface, generating
 * a proxy if needed.
*/
static public IActivityManager asInterface(IBinder obj) {
        if (obj == null) {
            return null;
        }
		//这里跟我们自己创建的AIDL中的方式是一样的，如果要访问的服务如果在当前进程的话，就直接返回，queryLocalInterface就是在本地查找
        IActivityManager in =
            (IActivityManager)obj.queryLocalInterface(descriptor);
        if (in != null) {
            return in;
        }
		//如果在本地没有查找到的话，就执行下面的这里，为其他进程的服务
        return new ActivityManagerProxy(obj);
    }
}
```
如果这个service是在当前的进程的话，IActivityManager in = (IActivityManager)obj.queryLocalInterface(descriptor);这边就得到一个IActivityManager对象
下面为这个接口的实现类，这里返回的是一个ActivityManagerService对象
![结果显示](/uploads/IActivityManager的实现类.png)
ActivityManagerService的类的继承关系为，所以可以看出他本身就是一个AIDL方式中的Stub类
```
public final class ActivityManagerService extends ActivityManagerNative
public abstract class ActivityManagerNative extends Binder implements IActivityManager
public interface IActivityManager extends IInterface 
```

总结：
IPC中的一种方式为AIDL，服务端要提供接口指定提供了什么功能，服务端要继承自动生成aidl文件类中的Stub，为一个IBinder对象，通过onBind函数，给系统返回一个
IBinder引用，远程的客户端通过服务端提供的aidl文件，在调用onBindService方法可以得到远程的IBinder引用，通过调用asInterface(binder),获得服务提供的接口对象
asInterface内部可以区分是否为在本进程内，如果是在本进程就直接返回当前的引用，如果是远程的就通过Proxy持有这个远程的Ibinder对象，客户端通过操作这个Proxy来调用
远程服务端中对应的方法