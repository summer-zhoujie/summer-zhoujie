---
layout:     post
title:      "Android Glide 3.7.0 源码解析(七) , 细说图形变换和解码"
subtitle:   ""
date:       2021-03-31 17:49:42
author:     "zhoujie"
catalog: true # 是否开启目录
tags:
    - glide3.7.0
    - 源码
---

# 一、概览
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210330135711769.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5,size_16,color_FFFFFF,t_70)

Glide 3.7.0 里面涉及的解码/图形变换主要是 **大小缩放**, **CenterCrop** , **FitCenter**, 其中大小缩放是基于**Downsampler(解码工具)** 实现的, 而剩余两个则是 **Transformation(图形变换)** 接口的两个实现类.
所以本文主要介绍3点:
* **CenterCrop**
* **FitCenter**
* **大小缩放**

> [Android Glide 3.7.0 源码解析 (二) , 从一次图片加载流程看源码](/2021/03/14/markdown-glide3.7.0_2/index.html)文中的流程可以看出是在 DecodeJob 里面进行 解码 --> 图形变换 的
> ![在这里插入图片描述](https://img-blog.csdnimg.cn/20210330135823576.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5,size_16,color_FFFFFF,t_70)
> 关于 fitCenter 和 centerCrop 的理解可以参考这篇文章的描述 [Android ImageView 的scaleType 属性图解](https://www.jianshu.com/p/32e335d5b842)

按照惯例先介绍原理框架, 免得看源码时候迷路
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210331101008287.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5,size_16,color_FFFFFF,t_70)
解码流程详述
* 读取图片的配置, width, height, config, orientation,
* 根据传入的目标 targetWidth  和 targetHeight , 计算出来目标采样率, 就是`缩放比例`
* 根据缩放比例开始解析原始图片流, 解析出缩放尺寸的图片
* 根据方向 ( `orientation` ) 信息对图片进行矩阵变换, 翻转/旋转图片

>解码的过程会伴随着大量对象池思想的使用, 关于对象池概念,参看[Android Glide 3.7.0 源码解析(四) , BitmapPool作用及原理](/2021/03/20/markdown-glide3.7.0_4/index.html)

图形转换流程
* 根据目标宽高计算出来合适的缩放比例和偏移量
* 然后通过矩阵变换实现图形变换

# 二、解码
还记得[Android Glide 3.7.0 源码解析 (二) , 从一次图片加载流程看源码](/2021/03/14/markdown-glide3.7.0_2/index.html)文中提到过 `DownSampler 这个类是将原始图片资源流解析成图片`, 我们的解码过程就是在这个类中进行的

```java
// Downsampler
	public Bitmap decode(InputStream is, BitmapPool pool, int outWidth, int outHeight, DecodeFormat decodeFormat) {
        final ByteArrayPool byteArrayPool = ByteArrayPool.get();
        final byte[] bytesForOptions = byteArrayPool.getBytes();
        final byte[] bytesForStream = byteArrayPool.getBytes();
        final BitmapFactory.Options options = getDefaultOptions();

        // Use to fix the mark limit to avoid allocating buffers that fit entire images.
        RecyclableBufferedInputStream bufferedStream = new RecyclableBufferedInputStream(is, bytesForStream);
        // Use to retrieve exceptions thrown while reading.
        // TODO(#126): when the framework no longer returns partially decoded Bitmaps or provides a way to determine
        // if a Bitmap is partially decoded, consider removing.
        ExceptionCatchingInputStream exceptionStream = ExceptionCatchingInputStream.obtain(bufferedStream);
        // Use to read data.
        // Ensures that we can always reset after reading an image header so that we can still attempt to decode the
        // full image even when the header decode fails and/or overflows our read buffer. See #283.
        MarkEnforcingInputStream invalidatingStream = new MarkEnforcingInputStream(exceptionStream);
        try {
            exceptionStream.mark(MARK_POSITION);
            int orientation = 0;
            try {
                orientation = new ImageHeaderParser(exceptionStream).getOrientation();
            } catch (IOException e) {
                if (Log.isLoggable(TAG, Log.WARN)) {
                    Log.w(TAG, "Cannot determine the image orientation from header", e);
                }
            } finally {
                try {
                    exceptionStream.reset();
                } catch (IOException e) {
                    if (Log.isLoggable(TAG, Log.WARN)) {
                        Log.w(TAG, "Cannot reset the input stream", e);
                    }
                }
            }

            options.inTempStorage = bytesForOptions;

            final int[] inDimens = getDimensions(invalidatingStream, bufferedStream, options);
            final int inWidth = inDimens[0];
            final int inHeight = inDimens[1];

            final int degreesToRotate = TransformationUtils.getExifOrientationDegrees(orientation);
            final int sampleSize = getRoundedSampleSize(degreesToRotate, inWidth, inHeight, outWidth, outHeight);

            final Bitmap downsampled = downsampleWithSize(invalidatingStream, bufferedStream, options,
            											  pool, inWidth, inHeight, sampleSize, decodeFormat);

            // BitmapFactory swallows exceptions during decodes and in some cases when inBitmap is non null, may catch
            // and log a stack trace but still return a non null bitmap. To avoid displaying partially decoded bitmaps,
            // we catch exceptions reading from the stream in our ExceptionCatchingInputStream and throw them here.
            final Exception streamException = exceptionStream.getException();
            if (streamException != null) {
                throw new RuntimeException(streamException);
            }

            Bitmap rotated = null;
            if (downsampled != null) {
                rotated = TransformationUtils.rotateImageExif(downsampled, pool, orientation);

                if (!downsampled.equals(rotated) && !pool.put(downsampled)) {
                    downsampled.recycle();
                }
            }

            return rotated;
        } finally {
            byteArrayPool.releaseBytes(bytesForOptions);
            byteArrayPool.releaseBytes(bytesForStream);
            exceptionStream.release();
            releaseOptions(options);
        }
    }
```
这个函数还蛮长的, 一个片段一个片段看

**对象池狂魔**

* `ByteArrayPool`典型的对象池的实现, bytesForOptions 赋值给了**options.inTempStorage**, [inTempStorage](https://developer.android.com/reference/kotlin/android/graphics/BitmapFactory.Options?hl=en#intempstorage)官方给的解释是解码的时候会用到的缓存, 这里用对象池管理回收,防止内存抖动, 看到此处回收的代码了吗 `byteArrayPool.releaseBytes(bytesForOptions)` ;
* 同理 bytesForStream 也被安排了, **RecyclableBufferedInputStream** 看名称就很容易能猜到了, 我们知道在解析流的时候, 如果要求这个流可以回溯读取(读过的内容再读取一遍), 一般需要一个Buffer来缓存从流中读出的数据, 而这里就把这个 Buffer 抽象出来交给 `ByteArrayPool` 管理了
* 再来看 `ExceptionCatchingInputStream exceptionStream = ExceptionCatchingInputStream.obtain(bufferedStream);` 这行代码, 是不是联想到 Message.obtain(), 没错这里也是个典型的对象池的概念, `exceptionStream.release();` 在此处回收进入对象池, 这里就不详述了,感兴趣可以自行跟进 **ExceptionCatchingInputStream** 看看
* 最后, 再来看 `final BitmapFactory.Options options = getDefaultOptions();` 和 `releaseOptions(options);` 这一组, 也是一个对象池实现
* 还没有结束 `downsampleWithSize(invalidatingStream, bufferedStream, options, pool, inWidth, inHeight, sampleSize, decodeFormat);` 这行里面的 **pool** , 就是一个 BitmapPool , 是 Bitmap 的对象池

> Android 3.0 之后可以将流中的图像数据解码在一个不用的已创建的 Bitmap 实例里面, 具体参见 [Android Bitmap(一), 资源重用](/2021/03/30/markdown-bitmap_1/index.html)

## 读取图片配置

```java
// DownSampler.decode

        RecyclableBufferedInputStream bufferedStream = new RecyclableBufferedInputStream(is, bytesForStream);
        ExceptionCatchingInputStream exceptionStream = ExceptionCatchingInputStream.obtain(bufferedStream);
        MarkEnforcingInputStream invalidatingStream = new MarkEnforcingInputStream(exceptionStream);

```

先来三层 InputStream 包装 (设计模式: 装饰者模式),
* **第一层 RecyclableBufferedInputStream** 实现流的回溯功能(mark/reset), 暴露 Buffer 方便接入外面的对象池管理;
* **第二层 ExceptionCatchingInputStream**  担任异常处理功能;
* **第三层 MarkEnforcingInputStream** 这一层是为了防止读取图片头部属性等数据的时候读超了 mark 标记的位数, 之后就无法 reset 了
> InputStream mark / reset 方法的解释
> ![在这里插入图片描述](https://img-blog.csdnimg.cn/2021033115434729.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5,size_16,color_FFFFFF,t_70)
> mark(int limit) 的作用是标记一段长度为limit的流, 使它可以被重新读取, 而 reset() 就是将当前的读取位置指向之前 mark() 的位置, 但当超限( 例如: readPos_2位置 )时就无法 reset() 了
> * 在**读取位置是 readPos_0** 时 , mark(int limit) 标记当前读取流的位置
> * 在**读取位置是 readPos_1** 时 , 调用 reset 回溯有效( readPos_1 <= limitPos ), 没有超出 limit 的限制, 会回到 markPos 再读一遍流
> * 在**读取位置是 readPos_2** 时, 调用 reset  无效 (readPos_1 > limitPos ), 超出 limit 限制
>
> 关于 RecyclableBufferedInputStream 如何实现 mark 和 reset 方法的, 参考 [Android Glide 3.7.0 源码解析(八) , RecyclableBufferedInputStream 的 mark/reset 实现](/2021/04/01/markdown-glide3.7.0_8/index.html)

```java
// DownSampler.decode

	try {
            exceptionStream.mark(MARK_POSITION);
            int orientation = 0;
            try {
                orientation = new ImageHeaderParser(exceptionStream).getOrientation();
            } catch (IOException e) {
                if (Log.isLoggable(TAG, Log.WARN)) {
                    Log.w(TAG, "Cannot determine the image orientation from header", e);
                }
            } finally {
                try {
                    exceptionStream.reset();
                } catch (IOException e) {
                    if (Log.isLoggable(TAG, Log.WARN)) {
                        Log.w(TAG, "Cannot reset the input stream", e);
                    }
                }
            }
            ...
     }
```
 紧接着就用到了 mark / reset 功能 , 读取头部信息里面存储的**方向信息**

> 关于 orientation 值代表的详细含义参考这篇文章:  [EXIF 方向参数 Orientation](https://my.oschina.net/u/4283333/blog/3423645)

```java
// DownSampler.decode

	final int[] inDimens = getDimensions(invalidatingStream, bufferedStream, options);
    final int inWidth = inDimens[0];
    final int inHeight = inDimens[1];

	public int[] getDimensions(MarkEnforcingInputStream is, RecyclableBufferedInputStream bufferedStream,
            BitmapFactory.Options options) {
        options.inJustDecodeBounds = true;
        decodeStream(is, bufferedStream, options);
        options.inJustDecodeBounds = false;
        return new int[] { options.outWidth, options.outHeight };
    }
```
这里获取了待解析图片的宽高

## 计算缩放比例

```java
// DownSampler.decode

	// 计算图片被旋转的角度
	final int degreesToRotate = TransformationUtils.getExifOrientationDegrees(orientation);
    final int sampleSize = getRoundedSampleSize(degreesToRotate, inWidth, inHeight, outWidth, outHeight);

    private int getRoundedSampleSize(int degreesToRotate, int inWidth, int inHeight, int outWidth, int outHeight) {
        int targetHeight = outHeight == Target.SIZE_ORIGINAL ? inHeight : outHeight;
        int targetWidth = outWidth == Target.SIZE_ORIGINAL ? inWidth : outWidth;

        final int exactSampleSize;
        if (degreesToRotate == 90 || degreesToRotate == 270) {
            // 90 和 270 度 需要把长宽对调来计算缩放比例
            exactSampleSize = getSampleSize(inHeight, inWidth, targetWidth, targetHeight);
        } else {
            exactSampleSize = getSampleSize(inWidth, inHeight, targetWidth, targetHeight);
        }

        // 去一个最大的 且  <= exactSampleSize 且 是2的次方
        final int powerOfTwoSampleSize = exactSampleSize == 0 ? 0 :
                Integer.highestOneBit(exactSampleSize);

        // powerOfTwoSampleSize == 0 代表不缩放,也就是返回 1 倍
        return Math.max(1, powerOfTwoSampleSize);
    }

    public static final Downsampler AT_LEAST = new Downsampler() {
        @Override
        protected int getSampleSize(int inWidth, int inHeight, int outWidth, int outHeight) {
        	// 按照目标的长宽比判定 至少需要缩放多少倍
            return Math.min(inHeight / outHeight, inWidth / outWidth);
        }

        @Override
        public String getId() {
            return "AT_LEAST.com.bumptech.glide.load.data.bitmap";
        }
    };

    public static final Downsampler AT_MOST = new Downsampler() {
        @Override
        protected int getSampleSize(int inWidth, int inHeight, int outWidth, int outHeight) {
        	// 按照目标的长宽比判定 至多需要缩放多少倍
            int maxIntegerFactor = (int) Math.ceil(Math.max(inHeight / (float) outHeight,
                inWidth / (float) outWidth));
            int lesserOrEqualSampleSize = Math.max(1, Integer.highestOneBit(maxIntegerFactor));
            return lesserOrEqualSampleSize << (lesserOrEqualSampleSize < maxIntegerFactor ? 1 : 0);
        }

        @Override
        public String getId() {
            return "AT_MOST.com.bumptech.glide.load.data.bitmap";
        }
    };
```
先根据目标宽高算出整数的缩放比例, 有两种计算方式(但其实查看 3.7.0的代码, 只用到了 AT_LEAST)
* AT_LEAST 取 sampleSize 的最小值, 意思是: 至少需要缩放多少倍
* AT_MOST 取 sampleSize 的最大值 (而且还是 ceil 的方式向上取整) 意思是最多需要缩放多少倍

这步计算我们拿到的 exactSampleSize 却不是最终的 sampleSize, [官文](https://developer.android.com/reference/android/graphics/BitmapFactory.Options#inSampleSize)里面有提到, sampleSize 需要是 2 的整数次方 且 大于一, 所以我们`需要在exactSampleSize 范围内找一个最大的满足 2 的整数次方`的最终 sampleSize , 并且与 1 进行比较

> sampleSize == 4 代表缩小 4 倍
> ![在这里插入图片描述](https://img-blog.csdnimg.cn/20210331163814297.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5,size_16,color_FFFFFF,t_70)

缩放比例就计算完了, 下一步

## 解析原图为对应缩放比例

```java
// DownSampler.decode

	final Bitmap downsampled = downsampleWithSize(invalidatingStream, bufferedStream,
												options, pool, inWidth, inHeight,
												sampleSize,decodeFormat);

	private Bitmap downsampleWithSize(MarkEnforcingInputStream is, RecyclableBufferedInputStream  bufferedStream,
            BitmapFactory.Options options, BitmapPool pool, int inWidth, int inHeight, int sampleSize,
            DecodeFormat decodeFormat) {

		// 读取 config
        Bitmap.Config config = getConfig(is, decodeFormat);
        // 初始化 options
        options.inSampleSize = sampleSize;
        options.inPreferredConfig = config;

		// 这里利用 BitmapPool 对象池 和 Bitmap 的重用机制, 做了一个Bitmap内存重用的东东
        if ((options.inSampleSize == 1 || Build.VERSION_CODES.KITKAT <= Build.VERSION.SDK_INT) && shouldUsePool(is)) {
            int targetWidth = (int) Math.ceil(inWidth / (double) sampleSize);
            int targetHeight = (int) Math.ceil(inHeight / (double) sampleSize);
            setInBitmap(options, pool.getDirty(targetWidth, targetHeight, config));
        }
        // 开始解析
        return decodeStream(is, bufferedStream, options);
    }

	private static void setInBitmap(BitmapFactory.Options options, Bitmap recycled) {
        if (Build.VERSION_CODES.HONEYCOMB <= Build.VERSION.SDK_INT) {
        	// 给 options 的 inBitmap 字段赋值, 可以将原始图片资源解析到一个不用的 Bitmap 对象中去
            options.inBitmap = recycled;
        }
	}

	private static Bitmap decodeStream(MarkEnforcingInputStream is, RecyclableBufferedInputStream bufferedStream,
            BitmapFactory.Options options) {
        ...
        final Bitmap result = BitmapFactory.decodeStream(is, null, options);
		...

        return result;
    }

```
* 读取 config
* 把 config 和之前算好的 sampleSize 赋值给 options
* 给 options 的 inBitmap 字段赋值, 可以将原始图片资源解析到一个不用的 Bitmap 对象中去
* 利用 Bitmap 的资源重用机制完成对原始图片的解码操作

> 关于Bitmap的重用机制可以参考, [Android Bitmap(一), 资源重用](/2021/03/30/markdown-bitmap_1/index.html)
> 关于对象池的概念可以参考, [Android Glide 3.7.0 源码解析(四) , BitmapPool作用及原理](/2021/03/20/markdown-glide3.7.0_4/index.html)

解码过程分析完毕, 接下来的图形转换就很简单了, 一共就两个函数

# 三、图形转换 fitCenter

```java
public class FitCenter extends BitmapTransformation {

    public FitCenter(Context context) {
        super(context);
    }

    public FitCenter(BitmapPool bitmapPool) {
        super(bitmapPool);
    }

    @Override
    protected Bitmap transform(BitmapPool pool, Bitmap toTransform, int outWidth, int outHeight) {
    	// 关键代码在这里
        return TransformationUtils.fitCenter(toTransform, pool, outWidth, outHeight);
    }

    @Override
    public String getId() {
        return "FitCenter.com.bumptech.glide.load.resource.bitmap";
    }
}

// TransformationUtils

	public static Bitmap fitCenter(Bitmap toFit, BitmapPool pool, int width, int height) {
        if (toFit.getWidth() == width && toFit.getHeight() == height) {
            if (Log.isLoggable(TAG, Log.VERBOSE)) {
                Log.v(TAG, "requested target size matches input, returning input");
            }
            return toFit;
        }

		// 计算缩放比例 2 代表放大 2 倍, 这里去最小值, 意思是保证能放的进去 ImageView 控件
        final float widthPercentage = width / (float) toFit.getWidth();
        final float heightPercentage = height / (float) toFit.getHeight();
        final float minPercentage = Math.min(widthPercentage, heightPercentage);


        final int targetWidth = (int) (minPercentage * toFit.getWidth());
        final int targetHeight = (int) (minPercentage * toFit.getHeight());

        if (toFit.getWidth() == targetWidth && toFit.getHeight() == targetHeight) {
            if (Log.isLoggable(TAG, Log.VERBOSE)) {
                Log.v(TAG, "adjusted target size matches input, returning input");
            }
            return toFit;
        }

		// 对象池获取一个旧的大小匹配的
        Bitmap.Config config = getSafeConfig(toFit);
        Bitmap toReuse = pool.get(targetWidth, targetHeight, config);
        if (toReuse == null) {
            toReuse = Bitmap.createBitmap(targetWidth, targetHeight, config);
        }
        // 设置透明属性, 官文描述在某些情况下可以提升Bitmap的绘制速度
        TransformationUtils.setAlpha(toFit, toReuse);

		// 矩阵变换控制缩放
        Canvas canvas = new Canvas(toReuse);
        Matrix matrix = new Matrix();
        matrix.setScale(minPercentage, minPercentage);
        Paint paint = new Paint(PAINT_FLAGS);
        canvas.drawBitmap(toFit, matrix, paint);

        return toReuse;
    }
```
以上源码,所见即所得,非常简单, 需要注意的是
* 缩放比例选取最小值, 是为的能放的进去界面组件, 因为是FitCenter
* TransformationUtils.setAlpha 设置是否包含透明像素的标志位, 某些情况下可以提升绘制速度,查看官文描述如下

>![在这里插入图片描述](https://img-blog.csdnimg.cn/20210331172045203.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5,size_16,color_FFFFFF,t_70)

# 四、图形转换 centerCrop

```java
// CenterCrop

	protected Bitmap transform(BitmapPool pool, Bitmap toTransform, int outWidth, int outHeight) {
        final Bitmap toReuse = pool.get(outWidth, outHeight, toTransform.getConfig() != null
                ? toTransform.getConfig() : Bitmap.Config.ARGB_8888);
        // 还是在 TransformationUtils 中进行处理
        Bitmap transformed = TransformationUtils.centerCrop(toReuse, toTransform, outWidth, outHeight);
        if (toReuse != null && toReuse != transformed && !pool.put(toReuse)) {
            toReuse.recycle();
        }
        return transformed;
    }

	public static Bitmap centerCrop(Bitmap recycled, Bitmap toCrop, int width, int height) {
        if (toCrop == null) {
            return null;
        } else if (toCrop.getWidth() == width && toCrop.getHeight() == height) {
            return toCrop;
        }

        final float scale;
        float dx = 0, dy = 0;
        Matrix m = new Matrix();
        // 这个算式改成除法比较好理解 toCrop.getWidth()/width > toCrop.getHeight()/height, 结合 CenterCrop 的属性理解
        if (toCrop.getWidth() * height > width * toCrop.getHeight()) {
        	// 宽度超限了,需要对宽度进行裁剪
        	// 按照高度比例进行缩放
            scale = (float) height / (float) toCrop.getHeight();
            dx = (width - toCrop.getWidth() * scale) * 0.5f;
        } else {
        	// 高度超限了,需要对高度进行裁剪
        	// 按照宽度比例进行缩放
            scale = (float) width / (float) toCrop.getWidth();
            dy = (height - toCrop.getHeight() * scale) * 0.5f;
        }

		// 先缩放
        m.setScale(scale, scale);
        // 再平移
        m.postTranslate((int) (dx + 0.5f), (int) (dy + 0.5f));


		// Bitmap重用机制
        final Bitmap result;
        if (recycled != null) {
            result = recycled;
        } else {
            result = Bitmap.createBitmap(width, height, getSafeConfig(toCrop));
        }

        // 同前文, 提速用的
        TransformationUtils.setAlpha(toCrop, result);

        Canvas canvas = new Canvas(result);
        Paint paint = new Paint(PAINT_FLAGS);
        // 绘制到 canvas 上
        canvas.drawBitmap(toCrop, m, paint);
        return result;
    }
```
代码比较简单,就是通过缩放和平移,摆好位置后,直接绘制到新的 Bitmap 上, 唯一需要注意的是`toCrop.getWidth()/width > toCrop.getHeight()/height`会选取一个比例小的按比例缩放, 把比例大的哪个平移裁剪掉


