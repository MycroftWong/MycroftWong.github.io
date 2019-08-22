---
title: Parcelable为什么效率高于Serializable
date: 2019-08-22 10:15:23
categories:
- Android
    - 序列化
tags:
- Android
- Parcelable
- Serializable
---

# Parcelable为什么效率高于Serializable

## 前言

在[WanAndroid](https://wanandroid.com/)上看到[每日一问 Parcelable 为什么效率高于 Serializable ？](https://www.wanandroid.com/wenda/show/9002)这篇文章，虽然知道`Parcelable`比`Serializable`效率高，但是一直不知道原因。这里总结一下。

## 相同点

`Parcelable`和`Serializable`都是用于数据传输（多用于应用内传输），特别是在`Android`组件之间传输时，非常常用。

### 不同点

#### 1. API不同
`Serializable`是`Java API`，而`Parcelable`是`Android API`，所以通常`Serializable`更通用些

#### 2. 目的不同

`Serializable`其实是进行`Java`对象序列化的，可以持久化，甚至在不同应用中传输，而`Parcelable`是`Android`为了解决对象传输效率的问题开发的，用于组件之间传输数据。

#### 3. 效率不同

`Serializable`使用的是反射机制，在序列化过程中会产生很多冗余对象，触发`GC`。

`Parcelable`则是将对象中所有的内容分解成可支持、可传递的基础属性，而且这些属性完全保存在内存中，效率很快。

#### 4. Parcelable的缺点

1. 不能持久化
2. 实现较为复杂

## 一句话总结

`Serializable`是利用反射进行对象序列化，开发简单但开销大效率低

`Parcelable`是将对象分解成基础属性，在内存中处理，高效但开发较为复杂