---
title: RxJava的使用
date: 2017-09-16
categories: "Android"
tags: "RxJava"
---
![](https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1514449648324&di=894c261ff0161c55c1d7d8219a9f5d44&imgtype=jpg&src=http%3A%2F%2Fwww.th7.cn%2Fd%2Ffile%2Fp%2F2016%2F01%2F25%2Ffc7a148d63317797a1e00ace4349728e.jpg)
# 概述
RxJava是一个实现异步操作的库，当初我们使是用AsyncTesk来进行异步交互的，现在RxJava是完完全全可以替代AsyncTesk的一种框架，当项目或逻辑越来越复杂时，它依旧能保持代码的可读性性，整洁性等。关于RxJava一个很重要的点就是响应式编程，响应式编程就是编程处理异步数据流。就是我们接收连续流动的数据–数据流–提供处理数据流的方法并将该方法应用到数据流。想象一下高速公路上汽车过收费站，公路就是流，汽车是事件（不断的行走），而收费站时接受事件的（不断的观察车辆）。此版本主要针对于RxJava2.X，假如对1.0版本不太熟悉也没关系，不影响2.0的使用。

# 成员介绍：
Observable：发射源，英文释义“可观察的”，在观察者模式中称为“被观察者”或“可观察对象”；

Observer：接收源，英文释义“观察者”，没错！就是观察者模式中的“观察者”，可接收Observable发射的数据；

Consumer：也是接收源，它跟Observer的区别是，Consumer只关心数据的结果，不关心它是否出错Error，和完成情况Complete；

Subscriber：“订阅者”，也是接收源，那它跟Observer有什么区别呢？Subscriber实现了Observer接口，比Observer多了一个最重要的方法unsubscribe( )，用来取消订阅，当你不再想接收数据了，可以调用unsubscribe( )方法停止接收，Observer 在 subscribe() 过程中,最终也会被转换成 Subscriber 对象，一般情况下，建议使用Subscriber作为接收源；

ObservableEmitter： Emitter是发射器的意思，那就很好猜了，这个就是用来发出事件的，它可以发出三种类型的事件，通过调用emitter的onNext(T value)、onComplete()和onError(Throwable error)就可以分别发出next事件、complete事件和error事件。

# 基本用法：
```java
//创建一个被观察者 Observable：
Observable<Integer> observable = Observable.create(new ObservableOnSubscribe<Integer>() {
    @Override
    public void subscribe(ObservableEmitter<Integer> emitter) throws Exception {
        emitter.onNext(1);
        emitter.onNext(2);
        emitter.onNext(3);
        emitter.onComplete();
    }
});
//创建一个观察者 Observer
Observer<Integer> observer = new Observer<Integer>() {
    @Override
    public void onSubscribe(Disposable d) {
        Log.d(TAG, "subscribe");
    }
    @Override
    public void onNext(Integer value) {
        Log.d(TAG, "" + value);
    }
    @Override
    public void onError(Throwable e) {
        Log.d(TAG, "error");
    }
    @Override
    public void onComplete() {
        Log.d(TAG, "complete");
    }
};
//调用subscribe()观察者才与被观察者建立连接
observable.subscribe(observer);
//注意：建立连接后才会开始发送数据
```
运行结果：
```java
D/TAG: subscribe
D/TAG: 1
D/TAG: 2
D/TAG: 3
D/TAG: complete
```
这里也可以连起来写
```java
Observable.create(new ObservableOnSubscribe<Integer>() {
    @Override
    public void subscribe(ObservableEmitter<Integer> emitter) throws Exception {
        emitter.onNext(1);
        emitter.onNext(2);
        emitter.onNext(3);
        emitter.onComplete();
    }
}).subscribe(new Observer<Integer>() {
    @Override
    public void onSubscribe(Disposable d) {
        Log.d(TAG, "subscribe");
    }
    @Override
    public void onNext(Integer value) {
        Log.d(TAG, "" + value);
    }
    @Override
    public void onError(Throwable e) {
        Log.d(TAG, "error");
    }
    @Override
    public void onComplete() {
        Log.d(TAG, "complete");
    }
});
```
以上两种写法是一样的。

complete的作用：
```java
Observable.create(new ObservableOnSubscribe<Integer>() {
     @Override
     public void subscribe(ObservableEmitter<Integer> emitter) throws Exception {
         Log.d(TAG, "emit 1");
         emitter.onNext(1);
         Log.d(TAG, "emit 2");
         emitter.onNext(2);
         Log.d(TAG, "emit 3");
         emitter.onNext(3);
         Log.d(TAG, "emit complete");
         emitter.onComplete();
         Log.d(TAG, "emit 4");
         emitter.onNext(4);
     }
 }).subscribe(new Observer<Integer>() {
     private Disposable mDisposable;
     private int i;
     @Override
     public void onSubscribe(Disposable d) {
         Log.d(TAG, "subscribe");
         mDisposable = d;
     }
     @Override
     public void onNext(Integer value) {
         Log.d(TAG, "onNext: " + value);
         i++;
         if (i == 2) {//当等于2时调用dispose()
             Log.d(TAG, "dispose");
             mDisposable.dispose();//调用dispose()被观察者仍然会继续发送剩余的事件.
             Log.d(TAG, "isDisposed : " + mDisposable.isDisposed());
         }
     }
     @Override
     public void onError(Throwable e) {
         Log.d(TAG, "error");
     }
     @Override
     public void onComplete() {
         Log.d(TAG, "complete");
     }
 });
```
调用dispose()被观察者仍然会继续发送剩余的事件.
我们让被观察者依次发送1,2,3,complete,4，被观察者因为在2的时候dispose，所以接受不到3,4。但不影响被观察者继续发送数据。
运行结果：
```java
D/TAG: subscribe
D/TAG: emit 1
D/TAG: onNext: 1
D/TAG: emit 2
D/TAG: onNext: 2
D/TAG: dispose
D/TAG: isDisposed : true
D/TAG: emit 3
D/TAG: emit complete
D/TAG: emit 4
```
以上我们简单的了解到了Rxjava的基本用法，我们知道RxJava的精髓在于它的异步操作。上面的事例全部都是在主线程中运行的。接下来看看如何实现线程的调度，实现异步操作。
```java
@Override                                                                                       
protected void onCreate(Bundle savedInstanceState) {                                            
    super.onCreate(savedInstanceState);                                                         
    setContentView(R.layout.activity_main);                                                     
                                                                                                
    Observable<Integer> observable = Observable.create(new ObservableOnSubscribe<Integer>() {   
        @Override                                                                               
        public void subscribe(ObservableEmitter<Integer> emitter) throws Exception {            
            Log.d(TAG, "Observable thread is : " + Thread.currentThread().getName());           
            Log.d(TAG, "emit 1");                                                               
            emitter.onNext(1);                                                                  
        }                                                                                       
    });                                                                                         
                                                                                                
    Consumer<Integer> consumer = new Consumer<Integer>() {                                      
        @Override                                                                               
        public void accept(Integer integer) throws Exception {                                  
            Log.d(TAG, "Observer thread is :" + Thread.currentThread().getName());              
            Log.d(TAG, "onNext: " + integer);                                                   
        }                                                                                       
    };                                                                                          
                                                                                                
    observable.subscribeOn(Schedulers.newThread())//子线程中进行耗时请求                                              
            .observeOn(AndroidSchedulers.mainThread())//结果在主线程中进行处理                                          
            .subscribe(consumer);                                                               
}
```
可以看出线程调度十分的方便，当逻辑越来越复杂的时候，更能体现其简洁的优点。
# 操作符

初次之外，还有很多的操作符，我们就最常用的操作符来分析一下：
### Map：是RxJava中最简单的一个变换操作符了，可以类似理解为当从接口获取Json字符串然后通过Gson转成对象bean。Map就是转换的作用
```java
Observable.create(new ObservableOnSubscribe<Integer>() {
    @Override
    public void subscribe(ObservableEmitter<Integer> emitter) throws Exception {//发射Interger值
        emitter.onNext(1);
        emitter.onNext(2);
        emitter.onNext(3);
    }
}).map(new Function<Integer, String>() {//这里理由Map将Interger转成String
    @Override
    public String apply(Integer integer) throws Exception {
        return "This is result " + integer;
    }
}).subscribe(new Consumer<String>() {
    @Override
    public void accept(String s) throws Exception {
        Log.d(TAG, s);
    }
});
```
### FlatMap：将一个发送事件的上游Observable变换为多个发送事件的Observables
```java
Observable.create(new ObservableOnSubscribe<Integer>() {
    @Override
    public void subscribe(ObservableEmitter<Integer> emitter) throws Exception {
        emitter.onNext(1);
        emitter.onNext(2);
        emitter.onNext(3);
    }
}).flatMap(new Function<Integer, ObservableSource<String>>() {
    @Override
    public ObservableSource<String> apply(Integer integer) throws Exception {
        final List<String> list = new ArrayList<>();
        for (int i = 0; i < 3; i++) {
            list.add("I am value " + integer);
        }
        return Observable.fromIterable(list).delay(10,TimeUnit.MILLISECONDS);//变换为多个发送事件的Observables
    }
}).subscribe(new Consumer<String>() {
    @Override
    public void accept(String s) throws Exception {
        Log.d(TAG, s);
    }
});
```
输出结果：
```java
D/TAG: I am value 1
D/TAG: I am value 1
D/TAG: I am value 1
D/TAG: I am value 3
D/TAG: I am value 3
D/TAG: I am value 3
D/TAG: I am value 2
D/TAG: I am value 2
D/TAG: I am value 2
```
可以看出flatMap是无序的。
假如需要结果为有序的话，则使用concatMap。使用方法相同。这里就不加演示了

### delay
延迟
```java
Observable.just(1, 2, 3)
                .delay(3, TimeUnit.SECONDS) // 延迟3s再发送，由于使用类似，所以此处不作全部展示
                .subscribe(new Observer<Integer>() {
                    @Override
                    public void onSubscribe(Disposable d) {

                    }

                    @Override
                    public void onNext(Integer value) {
                        Log.d(TAG, "接收到了事件"+ value  );
                    }

                    @Override
                    public void onError(Throwable e) {
                        Log.d(TAG, "对Error事件作出响应");
                    }

                    @Override
                    public void onComplete() {
                        Log.d(TAG, "对Complete事件作出响应");
                    }
                });
```
### retry
```Java
Observable.create(new ObservableOnSubscribe<Integer>() {
            @Override
            public void subscribe(ObservableEmitter<Integer> e) throws Exception {
                e.onNext(1);
                e.onNext(2);
                e.onError(new Exception("发生错误了"));
                e.onNext(3);
                 }
               })
                // 拦截错误后，判断是否需要重新发送请求
                .retry(new Predicate<Throwable>() {
                    @Override
                    public boolean test(@NonNull Throwable throwable) throws Exception {
                        // 捕获异常
                        Log.e(TAG, "retry错误: "+throwable.toString());

                        //返回false = 不重新重新发送数据 & 调用观察者的onError结束
                        //返回true = 重新发送请求（若持续遇到错误，就持续重新发送）
                        return true;
                    }
                })
                .subscribe(new Observer<Integer>() {
                    @Override
                    public void onSubscribe(Disposable d) {

                    }
                    @Override
                    public void onNext(Integer value) {
                        Log.d(TAG, "接收到了事件"+ value  );
                    }

                    @Override
                    public void onError(Throwable e) {
                        Log.d(TAG, "对Error事件作出响应");
                    }

                    @Override
                    public void onComplete() {
                        Log.d(TAG, "对Complete事件作出响应");
                    }
                });
```
*** repeatWhen
重复发送，轮询
```Java
Observable.just(1,2,4).repeatWhen(new Function<Observable<Object>, ObservableSource<?>>() {
            @Override
            // 在Function函数中，必须对输入的 Observable<Object>进行处理，这里我们使用的是flatMap操作符接收上游的数据
            public ObservableSource<?> apply(@NonNull Observable<Object> objectObservable) throws Exception {
                // 将原始 Observable 停止发送事件的标识（Complete（） /  Error（））转换成1个 Object 类型数据传递给1个新被观察者（Observable）
                // 以此决定是否重新订阅 & 发送原来的 Observable
                // 此处有2种情况：
                // 1. 若新被观察者（Observable）返回1个Complete（） /  Error（）事件，则不重新订阅 & 发送原来的 Observable
                // 2. 若新被观察者（Observable）返回其余事件，则重新订阅 & 发送原来的 Observable
                return objectObservable.flatMap(new Function<Object, ObservableSource<?>>() {
                    @Override
                    public ObservableSource<?> apply(@NonNull Object throwable) throws Exception {

                        // 情况1：若新被观察者（Observable）返回1个Complete（） /  Error（）事件，则不重新订阅 & 发送原来的 Observable
                        return Observable.empty();
                        // Observable.empty() = 发送Complete事件，但不会回调观察者的onComplete（）

                        // return Observable.error(new Throwable("不再重新订阅事件"));
                        // 返回Error事件 = 回调onError（）事件，并接收传过去的错误信息。

                        // 情况2：若新被观察者（Observable）返回其余事件，则重新订阅 & 发送原来的 Observable
                        // return Observable.just(1);
                       // 仅仅是作为1个触发重新订阅被观察者的通知，发送的是什么数据并不重要，只要不是Complete（） /  Error（）事件
                    }
                });

            }
        }).subscribeOn(Schedulers.io())               // 切换到IO线程进行网络请求
            .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Observer<Integer>() {
                    @Override
                    public void onSubscribe(Disposable d) {
                        Log.d(TAG, "开始采用subscribe连接");
                    }

                    @Override
                    public void onNext(Integer value) {
                        Log.d(TAG, "接收到了事件" + value);
                    }

                    @Override
                    public void onError(Throwable e) {
                        Log.d(TAG, "对Error事件作出响应：" + e.toString());
                    }

                    @Override
                    public void onComplete() {
                        Log.d(TAG, "对Complete事件作出响应");
                    }

                });
```


# 总结
以上就是Rxjava的基本用法，都是简单的示范，Rxjava还有更复杂的一些用法，将来有机会再说。后续会尝试进行Rxjava+OkHtto+Retrofi的网络框架封装。