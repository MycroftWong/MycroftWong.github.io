---
title: FileProvider
date: 2019-09-05 14:42:41
categories:
- Android
tags:
- Android
- FileProvider
---

# FileProvider

下面着重通过官方文档介绍`FileProvider`的使用：[FileProvider](https://developer.android.google.cn/reference/android/support/v4/content/FileProvider)

## 翻译

这里就不在引用原文了，下面是翻译的内容

`FileProvider`是`ContentProvider`的一个特殊子类，为了加强在应用之间安全的分享文件。通过创建`content://`的`uri`替换`file:///`的`uri`。

`content://`的`uri`允许授予临时的读写权限。当我们创建一个`Intent`，其中包含一个`content uri`，为了能够让对方的应用能够访问到这个`content uri`，可以使用`Intent.setFlags()`添加权限。如果收到`content uri`的是`Activity`，那么这些权限一直会存在，只要收到`content uri`的`Activity`栈处于活动状态。如果收到`content uri`的是`Service`，那么`Service`一直运行，权限就会一直在。

相对于`file:///`的`uri`，你想控制文件的控制权限，那么你必须修改系统的底层文件权限。你提供的这些权限将对所有的应用有效，一直保留到你更改他们。这种权限控制根本上是不安全的。

提供`content uri`增加了文件访问权限等级，使得`FileProvider`成为`Android`安全基础框架非常重要的部分。

关于`FileProvider`，通过下面5点介绍：

1. 定义`FileProvider`
2. 指定可用的文件
3. 获得一个文件的`content uri`
4. 为一个`uri`赋予临时权限
5. 提供`content uri`给其他应用

### 1. 定义`FileProvider`
因为`FileProvider`的默认功能就是为文件提供`content uri`，你不需要定义`FileProvider`的子类。而是，你在`manifest`中包含一个`FileProvider`。为了指定`FileProvider`组件，在`manifest`添加一个`provider`元素。这是`android:name`属性为`android.support.v4.content.FileProvider`（`androidx`为`androidx.core.content.FileProvider`）。设置`android:authorities`为`content uri`的域名。例如你的域名是`wang.mycroft`，你应该设置`authority`为`wang.mycroft.fileprovider`。设置`android:exported`属性为`false`，`FileProvider`不需要设置为公开的。设置`android:grantUriPermissions`属性为`true`，为了允许赋予文件的临时访问权限。如下：

```xml
<manifest>
    ...
    <application>
        ...
        <provider
            android:name="androidx.core.content.FileProvider"
            android:authorities="wang.mycroft.fileprovider"
            android:exported="false"
            android:grantUriPermissions="true">
            ...
        </provider>
        ...
    </application>
</manifest>
```

如果你想重写`FileProvider`方法的默认行为，那么继承`FileProvider`类，在`provider`中指定`android:name`为其的全路径类名。

### 2. 指定可用的文件

一个`FileProvider`只能为预先指定的文件夹下的文件提供`content uri`。为了指定一个文件夹，在`xml`中指定文件的存储路径，在`paths`下添加子属性。例如，下列的`path`元素告诉`FileProvider`你想要把私有文件区域下的`image/`子目录提供`content uri`。

```xml
<paths xmlns:android="http://schemas.android.com/apk/res/android">
    <files-path name="my_images" path="images/"/>
    ...
</paths>
```

`<paths>`元素必须包含一个或多个子元素：

```xml
<files-path name="name" path="path" />
```

代表`app`内部存储区域的`files/`子目录下的文件。此子目录的值和`Context.getFilesDir()`的返回值相同。

```xml
<cache-path name="name" path="path" />
```

代表`app`内部存储区域的缓存子目录。此子目录的值和`Context.getCacheDir()`的返回值相同。

```xml
<external-path name="name" path="path" />
```

代表外部存储区域的根目录。此子目录的值和`Environment.getExternalStorageDirectory()`的返回值相同。

```xml
<external-files-path name="name" path="path" />
```

