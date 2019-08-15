---
title: StringBuilder、StringBuffer的线程安全到底是什么
date: 2019-08-15 11:16:44
categories:
- Java
    - 基础知识
tags:
- Java
- String
- StringBuilder
- StringBuffer
---

# StringBuilder、StringBuffer的线程安全到底是什么

## 前言

昨天看到这个面试题：`String`、`StringBuilder`、`StringBuffer`之间的区别是什么？

这个问题本身很简单：

1. `String`是不可变的，对于字符串的拼接，都需要重新构造`String`对象
2. `StringBuilder`和`StringBuffer`主要解决`String`拼接字符串的问题，他们不需要重新构造对象
3. `StringBuilder`和`StringBuffer`的区别在于`StringBuffer`是**线程安全的**

但是根据这个问题可以引申出一些其他问题

## String为什么不可变

查看源码，`String`、`StringBuilder`、`StringBuffer`内部都是使用`char`数组实现的。

在`String`中，这个`char`数组是用`final`进行修饰的，所以一旦创建并不能改变其值（非常规方法是可以的）。

### 为什么要设计String不可变

`Java`将`String`保留在一个特殊的区域：字符串常量池。当创建一个`String`时，若在池中已经存在此字符串，则不会再创建，而是直接引用，若不存在，则创建。这是一种缓存策略。

## 为什么StringBuilder、StringBuffer可变

`StringBuilder`和`StringBuffer`都继承自`AbstractStringBuilder`，其中的`char`数组并没有被`final`修饰。在`append`之类的方法中，进行了整个数组的复制，它们引用了新的`char`数组，原来的`char`数组相当于被抛弃了。

## 为什么StringBuffer是线程安全的

### 什么是线程安全

定义：多线程环境中，能永远保证程序的正确性

### `StringBuffer`的线程安全到底是什么意思

在`StringBuilder`和`StringBuffer`的`append`之类的方法中，执行了一系列的代码。

这个是`AbstractStringBuilder`源码

```java
public AbstractStringBuilder append(String str) {
    if (str == null)
        return appendNull();
    int len = str.length();
    ensureCapacityInternal(count + len);
    str.getChars(0, len, value, count);
    count += len;
    return this;
}
```

这一段代码大致意思是：计算新的数组的长度、将原来的数组内容复制到新的数组中，在新的数组中添加内容，增加长度，返回。

在多线程环境中，可能进行了多个`append`操作，导致在其中一个`append`的执行过程中，已经在执行其他的`append`操作。如：前一个`append`生成新的数组后，还未将内容添加到新的数组时，后面一个`append`又生成了新的数组，导致**在新的数组中添加内容**这一步，在同一个位置添加了两次，若添加的内容超过新的数组长度，会抛出`StringIndexOutOfBoundsException`。

在`StringBuilder`中，并没有对`append`进行处理，所以它仍然是按照`AbstractStringBuilder`的逻辑执行。

但是在`StringBuffer`中，重写了所有的`append`之类的操作，添加了`synchronized`关键字，保证了这一段代码的原子性，保证了线程安全。

### 测试代码

```java
import java.util.Locale;
import java.util.Random;
import java.util.concurrent.Executor;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

public class Main {

    private static final Random seedRandom = new Random(4L);

    public static void main(String[] args) throws Exception {
        StringBuilder builder = new StringBuilder();
        StringBuffer buffer = new StringBuffer();

        ExecutorService executor = Executors.newCachedThreadPool();
        for (int i = 0; i < 50; i++) {
            executor.execute(() -> {
                Random random = new Random(seedRandom.nextLong());
                builder.append(random.nextInt(10));
                buffer.append(random.nextInt(10));
            });
        }

        TimeUnit.SECONDS.sleep(1);
        System.out.println(builder);
        System.out.println(builder.length());

        System.out.println(buffer);
        System.out.println(buffer.length());

        executor.shutdown();
    }
}
```

多执行这段代码几次，`StringBuffer`得到的结果长度会总是`50`，而`StringBuilder`偶尔会少于`50`次，这就是线程不安全的情况下出现的错误。

## 我的疑问

这样的线程安全有什么用，说实话，没有用过`StringBuffer`，一般的需求都是在单线程进行字符串的拼接。

我的疑问是：多线程使用`StringBuffer`怎么能够保证拼接的顺序呢？

我在使用过程中，对字符串拼接对顺序是有要求的，并不希望因为线程原因导致错误。而且字符串拼接的线程安全性，通常可以使用其他方法来解决。

## 总结：线程安全 != 保证拼接顺序

`StringBuffer`的线程安全只是保证在一次`append`操作中，不会有其他`append`介入，避免了拼接的过程中出现异常。

若在多线程环境中，`StringBuffer`并不像队列一般，能够将拼接按特定的顺序执行。
