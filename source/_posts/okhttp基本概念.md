---
title: okhttp基本概念
date: 2019-08-15 16:45:27
categories:
- 开源库
    - okhttp
tags:
- okhttp
- 源码
- http
- https
- 面试
- 责任链模式
---

# okhttp基本概念

[okhttp官网](https://github.com/square/okhttp)

> An HTTP client for Android, Kotlin, and Java. 

官方介绍：`Android`, `Kotlin`, `Java`的`HTTP`客户端

## 基本类

在使用`okhttp`时，我们一定会碰到的一些类的介绍


### 基本的网络请求

下面是使用`okhttp`一串最基本的网络请求

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

### 1. Request

> An HTTP request. Instances of this class are immutable if their {@link #body} is null or itself immutable.

翻译：代表有一个`HTTP`请求，这个类的实例是不可变的。

`Request`有多个属性：`url`，`method`，`header`，`body`，`tag`

字面意思很简单，需要解释一下的是`tag`，标记这个`request`，在`interceptor`，`event listener`，`callback`中都可以特殊处理标记的`request`。

`body`实际上是`RequestBody`对象，通常当请求`method`为`POST`，`PUT`之类的时可以使用。

### 2. RequestBody

`HTTP`的请求体，如`POST`提交表单。

常用到的有两个子类：`FormBody`，`MultipartBody`。前者多用于`POST`表单提交，后者多用于文件上传。

### 3. Call

> A call is a request that has been prepared for execution. A call can be canceled. As this object represents a single request/response pair (stream), it cannot be executed twice.

翻译：`Call`表示将`Request`请求为执行做准备。`Call`可以被取消。它只代表单独的一个请求-相应对，所以不能被执行两次。

`Call`是一个接口，包含的主要方法：`Response execute()`，`void enqueue(Callback)`，`cancel()`，分别表示同步执行、加入队列等待执行、取消执行。

### 4. Response

> An HTTP response. Instances of this class are not immutable: the response body is a one-shot value that may be consumed only once and then closed. All other properties are immutable.

翻译：`HTTP`响应。实例不可变：其中的`ResponseBody`是一次性的值，当被消费一次之后就会被关闭。其他所有属性都是不可变的。

`Response`封装了响应结果，其中主要的属性有：`code`，`message`，`request`，`header`，`body`。

`body`指的是`ResponseBody`。

### 5. ResponseBody

> A one-shot stream from the origin server to the client application with the raw bytes of the response body. Each response body is supported by an active connection to the webserver. This imposes both obligations and limits on the client application.

翻译：远程服务器响应客户端请求得到的一次性的二进制数据流。每个`ResponseBody`依赖于活跃的与服务器的连接。这种机制强行限制了客户端应用的使用。

`ResponseBody`实际上将返回的数据作为流来处理，最重要方法是`BufferedSource source()`，得到`BufferedSource`流接口，可以转换成真实数据。


## 责任链 Chain of Responsibility

> The chain of responsibility pattern let these object which can handle the request of customer become to a chain,and a handler of the responsibility chain handles the request of a customer or deliver the request of a customer to next handler of the chain of responsibility.

> 责任链模式将处理用户请求的对象形成一个链，责任链上的每个处理者要么处理用户的请求，要么把请求传递给责任链上的下一个处理者
