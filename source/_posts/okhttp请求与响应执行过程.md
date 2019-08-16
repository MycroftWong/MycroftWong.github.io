---
title: okhttp请求与响应执行过程
date: 2019-08-16 09:37:19
categories: 
- 开源库
    - okhttp
tags:
- okhttp
- 源码
- http
- https
- 面试
---

# okhttp请求与响应执行过程

## 前言

不考虑真正的网络请求部分，看一下从一个`Request`到`Response`的过程。

## 请求

再来看一下上一篇最简单的执行代码

```java
OkHttpClient httpClient = new OkHttpClient.Builder()
        .build();

Request request = new Request.Builder()
        .url("https://wanandroid.com/wxarticle/chapters/json")
        .get()
        .build();

Call call = httpClient.newCall(request);
Response response = call.execute();

System.out.println(response.body().string());
```

### OkHttpClient

> Factory for {@linkplain Call calls}, which can be used to send HTTP requests and read their responses.

翻译：`Call`的工厂类，用于发送`HTTP`请求，读取响应。

`OkHttpClient`实现类`Call.Factory`接口，下面是源码，实际上只构造一个`Call`实例。

```java
// Call.Factory
interface Factory {
    Call newCall(Request request);
}

// OkHttpClient 实现
@Override
public Call newCall(Request request) {
    return RealCall.newRealCall(this, request, false /* for web socket */);
}
```

`OkHttpClient`多用于配置，最重要的方法就是`Call newCall(Request)`，在这个方法中，实际上构造了一个`RealCall`，最后一个参数用于长连接，目前没有用到。并且构造了一个`Transmitter`。

```java
private RealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
    this.client = client;
    this.originalRequest = originalRequest;
    this.forWebSocket = forWebSocket;
}

static RealCall newRealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
    // Safely publish the Call instance to the EventListener.
    RealCall call = new RealCall(client, originalRequest, forWebSocket);
    call.transmitter = new Transmitter(client, call);
    return call;
}
```

### Transmitter

> Bridge between OkHttp's application and network layers. This class exposes high-level application layer primitives: connections, requests, responses, and streams.

翻译：是`OkHttp`应用层与网络层的桥梁。这个类向上面应用层暴露基本信息：连接、请求、响应、流。

`Transmitter`，可以直接翻译成**发射器**，将网络层的信息如资源、执行过程告诉应用层（即`OkHttpClient`配置的一些参数如`EventListener`。


### RealCall

我们得到了`Call`实例之后可以使用`execute()`同步执行，`enqueue(Callback)`异步执行。下面看看这两个方法的执行。


#### 同步执行

```java
@Override
public Response execute() throws IOException {
    // 判断是否已经执行，并且只能执行一次
    synchronized (this) {
        if (executed) throw new IllegalStateException("Already Executed");
        executed = true;
    }
    // 超时计时开始
    transmitter.timeoutEnter();
    // 发送开始执行的通知
    transmitter.callStart();
    try {
        // 加入执行队列中
        client.dispatcher().executed(this);
        // 开始执行链
        return getResponseWithInterceptorChain();
    } finally {
        // 完成执行，从队列中移除
        client.dispatcher().finished(this);
    }
}
```

前面是一些额外的处理，真正执行的代码在`try-finally`中。

##### RealCall.getResponseWithInterceptorChain()

为了不影响结构，先说一下`getResponseWithInterceptorChain()`方法，这也是最重要的方法。

```java
Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    // 构造Interceptor栈
    List<Interceptor> interceptors = new ArrayList<>();
    // 添加配置的Interceptor
    interceptors.addAll(client.interceptors());
    // 下面添加逻辑处理的的Interceptor
    interceptors.add(new RetryAndFollowUpInterceptor(client));
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    // 添加配置的networkInterceptor，不是长连接才添加
    if (!forWebSocket) {
        interceptors.addAll(client.networkInterceptors());
    }
    // 添加真正执行的网络连接的Interceptor
    interceptors.add(new CallServerInterceptor(forWebSocket));

    // 构造Interceptor链
    Interceptor.Chain chain = new RealInterceptorChain(interceptors, transmitter, null, 0,
            originalRequest, this, client.connectTimeoutMillis(),
            client.readTimeoutMillis(), client.writeTimeoutMillis());

    // 标记请求是否抛出异常
    boolean calledNoMoreExchanges = false;
    try {
        // 执行Interceptor责任链
        Response response = chain.proceed(originalRequest);
        // 若在执行过程前已经关闭了，则关闭Response并抛出异常
        if (transmitter.isCanceled()) {
            closeQuietly(response);
            throw new IOException("Canceled");
        }
        // 返回结果
        return response;
    } catch (IOException e) {
        // 标记发生了异常，抛出异常，并申请关闭连接
        calledNoMoreExchanges = true;
        throw transmitter.noMoreExchanges(e);
    } finally {
        // 若未发生异常，申请关闭连接
        if (!calledNoMoreExchanges) {
            transmitter.noMoreExchanges(null);
        }
    }
}
```

这一段代码，将一个`Request`处理得到`Response`。

##### 1. 构造Interceptor责任链

在知道`Interceptor`责任链的功能前，先不详说，只需要知道，这个责任链处理了`Request`得到`Response`。

##### 2. 执行责任链

这个是最重要的代码，这部分后面隐藏着整个`OkHttp`的核心。

##### 3. 结果处理

若执行过程中已经取消，那么清理资源。


#### 异步执行

```java
@Override
public void enqueue(Callback responseCallback) {
    // 判断是否已经执行，并且只能执行一次
    synchronized (this) {
        if (executed) throw new IllegalStateException("Already Executed");
        executed = true;
    }
    // 发送开始执行的通知
    transmitter.callStart();
    // 加入执行队列中
    client.dispatcher().enqueue(new AsyncCall(responseCallback));
}
```

没有实际的代码，只是在将一个`AsnycCall`添加到`Dispatcher`中的队列。


##### AsyncCall

`AsyncCall`是在`RealCall`的一个内部类，它继承自`NamedRunnable`，而`NamedRunnable`实现了`Runnable`接口，下面是源码，只是将当前线程设置了一个名字，并且将逻辑抽象成了`execute()`方法。

```java
public abstract class NamedRunnable implements Runnable {
    protected final String name;

    public NamedRunnable(String format, Object... args) {
        this.name = Util.format(format, args);
    }

    @Override
    public final void run() {
        // 设置当前线程的名字
        String oldName = Thread.currentThread().getName();
        Thread.currentThread().setName(name);
        try {
            execute();
        } finally {
            Thread.currentThread().setName(oldName);
        }
    }

    protected abstract void execute();
}
```

下面是`AsyncCall.execute()`实现

```java
@Override
protected void execute() {
    boolean signalledCallback = false;
    transmitter.timeoutEnter();
    try {
        Response response = getResponseWithInterceptorChain();
        signalledCallback = true;
        responseCallback.onResponse(RealCall.this, response);
    } catch (IOException e) {
        if (signalledCallback) {
            // Do not signal the callback twice!
            Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
        } else {
            responseCallback.onFailure(RealCall.this, e);
        }
    } finally {
        client.dispatcher().finished(this);
    }
}
```

这里也是调用了`RealCall.getResponseWithInterceptorChain()`方法，并且处理了回调。所以只需要将`AsyncCall`作为`Runnable`使用`ExecutorService`执行，就自然而然执行了。

### Dispatcher

> Policy on when async requests are executed. 
> Each dispatcher uses an {@link ExecutorService} to run calls internally. If you supply your own executor, it should be able to run {@linkplain #getMaxRequests the configured maximum} number of calls concurrently.

翻译：异步执行时的策略。每个`Dispatcher`在内部使用一个`ExecutorService`去执行`Call`。如果你提供自己的执行器，应该能够保证`getMaxRequests`得到的配置最大数量的请求。

查看源码，实际上内部有三个`ArrayDeque`双向队列用于管理`Call`，其中一个用于管理同步执行的`RealCall`，另外两个用于管理异步执行的`AsyncCall`。

先看一下在`RealCall.execute()`方法`Dispatcher`做了什么：

```java
/**
 * Used by {@code Call#execute} to signal it is in-flight.
 * 用于标记Call正在执行
 */
synchronized void executed(RealCall call) {
    runningSyncCalls.add(call);
}
/**
 * Used by {@code Call#execute} to signal completion.
 * 用于标记同步执行完毕
 */
void finished(RealCall call) {
    finished(runningSyncCalls, call);
}
private <T> void finished(Deque<T> calls, T call) {
    Runnable idleCallback;
    synchronized (this) {
        // 判断错误
        if (!calls.remove(call)) throw new AssertionError("Call wasn't in-flight!");
        // Dispatcher空闲时的回调接口
        idleCallback = this.idleCallback;
    }

    // 判断Dispatcher是否有在执行的任务，并推动队列执行
    boolean isRunning = promoteAndExecute();

    // 当Dispatcher没有在执行代码时，执行空闲借口回调
    if (!isRunning && idleCallback != null) {
        idleCallback.run();
    }
}
```

实际上只是先将`RealCall`添加到了这个双向队列中，标记它正在被执行，执行结束后，做移除它。

当然它还做了一些额外的工作，如回调空闲接口、通知其他在排队执行的任务开始执行。

下面看看`enqueue()`做了什么

```java
void enqueue(AsyncCall call) {
    synchronized (this) {
        // 添加到准备执行的双向队列中
        readyAsyncCalls.add(call);

        // Mutate the AsyncCall so that it shares the AtomicInteger of an existing running call to
        // the same host.
        // 转换AsyncCall，便于在多个运行同一host的Call中分享AtomicInteger，这个AtomicInteger标记在同一host上同时运行的数量
        if (!call.get().forWebSocket) {
            AsyncCall existingCall = findExistingCallWithHost(call.host());
            if (existingCall != null) call.reuseCallsPerHostFrom(existingCall);
        }
    }
    // 推动队列执行
    promoteAndExecute();
}
```

关于`promoteAndExecute()`方法的执行，可以在详解`Dispatcher`时深入，先看下这个方法的注释：

> Promotes eligible calls from {@link #readyAsyncCalls} to {@link #runningAsyncCalls} and runs them on the executor service. Must not be called with synchronization because executing calls can call into user code.

翻译：将符合条件的`Call`从准备队列`readyAsyncCalls`移动到执行中队列`runningAsyncCalls`，然后使用`ExecutorService`执行。禁止同步执行，因为正在执行的`Call`能够会调用到使用者的代码。

意思是如果满足条件（如未达到最大执行数、同一`host`上的请求没有达到最大数）时，启动那些等待执行的`Call`。这个方法间接使用`ExecutorService`启动了`AsyncCall`.

## 总结

无论是同步执行，还是异步执行，核心代码都是`RealCall.getResponseWithInterceptorChain()`，除开一些额外的配置，最重要的是`Interceptor`责任链，这个责任链将一个`Request`转换成了一个`Response`。所以要明白`OkHttp`，就是理解`Interceptor`责任链。
