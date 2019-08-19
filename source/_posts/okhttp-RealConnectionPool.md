---
title: okhttp RealConnectionPool
date: 2019-08-19 10:32:18
categories:
- 开源库
    - okhttp
tags:
- okhttp
- 开源
- http
- https
- 面试
- 责任链模式
- ConnectInterceptor
- RealConnectionPool
---

# okhttp RealConnectionPool

前一篇知道了`RealConnection`是真正建立连接的地方。现在我们看看`RealConnectionPool`是如何管理`RealConnection`的呢。

## 属性

先看看`RealConnectionPool`中的属性：

```java
private static final Executor executor = new ThreadPoolExecutor(0 /* corePoolSize */,
        Integer.MAX_VALUE /* maximumPoolSize */, 60L /* keepAliveTime */, TimeUnit.SECONDS,
        new SynchronousQueue<>(), Util.threadFactory("OkHttp ConnectionPool", true));

/**
 * The maximum number of idle connections for each address.
 */
private final int maxIdleConnections;
private final long keepAliveDurationNs;
private final Runnable cleanupRunnable = () -> {
    while (true) {
        long waitNanos = cleanup(System.nanoTime());
        if (waitNanos == -1) return;
        if (waitNanos > 0) {
            long waitMillis = waitNanos / 1000000L;
            waitNanos -= (waitMillis * 1000000L);
            synchronized (RealConnectionPool.this) {
                try {
                    RealConnectionPool.this.wait(waitMillis, (int) waitNanos);
                } catch (InterruptedException ignored) {
                }
            }
        }
    }
};

private final Deque<RealConnection> connections = new ArrayDeque<>();
final RouteDatabase routeDatabase = new RouteDatabase();
boolean cleanupRunning;
```

`Executor`这个线程池的功能只有一个，运行下面的`cleanupRunnable`，在其中调用了`cleanup(long)`。`cleanupRunning`则标记是否在执行清理任务。

使用了`Deque`来存储`RealConnection`，只是用来存储，不要在意数据结构，可以使用`Collection`来代替。

`RouteDatabase`用于记录与目标地址无法建立连接的错误路由，用于优化。

`maxIdelConnection`：最大空闲连接数。

`keepAliveDurationNs`：空闲连接的最长空置时间。

## 方法

再来看看方法

### cleanupRunnable.run()

```java
while (true) {
    long waitNanos = cleanup(System.nanoTime());
    if (waitNanos == -1) return;
    if (waitNanos > 0) {
        long waitMillis = waitNanos / 1000000L;
        waitNanos -= (waitMillis * 1000000L);
        synchronized (RealConnectionPool.this) {
            try {
                RealConnectionPool.this.wait(waitMillis, (int) waitNanos);
            } catch (InterruptedException ignored) {
            }
        }
    }
}
```

在这个方法中，循环调用了`cleanup(long)`方法，返回需要暂停的时间，如果是`-1`，表示不再进行清理任务。

### cleanup(long)

> Performs maintenance on this pool, evicting the connection that has been idle the longest if either it has exceeded the keep alive limit or the idle connections limit.

翻译：维护这个连接池。如果连接已经超过`Keep-Alive`的显示或空置时间了，则清理空置最长的连接。

> Returns the duration in nanos to sleep until the next scheduled call to this method. Returns -1 if no further cleanups are required.

翻译：返回等待下一次清理任务执行的间隔时间，暂停清理任务的线程。如果是`-1`，表示不再需要清理任务（如池中没有连接）。

```java
long cleanup(long now) {
    // 使用中的Connection数量
    int inUseConnectionCount = 0;
    // 空闲的Connection数量
    int idleConnectionCount = 0;
    // 空置时间最长的Connection
    RealConnection longestIdleConnection = null;
    // 最长的空置时间
    long longestIdleDurationNs = Long.MIN_VALUE;

    // Find either a connection to evict, or the time that the next eviction is due.
    // 找出需要被清理的Connection，或者下一次清理任务的时间
    synchronized (this) {
        // 循环对每个RealConnection进行检查，找出最长的控制时间
        for (Iterator<RealConnection> i = connections.iterator(); i.hasNext(); ) {
            RealConnection connection = i.next();

            // 如果这个RealConnection在使用中，则进行去检查下一个
            if (pruneAndGetAllocationCount(connection, now) > 0) {
                inUseConnectionCount++;
                continue;
            }

            idleConnectionCount++;

            // 循环计算出空置的最长时间
            long idleDurationNs = now - connection.idleAtNanos;
            if (idleDurationNs > longestIdleDurationNs) {
                longestIdleDurationNs = idleDurationNs;
                longestIdleConnection = connection;
            }
        }

        // 确定最长空置时间是否需要被清理的条件
        if (longestIdleDurationNs >= this.keepAliveDurationNs
                || idleConnectionCount > this.maxIdleConnections) {
            // 表示已经超时了，从列表中移除这个Connection，并且在下面关闭它的Socket
            connections.remove(longestIdleConnection);
        } else if (idleConnectionCount > 0) {
            // 返回下一次清理任务应该被执行的时间
            return keepAliveDurationNs - longestIdleDurationNs;
        } else if (inUseConnectionCount > 0) {
            // 表示并没有Connection空置，所以下一次检查时间是连接空置的最长时间
            return keepAliveDurationNs;
        } else {
            // 根本就没有连接，关闭清理任务
            cleanupRunning = false;
            return -1;
        }
    }

    closeQuietly(longestIdleConnection.socket());

    // Cleanup again immediately.
    return 0;
}
```

逻辑很清晰，最重要的是`pruneAndGetAllocationCount(RealConnection, long)`用于判断连接是否是空置的。

### pruneAndGetAllocationCount(RealConnection, long)

> Prunes any leaked transmitters and then returns the number of remaining live transmitters on {@code connection}. Transmitters are leaked if the connection is tracking them but the application code has abandoned them. Leak detection is imprecise and relies on garbage collection.

翻译：清除任何被释放掉的`Transmitter`，然后返回`RealConnection`上剩余存活的`Transmitter`的数量。如果应用程序舍弃了`Transimitter`但是`Connection`仍然在引用它，那么就会出现被释放。释放是不精准的，依赖于垃圾回收机制。

大致意思是，一个`RealConnection`上有多个`Transmitter`，如果没有在使用，那么`Transmitter`会被垃圾回收机制处理掉。这个方法就是计算还在使用的`Transmitter`的数量，确定`Connection`是否是空置的。

下面在看看源码：

```java
private int pruneAndGetAllocationCount(RealConnection connection, long now) {
    // 获取Transmitter列表
    List<Reference<Transmitter>> references = connection.transmitters;
    for (int i = 0; i < references.size(); ) {
        Reference<Transmitter> reference = references.get(i);

        // 判断是否被回收掉了
        if (reference.get() != null) {
            i++;
            continue;
        }

        // 出现一个泄漏的Transmitter，是个应用层错误
        TransmitterReference transmitterRef = (TransmitterReference) reference;
        String message = "A connection to " + connection.route().address().url()
                + " was leaked. Did you forget to close a response body?";
        Platform.get().logCloseableLeak(message, transmitterRef.callStackTrace);

        // 如果Transmitter被释放了，那么就没必要保存引用了
        references.remove(i);
        connection.noNewExchanges = true;

        // 如果Connection上没有Transmitter了，表示Connection处于空置状态，长时间空置则应该被回收
        if (references.isEmpty()) {
            connection.idleAtNanos = now - keepAliveDurationNs;
            return 0;
        }
    }

    // 返回Connection上剩余Transmitter的数量
    return references.size();
}
```

我们再重新审视一下`Transmitter`的概念

## Transmitter

> Bridge between OkHttp's application and network layers. This class exposes high-level application layer primitives: connections, requests, responses, and streams.

翻译：`OkHttp`应用层与网络层的桥梁。这个类暴露了上层应用层的基本属性：连接、请求、响应和流。

> This class supports {@linkplain #cancel asynchronous canceling}. This is intended to have the smallest blast radius possible. If an HTTP/2 stream is active, canceling will cancel that stream but not the other streams sharing its connection. But if the TLS handshake is still in progress then canceling may break the entire connection.

翻译：这个类支持异步取消。为了让取消动作影响最小，如果是`HTTP/2`的流是活跃的，那么取消动作只取消它自己的流，不会取消共享的连接。但是如果是正在进行`TLS`握手，那么就会取消整个连接。

在应用层，与网络层沟通使用的是`Transmitter`和`Exchange`。使用`Transmitter`可以建立连接。而在网络层，一个`RealConnection`上又有多个`Transmitter`，这多个`Transmitter`依赖于这个`RealConnection`，获得`Exchange`给应用层，表示进行一次请求/响应。

## 再看看其他方法

最重要的几个方法看完了，再来说说其他方法

```java
public synchronized int idleConnectionCount() {
    int total = 0;
    for (RealConnection connection : connections) {
        if (connection.transmitters.isEmpty()) total++;
    }
    return total;
}

public synchronized int connectionCount() {
    return connections.size();
}
```

这两个方法就不说了，只是查看`Connection`的数量

### transmitterAcquirePooledConnection(Address, Transmitter, List<Route>, boolean)

> Attempts to acquire a recycled connection to {@code address} for {@code transmitter}. Returns true if a connection was acquired.

翻译：尝试获取`Address`，`Transmitter`的一个循环使用的`RealConnection`。返回`true`表示获取到了。

```java
boolean transmitterAcquirePooledConnection(Address address, Transmitter transmitter,
                                            @Nullable List<Route> routes, boolean requireMultiplexed) {
    assert (Thread.holdsLock(this));
    for (RealConnection connection : connections) {
        // 要求是HTTP2多路复用
        if (requireMultiplexed && !connection.isMultiplexed()) continue;
        // 判断能否匹配
        if (!connection.isEligible(address, routes)) continue;
        // Transmitter绑定Connection上
        transmitter.acquireConnectionNoEvents(connection);
        return true;
    }
    return false;
}
```

这个方法提供给`ExchangeFinder`用于匹配`Transmitter`和`Connection`

### put(RealConnection)

```java
void put(RealConnection connection) {
    assert (Thread.holdsLock(this));
    if (!cleanupRunning) {
        cleanupRunning = true;
        executor.execute(cleanupRunnable);
    }
    connections.add(connection);
}
```

添加`RealConnection`到连接池中，并开始清理任务。这个方法提供给`ExchangeFinder`，在构造`RealConnection`进行连接，并添加到池中。

### connectionBecameIdle(RealConnection)

> Notify this pool that {@code connection} has become idle. Returns true if the connection has been removed from the pool and should be closed.

翻译：提醒连接池`Connection`处于空闲状态了。返回`true`表示`Connection`已经被从池中移除了，并且应该被关闭。

```java
boolean connectionBecameIdle(RealConnection connection) {
    assert (Thread.holdsLock(this));
    // Connection不再被使用，或者不允许Connection空置，那么就移除它
    if (connection.noNewExchanges || maxIdleConnections == 0) {
        connections.remove(connection);
        return true;
    } else {
        // 唤醒清理任务
        notifyAll();
        return false;
    }
}
```

`Transmitter`使用这个方法判断绑定的`RealConnection`是否应该被关闭。

### evictAll()

```java
public void evictAll() {
    List<RealConnection> evictedConnections = new ArrayList<>();
    synchronized (this) {
        for (Iterator<RealConnection> i = connections.iterator(); i.hasNext(); ) {
            RealConnection connection = i.next();
            if (connection.transmitters.isEmpty()) {
                connection.noNewExchanges = true;
                evictedConnections.add(connection);
                i.remove();
            }
        }
    }

    for (RealConnection connection : evictedConnections) {
        closeQuietly(connection.socket());
    }
}
```

删除并关闭所有的空置的`Connection`

### connectFailed(Route, IOException)

> Track a bad route in the route database. Other routes will be attempted first.

翻译：在路由数据库中记录错误的路由。其他的路由会优先尝试。

```java
public void connectFailed(Route failedRoute, IOException failure) {
    // Tell the proxy selector when we fail to connect on a fresh connection.
    if (failedRoute.proxy().type() != Proxy.Type.DIRECT) {
        Address address = failedRoute.address();
        address.proxySelector().connectFailed(
                address.url().uri(), failedRoute.proxy().address(), failure);
    }

    routeDatabase.failed(failedRoute);
}
```

## 总结

`RealConnectionPool`实际上很简单，维护`RealConnection`的集合，提供清理任务，提供外部处理这个集合的接口。

1. 多个`Transmitter`对应一个`RealConnection`
2. 一个`Transmitter`对应一个`ExchangeFinder`
3. 一个`Transmitter`对应一个`Exchange`

再来看看流程图：

![okhttp流程图](okhttp流程图.jpg)