---
title: okhttp实现token验证
date: 2019-09-21 16:51:47
categories:
- 开源库
    - okhttp
tags:
- okhttp
- 开源
- http
- token
---

# okhttp实现token验证

## 前言

公司目前的项目使用了`token`来验证用户。登陆之后会返回最新的`access token`，后续在每次请求`API`时，服务端会返回最新的`access token`，客户端进行保存。若一段时间内（假定是7天）没有进行操作，则需要重新登陆。

## 验证流程

![token验证流程](token验证流程.png)

在登陆成功之后，后续的`token`验证流流程如下：

1. 服务端返回`access token`，客户端保存
2. 客户端发起`HTTP`请求，携带`token`，添加`header` `Authorization: Bearer {access token}`
3. 服务端验证`access token`，若过期，则返回状态码`401`，若未过期，会添加`header` `NewToken: {new access token}`。
4. 若返回`401`，客户端需显示登陆页面，提示用户登陆，若验证通过，正常处理请求，并保存最新的`access token`，在后续请求中使用。

## 客户端的处理

### 思考

需要考虑的点：

1. 因为我们的`APP`是必须登陆之后才能使用，后续的接口必须依赖`access token`，故可以认为（除登录页和闪屏页部分页面的）所有界面都需要验证`access token`是否有效。
2. 若是在后续所有页面都进行`access token`验证的判断，侵入性较强，故`access token`验证肯定是需要进行同意处理的。
3. 在`okhttp`中，处理`access token`验证有两种方式：最常用的`Interceptor`和较少使用的`Authenticator`。另外，我们也可以统一`HTTP`请求和响应，不过这和`Interceptor`的功能重合，所以只需要考虑`Interceptor`和`Authenticator`。
4. 实际使用中，会发现`Authenticator`的调用是在`HTTP`响应之后（`RetryAndFollowUpInterceptor`的功能），若是使用`Authenticator`，则是在一次`HTTP`相应之后，添加`access token`再进行一次请求，不可取，所以最后的结论是在`Interceptor`中添加`access token`。
5. 在`okhttp`中，有两种添加`Interceptor`的方式，一种是在添加在`RetryAndFollowUpInterceptor`之前，一种是在`CallServerInterceptor`之前（即`networkInterceptors`），后一种是一定需要网络请求时才会调用，前者则是可能在缓存中就得到了值，对于一些静态资源，是不需要`access token`验证的，所以自然添加到`networkInterceptors`中。

### Interceptor

实现的代码：

```kotlin
object TokenInterceptor : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val oldRequest = chain.request()

        val newRequestBuilder = oldRequest.newBuilder()
            .header("User-Agent", "android/" + AppUtils.getAppVersionName())

        // 判断是否请求的我们公司的服务端
        if (isNeedToken(oldRequest)) {
            // 若存在token，添加token
            DataService.accessToken?.let {
                newRequestBuilder.header("Authorization", "Bearer $it")
            }

            val response = chain.proceed(newRequestBuilder.build())

            // 若是401，表示验证失败
            if (HttpURLConnection.HTTP_UNAUTHORIZED == response.code) {
                // 启动登陆界面
                ActivityUtils.startActivity(LoginActivity.getIntent(ActivityUtils.getTopActivity()))
            }

            // 若是返回新的token，则保存
            response.header("NewToken")?.let {
                DataService.accessToken = it
            }

            return response
        }

        return chain.proceed(newRequestBuilder.build())
    }

    private fun isNeedToken(request: Request): Boolean {
        // 匹配...
    }
}
```

若同时发起多个`HTTP`请求，上面的实现则会出现问题，并且这样的情况不可避免。通常在主页会有比较复杂的逻辑，会同时发起多个`HTTP`请求。下面是可能存在的问题：

1. 若同时发起多个`HTTP`请求，则会出现多个请求同时返回`401`的问题，同时启动多个登陆页面，当然这样的错误也可以避免，如设置登陆页面`Activity`的`launchMode`为`singleTask`。
2. 多个`HTTP`响应也会造成页面上可能会同时弹出多个`Toast`的情况。
3. 多个`HTTP`请求携带同样的`access token`，返回新的`access token`则不同，可能会因为网络延时，造成服务端最新的`access token`反而会被前一个`access token`覆盖。

综上，我给出的解决方案是进行同步：

```kotlin
object TokenInterceptor : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val oldRequest = chain.request()

        val newRequestBuilder = oldRequest.newBuilder()
            .header("User-Agent", "android/" + AppUtils.getAppVersionName())

        // 判断是否请求的我们公司的服务端
        if (isNeedToken(oldRequest)) {
            synchronized(this) {
                // 若存在token，添加token
                DataService.accessToken?.let {
                    newRequestBuilder.header("Authorization", "Bearer $it")
                }

                val response = chain.proceed(newRequestBuilder.build())

                // 若是401，表示验证失败
                if (HttpURLConnection.HTTP_UNAUTHORIZED == response.code) {
                    // 启动登陆界面
                    ActivityUtils.startActivity(LoginActivity.getIntent(ActivityUtils.getTopActivity()))
                }

                // 若是返回新的token，则保存
                response.header("NewToken")?.let {
                    DataService.accessToken = it
                }
            }

            return response
        }

        return chain.proceed(newRequestBuilder.build())
    }

    private fun isNeedToken(request: Request): Boolean {
        // 匹配...
    }
}
```

除了上面的实现，还需要设置登陆页面`Activity`的`launchMode`为`singleTask`，减少首页`Activity`请求失败后弹出`Toast`，另外可以在闪屏页面进行验证，不过设置超时时间尽量短。

不过使用同步就会使每个需要`token`验证的请求排队，对于`APP`并发量较少的情况来说没有问题，若是一些不需要`token`验证，并发量高的`HTTP`请求，如加载瀑布流图片，则可以进一步判断，不进入同步代码块，这样也能提高效率。

## 总结

实现看起来非常简单，不过这是要在理解`access token`验证的基础上，并且需要经历很多测试（我让服务端的大神设置`access token`过期时间为5分钟，这样可以频繁触发`access token`）。建议阅读后面的参考文章。

## 参考阅读

[理解OAuth 2.0](http://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html)

[OAuth 2.0 筆記 (6) Bearer Token 的使用方法](http://www.blog.chinaunix.net/uid-9162199-id-4694665.html)

[OAuth 2.0: Bearer Token Usage](https://www.cnblogs.com/XiongMaoMengNan/p/6785155.html)

[RxJava2 + Retrofit2 完全指南 之 Authenticator处理与Token静默刷新](https://www.jianshu.com/p/7a2d2d7497a1)

