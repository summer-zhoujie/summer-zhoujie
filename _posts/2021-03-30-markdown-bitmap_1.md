---
layout:     post
title:      "Android Bitmap(一), 资源重用"
subtitle:   ""
date:       2021-03-30 17:48:40
author:     "zhoujie"
catalog: true # 是否开启目录
tags:
    - Bitmap
    - 内存优化
---

# 一、为什么Bitmap需要资源重用
Android 中图片显示的实体其实是一个 Bitmap 对象, 每次图片显示时, 都会构建一个 Bitmap 对象, 不用时再销毁, 假设, 在一个长列表且列表的每项都有一个图片显示, 持续滑动这个列表, 内存中的行为就是, 持续的创建 Bitmap 对象和产生不用的 Bitmap 对象, 当量级到达一定程度, 会触发 GC, 这样持续滑动界面, 势必会频繁触发 GC, 导致界面卡顿

# 二、Bitmap 内存管理的演变
> 以下内容参考官文: [管理位图内存](https://developer.android.com/topic/performance/graphics/manage-memory#inBitmap)

*  **Android Android 2.2（API 级别 8）及以下**，当发生垃圾回收时，应用的线程会停止。这会导致延迟，从而降低性能。
* **Android 2.3** 添加了并发垃圾回收功能，这意味着系统不再引用位图后，很快就会回收内存。
* **Android 2.3.3（API 级别 10）及以下**，位图的后备像素数据存储在本地内存 ( 不是在虚拟机中是在Native中, 可以简单理解为Android设备内存 ) 中。它与存储在 Dalvik 堆中的位图本身是分开的。本地内存中的像素数据并不以可预测的方式释放，可能会导致应用短暂超出其内存限制并崩溃。
*  **Android 3.0（API 级别 11）~ Android 7.1（API 级别 25）**，像素数据会与关联的位图一起存储在 Dalvik 堆上。
* **Android 8.0（API 级别 26）及以上**，位图像素数据存储在原生堆 ( 又存回了Native ) 中。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210330153900146.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5,size_16,color_FFFFFF,t_70)

官文在介绍 Bitmap 重用之前, 为啥介绍内存管理背景?
* 猜测和资源重用有关, 官方介绍Android 3.0 及以后 Bitmap 才支持重用, 在 2.3.3 以前, 只能在不用的时候调用 Bitmap 的 recycle() 方法, 联想到上面的背景, 可能是当时 Bitmap 存储在 Native 上对重用的实现带来了一些难题, 而 Android 3.0~7.1 Bitmap 存储在 Dalvik中时, 这时候可以利用 BitmapFactory.Options.inBitmap 字段实现资源重用. 至于 Android 8.0 以上还支持重用, 则是难题被攻克了, 以上为猜测, 个中原因有待证实

为啥 Android 8.0 以后, Bitmap 的存储又挪回 Native 了呢?
* 应该是借鉴的 iOS 的操作, iOS的一个APP几乎能用近所有的可用内存（除去系统开支), 8.0之后，Android也向这个方向靠拢， 我们都知道 8.0 及以上的机器的内存高达4~8G, 而 Dalvik 虚拟机才能分配到多少, 至多几百兆, 这样势必会造成资源的浪费, 假设一个 4G 的机器, Dalvik 的 heap 分配了 512M, 那剩下的好几个G都浪费了, 如何解决这个问题? 最好的下手对象就是Bitmap，因为它是耗内存大户。我们把 Bitmap 的存储全部挪到 Native(机器存储) 去, 而不是放在 Dalvik 虚拟机分配了可怜的 heap 大小

# 三、如何资源重用
前文可知, 分两种情况, **3.0以下** 和 **3.0及以上**

## 3.0以下
只能使用 **Bitmap 的 recycle()** 方法来释放 Bitmap 对象, 并且需要自己管理 Bitmap 的生命周期( 自己记 Bitmap 的引用计数 ), 很麻烦, 资源利用率也不高

## 3.0及以上
引入了 **BitmapFactory.Options.inBitmap 字段**, 来完成对 Bitmap 重用的支持
> 参考 [inBitmap官方文档](https://developer.android.com/reference/android/graphics/BitmapFactory.Options#inBitmap)
> ![在这里插入图片描述](https://img-blog.csdnimg.cn/20210330155442822.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5,size_16,color_FFFFFF,t_70)

可得, 区分 **4.4及以上** 和 **3.0及以上~4.4以下** 两种不同的处理方式

4.4 <= api 需要满足如下条件
* 被重用的 Bitmap 对象是 mutable 的
* 被重用的 Bitmap 对象的 size >= 当前准备解析的

3.0 <= api  < 4.4
* 被重用的 Bitmap 对象是 mutable 的
* 被重用的 Bitmap 对象的 width 和 height 需要和当前准备解析的严格匹配
* BitmapFactory.Options.inSampleSize == 1
* 被解析的图片需要是 jpeg 或者 png 格式



首先, 我们需要有如下2个 BitmapPool, Bitmap 的缓存池子,分别对应上面的两种情况, 不用的 Bitmap都缓存在这里, 并限制缓存的上限, 和规定淘汰算法LRU
>具体实现原理参照 [Android Glide 3.7.0 源码解析(四) , BitmapPool作用及原理](/2021/03/20/markdown-glide3.7.0_4/index.html) 一文

```java
public interface BitmapPoolSize {

	// 从池子里获取一个大于等于指定大小的, 且config匹配的 Bitmap 实例
	Bitmap get(int size, Bitmap.Config config);

	// 放置一个不使用的 Bitmap 到池子里
	boolean put(Bitmap bitmap);

}
```
```java
public interface BitmapPoolAttribute {

	// 从池子里获取一个宽高严格, 且config匹配的 Bitmap 实例
	Bitmap get(int width, int height, Bitmap.Config config);

	// 放置一个不使用的 Bitmap 到池子里
	boolean put(Bitmap bitmap);

}
```

下面是一个需要重用旧的 Bitmap 的代码示例

```java
public Bitmap decodeBitmapFromFile(String pathName) {

        // 获取待解析图片的配置信息
        final BitmapFactory.Options options = new BitmapFactory.Options();
        options.inJustDecodeBounds = true;
        BitmapFactory.decodeFile(pathName, options);
        options.inJustDecodeBounds = false;
        int targetWidth = options.outWidth;
        int targetHeight = options.outHeight;
        int size = targetWidth * targetHeight;

        Bitmap cache;
        // 适配第一种情况
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {
            options.inMutable = true;
            cache = BitmapPoolSize.get(size, options.outConfig);
            options.inBitmap = cache;
        }

		// 适配第二种情况
		else if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.HONEYCOMB) {
            options.inMutable = true;
            cache = BitmapPoolAttribute.get(targetWidth, targetHeight, options.outConfig);
            if (options.inSampleSize == 1 &&
                (options.outMimeType.equals("image/jpeg") || options.equals("image/png"))) {
                options.inBitmap = cache;
            }
        }

        // 数据都被写入缓存的 cache 对象里去了
        return BitmapFactory.decodeFile(pathName, options);
}
```
至此重用解释完毕

# 四、FAQ
**inJustDecodeBounds** 这个属性好像没提到
> ![在这里插入图片描述](https://img-blog.csdnimg.cn/20210330170712826.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5,size_16,color_FFFFFF,t_70)

直译过来就是设置了这个属性之后, 在 decode 时, 不会真正的去解析 Bitmap, 而是去给 BitmapFactory.Options 中 out 打头的变量赋值, 来看看都有哪些变量
* outWidth 宽度
* outHeight 高度
* outMimeType 图片类型
* outConfig 图片配置
* outColorSpace pixed 数组 ( byte[] ) 的像素排列方式说明

# 五、参考
* 官文_管理位图内存: [https://developer.android.com/topic/performance/graphics/manage-memory#inBitmap](https://developer.android.com/topic/performance/graphics/manage-memory#inBitmap)
* 官文_inBitmap: [https://developer.android.com/reference/android/graphics/BitmapFactory.Options#inBitmap](https://developer.android.com/reference/android/graphics/BitmapFactory.Options#inBitmap)
* Android Developers 论坛: [https://groups.google.com/g/android-developers/c/Mp0MFVFi1Fo/m/e8ZQ9FGdWdEJ?pli=1](https://groups.google.com/g/android-developers/c/Mp0MFVFi1Fo/m/e8ZQ9FGdWdEJ?pli=1)
* Android Bitmap变迁与原理解析（4.x-8.x）: [https://toutiao.io/posts/ptdi4q/preview](https://toutiao.io/posts/ptdi4q/preview)

