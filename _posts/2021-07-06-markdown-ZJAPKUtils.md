---
layout:     post
title:      "ZJAPKUtils"
subtitle:   "1. apk文件是否安装 ; 2. 调起apk文件的安装; 3. 获取apk文件的包名"
date:       2021-07-06 16:39:16
author:     "zhoujie"
catalog: true # 是否开启目录
tags:
    - ZJUtils
---

[github地址](https://github.com/summer-zhoujie/ZJUtils)

# 1. 简介

public class ZJAPKUtils
extends java.lang.Object
apk文件工具

1. apk文件是否安装
2. 调起apk文件的安装
3. 获取apk文件的包名

# 2. apk文件是否安装
```java
public static boolean isApkFileInstalled(android.content.Context context,
                                         java.lang.String path)
```
`@param` path - apk文件路径<br>
`@param` context - 用于获取PackageManager<br>
`@return` true:已安装<br>

# 3. 调起安装APK文件
```java
public static void installApkFile(android.content.Context context,
                                  java.lang.String path)
```
`@param` path - apk文件路径

需要在Manifest中集成以下内容
```xml
<provider
 android:name="androidx.core.content.FileProvider"
 android:authorities="com.zj.tools.mylibrary.fileprovider"
 android:exported="false"
 android:grantUriPermissions="true">
     <meta-data
     android:name="android.support.FILE_PROVIDER_PATHS"
     android:resource="@xml/pg_file_path"
     tools:replace="android:resource" />
</provider>
```
再创建xml文件, 内容如下
```xml
 <?xml version="1.0" encoding="utf-8"?>
 <paths>
 <!--  APKUtils  这里填入自己APK文件放置的路径即可-->
     <external-files-path
         name="external_files_path"
         path="Download" />
 </paths>
```

# 4. 获取安装文件包名
`@param` path apk filepath <br>
`@return` packagename