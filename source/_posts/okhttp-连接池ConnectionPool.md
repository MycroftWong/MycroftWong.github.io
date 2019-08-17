---
title: okhttp 连接池ConnectionPool
date: 2019-08-17 20:10:10
- 开源库
    - okhttp
tags:
- okhttp
- 开源
- http
- https
- 面试
- 责任链模式
- Connection
- ConnectionPool
---

# okhttp 连接池ConnectionPool

## 前言

前面一篇，主要分析了`OkHttp`的整体设计，但是需要重提一句`OkHttp`是一个网络库。所以这篇和下一篇，来说一说`OkHttp`如何建立连接和如何交换数据的。

## ConnectInterceptor

从前面我们知道了，`ConnectInterceptor`承载着`OkHttp`与服务器建立网络连接的任务。下面看看`ConnectInterceptor`的源码：

```java
public final class ConnectInterceptor implements Interceptor {
    public final OkHttpClient client;

    public ConnectInterceptor(OkHttpClient client) {
        this.client = client;
    }

    @Override
    public Response intercept(Chain chain) throws IOException {
        RealInterceptorChain realChain = (RealInterceptorChain) chain;
        Request request = realChain.request();
        // 网络层与应用层的桥梁
        Transmitter transmitter = realChain.transmitter();

        // 我们需要保证网络能够满足网络请求。除GET方法外，我们都需要进行严格的检查
        boolean doExtensiveHealthChecks = !request.method().equals("GET");
        // 连接管理与事件的交换桥梁
        Exchange exchange = transmitter.newExchange(chain, doExtensiveHealthChecks);

        // 建立好连接之后，交给CallServerInterceptor读取数据
        return realChain.proceed(request, transmitter, exchange);
    }
}
```

从这段代码中，实际上并不能看出来什么，但是我们可以知道，这个跟`Transmitter`和`Exchange`关系莫大。

## Transmitter

> Bridge between OkHttp's application and network layers. This class exposes high-level application layer primitives: connections, requests, responses, and streams.

翻译：`OkHttp`应用层与网络层的桥梁。这个类暴露了上层应用层的基本属性：连接、请求、响应和流。

> This class supports {@linkplain #cancel asynchronous canceling}. This is intended to have the smallest blast radius possible. If an HTTP/2 stream is active, canceling will cancel that stream but not the other streams sharing its connection. But if the TLS handshake is still in progress then canceling may break the entire connection.

翻译：这个类支持异步取消。为了让取消动作影响最小，如果是`HTTP/2`的流是活跃的，那么取消动作只取消它自己的流，不会取消共享的连接。但是如果是正在进行`TLS`握手，那么就会取消整个连接。

从上面的说明，可以简单的了解，但是仍然不清楚它的作用，它既然是应用层和网络层的桥梁，那么它提供给网络层的是什么，又返回给应用层了什么呢。

上面已经说明了，它提供给网络层的是应用层的`Connection`，`Request`，`Response`和流。而在`ConnectInterceptor`方法中，我们可以看出，它给应用层返回的是一个`Exchange`。

我们可以发现，`Transmitter`的构造只有一个地方：

```java
static RealCall newRealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
    RealCall call = new RealCall(client, originalRequest, forWebSocket);
    call.transmitter = new Transmitter(client, call);
    return call;
}
```

在获得一个`RealCall`的时候，就生成了一个`Transmitter`。

## Exchange

> Transmits a single HTTP request and a response pair. This layers connection management and events on {@link ExchangeCodec}, which handles the actual I/O.

翻译：传输单个`HTTP`请求和响应。将连接管理部分和在`ExchangeCodec`的事件分层，`ExchangeCodec`处理`IO`。

大致意思应该清楚，`ExchangeCodec`负责`IO`操作，而`Exchange`则是处理`HTTP`请求和响应。

那么谁又负责连接管理部分呢？

### Exchange的构造

反向思考，`Exchange`是在哪里构造的呢？查看代码，只有一个地方，是在`Transmitter.newExchange(Interceptor.Chain, boolean)`

```java
/**
 * Returns a new exchange to carry a new request and response.
 * 返回一个Exchange负责请求、响应
 */
Exchange newExchange(Interceptor.Chain chain, boolean doExtensiveHealthChecks) {
    synchronized (connectionPool) {
        if (noMoreExchanges) {
            throw new IllegalStateException("released");
        }
        if (exchange != null) {
            throw new IllegalStateException("cannot make a new request because the previous response "
                    + "is still open: please call response.close()");
        }
    }

    ExchangeCodec codec = exchangeFinder.find(client, chain, doExtensiveHealthChecks);
    Exchange result = new Exchange(this, call, eventListener, exchangeFinder, codec);

    synchronized (connectionPool) {
        this.exchange = result;
        this.exchangeRequestDone = false;
        this.exchangeResponseDone = false;
        return result;
    }
}
```

除了前后的一些检查、复制代码，我们可以看到，这是通过一个`ExchangeFinder`，构造了一个`ExchangeCodec`负责`IO`操作，并使用这个`ExchangeCodec`构造了`Exchange`。

那么`Transmitter.newExchange(Interceptor.Chain, boolean)`什么时候被调用的呢？是在`ConnectInterceptor.intercept(Chain)`中调用的，并且交由`CallServerInterceptor`使用。而在`CallServerInterceptor`进行的是在建立好连接的基础上进行请求。那么我们就应该知道，`Exchange`内部肯定是已经建立好了连接。

特别注意的是，在构造`Exchange`时，有一个`ExchangeFinder`，它是做什么的呢？

## ExchangeFinder

> Attempts to find the connections for a sequence of exchanges. This uses the following strategies:
> If the current call already has a connection that can satisfy the request it is used. Using the same connection for an initial exchange and its follow-ups may improve locality.
> If there is a connection in the pool that can satisfy the request it is used. Note that it is possible for shared exchanges to make requests to different host names! See {@link RealConnection#isEligible} for details.
> If there's no existing connection, make a list of routes (which may require blocking DNS lookups) and attempt a new connection them. When failures occur, retries iterate the list of available routes.

翻译：尝试为一系列的`Exchange`找到`Connection`连接。使用下列的策略：
1. 如果当前有`Connection`能够满足请求`Request`，那么使用初始时的`Exchange`构造的`Connection`。
2. 如果连接池中有`Connection`能够满足请求`Request`。注意，共享同一`Connection`的`Exchange`可能向不同的主机名发送请求。
3. 如果没有存在的`Connection`，创建一个路由列表（获取需要使用阻塞`DNS`查询），并且尝试建立一个新的连接。如果发生错误，那么将根据路由列表重试。

这下明白了，建立连接的过程是在`ExchangeFinder`里面呀。看一下`ExchangeFinder`的代码：

```java
public ExchangeCodec find(
        OkHttpClient client, Interceptor.Chain chain, boolean doExtensiveHealthChecks) {
    int connectTimeout = chain.connectTimeoutMillis();
    int readTimeout = chain.readTimeoutMillis();
    int writeTimeout = chain.writeTimeoutMillis();
    int pingIntervalMillis = client.pingIntervalMillis();
    boolean connectionRetryEnabled = client.retryOnConnectionFailure();

    try {
        RealConnection resultConnection = findHealthyConnection(connectTimeout, readTimeout,
                writeTimeout, pingIntervalMillis, connectionRetryEnabled, doExtensiveHealthChecks);
        return resultConnection.newCodec(client, chain);
    } catch (RouteException e) {
        trackFailure();
        throw e;
    } catch (IOException e) {
        trackFailure();
        throw new RouteException(e);
    }
}

/**
 * Finds a connection and returns it if it is healthy. If it is unhealthy the process is repeated
 * until a healthy connection is found.
 * 找到可用的Connection。如果不可用，将重复查找直到找到为止。
 */
private RealConnection findHealthyConnection(int connectTimeout, int readTimeout,
                                                int writeTimeout, int pingIntervalMillis, boolean connectionRetryEnabled,
                                                boolean doExtensiveHealthChecks) throws IOException {
    while (true) {
        RealConnection candidate = findConnection(connectTimeout, readTimeout, writeTimeout,
                pingIntervalMillis, connectionRetryEnabled);

        // If this is a brand new connection, we can skip the extensive health checks.
        synchronized (connectionPool) {
            if (candidate.successCount == 0) {
                return candidate;
            }
        }

        // Do a (potentially slow) check to confirm that the pooled connection is still good. If it
        // isn't, take it out of the pool and start again.
        if (!candidate.isHealthy(doExtensiveHealthChecks)) {
            candidate.noNewExchanges();
            continue;
        }

        return candidate;
    }
}

/**
 * Returns a connection to host a new stream. This prefers the existing connection if it exists,
 * then the pool, finally building a new connection.
 * 返回一个Connection用于处理流。最好的先从已有的查找，再从连接池中查找，最后没找到则新建一个Connection
 */
private RealConnection findConnection(int connectTimeout, int readTimeout, int writeTimeout,
                                        int pingIntervalMillis, boolean connectionRetryEnabled) throws IOException {
    RealConnection result = null;
    // ...
    result.connect(connectTimeout, readTimeout, writeTimeout, pingIntervalMillis,
            connectionRetryEnabled, call, eventListener);
    // ...
    return result;
}
```

`ExchangeFinder.find(Interceptor.Chain, boolean)`得到`IO`操作的`ExchangeCodec`，在其中调用了`RealConnection findHealthyConnection(int, int, int, int, boolean, boolean)`，又通过`RealConnection finderConnection(int, int, int, int, boolean)`得到了一个`RealConnection`，然后调用`RealConnection.connect(int, int, int, int, boolean, Call, EventListener)`建立了真正的连接，然后通过这个`RealConnection`返回了最后的`ExchangeCodec`。

## RealConnection

现在知道了`RealConnection`才是真正的服务器连接。通过它进行连接服务器，建立好连接之后，通过`ExchangeCodec newCodec(OkHttpClient, Interceptor.Chain)`返回`ExchangeCodec`用于处理`IO`，并将其封装在`Exchange`中，供应用层调用（大部分是在`CallServerInterceptor`使用）。

而`RealConnection`实际上实现了`Connection`接口

## Connection

> The sockets and streams of an HTTP, HTTPS, or HTTPS+HTTP/2 connection. May be used for multiple HTTP request/response exchanges. Connections may be direct to the origin server or via a proxy.

