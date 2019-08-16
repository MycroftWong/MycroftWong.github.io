---
title: okhttp Interceptor责任链
date: 2019-08-16 13:47:15
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

# okhttp Interceptor责任链

## 责任链

> 责任链模式将处理用户请求的对象形成一个链，责任链上的每个处理者要么处理用户的请求，要么把请求传递给责任链上的下一个处理者

实例：
1. 请求
2. 处理者：处理请求的对象
3. 链：处理者形成的链表

说明：
1. 一个请求交给链表上第一个处理者处理
2. 第一个处理者处理不了，交由下一个处理者
3. 如果链上某一处理者能够处理，则直接返回处理结果，不再交由后面的处理者处理
4. 如果最后一个处理者也无法处理，则返回无效的结果
5. 后面处理者处理的结果，前面的处理者也可以操作。

## RealCall.getResponseWithInterceptorChain()

源码：

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

下面是`Interceptor`责任链流程图

![Interceptor 责任链](Interceptor责任链.jpg)

在这个链表中，可以直接自行处理`Request`得到`Response`，不交由下一级处理者的`Interceptor`有：`CacheInterceptor`，`CallServerInterceptor`。其他的`Interceptor`用于其他处理，如`ConnectInterceptor`用于连接服务器。

下面分别分析这些`Interceptor`的作用

## RetryAndFollowUpInterceptor

> This interceptor recovers from failures and follows redirects as necessary. It may throw an {@link IOException} if the call was canceled.

翻译：这个`Interceptor`用于失败重试和必要时重定向。如果`Call`被取消，可能会抛出`IOException`。

具体我就不深入代码了，这更多的涉及`HTTP`协议的相关内容，其实知道这个类的作用就行了。可以简单的了解几点：
1. 重定向最大次数为20次，`Chrome`是21次，`Firefox`，`curl`，`wget`是20次，`Safari`是16次，`HTTP/1.0`推荐是5次。
2. 不能重试的情况
    * 应用层禁止重试
    * 无法发送`RequestBody`：只有当`RequestBody`是被缓存的（`Buffered`）才能重试（可能`RequestBody`是一次性的）
    * 无法恢复的异常，如`ProtocolException`协议异常
    * 没有更多的线路去重试

## BridgeInterceptor

> Bridges from application code to network code. First it builds a network request from a user request. Then it proceeds to call the network. Finally it builds a user response from the network response.

翻译：应用层代码和网络层代码的桥梁。首先它从用户请求构建一个网络层的`Request`，然后它交由网络层处理，最后从网络层的`Response`处理得到一个应用层的`Response`。

这段话很直接，实际上，它处理传送过来的应用`Request`，添加一些参数，然后对交由网络层处理得到的`Response`再进一步处理，返回给应用层。

`BridgeInterceptor`对`Request`的处理：
1. 如果有请求体`RequestBody`，则根据请求体内容，添加`header`：`Content-Type`、`Content-Length`、`Transfer-Encoding`
2. `header`中添加`Host`
3. 若`header`中没有`Connection`，则设置为`Keep-Alive`
4. 若`header`中没有`Accept-Encoding`和`Range`，设置`Accept-Encoding`为`gzip`
5. 添加`cookie`
6. 添加`User-Agent`

`BridgeInterceptor`对`Response`的处理：
1. 复制网络层的`Response`内容，得到新的`Response`
2. 处理`cookie`
3. 处理`gzip`响应结果

## CacheInterceptor

> Serves requests from the cache and writes responses to the cache.

翻译：使用缓存处理请求，将结果写入缓存。

字面意思很简单。下面是需要注意的点：

1. 如果只能从配置了只能从`Cache`中获取结果，但是没有缓存结果，将返回`504`错误。
2. 如果可以从缓存中获取数据，并且有缓存，则直接返回缓存数据
3. 进行网络请求，如果返回`304`表示结果与上次请求相同，返回缓存结果，同时更新缓存。
4. 如有必要，将得到的网络请求结果，写入缓存

可以看出，`CacheInterceptor`是可以处理`Request`得到`Response`的，如果可以，不必交由下一个`Interceptor`进行处理。

## ConnectInterceptor

> Opens a connection to the target server and proceeds to the next interceptor.

翻译：为目标服务器打开一个连接`Connection`，并且交由下一个`Interceptor`进行处理。

从这段话就可以看出`ConnectIntercetpor`则不产生直接结果。并且只是用于建立连接。

## CallServerInterceptor

> This is the last interceptor in the chain. It makes a network call to the server.

翻译：这是链中最后一个`Interceptor`。它向服务器发送一个网络请求。

它实际完成的工作：
1. 写入请求头
2. 写入请求体
3. 读取请求头
4. 读取请求体
5. 处理请求结果

除了控制逻辑之外，实际处理请求的是`Exchange`

### Exchange

> Transmits a single HTTP request and a response pair. This layers connection management and events on {@link ExchangeCodec}, which handles the actual I/O.

翻译：发送单独`HTTP Request-Response`对。它将连接层管理与事件分层，`ExchangeCodec`处理真实的`IO`操作。

### ExchangeCodec

> Encodes HTTP requests and decodes HTTP responses.

翻译：编码`HTTP`请求，解码`HTTP`响应。

实际上就是转换对象/实例与`HTTP`协议内容。

## 总结

这样子分别理清所有`Interceptor`的功能，一下就豁然开朗了。不需要在意代码细节，整个流程真的是非常的漂亮。我在编码，而`OkHttp`的开发者是在做艺术。

同时，我们通过`OkHttpClient`添加的`interceptor`和`networkInterceptor`分别是在`RetryAndFollowUpInterceptor`、`CallServerInterceptor`之前。所以可以明白：

1. 如果使用的是缓存，则`interceptor`可以被调用到，而`networkdInterceptor`是不会调用到的
2. 我们通常可以在`networkInterceptor`中进行一些特殊的处理，如`access token`过期时，使用`refresh token`刷新`access token`，配合`Request`的`tag`可以区分普通请求和刷新`access token`请求
3. 我们可以在`interceptor`中加密数据
4. 甚至我们可以自己返回`Response`，拦截责任链。