---
title: okhttp 缓存
date: 2019-08-17 10:16:18
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
- Cache
- 缓存
---

# okhttp 缓存

## 前言

缓存的使用可以减少我们程序请求服务器、读取文件等耗时`IO`的次数，能够极大的提高程序的运行速度、性能，除了在`OkHttp`中使用了缓存，在很多优秀的库中都使用了缓存，图片库最为明显，如`glide`，`fresco`在这方面都是很极致的使用者，又如`Android`控件的`ListView`，`RecyclerView`使用缓存很大的提高了界面的流畅度。

所以，理解缓存、使用缓存是必备的知识。

在`OkHttp`中，`CacheInterceptor`担负着缓存的作用。这一章，深入讨论`CacheInterceptor`的使用。

## 缓存的操作

首先应该明白缓存应该具备的最基本的操作：添加、删除、更新、获取。这很像`CRUD`。当然在实际情况中，我们会额外添加操作，如设置缓存的大小。

## CacheInterceptor

> Serves requests from the cache and writes responses to the cache.

翻译：使用缓存处理请求，将结果写入缓存。

下面是`CacheInterceptor`的源码（删除了非逻辑代码）。

```java
public final class CacheInterceptor implements Interceptor {
    // OkHttp的内部缓存接口
    @Nullable
    final InternalCache cache;

    public CacheInterceptor(@Nullable InternalCache cache) {
        this.cache = cache;
    }

    @Override
    public Response intercept(Chain chain) throws IOException {
        // 获取是否有对应Request的Response缓存对象
        Response cacheCandidate = cache != null
                ? cache.get(chain.request())
                : null;

        long now = System.currentTimeMillis();

        // 缓存策略，弄清楚是否到底使用网络、缓存，还是两者都使用
        CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();
        // 得到的网络请求，若不使用网络（客户端强制使用缓存）则为null
        Request networkRequest = strategy.networkRequest;
        // 得到对应的缓存响应，用于返回或者需要被更新，如果这次请求不使用缓存，则为null
        Response cacheResponse = strategy.cacheResponse;

        if (cache != null) {
            cache.trackResponse(strategy);
        }

        // 如果应用不使用缓存，清理资源
        if (cacheCandidate != null && cacheResponse == null) {
            closeQuietly(cacheCandidate.body());
        }

        // 如果不使用网络请求，并且没有缓存，则返回504错误
        if (networkRequest == null && cacheResponse == null) {
            return new Response.Builder()
                    .request(chain.request())
                    .protocol(Protocol.HTTP_1_1)
                    .code(504)
                    .message("Unsatisfiable Request (only-if-cached)")
                    .body(Util.EMPTY_RESPONSE)
                    .sentRequestAtMillis(-1L)
                    .receivedResponseAtMillis(System.currentTimeMillis())
                    .build();
        }

        // 如果不使用网络，并且有缓存，则返回缓存结果
        if (networkRequest == null) {
            return cacheResponse.newBuilder()
                    .cacheResponse(stripBody(cacheResponse))
                    .build();
        }

        // 前面处理了缓存，现在开始网络请求
        Response networkResponse = null;
        try {
            // 交给下一个Interceptor处理，最后会交给CallServerInterceptor返回一个从网络请求得到的Response
            networkResponse = chain.proceed(networkRequest);
        } finally {
            // 如果出现网络错误，清理资源
            if (networkResponse == null && cacheCandidate != null) {
                closeQuietly(cacheCandidate.body());
            }
        }

        // 获得网络结果，如果是304，则更新、并返回缓存结果
        if (cacheResponse != null) {
            if (networkResponse.code() == HTTP_NOT_MODIFIED) {
                Response response = cacheResponse.newBuilder()
                        .headers(combine(cacheResponse.headers(), networkResponse.headers()))
                        .sentRequestAtMillis(networkResponse.sentRequestAtMillis())
                        .receivedResponseAtMillis(networkResponse.receivedResponseAtMillis())
                        .cacheResponse(stripBody(cacheResponse))
                        .networkResponse(stripBody(networkResponse))
                        .build();
                networkResponse.body().close();

                // 在组合和网络结果的header之后更新缓存
                cache.trackConditionalCacheHit();
                cache.update(cacheResponse, response);
                return response;
            } else {
                // 清理资源
                closeQuietly(cacheResponse.body());
            }
        }

        // 构造返回的结果Response
        Response response = networkResponse.newBuilder()
                .cacheResponse(stripBody(cacheResponse))
                .networkResponse(stripBody(networkResponse))
                .build();

        if (cache != null) {
            // 若根据HTTP协议可以缓存，那么进行缓存
            if (HttpHeaders.hasBody(response) && CacheStrategy.isCacheable(response, networkRequest)) {
                // Offer this request to the cache.
                CacheRequest cacheRequest = cache.put(response);
                return cacheWritingResponse(cacheRequest, response);
            }

            // 如果是不可缓存的method，则移除缓存
            if (HttpMethod.invalidatesCache(networkRequest.method())) {
                try {
                    cache.remove(networkRequest);
                } catch (IOException ignored) {
                    // The cache cannot be written.
                }
            }
        }

        // 返回结果
        return response;
    }
}
```

主要的逻辑是：

首先判断是否需要使用缓存结果：

1. 需要，是否有缓存结果
    1. 有，返回
    2. 无，返回`504`错误
2. 不需要，进行网络请求，分析。判断是否之前有缓存，并且网络结果`code`是`304`（资源没有修改过）
    1. 是，更新缓存，并且返回更新后的缓存结果
    2. 否，返回网络结果，判断是否可以将网络结果缓存
        1. 是，缓存网络返回结果
        2. 否，不缓存

这些条件的由来，是对`HTTP`协议的理解。根据`HTTP`协议，判断是否使用缓存、是否更新缓存、是否删除缓存、缓存是否过期等。

## InternalCache

了解了`OkHttp`的缓存逻辑，我们来看看实际它是如何缓存的。在`CacheInterceptor`使用了`InternalCache`这个接口来进行缓存。

> OkHttp's internal cache interface. Applications shouldn't implement this: instead use {@link okhttp3.Cache}.

翻译：`OkHttp`的内部缓存接口。应用层应该使用`Cache`类。

下面是`InternalCache`的代码

```java
public interface InternalCache {
    @Nullable
    Response get(Request request) throws IOException;

    @Nullable
    CacheRequest put(Response response) throws IOException;

    void remove(Request request) throws IOException;

    void update(Response cached, Response network);

    void trackConditionalCacheHit();

    void trackResponse(CacheStrategy cacheStrategy);
}
```

除了两个用于`log`的方法，其他四个接口满足最基本的缓存要求。

## Cache

> Caches HTTP and HTTPS responses to the filesystem so they may be reused, saving time and bandwidth.

翻译：使用文件缓存`HTTP`和`HTTPS`响应，节约时间和带宽。

代码很长，说明主要的几个点：

### 1. `Cache`内部有一个`OkHttp`需要的`InternalCache`匿名对象

在这个匿名对象中，所有方法都是代理，都调用`Cache`本身的方法。下面是源码：

```java
final InternalCache internalCache = new InternalCache() {
    @Override
    public @Nullable
    Response get(Request request) throws IOException {
        return Cache.this.get(request);
    }

    @Override
    public @Nullable
    CacheRequest put(Response response) throws IOException {
        return Cache.this.put(response);
    }

    @Override
    public void remove(Request  request) throws IOException {
        Cache.this.remove(request);
    }

    @Override
    public void update(Response cached, Response network) {
        Cache.this.update(cached, network);
    }

    @Override
    public void trackConditionalCacheHit() {
        Cache.this.trackConditionalCacheHit();
    }

    @Override
    public void trackResponse(CacheStrategy cacheStrategy) {
        Cache.this.trackResponse(cacheStrategy);
    }
};
```

### 2. `Cache`使用`DiskLruCache`来管理缓存写入到文件的内容

```java
final DiskLruCache cache;

public Cache(File directory, long maxSize) {
    this(directory, maxSize, FileSystem.SYSTEM);
}

Cache(File directory, long maxSize, FileSystem fileSystem) {
    this.cache = DiskLruCache.create(fileSystem, directory, VERSION, ENTRY_COUNT, maxSize);
}
```

#### `DiskLruCache`

> A cache that uses a bounded amount of space on a filesystem. Each cache entry has a string key and a fixed number of values. Each key must match the regex <strong>[a-z0-9_-]{1,64}</strong>. Values are byte sequences, accessible as streams or files. Each value must be between {@code 0} and {@code Integer.MAX_VALUE} bytes in length.

翻译：这是一个在文件系统中使用了有限空间的缓存。每一个缓存都有一个字符串的键和一定数量的值。每个键必须满足正则表达式：`[a-z0-9_-]{1,64}`。值是`byte`序列，通过文件或者流获取。每个值的长度是`0-Integer.MAX_VALUE`之间的`byte`。

> The cache stores its data in a directory on the filesystem. This directory must be exclusive to the cache; the cache may delete or overwrite files from its directory. It is an error for multiple processes to use the same cache directory at the same time.

翻译：缓存将数据存入一个文件夹内的文件中。这个文件夹必须是这个缓存独享的。缓存需要删除或者重写其中的文件。同时在同一个缓存目录使用多个操作会造成错误。

很清楚，`DiskLruCache`是磁盘缓存的实现。

读取缓存时，我们得到的是一个`DiskLruCache.Snapshot`，我们通过其中的`okio.Source`读取缓存的内容。写入缓存时，我们得到一个`DiskLruCache.Editor`，通过其中`okio.Sink`写入缓存的内容。

这样，我们就可以将`Request`、`Response`的内容写入进行缓存，同时，将读取的缓存内容转换为`Request`、`Response`对象。

## 总结

下图是`OkHttp`缓存的调用关系：

1. `CacheInterceptor`通过`InternalCache`操作缓存，交换对象是`Request`和`Response`
2. `InternalCache`则只是代理了`Cache`
3. `Cache`将`Request`和`Response`的内容转换成明文，通过`DiskLruCache`保存到文件中

![okhttp Cache](okhttpCache.jpg)

`OkHttp`在这里非常合理的使用了接口：

1. `InternalCache`定义了`CacheInterceptor`操作缓存的接口，不提供额外的功能
2. `Cache`则是向应用层提供了操作接口，隐藏了代理`InternalCache`的方法

## 特别感谢

[OKHttp源码解析(六)--中阶之缓存基础](https://www.jianshu.com/p/b32d13655be7)
