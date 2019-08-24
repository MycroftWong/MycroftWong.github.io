---
title: okhttp系列
date: 2019-08-15 16:43:07
top: true
summary: OkHttp架构分析<br/>OkHttp是一个非常好用的网络数据库，本系列文章以OkHttp 3.14.2版本，也是最后一个Java版本
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
---

# okhttp系列

## 前言

今天看到一个面试题：`okhttp`的原理

刚好有时间，搜索了一下，其中一篇文章瞬间点醒我，解决了我对`okhttp`部分一直不解的问题。

最近有时间更新完自己的`okhttp`源码解析系列

## 系列文章

1. {% post_link okhttp基本概念 %}
2. {% post_link okhttp请求与响应执行过程 %}
3. {% post_link okhttp-Interceptor责任链 %}
4. {% post_link okhttp-Interceptor接口 %}
5. {% post_link okhttp-缓存 %}
6. {% post_link okhttp-连接 %}
7. {% post_link okhttp-RealConnectionPool %}

## 流程

下面是`okhttp`的流程图，其中最核心的是`interceptor`的责任链机制，这也是上面面试题的答案：`okhttp`的原理。

![okhttp流程图](okhttp流程图.jpg)

## 参考文章

[OKhttp源码解析详解系列](https://www.jianshu.com/p/d98be38a6d3f)

[Android开源框架源码鉴赏：Okhttp](https://juejin.im/post/5a704ed05188255a8817f4c9#heading-0)

[Okhttp3源码分析](https://www.jianshu.com/p/b0353ed71151)

[OkHttp3架构分析](https://www.jianshu.com/p/9deec36f2759)

[Android OkHttp3源码分析](https://blog.csdn.net/baidu_36959886/article/details/90900261)

[OKHttp源码解析(一)--初阶](https://www.jianshu.com/p/82f74db14a18)
