---
layout: post
title: "使用FileProvider解决file:// URI引起的FileUriExposedException"
published: true
categories: Android
tags: [file, FileUriExposedException, FileProvider]
comments: true
---

## 问题
以下是一段简单的代码，它调用系统的相机app来拍摄照片：

``` java
void takePhoto(String cameraPhotoPath) {
    File cameraPhoto = new File(cameraPhotoPath);
    Intent takePhotoIntent = new Intent(MediaStore.ACTION_IMAGE_CAPTURE);
    takePhotoIntent.putExtra(MediaStore.EXTRA_OUTPUT, Uri.fromFile(cameraPhoto));
    startActivityForResult(takePhotoIntent, REQUEST_TAKE_PHOTO);
}
```

在一般情况下，运行没有任何问题；可是当把targetSdkVersion指定成24及之上并且在API>=24的设备上运行时，会抛出异常：

```
android.os.FileUriExposedException: 		file:///storage/emulated/0/DCIM/IMG_20170125_144112.jpg exposed beyond app through ClipData.Item.getUri()
    at android.os.StrictMode.onFileUriExposed(StrictMode.java:1799)
    at android.net.Uri.checkFileUriExposed(Uri.java:2346)
    at android.content.ClipData.prepareToLeaveProcess(ClipData.java:832)
    at android.content.Intent.prepareToLeaveProcess(Intent.java:8909)
    ...
```

<!--more-->

---

## 原因
我们来看一下`FileUriExposedException`的[文档](https://developer.android.com/reference/android/os/FileUriExposedException.html)：

> The exception that is thrown when an application exposes a `file://` Uri to another app.
>
> This exposure is discouraged since the receiving app may not have access to the shared path. For example, the receiving app may not have requested the `READ_EXTERNAL_STORAGE` runtime permission, or the platform may be sharing the Uri across user profile boundaries.
>
> Instead, apps should use `content://` Uris so the platform can extend temporary permission for the receiving app to access the resource.
>
> This is only thrown for applications targeting N or higher. Applications targeting earlier SDK versions are allowed to share `file://` Uri, but it's strongly discouraged.

总而言之，就是Android不再允许在app中把`file://`Uri暴露给其他app，包括但不局限于通过Intent或ClipData 等方法。

原因在于使用`file://`Uri会有一些风险，比如：

* 文件是私有的，接收`file://`Uri的app无法访问该文件。
* 在Android6.0之后引入运行时权限，如果接收`file://`Uri的app没有申请READ_EXTERNAL_STORAGE权限，在读取文件时会引发崩溃。

因此，google提供了`FileProvider`，使用它可以生成`content://`Uri来替代`file://`Uri。

---

## 解决方案
先上解决方案，感兴趣的同学可以看下一节`FileProvider`的具体讲解。

首先在`AndroidManifest.xml`中添加provider

``` xml
<provider
    android:name="android.support.v4.content.FileProvider"
    android:authorities="${applicationId}.fileprovider"
    android:exported="false"
    android:grantUriPermissions="true">
    <meta-data
        android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/provider_paths" />
</provider>
```

res/xml/provider_paths.xml

``` xml
<?xml version="1.0" encoding="utf-8"?>
<paths xmlns:android="http://schemas.android.com/apk/res/android">
    <external-path name="external_files" path="."/>
</paths>
```

然后修改代码

``` java
void takePhoto(String cameraPhotoPath) {
    File cameraPhoto = new File(cameraPhotoPath);
    Intent takePhotoIntent = new Intent(MediaStore.ACTION_IMAGE_CAPTURE);
    Uri photoUri = FileProvider.getUriForFile(
                this,
                getPackageName() + ".provider",
                cameraPhoto);
    takePhotoIntent.putExtra(MediaStore.EXTRA_OUTPUT, photoUri);
    startActivityForResult(takePhotoIntent, REQUEST_TAKE_PHOTO);
}
```

---

## FileProvider
使用`content://`Uri的优点：

* 它可以控制共享文件的读写权限，只要调用`Intent.setFlags()`就可以设置对方app对共享文件的访问权限，并且该权限在对方app退出后自动失效。相比之下，使用`file://`Uri时只能通过修改文件系统的权限来实现访问控制，这样的话访问控制是它对_所有_ app都生效的，不能区分app。
* 它可以隐藏共享文件的真实路径。

### 定义FileProvider
在`AndroidManifest.xml`的`<application>`节点中添加`<provider>`

``` xml
<provider
    android:name="android.support.v4.content.FileProvider"
    android:authorities="${applicationId}.fileprovider"
    android:exported="false"
    android:grantUriPermissions="true">
    <meta-data
        android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/provider_paths" />
</provider>
```

* `android:authorities`是用来标识provider的唯一标识，在同一部手机上一个"authority"串只能被一个app使用，冲突的话会导致app无法安装。我们可以利用[manifest placeholders](https://developer.android.com/studio/build/manifest-build-variables.html)来保证authority的唯一性。
* `android:exported`必须设置成`false`，否则运行时会报错`java.lang.SecurityException: Provider must not be exported`。
* `android:grantUriPermissions`用来控制共享文件的访问权限，也可以在java代码中设置。

### 指定路径和转换规则
FileProvider会隐藏共享文件的真实路径，将它转换成`content://`Uri路径，因此，我们还需要设定转换的规则。`android:resource="@xml/provider_paths"`这个属性指定了规则所在的文件。

res/xml/provider_paths.xml：

``` xml
<paths xmlns:android="http://schemas.android.com/apk/res/android">
    <files-path  path="." name="root" />
    <files-path  path="images/" name="my_images" />
    <files-path  path="audios/" name="my_audios" />
    ...
</paths>
```

`<paths>`中可以定义以下子节点

子节点 | 对应路径
- | -
files-path | Context.getFilesDir()
cache-path | Context.getCacheDir()
external-path | Environment.getExternalStorageDirectory()
external-files-path | Context.getExternalFilesDir(null)
external-cache-path | Context.getExternalCacheDir()

<br/>
`file://`到`content://`的转换规则：

1. 替换前缀：把`file://`替换成`content://${android:authorities}`。
2. 匹配和替换
      1. 遍历<paths>的子节点，找到最大能匹配上文件路径前缀的那个子节点。
      2. 用path的值替换掉文件路径里所匹配的内容。
3. 文件路径剩余的部分保持不变。


需要注意的是，文件的路径必须包含在xml中，也就是2.1中必须能找到一个匹配的子节点，否则会抛出异常：

``` 
java.lang.IllegalArgumentException: Failed to find configured root that contains /data/data/com.xxx/cache/test.txt
    at android.support.v4.content.FileProvider$SimplePathStrategy.getUriForFile(FileProvider.java:679)
    at android.support.v4.content.FileProvider.getUriForFile(FileProvider.java:378)
    ...
```

### 代码中生成Content Uri

``` java
File imagePath = new File(Context.getFilesDir(), "images");
File newFile = new File(imagePath, "2016/pic.png");
Uri contentUri = getUriForFile(getContext(), getPackageName() + ".fileprovider", newFile);
```

### 设置文件的访问权限
有两种设置权限的办法：

* 调用`Context.grantUriPermission(package, uri, modeFlags)`。这样设置的权限只有在手动调用`Context.revokeUriPermission(uri, modeFlags)`或系统重启后才会失效。
* 调用`Intent.setFlags()`来设置权限。权限失效的时机：接收`Intent`的`Activity`所在的stack销毁时。

### 题外话：ContentProvider的一个小技巧
许多Android SDK在使用前往往需要使用者调用初始化代码：`SomeSdk.init(Context)`，其实借助`ContentProvider`就可以把这一步省略掉。

原理是在SDK的`AndroidManifest`里注册一个`ContentProvider`，在它的`OnCreate`中进行SDK的初始化工作。`ContentProvider`会在app进程被创建时创建，并且在它的`onCreate`里可以拿到Context的引用。

具体实现可以参照[How does Firebase initialize on Android?](https://firebase.googleblog.com/2016/12/how-does-firebase-initialize-on-android.html)

---

参考文档

* [FileProvider](https://developer.android.com/reference/android/support/v4/content/FileProvider.html)
* [file:// scheme is now not allowed to be attached with Intent on targetSdkVersion 24 (Android Nougat). And here is the solution.](https://inthecheesefactory.com/blog/how-to-share-access-to-file-with-fileprovider-on-android-nougat/en)
* [How does Firebase initialize on Android?](https://firebase.googleblog.com/2016/12/how-does-firebase-initialize-on-android.html)
