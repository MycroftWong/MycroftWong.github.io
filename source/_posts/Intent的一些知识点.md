---
title: Intent的一些知识点
date: 2019-12-03 15:47:06
categories:
- Android
tags:
- Android
- Intent
---

# Intent的一些知识点

## Intent的用途

`Intent`是一个消息传递对象。使用`Intent`向系统请求操作，主要包括：

1. 启动`Activity`
2. 启动`Service`
3. 发送`Broadcast`

下图展示了启动`Activity`时，`Intent`在两个`Activity`之间如何传递的

![startActivity过程](intent_start_activity.png)

## Intent类型

官方的说法：

* 显式`Intent`：通过提供目标应用的软件包名称或完全限定的组件类名来指定可处理`Intent`的应用
* 隐式`Intent`：不会指定特定的组件，而是声明要执行的常规操作，从而允许其他应用中的组件来处理

是否指定组件是区别显式和隐式`Intent`的关键。显式`Intent`启动指定的组件，而隐式`Intent`通过声明执行的操作向系统请求操作。

### 构造显式Intent

可通过`Intent`构造器、`setComponent(ComponentName)`、`setClassName(Context, String)`、`setClassName(String, String)`、`setClass(Context, Class<?>)`指定组件。

## Activity如何声明接收隐式Intent

`Activity`想要接收隐式`Intent`，必须在`manifest`中的声明中添加`<intent-filter>`元素，在`<intent-filter>`内部，可使用`action`、`data`、`category`三个元素的一个或多个指定要接收的`Intent`类型。

### action

前提条件：一个`Activity`可以声明一个或多个`action`，未声明`action`表示不接受隐式`Intent`；一个`Intent`不设置或只能设置一个`action`。

匹配条件：`Activity`声明的`action`必须包含想要接收的隐式`Intent`的`action`。

### category

前提条件：一个`Activity`可以声明一个或多个`category`，一个`Intent`也可以包含一个或多个`category`。

重点：**`Intent`不存在没有`category`的情况，因为使用`startActivity`或`startActivityForResult`启动隐式`Activity`时会自动将`CATEGORY_DEFAULT`应用到`Intent`中，所以`Activity`中至少有一个`android.intent.category.DEFAULT`**

匹配条件：`Activity`声明的所有`category`必须包含`Intent`的所有`category`。

### data

`data`分为两部分：`MIME`和`URI`。`MIME`和`URI`各自有匹配的规则，非常简单，这里就不说了。

前提条件：一个`Activity`可以声明0个、一个或多个`data`，一个`Intent`也可以包含0个或一个`data`。

1. 当`Activity`未声明`data`时，只能接受无`data`的`Intent`
2. 只声明了`URI`的`Activity`，只能接受只有且能够匹配`URI`部分的`data`的`Intent`
3. 只声明了`MIME`的`Activity`，接受能够匹配`MIME`的`data`的`Intent`，或是能够匹配`MIME`，`URI`是`content`或`file`类型数据的`Intent`
4. 声明了`URI`和`MIME`的`Activity`，只能接受`URI`和`MIME`同时匹配的`Intent`

注意其中有点差异的是，之声明了`MIME`部分的`Activity`，可以接受`content`或`file`类型`URI`数据的`Intent`。

## 一个可接受隐式Intent的Activity的过滤器的友好设计

1. 声明一个或多个想要接收的`action`
2. 声明至少一个`category`————`android.intent.category.DEFAULT`，若想要被浏览器启动，还需要声明`android.intent.category.BROWSABLE`
3. 声明一个或多个想要接收的`data`，若想要接收文件类型的`data`，可以只声明`MIME`
4. 可声明多个`<intent-filter>`接收多种过滤器
5. （很少使用）若是不想其他`app`启动，则可以添加`"exported"=false`。

## 总结

这个规则其实很简单，我们通常会在打开相机、访问文件时使用隐式`Intent`调动系统组件。反过来，我们想在其他应用如浏览器打开我们的`app`时，就需要自己声明过滤器，这时就要站在编写`<intent-filter>`的角度来看待`Intent`的过滤规则，这是之前一直困扰我，没能够友好理解`Intent`的关键。
