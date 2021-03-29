---
layout: pager
title: C++智能指针
date: 2018-12-15 15:40:37
tags: [C++,智能指针]
description: C++智能指针
---
### 简介
> 接触Aria2项目已经有大半年多了，对于项目的源码实现思路都有更深入的了解，通过接触Aria2 也让我了解了C++ 11的更多高级的特性，比如智能指针，该项目大量的采用智能指针的方式，可以做到很巧妙的不用处理内存释放的问题，整个项目源码的修改都是由我完成的，由于对C++ 高级的特性不是很熟悉，导致每次写代码都写的非常小心，在写代码之前，都要在源码里面看看别人是怎么样写的，自己参考着写，导致对自己写的代码没有多大的信心，所以这篇文章会大致讲解下我对智能指针的理解,在了解什么智能指针之前要先明白什么是引用


### 什么是引用
#### 引用的特性
> 引用：就是某一变量（目标）的一个别名，对引用的操作与对变量直接操作完全一样。
引用的声明方法：类型标识符 &引用名=目标变量名；
如下：定义引用ra，它是变量a的引用，即别名。
int a;
int &ra=a;
（1）&在此不是求地址运算符，而是起标识作用。
（2）类型标识符是指目标变量的类型。
（3）声明引用时，必须同时对其进行初始化。
（4）引用声明完毕后，相当于目标变量有两个名称即该目标原名称和引用名，且不能再把该引用名作为其他变量名的别名。

#### 引用的本质
> 引用的本质是占内存空间的，就是一个常量指针，所以必须要在定义的时候，分配内存空间的时候，完成赋值，之后就不允许再次的赋值，只是编译器帮我们做了处理,接下来会证明引用是占用内存空间的，并且本质是一个常量指针

下面是证明
```C++
#include <iostream>
using namespace std;

//定义一个全局的变量num
int num = 99;

class A
{
public:
	A();
public:
	int n;
	int &r;
};

//构造函数赋值,引用必须在初始化成员列表中赋值，并且赋值之后，就不允许改变了，因为初始化成员列表中才是定义这个变量，在类里面的只是声明，由于引用本质为常量指针，所以必须在定义的时候
//就要执行赋值操作，在初始化成员列表中赋值跟在构造函数里面赋值是完全不同的，在构造函数里面更像是一种计算操作
A::A() :n(0), r(num)
{

}

//从而可以退出，引用是占用内存空间的，他的本质是一个常量指针，不管是什么类型的指针，都是占用4个字节的地址，而且只有在初始化的时候赋值,本质是一个常量指针，之后就不允许赋值了
//其实引用只是对指针进行了简单的封装，它的底层依然是通过指针实现的，引用占用的内存和指针占用的内存长度,是一样的，在32位环境下是4个字节，在64位环境下是8个字节，之所以不能获取引用的地址，是因为编译器进行了内部的转换
void main()
{
	A *a = new A();
	cout << sizeof(A) << endl;//输出A类型的大小 , 8
	printf("r == %x\n", (int *)a + 1);//输出引用的地址 , 7df394
	printf("a == %x\n", a);//输出a的地址 , 7df390
	printf("n == %x\n", &(a->n));//输出n的地址 , 7df390，这个先声明，所以先压栈
	printf("r == %x\n", *((int *)a + 1));//输出引用r的值  , 23d000
	printf("num == %x\n", &num);//输出num变量的地址  ,23d000

	//使用&r取地址时，编译器会对代码进行隐式的转换，使得代码输出的是 r 的内容（a 的地址），而不是 r 的地址，
	//这就是为什么获取不到引用变量的地址的原因。也就是说，不是变量 r 不占用内存，而是编译器不让获取它的地址。
	system("pause");
}
```
打印的结果为
![结果显示](/uploads/C++智能指针/引用的结果.jpg)


### 智能指针的由来
> C++程序设计中使用堆内存是非常频繁的操作，堆内存的申请和释放都由程序员自己管理。程序员自己管理堆内存可以提高了程序的效率，但是整体来说堆内存的管理是麻烦的，C++11中引入了智能指针的概念，方便管理堆内存。使用普通指针，容易造成堆内存泄露（忘记释放），二次释放，程序发生异常时内存泄露等问题等，使用智能指针能更好的管理堆内存。

### 智能指针的作用
> 1.从较浅的层面看，智能指针是利用了一种叫做RAII（资源获取即初始化）的技术对普通的指针进行封装，这使得智能指针实质是一个对象，行为表现的却像一个指针。
2.智能指针的作用是防止忘记调用delete释放内存和程序异常的进入catch块忘记释放内存。另外指针的释放时机也是非常有考究的，多次释放同一个指针会造成程序崩溃，这些都可以通过智能指针来解决。
3.智能指针还有一个作用是把值语义转换成引用语义。C++和Java有一处最大的区别在于语义不同，在Java里面下列代码：

```C
Animal a = new Animal();

Animal b = a;

你当然知道，这里其实只生成了一个对象，a和b仅仅是把持对象的引用而已。但在C++中不是这样，

Animal a;

Animal b = a;

这里却是就是生成了两个对象。
```

### 智能指针的使用
> 智能指针在C++11版本之后提供，包含在头文件<memory>中，shared_ptr、unique_ptr、weak_ptr

### 智能指针的本质
> 智能指针本质是一个普通的变量，他通过内部持有了你原有的对象，然后通过引用计数的方式来决定是否要删除这个原有的对象，比如当引用计数为0的时候，就要删除，当智能指针拷贝或者复制的时候会让他们的引用计数加一

### 智能指针的设计和实现
> 在介绍 shared_ptr、unique_ptr 还是先大概的了解下他们是怎么实现的，下面是模拟智能指针的实现，智能指针类将一个计数器与类指向的对象相关联，引用计数跟踪该类有多少个对象共享同一指针。每次创建类的新对象时，初始化指针并将引用计数置为1；当对象作为另一对象的副本而创建时，拷贝构造函数拷贝指针并增加与之相应的引用计数；对一个对象进行赋值时，赋值操作符减少左操作数所指对象的引用计数（如果引用计数为减至0，则删除对象），并增加右操作数所指对象的引用计数；调用析构函数时，构造函数减少引用计数（如果引用计数减至0，则删除基础对象）能指针就是模拟指针动作的类。所有的智能指针都会重载 -> 和 * 操作符。智能指针还有许多其他功能，比较有用的是自动销毁。这主要是利用栈对象的有限作用域以及临时对象（有限作用域实现）析构函数释放内存。


```C++
#include <iostream>
#include <memory>

template<typename T>
class SmartPointer {
private:
	//真实的指针对象
    T* _ptr;
	//用来做为引用计数
    size_t* _count;
public:
    SmartPointer(T* ptr = nullptr) :
            _ptr(ptr) {
        if (_ptr) {
            _count = new size_t(1);
        } else {
            _count = new size_t(0);
        }
    }

    SmartPointer(const SmartPointer& ptr) {
        if (this != &ptr) {
            this->_ptr = ptr._ptr;
            this->_count = ptr._count;
            (*this->_count)++;
        }
    }

    SmartPointer& operator=(const SmartPointer& ptr) {
        if (this->_ptr == ptr._ptr) {
            return *this;
        }

        if (this->_ptr) {
            (*this->_count)--;
            if (this->_count == 0) {
                delete this->_ptr;
                delete this->_count;
            }
        }

        this->_ptr = ptr._ptr;
        this->_count = ptr._count;
        (*this->_count)++;
        return *this;
    }

    T& operator*() {
        assert(this->_ptr == nullptr);
        return *(this->_ptr);

    }

    T* operator->() {
        assert(this->_ptr == nullptr);
        return this->_ptr;
    }

    ~SmartPointer() {
        (*this->_count)--;
        if (*this->_count == 0) {
            delete this->_ptr;
            delete this->_count;
        }
    }

    size_t use_count(){
        return *this->_count;
    }
};

int main() {
    {
        SmartPointer<int> sp(new int(10));
        SmartPointer<int> sp2(sp);
        SmartPointer<int> sp3(new int(20));
        sp2 = sp3;
        std::cout << sp.use_count() << std::endl;
        std::cout << sp3.use_count() << std::endl;
    }
    //delete operator
}
```

### shared_ptr
> shared_ptr使用引用计数，每一个shared_ptr的拷贝都指向相同的内存。每使用他一次，内部的引用计数加1，每析构一次，内部的引用计数减1，减为0时，删除所指向的堆内存。hared_ptr内部的引用计数是安全的，但是对象的读取需要加锁。

```C++
#include "stdafx.h"
#include <iostream>
#include <future>
#include <thread>

using namespace std;
class Person
{
public:
    Person(int v) {
        value = v;
        std::cout << "Cons" <<value<< std::endl;
    }
    ~Person() {
        std::cout << "Des" <<value<< std::endl;
    }
    int value;
};

int main()
{
    std::shared_ptr<Person> p1(new Person(1));// Person(1)的引用计数为1

    std::shared_ptr<Person> p2 = std::make_shared<Person>(2);

    p1.reset(new Person(3));// 首先生成新对象，然后引用计数减1，引用计数为0，故析构Person(1)
                            // 最后将新对象的指针交给智能指针

    std::shared_ptr<Person> p3 = p1;//现在p1和p3同时指向Person(3)，Person(3)的引用计数为2

    p1.reset();//Person(3)的引用计数为1
    p3.reset();//Person(3)的引用计数为0，析构Person(3)
    return 0;
}
reset()包含两个操作。当智能指针中有值的时候，调用reset()会使引用计数减1.当调用reset(new xxx())重新赋值时，智能指针首先是生成新对象，然后将就对象的引用计数减1
(当然，如果发现引用计数为0时，则析构旧对象)，然后将新对象的指针交给智能指针保管。
```

### unique_ptr的使用
> unique_ptr“唯一”拥有其所指对象，同一时刻只能有一个unique_ptr指向给定对象（通过禁止拷贝语义、只有移动语义来实现）。相比与原始指针unique_ptr用于其RAII的特性，使得在出现异常的情况下，动态资源能得到释放。unique_ptr指针本身的生命周期：从unique_ptr指针创建时开始，直到离开作用域。离开作用域时，若其指向对象，则将其所指对象销毁(默认使用delete操作符，用户可指定其他操作)。unique_ptr指针与其所指对象的关系：在智能指针生命周期内，可以改变智能指针所指对象，如创建智能指针时通过构造函数指定、通过reset方法重新指定、通过release方法释放所有权、s通过移动语义转移所有权，

```C++
#include <iostream>
#include <memory>

int main() {
    {
        std::unique_ptr<int> uptr(new int(10));  //绑定动态对象
        //std::unique_ptr<int> uptr2 = uptr;  //不能賦值
        //std::unique_ptr<int> uptr2(uptr);  //不能拷貝
        std::unique_ptr<int> uptr2 = std::move(uptr); //轉換所有權
        uptr2.release(); //释放所有权
    }
    //超過uptr的作用域，內存釋放，因为智能指针本质是一个普通的对象，到了这里智能指针变量就会被释放了，导致他虚构函数的执行，从而释放他原有的真实的对象
}

可以看出来unique_ptr本质跟shared_ptr没有多大的区别，唯一的区别就是不能赋值，不能拷贝，通过move操作转换之后，就会转换所有权
```

### 实操

了解了引用的本质，跟智能指针的本质之后，写代码就不会有那么多的疑问了，比如
```C++

//检查执行Tracker请求
std::unique_ptr<AnnRequest> TrackerWatcherCommand::checkAndExuteTracke(DownloadEngine* e,std::shared_ptr<AnnounceTier>& annouceItem)
{
    //创造用于连接tracker的请求，是要按照规范来的
    std::string uri = btAnnounce_->getAnnounceUrl(annouceItem);
    LOGD("TrackerWatcherCommand::createAnnounce announceUrl %s", uri.c_str());
    uri_split_result res;
    memset(&res, 0, sizeof(res));
    if (uri_split(&res, uri.c_str()) == 0) {
        // Without UDP tracker support, send it to normal tracker flow  and make it fail.
        std::unique_ptr<AnnRequest> treq;
        //根据tracker的url类型，创建不同的类型的请求，比如udp，http
        if (udpTrackerClient_ && uri::getFieldString(res, USR_SCHEME, uri.c_str()) == "udp") {
            uint16_t localPort;
            //修改为UDP端口 这个主要是用来告诉tracker服务器收集当前连接用户的peer信息，好让下一个用户连接上tracker服务器之后，他可以返回当前已经连接tracker服务器peer信息
            localPort = e->getBtRegistry()->getUdpPort();
            //创建udp类型的Tracker连接
            treq = createUDPAnnRequest(uri::getFieldString(res, USR_HOST, uri.c_str()), res.port, localPort, annouceItem);
        } else {
            //创建Http类型的Tracker连接
            treq = createHTTPAnnRequest(uri,annouceItem);
        }
        return std::move(treq);
    }
    return nullptr;
}

比如这里的返回值为什么不是返回一个引用，因为返回值是来自局部变量， std::unique_ptr<AnnRequest> treq 当这个函数执行完毕之后，这个智能指针变量就会被销毁，如果返回一个引用的化，
引用的本质是一个指针，那么外部得到这个返回值进行访问的时候，肯定是闪退的，因为他访问了一个非法的地址，所以这里不能返回引用

再比如：
//当前还允许tracker 并发执行的数量没有达到最大的限度
if(btAnnounce_->getMaxConcurrentSize() > 0)
{
   //获取到当前可以执行的 tracker 请求的 AnnounceTier 集合,注意这里的needExuteAnnounceList 为临时变量
   std::deque<std::shared_ptr<AnnounceTier>> needExuteAnnounceList = btAnnounce_->getCanExeuteUrlList();
   //当前最多可以执行的tracker 请求的并发数不应该大于 当前种子最多的tracker 并发数
   int minSize = btAnnounce_->getMaxConcurrentSize() > needExuteAnnounceList.size() ? needExuteAnnounceList.size() : btAnnounce_->getMaxConcurrentSize();
   //遍历将每一个AnnounceTier，变成对应的AnnRequest
   for(int i = 0; i< minSize;i++)
   {
       std::shared_ptr<AnnounceTier> tierItem = needExuteAnnounceList.front();
       auto request = checkAndExuteTracke(e_,tierItem);
       //开始执行请求
       request->issue(e_);
       //设置并发数量减一处理
       btAnnounce_->setConcurrentSize((btAnnounce_->getMaxConcurrentSize() - 1));
       //设置为请求开始了
       tierItem->setAnnounceExute();
       //将当前的请求添加到发送队列中
       trakerRequestList_.push_back(std::move(request));
       //弹出
       needExuteAnnounceList.pop_front();
       LOGD("tracker request created trakerRequestList_ size %d",trakerRequestList_.size());
   }
}

std::unique_ptr<AnnRequest> TrackerWatcherCommand::checkAndExuteTracke(DownloadEngine* e,std::shared_ptr<AnnounceTier>& annouceItem)
{
	....
}
这里的annouceItem 可以为一个引用，这里就应该要使用一个引用了，因为引用本质是一个地址，如果不使用引用，使用 std::shared_ptr<AnnounceTier> 那就相当于是重新的新建了一个变量
执行了浅拷贝的操作，至于为什么能使用引用，是因为tierItem 的作用域已经足够，虽然是局部变量


通过使用指针指针，很多事情都不用自己管理，比如 我们在类成员中定义这样的成员
//用于存储当前tracker 请求的集合
std::deque<std::unique_ptr<AnnRequest>> trakerRequestList_;

对这个集合的移除我们可以像普通变量来操作，因为智能指针本质就是一个普通的对象，从集合中移除了，也就释放了
trakerRequestList_.erase(std::remove_if(std::begin(trakerRequestList_), std::end(trakerRequestList_),
                                            [&](const std::unique_ptr<AnnRequest> &ent) {
                                                if(ent->stopped())
                                                {
                                                    //移除的时候，让并发数量加一
                                                    btAnnounce_->setConcurrentSize((btAnnounce_->getMaxConcurrentSize() + 1));
                                                }
                                                return ent->stopped();
                                            }),
                             std::end(trakerRequestList_));


对于能否使用引用，本质要看这个变量的声明周期，再看一个

 //当前还允许tracker 并发执行的数量没有达到最大的限度
  if(btAnnounce_->getMaxConcurrentSize() > 0)
  {
      //获取到当前可以执行的 tracker 请求的 AnnounceTier 集合
      std::deque<std::shared_ptr<AnnounceTier>> needExuteAnnounceList = btAnnounce_->getCanExeuteUrlList();
      //当前最多可以执行的tracker 请求的并发数不应该大于 当前种子最多的tracker 并发数
      int minSize = btAnnounce_->getMaxConcurrentSize() > needExuteAnnounceList.size() ? needExuteAnnounceList.size() : btAnnounce_->getMaxConcurrentSize();
      //遍历将每一个AnnounceTier，变成对应的AnnRequest
      for(int i = 0; i< minSize;i++)
      {
          std::shared_ptr<AnnounceTier> tierItem = needExuteAnnounceList.front();
          auto request = checkAndExuteTracke(e_,tierItem);
          //开始执行请求
          request->issue(e_);
          //设置并发数量减一处理
          btAnnounce_->setConcurrentSize((btAnnounce_->getMaxConcurrentSize() - 1));
          //设置为请求开始了
          tierItem->setAnnounceExute();
          //将当前的请求添加到发送队列中
          trakerRequestList_.push_back(std::move(request));
          //弹出
          needExuteAnnounceList.pop_front();
          LOGD("tracker request created trakerRequestList_ size %d",trakerRequestList_.size());
      }
  }

  //请求中，判断是否获取到了结果
  for (auto &requestItem : trakerRequestList_) {
     if (requestItem->stopped())//如果当前请求已经停止
     {
        LOGD("trakcr request stop url %s",requestItem->getAnnounceTier()->getUrl().c_str());
        if (requestItem->success()) {//当前请求成功
             if (requestItem->processResponse(btAnnounce_))
             {
                 requestItem->getAnnounceTier()->trackerSuccess();
                 //根据请求traker 返回的peer 执行连接
                 addConnection();
             } else {
                 //解析失败
                 requestItem->getAnnounceTier()->trackerFailer(requestItem->getCurrentTrackerEvent());
             }
        } else {
             //由BtAnnounce 告知哪个tracker 请求失败
            requestItem->getAnnounceTier()->trackerFailer(requestItem->getCurrentTrackerEvent());
        }
     }
  }			

  //获取当前请求对应的 AnnounceTier,这边不能返回一个引用，因为是局部变量
  virtual std::shared_ptr<AnnounceTier> getAnnounceTier() = 0;
  
  我原本想在getAnnounceTier 的时候返回一个引用，如果可行的化，性能会更加高，但是这是错误的，因为这个 AnnounceTier 是由 needExuteAnnounceList 局部的成员集合传递进去的，
  而且下面的getAnnounceTier() 跟上面的 needExuteAnnounceList 是异步的，所以会导致 等你获取这个引用的时候，其实这个引用已经被释放掉了，导致访问了非法的地址，导致闪退
```


### 总结
应该大量的使用智能指针，对于是否应该要使用引用，看这个变量的生命周期，如果生命周期允许的化，可以使用