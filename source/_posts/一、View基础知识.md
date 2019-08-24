---
title: 一、View基础知识
date: 2019-08-24 11:41:12
categories:
- Android
    - View
tags:
- Android
- View
- measure
- layout
- draw
- touch event
- 面试
---

# View基础知识

## 前言

`View`基础知识点，完全可以参考文章[自定义View基础 - 最易懂的自定义View原理系列（1）](https://www.jianshu.com/p/146e5cec4863)，这篇文章已经非常的详细。

我这里将我理解的东西总结一下。

## View与ViewGroup

### View

`View`的代码有近3万行，文件大小有`1M`，这是一个非常庞大的代码，我想谁也不愿意读完全部代码来分析它的机制，而是更愿意通过官方的文档来理解。

> This class represents the basic building block for user interface components. A View occupies a rectangular area on the screen and is responsible for drawing and event handling. View is the base class for <em>widgets</em>, which are used to create interactive UI components (buttons, text fields, etc.). The {@link android.view.ViewGroup} subclass is the base class for <em>layouts</em>, which are invisible containers that hold other Views (or other ViewGroups) and define their layout properties.

翻译：`View`代表了用户界面组件的基础构建模块。`View`占据屏幕上的一块矩形区域，负责绘制和处理事件（屏幕触摸事件）。`View`是`widget`的基类，用于创造交互式的`UI`控件，如`Button`等。子类`ViewGroup`是`layout`的基类，`layout`是不可见的容器，用于包含其他`View`或`ViewGroup`，并且定义布局属性。

大致意思是（**重点**），`View`作为交互式组件，而`ViewGroup`则是用于布局。

### ViewGroup

> A <code>ViewGroup</code> is a special view that can contain other views (called children.) The view group is the base class for layouts and views containers. This class also defines the {@link android.view.ViewGroup.LayoutParams} class which serves as the base class for layouts parameters.

翻译：`ViewGroup`是特殊的`View`，用于包含其他的`View`（被称为`children`）。`ViewGroup`是`layout`的基类，也可以说是`View`的容器。在其中定义了`ViewGroup.LayoutParams`类，是`layout parameters`的基类。

这里再次说明了`ViewGroup`的任务是布局。而`ViewGroup.LayoutParams`则是封装了布局的参数。

### 强调

我再换一种说法解释上面的内容：

1. `View`用于显示内容，着重在于`measure`，`draw`和`touch event`
2. `ViewGroup`用于控制布局，着重在于`layout`，`measure`和`touch event`
3. `View`负责自身的显示
4. `ViewGroup`着重`children`如何在`ViewGroup`中显示

下面是`View`与`ViewGroup`的树形结构

![View树结构](View树结构.jpg)

## Android坐标系

`Android`的坐标系和我们数学上的坐标系有出入。

`Android`坐标系以屏幕左上角为起点，向右为`X`轴增大方向，向下为`Y`轴增大方向。如下图：

![Android坐标系](Android坐标系.png)

`View`在`ViewGroup`上的坐标，则是由`top`，`bottom`，`left`，`right`决定。如下图，`ABCD`四个点的坐标分别是`(left, top)`，`(right, top)`，`(right, bottom)`，`(left, bottom)`。**注意，是相对于其`parent`的坐标，并不是相对于屏幕左上角的坐标**。在`View`显示在屏幕上之后，可以通过对应的`get`方法获得值。

![View坐标](View坐标.webp)

## Android角度

回顾一下角度与弧度的概念：

![角度与弧度](角度与弧度.webp)

在`Android`坐标系中，顺时针为角度增大方向。如下图

![屏幕坐标系角度增大方向](屏幕坐标系角度增大方向.webp)

## 总结

这是`View`必须了解的一些基础知识，在后面的使用中一定会用到。

## 参考文章

[自定义View基础 - 最易懂的自定义View原理系列（1）](https://www.jianshu.com/p/146e5cec4863)


[Android自定义View之时钟](https://blog.csdn.net/z_Xiaozuo/article/details/82560260)