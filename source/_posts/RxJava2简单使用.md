---
layout: pager
title: RxJava2简单使用
date: 2018-05-28 16:16:20
tags: [Android,RxJava,RxAndroid]
description:  RxJava2简单使用
---

### 概述

> RxJava2简单使用

<!--more-->

### 为什么要学 RxJava?
> 提升开发效率，降低维护成本一直是开发团队永恒不变的宗旨。近两年来国内的技术圈子中越来越多的开始提及 RxJava ，越来越多的应用和面试中都会有 RxJava ，而就目前的情况，Android 的网络库基本被 Retrofit + OkHttp 一统天下了，而配合上响应式编程 RxJava 可谓如鱼得水。想必大家肯定被近期的 Kotlin 炸开了锅，发现其中有个非常好的优点就是简洁，支持函数式编程。是的， RxJava 最大的优点也是简洁，但它不止是简洁，而且是** 随着程序逻辑变得越来越复杂，它依然能够保持简洁 **（这货洁身自好呀有木有）。

### 什么是响应式编程？
> 上面我们提及了响应式编程，不少新司机对它可谓一脸懵逼，那什么是响应式编程呢？响应式编程是一种基于异步数据流概念的编程模式。数据流就像一条河：它可以被观测，被过滤，被操作，
或者为新的消费者与另外一条流合并为一条新的流。

响应式编程的一个关键概念是事件
> 事件可以被等待，可以触发过程，也可以触发其它事件。事件是唯一的以合适的方式将我们的现实世界映射到我们的软件中：如果屋里太热了我们就打开一扇窗户。同样的，当我们的天气app从服务端获取到新的天气数据后，我们需要更新app上展示天气信息的UI；汽车上的车道偏移系统探测到车辆偏移了正常路线就会提醒驾驶者纠正，就是是响应事件。

今天，响应式编程最通用的一个场景是UI：我们的移动App必须做出对网络调用、用户触摸输入和系统弹框的响应。在这个世界上，软件之所以是事件驱动并响应的是因为现实生活也是如此。

### create操作符
最常见的操作符，用于生产一个发射对象
```java
Observable.create(new ObservableOnSubscribe<Integer>() {
            @Override
            public void subscribe(@NonNull ObservableEmitter<Integer> e) throws Exception {
                mRxOperatorsText.append("Observable emit 1" + "\n");
                Log.e(TAG, "Observable emit 1" + "\n");
                e.onNext(1);
                mRxOperatorsText.append("Observable emit 2" + "\n");
                Log.e(TAG, "Observable emit 2" + "\n");
                e.onNext(2);
                mRxOperatorsText.append("Observable emit 3" + "\n");
                Log.e(TAG, "Observable emit 3" + "\n");
                e.onNext(3);
                e.onComplete();
                mRxOperatorsText.append("Observable emit 4" + "\n");
                Log.e(TAG, "Observable emit 4" + "\n" );
                e.onNext(4);
            }
        }).subscribe(new Observer<Integer>() {
            private int i;
            private Disposable mDisposable;

            @Override
            public void onSubscribe(@NonNull Disposable d) {
                mRxOperatorsText.append("onSubscribe : " + d.isDisposed() + "\n");
                Log.e(TAG, "onSubscribe : " + d.isDisposed() + "\n" );
                mDisposable = d;
            }

            @Override
            public void onNext(@NonNull Integer integer) {
                mRxOperatorsText.append("onNext : value : " + integer + "\n");
                Log.e(TAG, "onNext : value : " + integer + "\n" );
                i++;
                if (i == 2) {
                    // 在RxJava 2.x 中，新增的Disposable可以做到切断的操作，让Observer观察者不再接收上游事件,所以2之后的事件就会被中断接受
                    mDisposable.dispose();
                    mRxOperatorsText.append("onNext : isDisposable : " + mDisposable.isDisposed() + "\n");
                    Log.e(TAG, "onNext : isDisposable : " + mDisposable.isDisposed() + "\n");
                }
            }

            @Override
            public void onError(@NonNull Throwable e) {
                mRxOperatorsText.append("onError : value : " + e.getMessage() + "\n");
                Log.e(TAG, "onError : value : " + e.getMessage() + "\n" );
            }

            @Override
            public void onComplete() {
                mRxOperatorsText.append("onComplete" + "\n");
                Log.e(TAG, "onComplete" + "\n" );
            }
});
```
![结果显示](/uploads/RxJava简单使用/create.png)

### Zip操作符

Zip操作符 用来合并事件专用，分别从俩个上游事件中各取一个组合，一个事件只能被使用一次，顺序严格按照事件发送的顺序 最终下游事件收到的是上游事件最少的数目相同,必须俩俩配对，多余的舍弃

```java
Observable.zip(getStringObservable(), getIntegerObservable(), new BiFunction<String, Integer, String>() {
    @Override
    public String apply(@NonNull String s, @NonNull Integer integer) throws Exception {
            return s + integer;
        }
    }).subscribe(new Consumer<String>() {
    @Override
    public void accept(@NonNull String s) throws Exception {
        mRxOperatorsText.append("zip : accept : " + s + "\n");
    }
});

private Observable<String> getStringObservable() {
    return Observable.create(new ObservableOnSubscribe<String>() {
        @Override
        public void subscribe(@NonNull ObservableEmitter<String> e) throws Exception {
            if (!e.isDisposed()) {
                e.onNext("A");
                mRxOperatorsText.append("String emit : A \n");
                e.onNext("B");
                mRxOperatorsText.append("String emit : B \n");
                e.onNext("C");
                mRxOperatorsText.append("String emit : C \n");
            }
        }
    });
}

private Observable<Integer> getIntegerObservable() {
    return Observable.create(new ObservableOnSubscribe<Integer>() {
        @Override
        public void subscribe(@NonNull ObservableEmitter<Integer> e) throws Exception {
            if (!e.isDisposed()) {
                e.onNext(1);
                mRxOperatorsText.append("Integer emit : 1 \n");
                e.onNext(2);
                mRxOperatorsText.append("Integer emit : 2 \n");
                e.onNext(3);
                mRxOperatorsText.append("Integer emit : 3 \n");
                e.onNext(4);
                mRxOperatorsText.append("Integer emit : 4 \n");
                e.onNext(5);
                mRxOperatorsText.append("Integer emit : 5 \n");
            }
        }
    });
}
```
![结果显示](/uploads/RxJava简单使用/zip.png)

### Map操作符

Map操作符，可以针对上游发送的每一个事件应用一个函数，使得每一个事件都按照指定的函数去变化

```java
Observable.create(new ObservableOnSubscribe<Integer>() {
            @Override
            public void subscribe(@NonNull ObservableEmitter<Integer> e) throws Exception {
                e.onNext(1);
                e.onNext(2);
                e.onNext(3);
            }
        }).map(new Function<Integer, String>() {
            @Override
            public String apply(@NonNull Integer integer) throws Exception {
                return "This is result " + integer;
            }
        }).subscribe(new Consumer<String>() {
            @Override
            public void accept(@NonNull String s) throws Exception {
                mRxOperatorsText.append("accept : " + s +"\n");
            }
        });

```
![结果显示](/uploads/RxJava简单使用/map.png)

### FlatMap操作符

FlatMap将一个发送事件的上游Observable变成多个发送事件的Observables 然后将他们发射的结果合并后放进一个单独的Observable里面,注意FlatMap不保证事件的顺序

```java
Observable.create(new ObservableOnSubscribe<Integer>() {
            @Override
            public void subscribe(@NonNull ObservableEmitter<Integer> e) throws Exception {
                e.onNext(1);
                e.onNext(2);
                e.onNext(3);
            }
        }).flatMap(new Function<Integer, ObservableSource<String>>() {
            @Override
            public ObservableSource<String> apply(@NonNull Integer integer) throws Exception {
                List<String> list = new ArrayList<>();
                for (int i = 0; i < 3; i++) {
                    list.add("I am value " + integer);
                }
                int delayTime = (int) (1 + Math.random() * 10);
                return Observable.fromIterable(list).delay(delayTime, TimeUnit.MILLISECONDS);//返回一个Observable对象
            }
        }).subscribeOn(Schedulers.newThread())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Consumer<String>() {
                    @Override
                    public void accept(@NonNull String s) throws Exception {
                        mRxOperatorsText.append("flatMap : accept : " + s + "\n");
                    }
});

```
![结果显示](/uploads/RxJava简单使用/flatMap.png)

### concatMap操作符

concatMap作用和flatMap几乎一模一样，唯一的区别是它能保证事件的顺序

```java
 
Observable.create(new ObservableOnSubscribe<Integer>() {
            @Override
            public void subscribe(@NonNull ObservableEmitter<Integer> e) throws Exception {
                e.onNext(1);
                e.onNext(2);
                e.onNext(3);
            }
        }).concatMap(new Function<Integer, ObservableSource<String>>() {
            @Override
            public ObservableSource<String> apply(@NonNull Integer integer) throws Exception {
                List<String> list = new ArrayList<>();
                for (int i = 0; i < 3; i++) {
                    list.add("I am value " + integer);
                }
                int delayTime = (int) (1 + Math.random() * 10);
                return Observable.fromIterable(list).delay(delayTime, TimeUnit.MILLISECONDS);
            }
        }).subscribeOn(Schedulers.newThread())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Consumer<String>() {
                    @Override
                    public void accept(@NonNull String s) throws Exception {
                        mRxOperatorsText.append("concatMap : accept : " + s + "\n");
                    }
});


```
![结果显示](/uploads/RxJava简单使用/concatMap.png)

### doOnNext操作符

让订阅者在接受到数据之前干点事情的操作符

```java
Observable.just(1, 2, 3, 4)
                .doOnNext(new Consumer<Integer>() {
                    @Override
                    public void accept(@NonNull Integer integer) throws Exception {
                        mRxOperatorsText.append("doOnNext 保存 " + integer + "成功" + "\n");
                    }
                }).subscribe(new Consumer<Integer>() {
            @Override
            public void accept(@NonNull Integer integer) throws Exception {
                mRxOperatorsText.append("doOnNext :" + integer + "\n");
            }
});

```
![结果显示](/uploads/RxJava简单使用/doOnOneNext.png)

### filter操作符

Filter 操作符，过滤操作符，取正确的值，当filter函数中返回true的就会被获取，返回false的就会被过滤掉

```java
Observable.just(1, 20, 65, -5, 7, 19)
                .filter(new Predicate<Integer>() {
                    @Override
                    public boolean test(@NonNull Integer integer) throws Exception {
                        return integer >= 10;
                    }
                }).subscribe(new Consumer<Integer>() {
            @Override
            public void accept(@NonNull Integer integer) throws Exception {
                mRxOperatorsText.append("filter : " + integer + "\n");
            }
});

```
![结果显示](/uploads/RxJava简单使用/filter.png)

### skip操作符

skip操作符，接受一个long型参数，代表跳过多少个数目的事件再开始接收 ,这里是跳过前面的俩个，才开始接收，所以打印3,4,5

```java
Observable.just(1,2,3,4,5)
                .skip(2)
                .subscribe(new Consumer<Integer>() {
                    @Override
                    public void accept(@NonNull Integer integer) throws Exception {
                        mRxOperatorsText.append("skip : "+integer + "\n");
                    }
});
```
![结果显示](/uploads/RxJava简单使用/skip.png)

### take操作符

用于指定订阅者最多收到多少数据 ,这里是指定最多能收到2个数据，所以会打印 前面的俩个内容 也即是 1,2

```java
Flowable.fromArray(1,2,3,4,5)
                .take(2)
                .subscribe(new Consumer<Integer>() {
                    @Override
                    public void accept(@NonNull Integer integer) throws Exception {
                        mRxOperatorsText.append("take : "+integer + "\n");
                    }
});

```
![结果显示](/uploads/RxJava简单使用/take.png)

### timer操作符

Timer 操作符既可以延迟执行一段逻辑，也可以间隔的执行一段逻辑，要注意的是timer和Interval都默认执行在一个新线程上

```java
mRxOperatorsText.append("timer start : " + TimeUtil.getNowStrTime() + "\n");
Observable.timer(2, TimeUnit.SECONDS)
                .subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())//默认执行在新线程上，所以要切换到主线程来更新UI操作
                .subscribe(new Consumer<Long>() {
                    @Override
                    public void accept(@NonNull Long aLong) throws Exception {
                        mRxOperatorsText.append("timer :" + aLong + " at " + TimeUtil.getNowStrTime() + "\n");
                    }
});

```
![结果显示](/uploads/RxJava简单使用/timer.png)

### interval操作符

Interval可以间隔的执行操作，默认在新线程，所以对于返回的结果要执行切换线程

```java
mRxOperatorsText.append("interval start : " + TimeUtil.getNowStrTime() + "\n");

mDisposable = Observable.interval(3, 2, TimeUnit.SECONDS)
                .subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread()) // 由于interval默认在新线程，所以我们应该切回主线程
                .subscribe(new Consumer<Long>() {
                    @Override
                    public void accept(@NonNull Long aLong) throws Exception {
                        mRxOperatorsText.append("interval :" + aLong + " at " + TimeUtil.getNowStrTime() + "\n");
                    }
});

```
![结果显示](/uploads/RxJava简单使用/interval.png)

### just操作符

just操作符，接收可变的参数，依次发送事件，观察者会依次的收到onNext事件

```java
Observable.just("1", "2", "3")
                .subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Consumer<String>() {
                    @Override
                    public void accept(@NonNull String s) throws Exception {
                        mRxOperatorsText.append("accept : onNext : " + s + "\n");
                    }
});
```
![结果显示](/uploads/RxJava简单使用/just.png)

### Single操作符

Single 操作符，只会接收一个参数，而SingleObserver只会调用onError或者onSuccess回调

```java
Single.just(new Random().nextInt())
                .subscribe(new SingleObserver<Integer>() {
                    @Override
                    public void onSubscribe(@NonNull Disposable d) {

                    }

                    @Override
                    public void onSuccess(@NonNull Integer integer) {
                        mRxOperatorsText.append("single : onSuccess : "+integer+"\n");
                    }

                    @Override
                    public void onError(@NonNull Throwable e) {
                        mRxOperatorsText.append("single : onError : "+e.getMessage()+"\n");
                    }
});
```
![结果显示](/uploads/RxJava简单使用/Single.png)

### concat操作符

concat连接操作符，可接收Observable的可变参数，或者Observable的集合 ,可以保证顺序

```java
Observable.concat(Observable.just(1,2,3), Observable.just(4,5,6))
                .subscribe(new Consumer<Integer>() {
                    @Override
                    public void accept(@NonNull Integer integer) throws Exception {
                        mRxOperatorsText.append("concat : "+ integer + "\n");
                    }
});

```
![结果显示](/uploads/RxJava简单使用/concat.png)

### distinct操作符

distinct操作符，简单的去重操作符,就是将重复的事件过滤掉

```java
Observable.just(1, 1, 1, 2, 2, 3, 4, 5)
                .distinct()
                .subscribe(new Consumer<Integer>() {
                    @Override
                    public void accept(@NonNull Integer integer) throws Exception {
                        mRxOperatorsText.append("distinct : " + integer + "\n");
                    }
                });
```
![结果显示](/uploads/RxJava简单使用/distinct.png)

### buffer操作符

buffer操作符，将数据按skip分成最长不超过 count的 buffer，然后生成一个Observable

```java
Observable.just(1, 2, 3, 4, 5)
                .buffer(3, 2)
                .subscribe(new Consumer<List<Integer>>() {
                    @Override
                    public void accept(@NonNull List<Integer> integers) throws Exception {
                        mRxOperatorsText.append("buffer size : " + integers.size() + "\n");
                        mRxOperatorsText.append("buffer value : ");
                        for (Integer i : integers) {
                            mRxOperatorsText.append(i + "");
                        }
                        mRxOperatorsText.append("\n");
                    }
                });
```
![结果显示](/uploads/RxJava简单使用/buffer.png)

### debounce操作符

debounce 操作符，可以用来过滤掉发射速率过快的数据项，这里的意思就是用来过滤小于500毫秒的事件，所以大于500毫秒的才会打印出来

```java
Observable.create(new ObservableOnSubscribe<Integer>() {
            @Override
            public void subscribe(@NonNull ObservableEmitter<Integer> emitter) throws Exception {
                // send events with simulated time wait
                emitter.onNext(1); // skip
                Thread.sleep(400);
                emitter.onNext(2); // deliver
                Thread.sleep(505);
                emitter.onNext(3); // skip
                Thread.sleep(100);
                emitter.onNext(4); // deliver
                Thread.sleep(605);
                emitter.onNext(5); // deliver
                Thread.sleep(510);
                emitter.onComplete();
            }
        }).debounce(500, TimeUnit.MILLISECONDS)
                .subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Consumer<Integer>() {
                    @Override
                    public void accept(@NonNull Integer integer) throws Exception {
                        mRxOperatorsText.append("debounce :" + integer + "\n");
                    }
});

```
![结果显示](/uploads/RxJava简单使用/debonce.png)

### defer操作符

defer操作符，如果该Observable没有被订阅，就不会生成新的Observable

```java
Observable<Integer> observable = Observable.defer(new Callable<ObservableSource<Integer>>() {
            @Override
            public ObservableSource<Integer> call() throws Exception {
                return Observable.just(1, 2, 3);
            }
        });

        observable.subscribe(new Observer<Integer>() {
            @Override
            public void onSubscribe(@NonNull Disposable d) {

            }

            @Override
            public void onNext(@NonNull Integer integer) {
                mRxOperatorsText.append("defer : " + integer + "\n");
            }

            @Override
            public void onError(@NonNull Throwable e) {
                mRxOperatorsText.append("defer : onError : " + e.getMessage() + "\n");
            }

            @Override
            public void onComplete() {
                mRxOperatorsText.append("defer : onComplete\n");
            }
        });

```
![结果显示](/uploads/RxJava简单使用/defer.png)

### last操作符

Last操作符，取出最后的一个值，参数是没有值的时候默认值,不管参数是怎么设置的，反正就只会取出事件中的最后一个值

```java
Observable.just(1, 2, 3)
                .last(1)
                .subscribe(new Consumer<Integer>() {
                    @Override
                    public void accept(@NonNull Integer integer) throws Exception {
                        mRxOperatorsText.append("last : " + integer + "\n");
                        Log.e(TAG, "last : " + integer + "\n");
                    }
});

```
![结果显示](/uploads/RxJava简单使用/last.png)

### merge操作符

merge操作符，将多个的Observable合起来，可以接收可变参数，也支持使用迭代器集合 ,注意它和concat的区别在于，不用等到发射器A发送完所有的事件再进行发射器B的发送

```java
Observable.merge(Observable.just(1, 2), Observable.just(3, 4, 5))
                .subscribe(new Consumer<Integer>() {
                    @Override
                    public void accept(@NonNull Integer integer) throws Exception {
                        mRxOperatorsText.append("merge :" + integer + "\n");
                    }
});
```
![结果显示](/uploads/RxJava简单使用/merge.png)

### reduce操作符

reduce操作符，就是一次用一个方法处理一个值，这里就是将所有的输入事件执行相加，只会得到最后的一个结果

```java
Observable.just(1, 2, 3)
                .reduce(new BiFunction<Integer, Integer, Integer>() {
                    @Override
                    public Integer apply(@NonNull Integer integer, @NonNull Integer integer2) throws Exception {
                        return integer + integer2;
                    }
                }).subscribe(new Consumer<Integer>() {
            @Override
            public void accept(@NonNull Integer integer) throws Exception {
                mRxOperatorsText.append("reduce : " + integer + "\n");
            }
});
```
![结果显示](/uploads/RxJava简单使用/redouce.png)

### scan操作符

scan操作符，跟reduce操作符差不多，区别在于reduce只会输出结果，而scan会将过程中的每一个结果输出

```java
Observable.just(1, 2, 3)
                .scan(new BiFunction<Integer, Integer, Integer>() {
                    @Override
                    public Integer apply(@NonNull Integer integer, @NonNull Integer integer2) throws Exception {
                        return integer + integer2;
                    }
                }).subscribe(new Consumer<Integer>() {
            @Override
            public void accept(@NonNull Integer integer) throws Exception {
                mRxOperatorsText.append("scan " + integer + "\n");
            }
});

```
![结果显示](/uploads/RxJava简单使用/scan.png)

### PublishSubject操作符

创建一个PublishSubject对象，PublishSubject的特点就是会通知每一个观察者，调用对应的onNext函数

```java
mRxOperatorsText.append("PublishSubject\n");
PublishSubject<Integer> publishSubject = PublishSubject.create();
publishSubject.subscribe(new Observer<Integer>() {
            @Override
            public void onSubscribe(@NonNull Disposable d) {
                mRxOperatorsText.append("First onSubscribe :"+d.isDisposed()+"\n");
            }

            @Override
            public void onNext(@NonNull Integer integer) {
                mRxOperatorsText.append("First onNext value :"+integer + "\n");
            }

            @Override
            public void onError(@NonNull Throwable e) {
                mRxOperatorsText.append("First onError:"+e.getMessage()+"\n");
            }

            @Override
            public void onComplete() {
                mRxOperatorsText.append("First onComplete!\n");
            }
});


这里通过PublishSubject发送事件，因为这个时候只有一个观察者订阅了该事件，所以只有第一个观察者能接受到事件的发送
publishSubject.onNext(1);
publishSubject.onNext(2);
publishSubject.onNext(3);

publishSubject.subscribe(new Observer<Integer>() {
            @Override
            public void onSubscribe(@NonNull Disposable d) {
                mRxOperatorsText.append("Second onSubscribe :"+d.isDisposed()+"\n");
            }

            @Override
            public void onNext(@NonNull Integer integer) {
                mRxOperatorsText.append("Second onNext value :"+integer + "\n");
            }

            @Override
            public void onError(@NonNull Throwable e) {
                mRxOperatorsText.append("Second onError:"+e.getMessage()+"\n");
            }

            @Override
            public void onComplete() {
                mRxOperatorsText.append("Second onComplete!\n");
            }
});

到了这里就有俩个观察者订阅了该事件，所以这个时候执行事件的发送，就可以分别的到达俩个观察者中
publishSubject.onNext(4);
publishSubject.onNext(5);
publishSubject.onComplete();

```
![结果显示](/uploads/RxJava简单使用/publishSubject.png)

### AsyncSubject操作符

跟PublisShSubject一样既有将事件发送给每一个观察者的能力，当执行事件发送的时候，但是，他在调用onComplete之前，除了subscribe()函数，其他的操作都会被缓存
在调用onComplete之后，只有最后一个onNext会生效

```java
mRxOperatorsText.append("AsyncSubject\n");
AsyncSubject<Integer> asyncSubject = AsyncSubject.create();
asyncSubject.subscribe(new Observer<Integer>() {
            @Override
            public void onSubscribe(@NonNull Disposable d) {
                mRxOperatorsText.append("First onSubscribe :"+d.isDisposed()+"\n");
            }

            @Override
            public void onNext(@NonNull Integer integer) {
                mRxOperatorsText.append("First onNext value :"+integer + "\n");
            }

            @Override
            public void onError(@NonNull Throwable e) {
                mRxOperatorsText.append("First onError:"+e.getMessage()+"\n");
            }

            @Override
            public void onComplete() {
                mRxOperatorsText.append("First onComplete!\n");
            }
        });

AsyncSubject 在调用onComplete之前的事件都会被缓存，除了onSunscribe函数
asyncSubject.onNext(1);
asyncSubject.onNext(2);
asyncSubject.onNext(3);

asyncSubject.subscribe(new Observer<Integer>() {
            @Override
            public void onSubscribe(@NonNull Disposable d) {
                mRxOperatorsText.append("Second onSubscribe :"+d.isDisposed()+"\n");
            }

            @Override
            public void onNext(@NonNull Integer integer) {
                mRxOperatorsText.append("Second onNext value :"+integer + "\n");
            }

            @Override
            public void onError(@NonNull Throwable e) {
                mRxOperatorsText.append("Second onError:"+e.getMessage()+"\n");
            }

            @Override
            public void onComplete() {
                mRxOperatorsText.append("Second onComplete!\n");
            }
});

在调用onComplete之后，会传递最后的一个事件
asyncSubject.onNext(4);
asyncSubject.onNext(5);
asyncSubject.onComplete();
```
![结果显示](/uploads/RxJava简单使用/AsyncSubject.png)


### BehaviorSubject操作符

BehaviorSubject操作也有跟PublishSubject 一样的功能既有将事件发送给每一个订阅者的能力，但是使用BehaviorSubject，会缓存第一个订阅者的事件
当第二个订阅者订阅的时候，当执行了onSubscribe之后，就会先将缓存的第一个订阅者的事件，发送给这个订阅者，之后再执行当前订阅事件的分发操作

```java
mRxOperatorsText.append("BehaviorSubject\n");
BehaviorSubject<Integer> behaviorSubject = BehaviorSubject.create();
behaviorSubject.subscribe(new Observer<Integer>() {
            @Override
            public void onSubscribe(@NonNull Disposable d) {
                mRxOperatorsText.append("First onSubscribe :"+d.isDisposed()+"\n");
            }

            @Override
            public void onNext(@NonNull Integer integer) {
                mRxOperatorsText.append("First onNext value :"+integer + "\n");
            }

            @Override
            public void onError(@NonNull Throwable e) {
                mRxOperatorsText.append("First onError:"+e.getMessage()+"\n");
            }

            @Override
            public void onComplete() {
                mRxOperatorsText.append("First onComplete!\n");
            }
        });

behaviorSubject.onNext(1);
behaviorSubject.onNext(2);
behaviorSubject.onNext(3);

behaviorSubject.subscribe(new Observer<Integer>() {
            @Override
            public void onSubscribe(@NonNull Disposable d) {
                mRxOperatorsText.append("Second onSubscribe :"+d.isDisposed()+"\n");
            }

            @Override
            public void onNext(@NonNull Integer integer) {
                mRxOperatorsText.append("Second onNext value :"+integer + "\n");
            }

            @Override
            public void onError(@NonNull Throwable e) {
                mRxOperatorsText.append("Second onError:"+e.getMessage()+"\n");
            }

            @Override
            public void onComplete() {
                mRxOperatorsText.append("Second onComplete!\n");
            }
        });

behaviorSubject.onNext(4);
behaviorSubject.onNext(5);
behaviorSubject.onComplete();
```
![结果显示](/uploads/RxJava简单使用/BehaviorSubject.png)

### Completable操作符

Completable 操作符，只关心结果，也即是说CompleteAble 是没有onNext的，要不成功，要不出错,在Subscribe后的某一个时间点返回结果

```java
mRxOperatorsText.append("Completable\n");
Completable.timer(1, TimeUnit.SECONDS)
                .subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new CompletableObserver() {
                    @Override
                    public void onSubscribe(@NonNull Disposable d) {
                        mRxOperatorsText.append("onSubscribe : d :" + d.isDisposed() + "\n");
                    }

                    @Override
                    public void onComplete() {
                        mRxOperatorsText.append("onComplete\n");
                    }

                    @Override
                    public void onError(@NonNull Throwable e) {
                        mRxOperatorsText.append("onError :" + e.getMessage() + "\n");
                    }
});

```
![结果显示](/uploads/RxJava简单使用/Completable.png)

### Flowable操作符

reduce 操作符会会将最终的结果返回，可以指定一个基数，这里就是将所有的值相加，在100的基础上

```java
Flowable.just(1,2,3,4)
                .reduce(100, new BiFunction<Integer, Integer, Integer>() {
                    @Override
                    public Integer apply(@NonNull Integer integer, @NonNull Integer integer2) throws Exception {
                        return integer+integer2;
                    }
                }).subscribe(new Consumer<Integer>() {
            @Override
            public void accept(@NonNull Integer integer) throws Exception {
                mRxOperatorsText.append("Flowable :"+integer+"\n");
            }
});
```
![结果显示](/uploads/RxJava简单使用/flowable.png)


### flatMap使用场景

多个网络请求依次依赖，比如：
1、注册用户前先通过接口A获取当前用户是否已注册，再通过接口B注册;
2、注册后自动登录，先通过注册接口注册用户信息，注册成功后马上调用登录接口进行自动登录。

```java
Rx2AndroidNetworking.get("http://www.tngou.net/api/food/list")
                .addQueryParameter("rows", 1 + "")
                .build()
                .getObjectObservable(FoodList.class) // 发起获取食品列表的请求，并解析到FootList
                .subscribeOn(Schedulers.io())        // 在io线程进行网络请求
                .observeOn(AndroidSchedulers.mainThread()) // 在主线程处理获取食品列表的请求结果
                .doOnNext(new Consumer<FoodList>() {
                    @Override
                    public void accept(@NonNull FoodList foodList) throws Exception {
                        // 先根据获取食品列表的响应结果做一些操作
                        mRxOperatorsText.append("accept: doOnNext :" + foodList.toString()+"\n");
                    }
                })
                .observeOn(Schedulers.io()) // 回到 io 线程去处理获取食品详情的请求
                .flatMap(new Function<FoodList, ObservableSource<FoodDetail>>() {
                    @Override
                    public ObservableSource<FoodDetail> apply(@NonNull FoodList foodList) throws Exception {
                        if (foodList != null && foodList.getTngou() != null && foodList.getTngou().size() > 0) {
                            return Rx2AndroidNetworking.post("http://www.tngou.net/api/food/show")
                                    .addBodyParameter("id", foodList.getTngou().get(0).getId() + "")
                                    .build()
                                    .getObjectObservable(FoodDetail.class);
                        }
                        return null;

                    }
                })
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Consumer<FoodDetail>() {
                    @Override
                    public void accept(@NonNull FoodDetail foodDetail) throws Exception {
                        mRxOperatorsText.append("accept: success ：" + foodDetail.toString()+"\n");
                    }
                }, new Consumer<Throwable>() {
                    @Override
                    public void accept(@NonNull Throwable throwable) throws Exception {
                        mRxOperatorsText.append("accept: error :" + throwable.getMessage()+"\n");
                    }
                });

```
### map使用场景

```java
mRxOperatorsText.append("RxNetworkActivity\n");
Rx2AndroidNetworking.get("http://api.avatardata.cn/MobilePlace/LookUp?key=ec47b85086be4dc8b5d941f5abd37a4e&mobileNumber=13021671512")
                .build()
                .getObjectObservable(MobileAddress.class)
                .observeOn(AndroidSchedulers.mainThread()) // 为doOnNext() 指定在主线程，否则报错
                .doOnNext(new Consumer<MobileAddress>() {//在返回结果之前，可以指定执行一些东西
                    @Override
                    public void accept(@NonNull MobileAddress data) throws Exception {;
                        mRxOperatorsText.append("\ndoOnNext:"+Thread.currentThread().getName()+"\n" );
                        mRxOperatorsText.append("doOnNext:"+data.toString()+"\n");
                    }
                })
                .map(new Function<MobileAddress, ResultBean>() {
                    @Override
                    public ResultBean apply(@NonNull MobileAddress mobileAddress) throws Exception {
                        mRxOperatorsText.append("\n");
                        mRxOperatorsText.append("map:"+Thread.currentThread().getName()+"\n" );
                        return mobileAddress.getResult();
                    }
                })
                .subscribeOn(Schedulers.io())//指定订阅的时候发生在Io线程可以用来执行一下IO操作
                .subscribe(new Consumer<ResultBean>() {
                    @Override
                    public void accept(@NonNull ResultBean data) throws Exception {
                        mRxOperatorsText.append("\nsubscribe 成功:"+Thread.currentThread().getName()+"\n" );
                        mRxOperatorsText.append("成功:" + data.toString() + "\n");
                    }
                }, new Consumer<Throwable>() {
                    @Override
                    public void accept(@NonNull Throwable throwable) throws Exception {
                        mRxOperatorsText.append("\nsubscribe 失败:"+Thread.currentThread().getName()+"\n" );
                        mRxOperatorsText.append("失败："+ throwable.getMessage()+"\n");
                    }
});
```

### debounce使用场景

debounce 操作符可以过滤掉发射频率过快的数据项

```java
RxView.clicks(mRxOperatorsBtn)
                .debounce(2,TimeUnit.SECONDS) // 过滤掉发射频率小于2秒的发射事件
                .subscribe(new Consumer<Object>() {
                    @Override
                    public void accept(@NonNull Object o) throws Exception {
                        clickBtn();
                    }
});

@SuppressLint("CheckResult")
private void clickBtn() {
        Rx2AndroidNetworking.get("http://www.tngou.net/api/food/list")
                .addQueryParameter("rows",2+"") // 只获取两条数据
                .build()
                .getObjectObservable(FoodList.class)
                .subscribeOn(Schedulers.io())  // 在 io 线程进行网络请求
                .observeOn(AndroidSchedulers.mainThread()) // 在主线程进行更新UI等操作
                .subscribe(new Consumer<FoodList>() {
                    @Override
                    public void accept(@NonNull FoodList foodList) throws Exception {
                        mRxOperatorsText.append("accept: 获取数据成功:"+foodList.toString()+"\n" );
                    }
                }, new Consumer<Throwable>() {
                    @Override
                    public void accept(@NonNull Throwable throwable) throws Exception {
                        mRxOperatorsText.append("accept: 获取数据失败："+throwable.getMessage() +"\n");
                    }
                });
}
```

### 线程切换使用场景
```java
Observable.create(new ObservableOnSubscribe<Response>() {
            @Override
            public void subscribe(@NonNull ObservableEmitter<Response> e) throws Exception {
                Builder builder = new Builder()
                        .url("http://api.avatardata.cn/MobilePlace/LookUp?key=ec47b85086be4dc8b5d941f5abd37a4e&mobileNumber=13021671512")
                        .get();
                Request request = builder.build();
                Call call = new OkHttpClient().newCall(request);
                Response response = call.execute();
                e.onNext(response);
            }
        }).map(new Function<Response, MobileAddress>() {
                    @Override
                    public MobileAddress apply(@NonNull Response response) throws Exception {
                        if (response.isSuccessful()) {
                            ResponseBody body = response.body();
                            if (body != null) {
                                return new Gson().fromJson(body.string(), MobileAddress.class);
                            }
                        }
                        return null;
                    }
                }).observeOn(AndroidSchedulers.mainThread())//指定结果返回在主线程
                .doOnNext(new Consumer<MobileAddress>() {//在返回结果之前可以做点一下其他的事情
                    @Override
                    public void accept(@NonNull MobileAddress s) throws Exception {
                        mRxOperatorsText.append("\ndoOnNext 线程:" + Thread.currentThread().getName() + "\n");
                        mRxOperatorsText.append("doOnNext: 保存成功：" + s.toString() + "\n");

                    }
                }).subscribeOn(Schedulers.io())//指定订阅的事件发生的线程
                .subscribe(new Consumer<MobileAddress>() {//最后的返回结果
                    @Override
                    public void accept(@NonNull MobileAddress data) throws Exception {
                        mRxOperatorsText.append("\nsubscribe 线程:" + Thread.currentThread().getName() + "\n");
                        mRxOperatorsText.append("成功:" + data.toString() + "\n");
                    }
                }, new Consumer<Throwable>() {
                    @Override
                    public void accept(@NonNull Throwable throwable) throws Exception {
                        mRxOperatorsText.append("\nsubscribe 线程:" + Thread.currentThread().getName() + "\n");
                        mRxOperatorsText.append("失败：" + throwable.getMessage() + "\n");
                    }
});
```

### zip使用场景

结合多个接口的数据再更新 UI

```java
Observable<MobileAddress> observable1 = Rx2AndroidNetworking.get("http://api.avatardata.cn/MobilePlace/LookUp?key=ec47b85086be4dc8b5d941f5abd37a4e&mobileNumber=13021671512")
                .build()
                .getObjectObservable(MobileAddress.class);

        Observable<CategoryResult> observable2 = Network.getGankApi()
                .getCategoryData("Android",1,1);

        Observable.zip(observable1, observable2, new BiFunction<MobileAddress, CategoryResult, String>() {
            @Override
            public String apply(@NonNull MobileAddress mobileAddress, @NonNull CategoryResult categoryResult) throws Exception {
                return "合并后的数据为：手机归属地："+mobileAddress.getResult().getMobilearea()+"人名："+categoryResult.results.get(0).who;
            }
        }).subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Consumer<String>() {
                    @Override
                    public void accept(@NonNull String s) throws Exception {
                        Log.e(TAG, "accept: 成功：" + s+"\n");
                    }
                }, new Consumer<Throwable>() {
                    @Override
                    public void accept(@NonNull Throwable throwable) throws Exception {
                        Log.e(TAG, "accept: 失败：" + throwable+"\n");
                    }
});
```

### concat使用场景

Concat 先读取缓存数据并展示UI再获取网络数据刷新UI
1、concat 可以做到不交错的发射两个甚至多个 Observable 的发射物;
2、并且只有前一个 Observable 终止（onComplete）才会订阅下一个 Observable

```java
Observable<FoodList> cache = Observable.create(new ObservableOnSubscribe<FoodList>() {
            @Override
            public void subscribe(@NonNull ObservableEmitter<FoodList> e) throws Exception {
                FoodList data = CacheManager.getInstance().getFoodListData();

                // 在操作符 concat 中，只有调用 onComplete 之后才会执行下一个 Observable
                if (data != null){ // 如果缓存数据不为空，则直接读取缓存数据，而不读取网络数据
                    isFromNet = false;
                    runOnUiThread(new Runnable() {
                        @Override
                        public void run() {
                            mRxOperatorsText.append("\nsubscribe: 读取缓存数据:\n");
                        }
                    });

                    e.onNext(data);
                }else {
                    isFromNet = true;
                    runOnUiThread(new Runnable() {
                        @Override
                        public void run() {
                            mRxOperatorsText.append("\nsubscribe: 读取网络数据:\n");
                        }
                    });
                    e.onComplete();
                }
            }
        });

        Observable<FoodList> network = Rx2AndroidNetworking.get("http://www.tngou.net/api/food/list")
                .addQueryParameter("rows",10+"")
                .build()
                .getObjectObservable(FoodList.class);

        // 两个 Observable 的泛型应当保持一致
        Observable.concat(cache,network)
                .subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Consumer<FoodList>() {
                    @Override
                    public void accept(@NonNull FoodList tngouBeen) throws Exception {
                        if (isFromNet){
                            mRxOperatorsText.append("accept : 网络获取数据设置缓存: \n");
                            CacheManager.getInstance().setFoodListData(tngouBeen);
                        }
                        mRxOperatorsText.append("accept: 读取数据成功:" + tngouBeen.toString()+"\n");
                    }
                }, new Consumer<Throwable>() {
                    @Override
                    public void accept(@NonNull Throwable throwable) throws Exception {
                        mRxOperatorsText.append("accept: 读取数据失败："+throwable.getMessage()+"\n");
                    }
});
```