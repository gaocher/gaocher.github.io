---
title: Mono/FluxCreate —— 传统代码转为Reactive的桥梁
date: 2020-01-05 21:32:13
tags:
category: 
    - Project Reactor
---
`Project Reactor`提供了很多创建Mono/Flux的静态方法，而最常用的就是Mono#create方法，通过该方法能把以前命令式的程序转化为Reactive的编程方式。
众所周知，Reactive Programming是一种Pull-Push模型，其中Pull用于实现back-pressure，push则是常见的推模型，也是reactive programming的重点（这里不再深入讲解pull/push模型两者的区别）。下面以一个常见的Pull模型迭代器Iterator来说明如何将传统代码转为Reactive的代码。

# Iterator -> Flux
```java
//创建一个迭代器
Iterator it = Arrays.asList<>(1,2,3).iterator();

//使用迭代器
while(it.hasNext()) {
    //模拟业务逻辑 —— 这里直接打印value
    System.out.println(it.next());
}
```
上面是一个常见的迭代器使用方式，下面看看是如何将迭代器转换成Flux的:
```java
//创建迭代器
Iterator it = Arrays.asList<>(1,2,3).iterator();

Flux<Integer> iteratorFlux = Flux.create(sink -> {
    while (it.hasNext()) {
        sink.next(it.next()); //利用FluxSink实现data的Push
    }
    sink.complete();  //发送结束的Signal
});

//进行订阅，进行业务逻辑操作
iteratorFlux.subscribe(System.out::println);
```

# MonoCreate常见的两者使用方式
传统命令式编程除了Iterator的Pull模式外，通常还有Observable以及Callback这两种Push模式，下面分别举例讲讲这两种模式。

## Observable -> MonoCreate
Observable原始代码举例：
```java
Observable observable = new Observable() {
    //需要重写Observable，默认是setChanged与notifyObservers分离，实现先提交再通知的效果
    //这里为了简单起见，将通知与提交放在了一起
    @Override
    public void notifyObservers(Object arg) {
    setChanged();
    super.notifyObservers(arg);
    }
};
Observer first = (ob,value) -> {
    System.out.println("value is " + value);
};
observable.addObserver(first);
observable.notifyObservers("42");

//    after some time, cancel observer to dispose resource
observable.deleteObserver(first);
```
MonoCreate的转化示例：
```java
Mono<Object> observableMono = Mono.create(sink -> {
    Observer first = (ob, value) -> {
        sink.success(value);
    };
    observable.addObserver(first);
    observable.notifyObservers("42");
    sink.onDispose(() -> observable.deleteObserver(first));
});
observableMono.subscribe(v -> System.out.println("value is " + v));
```

## Callback -> MonoCreate
```java
//callback example
FutureCallback<HttpResponse> callback = new FutureCallback<HttpResponse>() {
    @Override
    public void completed(HttpResponse result) {
        System.out.println("Response: " + result.getStatusLine());
    }

    @Override
    public void failed(Exception ex) {
        System.out.println("Fail in " + ex);
    }

    @Override
    public void cancelled() {
        System.out.println("Cancelled");
    }
};

CloseableHttpAsyncClient httpclient = HttpAsyncClients.createDefault();
httpclient.start();

HttpGet request = new HttpGet("http://www.example.com/");
httpclient.execute(request, callback);
```
MonoCreate的转化示例：
```java
CloseableHttpAsyncClient httpclient = HttpAsyncClients.createDefault();
httpclient.start();

Mono<HttpResponse> responseMono = Mono.create(monoSink -> {
    //创建response callback的处理类，并传入monoSink供使用
    CallbackHandler callbackHandler = new CallbackHandler(monoSink);
    HttpGet getRequest = new HttpGet("http://www.example.com/");
    httpclient.execute(getRequest, callbackHandler.getResponseCallback());
});
responseMono.subscribe(response -> System.out.println("Response: " + response.getStatusLine()));

@Data
static class CallbackHandler {
    private MonoSink monoSink;
    private FutureCallback<HttpResponse> responseCallback;

    public CallbackHandler(MonoSink monoSink) {
        this.monoSink = monoSink;
        responseCallback = new FutureCallback<HttpResponse>() {
            @Override
            public void completed(HttpResponse result) {
                monoSink.success(result);
            }

            @Override
            public void failed(Exception ex) {
                monoSink.error(ex);
            }

            @Override
            public void cancelled() {
                monoSink.onDispose(() -> System.out.println("cancelled"));
            }
        };
    }
}

```
# MonoSink
从前面已经可以看到，将传统代码转变为Reactive方式的关键是在于sink，在创建Mono/FluxCreate的时候，Mono/Flux都会提供相应的sink给使用方来使用。MonoSink相比FluxSink要简单的多，为了简单起见，我们先从MonoSink来了解sink的运行原理（FluxSink会专门另开一篇来说明）。下面就来探探Mono下的MonoSink究竟到底是什么。

再深入MonoSink之前，我们先来看看MonoCreate是怎么使用MonoSink的，对于Reactor来说，所有的入口都是`subscribe`方法，所以先来看看MonoCreate的subscribe方法：
```java
public void subscribe(CoreSubscriber<? super T> actual) {
    //1. 创建MonoSink实例，供MonoCreate来使用
    //如变量名字emitter一样，MonoSink的作用其实就是信号的发射器（signal emitter）
    DefaultMonoSink<T> emitter = new DefaultMonoSink<>(actual);

    //2. emitter除了是sink外，也实现了subscription，供Subscriber使用
    //这一步，调用Subscriber的onSubscribe方法，其内部则会调用subscription的request方法 （后续会重点说DefaultMonoSink的request方法）
    actual.onSubscribe(emitter);

    try {
        //3. callback就是在Mono.create时候传入的Mono构造器
        //此步骤即调用Mono构造器函数，并将sink传入
        callback.accept(emitter);
    }
    catch (Throwable ex) {
        emitter.error(Operators.onOperatorError(ex, actual.currentContext()));
    }
}
```
从上面的源代码可以看出，整个MonoCreate订阅过程很简单，主要是分为三个步骤：
1. 创建DefaultMonoSink *(通过这一步可以看出，一个Subscriber是独占一个MonoSink的)*
2. 实现Subscriber的onSubscribe的方法
3. 调用Mono#create的构造器函数

以上三个步骤是从整体视角来看的，我们再进一步进入DefaultMonoSink，以它的内部视角，来看看到底作为signal emitter的MonoSink做了些什么。

## MonoSink 内部状态
MonoSink内部主要有4个状态:
```java
volatile int state; //初始默认状态0，即未调用Request且未赋值

static final int NO_REQUEST_HAS_VALUE  = 1; //未调用Request但已经赋值
static final int HAS_REQUEST_NO_VALUE  = 2; //调用了Request但还未赋值
static final int HAS_REQUEST_HAS_VALUE = 3; //调用了Request且已经赋值了
```
这三个状态主要取决于request和success(或者error)的调用时机，调用了request方法则会是`HAS_REQUEST`，调用了success(或者error)方法则会是`HAS_VALUE`，其中request方法调用是由Subscriber#onSubscribe调用的，success或者error则是由具体使用者来调用的，如Callback。由于success或者error调用时机往往不可能确定（通常是异步的），所以才产生了上述4种状态。

以同步的角度思考，通常是先调用request然后再调用success或者error方法，其中success会对应调用Subscriber的onNext与onComplete方法，error方法则会调用对应的Subscriber#onError方法。但事情往往没这么简单，就如前面提到的，request方法与success/error方法是乱序的，很有可能在request的时候，success/error方法已经调用结束了。为了解决这个问题，每个方法都引入了for-loop加CAS的多线程操作，变得相对复杂了，但只要知道其内部原理，再复杂的代码看起来就都有线索了，下面以request方法为例，来讲讲是MonoSink是如何解决多线程问题的。

## MonoSink request方法解释
```java
public void request(long n) {
    if (Operators.validate(n)) {
        LongConsumer consumer = requestConsumer;
        //1. 如果传入了requestConsumer，则调用
        //requestConsumer是通过onRequest方法传入的
        if (consumer != null) {
            consumer.accept(n);
        }
        //2. 进入for loop来实现自旋
        for (; ; ) {
            int s = state;
            //2.1 HAS_Request: 已经调用过了，直接退出
            if (s == HAS_REQUEST_NO_VALUE || s == HAS_REQUEST_HAS_VALUE) {
                return;
            }
            if (s == NO_REQUEST_HAS_VALUE) {
                // 2.2 double check 是否已经有值
                // 如果是，执行onNext/onComplete方法，并设置完成状态: HAS_REQUEST_HAS_VALUE
                // 如果不是，double check失败，直接退出，说明有别的线程已经执行了该方法了
                if (STATE.compareAndSet(this, s, HAS_REQUEST_HAS_VALUE)) {
                    try {
                        actual.onNext(value);
                        actual.onComplete();
                    }
                    finally {
                        //释放资源 - 具体调用的disposable对象由onDisposable方法赋值
                        disposeResource(false);
                    }
                }
                return;
            }
            //2.3 正常流程，值没有被赋值，设置为HAS_REQUEST_NO_VALUE
            if (STATE.compareAndSet(this, s, HAS_REQUEST_NO_VALUE)) {
                return;
            }
        }
    }
}
```

## MonoSink回调方法
MonoSink除了request、success、error方法外，还提供了几个回调函数，以供使用者使用，主要有：
```java
//request的时候会被调用，获取request的数量N
MonoSink<T> onRequest(LongConsumer consumer);

//Subscriber调用subscription.cancel是会调用该Disposable方法
MonoSink<T> onCancel(Disposable d);

//与onCancel类似，区别是，除了onCancel方法，在onComplete以及onError也会调用该Disposable方法
MonoSink<T> onDispose(Disposable d);
```
这里简单讲一下Reactor的代码命名规范，对于回调函数都是以onXXX方式命名，注意调用该onXXX方式的时候，**并不是直接调用，而只是传入该回调方法，待对应的事件信号发生时，才会真的被调用**。这也是声明式编程的一个特色，先声明再执行。

# 总结
本文首先描述了传统命令式的代码如何转换为Reactive方式的代码，然后就其内部MonoSink就行了深入的了解，重点讲解了其实现形式，通过对MonoSink的剖析，能够更具体的对Mono整体的使用方式的了解。