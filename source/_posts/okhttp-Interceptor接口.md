---
title: okhttp Interceptor接口
date: 2019-08-16 19:53:36
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
- Interceptor
- Chain
---

# okhttp Interceptor接口

## 前言

前面一直在说`Interceptor`责任链，那`Interceptor`到底是什么呢

## Interceptor

> Observes, modifies, and potentially short-circuits requests going out and the corresponding responses coming back in. Typically interceptors add, remove, or transform headers on the request or response.

翻译：观察、修改、并有可能拦截`Request`，而且可以返回自定义的`Response`。通常`Interceptor`作用是在`Request`和`Response`上添加、删除、转换`header`。

下面是`Interceptor`的源码

```java
public interface Interceptor {
    // 得到Chain对象，获取请求必要的参数如Request，Call，处理得到Response
    Response intercept(Chain chain) throws IOException;

    interface Chain {
        // 得到责任链上的Request
        Request request();

        // 在Interceptor中调用，将其交由责任链中的下一个Interceptor处理
        Response proceed(Request request) throws IOException;

        /**
         * Returns the connection the request will be executed on. This is only available in the chains
         * of network interceptors; for application interceptors this is always null.
         */
        // 返回将会被用于执行的Connection。只有在责任链是networkInterceptor是会得到值，在应用层interceptor总是返回null
        @Nullable
        Connection connection();

        // 得到Call对象
        Call call();

        // 连接超时时间
        int connectTimeoutMillis();

        // 得到一个新的责任链，设置了连接超时
        Chain withConnectTimeout(int timeout, TimeUnit unit);

        // 读取超时时间
        int readTimeoutMillis();

        // 得到一个新的责任链，设置了读取超时
        Chain withReadTimeout(int timeout, TimeUnit unit);

        // 写入超时时间
        int writeTimeoutMillis();

        // 得到一个新的责任链，设置了写入超时
        Chain withWriteTimeout(int timeout, TimeUnit unit);
    }
}
```

`Interceptor`的代码很简单，通常，我们需要实现`Interceptor`实现我们自己的逻辑时，我们会调用`Chain.proceed(Request)`方法，表示交由下一个`Interceptor`进行处理。我们可以处理`Request`，也可以处理由下一个`Interceptor`返回的结果`Response`。如果我们自己能够处理（这样的情况极少），那么我们可以不用调用`Chain.proceed(Request)`方法，交由下一`Interceptor`处理。

## Interceptor.Chain

`Chain`是什么呢，也没有注释，我们来看一下`Chain`的唯一实现：`RealInterceptorChain`。

### RealInterceptorChain

> A concrete interceptor chain that carries the entire interceptor chain: all application interceptors, the OkHttp core, all network interceptors, and finally the network caller.
> If the chain is for an application interceptor then {@link #connection} must be null. Otherwise it is for a network interceptor and {@link #connection} must be non-null.

翻译：这是一个确切的`Interceptor.Chain`实现类，负责整个`Interceptor`责任链的传递：通过应用层的`interceptor`，`OkHttp`的核心，所有的`networkInterceptor`，最后进行网络请求。如果是应用层的`interceptor`，那么`connection()`方法返回的是`null`，否则是一个`networkInterceptor`，返回的则不是`null`。

```java
public final class RealInterceptorChain implements Interceptor.Chain {
    // 责任链列表
    private final List<Interceptor> interceptors;
    // 应用层与网络层的桥梁
    private final Transmitter transmitter;
    // 连接管理与事件的交换桥梁
    private final @Nullable Exchange exchange;
    // 责任链的位置
    private final int index;
    // 请求
    private final Request request;
    // 执行器
    private final Call call;
    // 连接超时
    private final int connectTimeout;
    // 读取超时
    private final int readTimeout;
    // 写入超时
    private final int writeTimeout;
    // 标记proceed被执行的次数，保证只能被执行一次
    private int calls;

    public RealInterceptorChain(List<Interceptor> interceptors, Transmitter transmitter,
                                @Nullable Exchange exchange, int index, Request request, Call call,
                                int connectTimeout, int readTimeout, int writeTimeout) {
        this.interceptors = interceptors;
        this.transmitter = transmitter;
        this.exchange = exchange;
        this.index = index;
        this.request = request;
        this.call = call;
        this.connectTimeout = connectTimeout;
        this.readTimeout = readTimeout;
        this.writeTimeout = writeTimeout;
    }

    // 删除了与逻辑无关的代码

    @Override
    public Response proceed(Request request) throws IOException {
        return proceed(request, transmitter, exchange);
    }

    public Response proceed(Request request, Transmitter transmitter, @Nullable Exchange exchange)
            throws IOException {
        // 不能超过责任链数量
        if (index >= interceptors.size()) throw new AssertionError();

        // 标记当前Interceptor.Chain已经被执行过了
        calls++;

        // 保证需要执行网络请求时，Exchange已经准备好了
        if (this.exchange != null && !this.exchange.connection().supportsUrl(request.url())) {
            throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
                    + " must retain the same host and port");
        }

        // 保证Chain.proceed()只被执行过一次
        if (this.exchange != null && calls > 1) {
            throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
                    + " must call proceed() exactly once");
        }

        // 调用责任链的下一个Interceptor
        RealInterceptorChain next = new RealInterceptorChain(interceptors, transmitter, exchange,
                index + 1, request, call, connectTimeout, readTimeout, writeTimeout);
        Interceptor interceptor = interceptors.get(index);
        Response response = interceptor.intercept(next);

        // 确保下一个Interceptor被执行过了`Chain.proceed()`
        if (exchange != null && index + 1 < interceptors.size() && next.calls != 1) {
            throw new IllegalStateException("network interceptor " + interceptor
                    + " must call proceed() exactly once");
        }

        // 确保Interceptor返回的Response不是null
        if (response == null) {
            throw new NullPointerException("interceptor " + interceptor + " returned null");
        }

        // 保证Interceptor返回的ResponseBody不是null
        if (response.body() == null) {
            throw new IllegalStateException(
                    "interceptor " + interceptor + " returned a response with no body");
        }

        // 返回Response
        return response;
    }
}
```

从代码可以看出，核心部分是执行下一个责任链，前后做了大量的判断，以保证责任链正确的执行。

### Chain是什么

这下我们知道：`Chain`保证在责任链上`Interceptor`正确的执行。如果`Interceptor`调用了`Chain.proceed(Request)`方法，表示继续交由下一`Interceptor`处理，当然`Interceptor`也可以自己处理得到`Response`直接返回，不再传递下去。
