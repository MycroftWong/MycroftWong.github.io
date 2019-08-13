---
title: Android包优化个人总结
date: 2019-08-13 14:05:17
categories: 
- Android
    - 优化
tags:
- Android
- 优化
- apk
---

# Android包优化个人总结

## 前言

打包`apk`对`app`运行本身会有很多用处，减少代码量，减少资源量，减少资源名等，都可以减少`apk`的大小。如果`apk`足够大，会发现，`release`包比`debug`包启动都会快很多。

所以优化非常有必要，同时为了安全，还需要数据加密和`apk`加固

## 优化项

### 基本配置

```
release {
    // 删除无用资源
    shrinkResources true
    // 开启混淆
    minifyEnabled true
    // apk对齐
    zipAlignEnabled true
    // 指定混淆规则文件
    proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
    // 只使用中文资源，xhdpi图片资源
    resConfigs "zh", "xhdpi"
    // 配置主dex的规则，将首页和第二页需要的代码放在主dex中有利于提升app启动效率
    multiDexKeepProguard file("maindexlist.pro")
   // 配置签名
    signingConfig signingConfigs.release
    // 关闭所有debug
    jniDebuggable false
    renderscriptDebuggable false
    debuggable false
}
```

### 一、混淆

一些通用的配置
```
-optimizationpasses 5
-dontusemixedcaseclassnames
-dontskipnonpubliclibraryclasses
-dontskipnonpubliclibraryclassmembers
-dontpreverify
-verbose
-printmapping proguardMapping.txt
-optimizations !code/simplification/cast,!field/*,!class/merging/*
-keepattributes *Annotation*,InnerClasses
-keepattributes Signature
-keepattributes SourceFile,LineNumberTable
```

#### 1. 通用混淆配置

1. 不混淆`Android`组件类，`View`类等需要在配置中使用的类

```
-keep public class * extends android.app.Activity
-keep public class * extends android.app.Application
-keep public class * extends android.app.Service
-keep public class * extends android.content.BroadcastReceiver
-keep public class * extends android.content.ContentProvider
-keep public class * extends android.app.backup.BackupAgentHelper
-keep public class * extends android.preference.Preference
-keep public class com.android.vending.licensing.ILicensingService
```
2. 不混淆通用的如`Serializable`, `Parcelable`子类

```
-keepclassmembers class * implements android.os.Parcelable {
    static ** CREATOR;
}
-keepclassmembers class * implements android.os.Parcelable {
    public static final android.os.Parcelable$Creator CREATOR;
}

-keepnames class * implements java.io.Serializable
-keepclassmembers class * implements java.io.Serializable {
    static final long serialVersionUID;
    private static final java.io.ObjectStreamField[] serialPersistentFields;
    !static !transient <fields>;
    !private <fields>;
    !private <methods>;
    private void writeObject(java.io.ObjectOutputStream);
    private void readObject(java.io.ObjectInputStream);
    java.lang.Object writeReplace();
    java.lang.Object readResolve();
}
```
3. 枚举类

```
-keepclassmembers enum * {
    public static **[] values();
    public static ** valueOf(java.lang.String);
}
```

4. 含有`native`方法的类

```
-keepclasseswithmembernames,includedescriptorclasses class * {
    native <methods>;
}
-keepclasseswithmembernames class * {
    native <methods>;
}
```

5. `androidx.annotation.Keep`标注的类、方法、域

```
# AndroidX
-keep class androidx.annotation.Keep
-keep @androidx.annotation.Keep class * {*;}
-keepclasseswithmembers class * {
    @androidx.annotation.Keep <methods>;
}
-keepclasseswithmembers class * {
    @androidx.annotation.Keep <fields>;
}
-keepclasseswithmembers class * {
    @androidx.annotation.Keep <init>(...);
}

# 非AndroidX
-keep class android.support.annotation.Keep
-keep @android.support.annotation.Keep class * {*;}
-keepclasseswithmembers class * {
    @android.support.annotation.Keep <methods>;
}
-keepclasseswithmembers class * {
    @android.support.annotation.Keep <fields>;
}
-keepclasseswithmembers class * {
    @android.support.annotation.Keep <init>(...);
}
```

6. `R`类中的所有域（后面可以使用`AndResGuard`进行处理）

```
-keep class **.R$* {
    public static <fields>;
}
-keep class **.R
```

#### 2. 库相关的类

引用的库中需要保留一些代码，通常只需要去相对应的库文档中查找即可，下面举一些常用的库

1. [Glide](https://github.com/bumptech/glide)

```
-keep public class * implements com.bumptech.glide.module.GlideModule
-keep public class * extends com.bumptech.glide.module.AppGlideModule
-keep public enum com.bumptech.glide.load.ImageHeaderParser$** {
  **[] $VALUES;
  public *;
}

# for DexGuard only
-keepresourcexmlelements manifest/application/meta-data@value=GlideModule
```

2. [EventBus](https://github.com/greenrobot/EventBus)

```
-keepclassmembers class * {
    @org.greenrobot.eventbus.Subscribe <methods>;
}
-keep enum org.greenrobot.eventbus.ThreadMode { *; }
 
# Only required if you use AsyncExecutor
-keepclassmembers class * extends org.greenrobot.eventbus.util.ThrowableFailureEvent {
    <init>(java.lang.Throwable);
}
```

3. [Okio](https://github.com/square/okio)

```
# Animal Sniffer compileOnly dependency to ensure APIs are compatible with older versions of Java.
-dontwarn org.codehaus.mojo.animal_sniffer.*
```

4. [Okhttp](https://github.com/square/okhttp)

```
# JSR 305 annotations are for embedding nullability information.
-dontwarn javax.annotation.**

# A resource is loaded with a relative path so the package of this class must be preserved.
-keepnames class okhttp3.internal.publicsuffix.PublicSuffixDatabase

# Animal Sniffer compileOnly dependency to ensure APIs are compatible with older versions of Java.
-dontwarn org.codehaus.mojo.animal_sniffer.*

# OkHttp platform used only on JVM and when Conscrypt dependency is available.
-dontwarn okhttp3.internal.platform.ConscryptPlatform
```

5. [Retrofit](https://github.com/square/retrofit)

除了下面官方的配置之外，因为在`json`转`model`时使用了反射，所以不能混淆`model`类、域

```
# Retrofit does reflection on generic parameters. InnerClasses is required to use Signature and
# EnclosingMethod is required to use InnerClasses.
-keepattributes Signature, InnerClasses, EnclosingMethod

# Retrofit does reflection on method and parameter annotations.
-keepattributes RuntimeVisibleAnnotations, RuntimeVisibleParameterAnnotations

# Retain service method parameters when optimizing.
-keepclassmembers,allowshrinking,allowobfuscation interface * {
    @retrofit2.http.* <methods>;
}

# Ignore annotation used for build tooling.
-dontwarn org.codehaus.mojo.animal_sniffer.IgnoreJRERequirement

# Ignore JSR 305 annotations for embedding nullability information.
-dontwarn javax.annotation.**

# Guarded by a NoClassDefFoundError try/catch and only used when on the classpath.
-dontwarn kotlin.Unit

# Top-level functions that can only be used by Kotlin.
-dontwarn retrofit2.KotlinExtensions
-dontwarn retrofit2.KotlinExtensions$*

# With R8 full mode, it sees no subtypes of Retrofit interfaces since they are created with a Proxy
# and replaces all potential values with null. Explicitly keeping the interfaces prevents this.
-if interface * { @retrofit2.http.* <methods>; }
-keep,allowobfuscation interface <1>
```

### 二、减少资源名
> AndResGuard是一个帮助你缩小APK大小的工具，他的原理类似Java Proguard，但是只针对资源。他会将原本冗长的资源路径变短，例如将res/drawable/wechat变为r/d/a。

当项目逐渐增大，包里面的资源越来越多，名字也越来越长（名字清晰当然易于阅读），`AndResGuard`的作用就越大，当使用`AndResGuard`后，会发现`apk`明显的缩短，并且某种程度上，进行了混淆。

官方地址：[AndResGuard](https://github.com/shwenzhang/AndResGuard)

#### 1. 配置
在项目`build.gradle`中添加插件
```gradle
buildscript {
    dependencies {
        classpath 'com.tencent.mm:AndResGuard-gradle-plugin:1.2.17'
    }
}
```
在主模块`build.gradle`中添加，其中注意一定要将需要保留的资源文件名加入白名单，另外如果使用了如友盟、融云等库时，因为在这些库中使用资源文件名获取资源，所以一定要添加白名单，不然会出现资源找不到的问题。（我更喜欢用`LeanCloud`）
```gradle
andResGuard {
    // mappingFile = file("./resource_mapping.txt")
    mappingFile = null
    use7zip = true
    useSign = true
    // 打开这个开关，会keep住所有资源的原始路径，只混淆资源的名字
    keepRoot = false
    // 设置这个值，会把arsc name列混淆成相同的名字，减少string常量池的大小
    fixedResName = "arg"
    // 打开这个开关会合并所有哈希值相同的资源，但请不要过度依赖这个功能去除去冗余资源
    mergeDuplicatedRes = true
    whiteList = [
        // for your icon
        "R.drawable.icon",
        // for fabric
        "R.string.com.crashlytics.*",
        // for google-services
        "R.string.google_app_id",
        "R.string.gcm_defaultSenderId",
        "R.string.default_web_client_id",
        "R.string.ga_trackingId",
        "R.string.firebase_database_url",
        "R.string.google_api_key",
        "R.string.google_crash_reporting_api_key"
    ]
    compressFilePattern = [
        "*.png",
        "*.jpg",
        "*.jpeg",
        "*.gif",
    ]
    sevenzip {
         artifact = 'com.tencent.mm:SevenZip:1.2.17'
         //path = "/usr/local/bin/7za"
    }

    /**
    * 可选： 如果不设置则会默认覆盖assemble输出的apk
    **/
    // finalApkBackupPath = "${project.rootDir}/final.apk"

    /**
    * 可选: 指定v1签名时生成jar文件的摘要算法
    * 默认值为“SHA-1”
    **/
    // digestalg = "SHA-256"
}
```

#### 2. 启动
> 使用Android Studio的同学可以再 andresguard 下找到相关的构建任务; 命令行可直接运行./gradlew resguard[BuildType | Flavor]， 这里的任务命令规则和assemble一致。


**详细使用查看官方地址[AndResGuard](https://github.com/shwenzhang/AndResGuard)**

### 三、删除无用资源

虽然可以配置在打包时删除无用资源，但是`Android Studio`本身就提供找出未使用资源的方法

流程：`Analyze -> Run Inspection by Name -> 输入Unused resources -> 选择需要查找的范围 -> OK`

等待几分钟，就可以发现目前没有使用到的资源，根据自身情况删除，如果存在相互引用，可以多运行几次

如下图：

![run inspection by name](android_studio_run_insepection.png)

![unused resources](android_studio_unused_resources.png)

![start unused resources](android_studio_start_unused_resources.png)

### 四、代码审查

这篇文章不讨论代码相关的事情，使用代码审查可以找出代码中的问题，可能引用了一些没有使用到的资源，同时建议开启阿里`Java`代码指导插件`Alibaba Java Coding Guidelines`，代码的质量就是我们程序员的脸面，当别人看你代码时，即使功能不强大，但是代码一定要漂亮。

使用方法：`Analyze -> Inspect Code`，运行结束可以在下方看到`Android Studio`帮忙找出的问题，虽然它没有那么智能，有些东西也不需要修改，但是我们可以做修改参考意见。

另外，还可以多看看`Analyze`下的一些功能，可以帮助提升代码质量。一经检查，发现修改根本停不下来...

