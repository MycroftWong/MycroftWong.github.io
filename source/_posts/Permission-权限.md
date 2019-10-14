---
title: Permission 权限
date: 2019-10-14 09:30:30
categories:
- Android
    - 基础
tags:
- Permission
- 权限
- ContentProvider
---

# Permission 权限

`Android`引入权限的目的是保护用户隐私。`APP`一定要在访问用户敏感数据（如短信服务）和系统功能（如相机、网络）时申请权限。对于不同的功能，系统可能会自动通过权限申请，也可能提示用户批准权限。

安卓系统安全架构的中心点是禁止任何`APP`默认拥有对其他`APP`、系统或是用户拥有不利影响操作的权限。

## 申请权限

`APP`功能需要的权限一定要在`manifest`中声明。如下`APP`需要发送短信功能的权限时：

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
          package="com.example.snazzyapp">

    <uses-permission android:name="android.permission.SEND_SMS"/>

    <application ...>
        ...
    </application>
</manifest>
```

1. 如果在`manifest`中申请的权限是**普通权限**，那么系统将自动同意
2. 如果在`manifest`中申请的权限是**危险权限**，系统将提示用户同意`APP`的权限申请

关于**普通权限**和**危险权限**的区别，查看后面的**保护等级**。

### 危险权限的申请提示

根据`Android`版本的不同和`APP`的目标版本的不同，`Android`提示用户批准危险权限的方式也不同。

#### 运行时申请 `Android 6.0`及以上

`Android`版本在`6.0`及以上，并且`APP`目标版本在23及以上时，在安装时，系统不会提醒用户批准权限（荣耀手机会在安装完成时，显示一个用户可以操作权限的界面），用户需要在运行时申请权限。如下图左，系统将弹出`Dialog`供用户操作。若用户已经拒绝过一次，`APP`再次申请该权限是，`Dialog`则会显示“不再提醒”的勾选框，以后`APP`申请权限时，将不在提示用户申请，自动拒绝，如下图右（现在很多系统会在第一次申请时就显示“不再提醒”的勾选框）：

![申请运行时权限](runtime_permission_request.png)

另外，及时用户通过了权限，但是也可以在之后在系统设置中关闭权限，所以一定要在运行时每次需要就申请。

#### 安装时申请

`Android`版本在`5.1.1`及以下或是`APP`目标版本在22及以下时，系统将在安装时询问用户批准所有的危险权限。若用户同意，那么系统会批准`APP`声明的所有权限，若用户不同意，`APP`将无法安装。（某些定制系统会允许安装，但是需要特殊的方法在运行时申请权限）。

### 获取用户敏感信息的申请提示

一些`APP`可能需要获取依赖于短信的用户敏感信息。如果想申请这些权限，一定要在运行时申请权限之前，提示用户在系统设置中，修改“默认应用”，如默认的短信服务、电话服务。

## 可选的硬件功能的权限

访问硬件功能时需要申请应用权限。然而，不是所有的安卓设备有特定的硬件功能呢。所以，加入你申请相机权限，那么在`manifest`中一定要使用`<uses-feature>`标记声明需要的硬件功能。如下：

```xml
<uses-feature android:name="android.hardware.camera" android:required="false" />
```

如果硬件功能的需求设置为`true`，那么如果硬件上没有该功能，那么系统将不允许安装该`APP`，如果为`false`，那么最好在运行时需要改功能前，使用`PackageManager.hasSystemFeature()`判断设备是否有改功能。

## 权限的强制执行

权限不仅仅用于申请系统的功能，`APP`的`Service`也可以强制执行想使用它的自定义权限。关于如果自定义权限，可查看**自定义的`APP`权限**。

### Activity权限的强制执行

可以在`<activity>`标记中添加`android:permission`属性，限制谁可以启动该`Activity`。在`Context.startActivity()`和`Activity.startActivityForResult()`会检查权限，如果调用者没有该权限，调用时将抛出`SecurityException`。

### Service权限的强制执行

和`Activity`一样，可以在`<service>`设置需要的权限。将在`Context.startService()`、`Context.stopService()`和`Context.bindService()`中检查权限。

### Broadcast权限的强制执行

和`Activity`一样，可以在`<receiver>`设置需要的权限。系统将在`Context.sendBroadcast()`返回后检查权限，若通过，系统将发送，若未通过，系统不会抛出异常，但也不会发送该`Broadcast`。

动态注册的`BroadcastReceiver`也可以在注册时的代码中设置权限。

**重点**：发送者和接收者都需要申请权限。

#### 发送时的权限

当发送广播`Broadcast`时，可以指定权限参数。只有拥有该权限的接收者能够收到该广播。如下

```kotlin
sendBroadcast(Intent("com.example.NOTIFY"), Manifest.permission.SEND_SMS)
```

接收者一定要获取该权限：

```xml
<uses-permission android:name="android.permission.SEND_SMS"/>
```

不仅仅可以使用`Android`自带的权限，也可以使用自定义的权限。

#### 接收时的权限

当在`<receiver>`（接收者）声明时指定权限（危险的权限），那么想要接收者收到该通知，那么一定要在`manifest`中使用`<uses-permission>`声明接收者所需要的权限。如下：

```kotlin
<receiver android:name=".MyBroadcastReceiver"
          android:permission="android.permission.SEND_SMS">
    <intent-filter>
        <action android:name="android.intent.action.AIRPLANE_MODE"/>
    </intent-filter>
</receiver>
```

动态注册类似

#### BroadcastReceiver权限总结

1. 在发送时指定权限，指的是接收者必须拥有该权限，并且需要在发送者的`manifest`中定义该权限。
2. 接收者指定权限，指的是发送者必须拥有并携带权限，并且需要在接收者的`manifest`中定义该权限。

#### Android 8.0 权限限制

为了避免像`IM SDK`这种进行相互启动的机制。从`Android 8.0`开始，对广播进行了极大限制，自定义的`ACTION`不再能够使用。所以，当我在荣耀手机（`Android 9.0`）上测试时，使用了自定义的`ACTION`发送广播，接收者始终没有收到。解决方案是使用动态注册。不过这样就实现不了很多功能了。

### ContentProvider权限的强制执行

在`<provider>`上添加`android:permission`属性声明想要操作`ContentProvider`数据的权限。`ContentProvder`拥有额外的一个重要的安全特性，被称为`URI`权限，后面将说明。不同于其他的组件，`ContentProvider`有两个单独的权限属性可以设置`android:readPermission`和`android:writePermission`。注意，拥有写权限，并不意味着拥有读权限，这两者是独立的。

当首次检索一个`ContentProvider`，进行对应操作时将进行权限检查，若未通过，将抛出`SecurityException`异常。使用`ContentResolver.query()`时，需要读权限，使用`ContentResolver.insert()`、`ContentResolver.update()`、`ContentResolver.delete()`时，需要写权限。

### URI 权限

标准的权限系统对`ContentProvider`的功能有限。`ContentProvider`获取想保护它自己的读写权限，并且使其提供的`URI`只被单独需要的`APP`使用。

典型的例子是邮箱应用。应该通过权限机制保护用户的隐私数据。如想要展示邮箱应用中的图片时，提供一个`URI`给图片预览`APP`，那么当不再预览时，应该收回该权限，因为没有任何理由再让图片预览`APP`获取其操作了。

解决方法是为每个`URI`设置单独的权限，如当启动一个`Activity`时，设置`Intent.FLAG_GRANT_READ_URI_PERMISSION`和/或`Intent.FLAG_GRANT_WRITE_URI_PERMISSION`。这样保证接收的`Activity`能够获取`Intent`中指定的`URI`数据，而不管是否操作`ContentProvider`的权限。

关于`URI`权限的使用，可以参考{% post_link FileProvider %}

### 其他的权限强制执行

参考：[权限概览](https://developer.android.google.cn/guide/topics/permissions/overview)

## 权限级别

权限被分为3个级别：
1. 普通权限：对于用户隐私或是设备操作没有危险的权限，系统将在安装时自动通过该权限
2. 签名权限：系统将在安装时通过的权限，但是只有相同签名的`APP`才能够获取。注意，一些签名权限并不用于第三方`APP`
3. 危险权限：可能会影响到用户隐私或是设备的普通操作

## 特殊权限

某些权限并不是与普通权限和危险权限表现相同。`SYSTEM_ALERT_WINDOW`和`WRITE_SETTINGS`两个权限相当的敏感，大部分`APP`不应该使用它们。如果`APP`需要使用，那么需要在`manifest`中声明，并且发送`Intent`请求用户验证。如下：

```kotlin
if (PackageManager.PERMISSION_GRANTED != ContextCompat.checkSelfPermission(
        this, Manifest.permission.WRITE_SETTINGS
    )
) {
    startActivityForResult(Intent(Settings.ACTION_MANAGE_WRITE_SETTINGS).apply {
        data = Uri.parse("package:$packageName")
    }, 2)
}

override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
    super.onActivityResult(requestCode, resultCode, data)
    when (requestCode) {
        2 -> if (Settings.System.canWrite(this)) {
            Toast.makeText(this, "获取到了WRITE_SETTINGS权限", Toast.LENGTH_LONG).show()
        } else {
            Toast.makeText(this, "没有获取到了WRITE_SETTINGS权限", Toast.LENGTH_LONG).show()
        }
    }
}
```

## 权限组

权限根据设备特性或功能被分为不同的组。在这种系统下，权限请求被组织为组级别处理。并且单个清单组对应多个权限声明。例如，`SMS`权限组包含`READ_SMS`和`RECEIVE_SMS`权限。这样对权限进行分组，让用户做出更有意义的选择，而不是被复杂的权限请求弄得不知所措。

所有危险的权限都属于权限组。无论保护级别如何，任何权限都可以属于权限组。 但是，如果权限很危险，则权限的组只会影响用户体验。

当设备运行的是`Android 6.0`及以上，并且`APP`目标版本是23及以上时，当请求一个危险权限时，系统的表现：当我们请求权限时，系统检查`APP`是否已经获得同组下的其他权限，若没有，将使用系统`Dialog`提示用户。若`APP`已经又有了同组下的危险权限，那么系统将直接通过新申请的权限，不必再提示用户。

## 请求应用权限

非常简单，直接可以参考[请求应用权限](https://developer.android.google.cn/training/permissions/requesting)

## 定义自定义权限

自定义权限非常简单，重要的是如何使用权限。如下所示：

```xml
<permission
    android:name="com.example.myapp.permission.DEADLY_ACTIVITY"
    android:label="@string/permlab_deadlyActivity"
    android:description="@string/permdesc_deadlyActivity"
    android:permissionGroup="android.permission-group.COST_MONEY"
    android:protectionLevel="dangerous" />
```

## 总结

理解设计权限的目的，就非常容易理解权限的使用。自定义权限较少，大多数时候我们是在使用权限。当然自定义权限也可以使用到，不过这通常是在`Service`中。另外注意`Android 8.0`开始，`ContentProvider`的静态声明受到了极大限制，对于推送功能的实现，目前都需要根据厂商的渠道。