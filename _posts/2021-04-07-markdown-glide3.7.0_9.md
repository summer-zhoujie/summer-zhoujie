---
layout:     post
title:      "Android Glide 3.7.0 源码解析(九) ,  gif 的加载实现"
subtitle:   ""
date:       2021-04-07 19:27:10
author:     "zhoujie"
catalog: true # 是否开启目录
tags:
    - glide3.7.0
    - 源码
---


# 一、涉及类目
GlideDrawableImageViewTarget.java
GifDrawable.java
GifFrameLoader.java
GifDecoder.java

# 二、原理概述

老规矩先介绍原理的框架,免得看源代码迷路

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210405203102128.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5,size_16,color_FFFFFF,t_70)

* `GlideDrawableImageViewTarget` 会调用加载的 `GifDrawable` 来启动动画
* `GifDrawable` 会在 `draw()` 中绘制当前帧, 并委托 `GifFrameLoader` 去加载下一帧
* `GifFrameLoader` 依赖 `GifDecoder` 加载完成下一帧通知 `GifDrawable` 刷新视图

GifDrawable 其实是重写的 Drawable, 通过其 invalidateSelf() 通知界面重绘自己, 且在 draw() 方法中完成重绘, 还需要管理 loop, 用以控制结束循环
GifFrameLoader 负责控制加载每一帧的时间间隔, 还负责管理加载位置
GifDecoder 负责加载 gif 的帧

# 三、源码细节

先把下面这两步的代码看了

**`GlideDrawableImageViewTarget` 调用加载的 `GifDrawable` 来启动动画**
**`GifDrawable` 会在 `draw()` 中绘制当前帧, 并委托 `GifFrameLoader` 去加载下一帧**

```java
// GlideDrawableImageViewTarget

    public void onResourceReady(GlideDrawable resource, GlideAnimation<? super GlideDrawable> animation) {
        if (!resource.isAnimated()) {
            float viewRatio = view.getWidth() / (float) view.getHeight();
            float drawableRatio = resource.getIntrinsicWidth() / (float) resource.getIntrinsicHeight();
            if (Math.abs(viewRatio - 1f) <= SQUARE_RATIO_MARGIN
                    && Math.abs(drawableRatio - 1f) <= SQUARE_RATIO_MARGIN) {
                resource = new SquaringDrawable(resource, view.getWidth());
            }
        }
        super.onResourceReady(resource, animation);
        this.resource = resource;
        // 这里调用了 GifDrawable 的 start 方法
        resource.setLoopCount(maxLoopCount);
        resource.start();
    }


// GlideDrawable

    public void start() {
        // 状态置换跳过不看
        isStarted = true;
        resetLoopCount();
        if (isVisible) {
            // 真正的使能代码
            startRunning();
        }
    }

    private void startRunning() {
        // gif 只有 1 帧, 开始即结束
        if (decoder.getFrameCount() == 1) {
            // 通知界面重绘自己, 结束了
            invalidateSelf();
        }

        // 不只 1 帧
        else if (!isRunning) {
            isRunning = true;
            // frameLoader 开始工作啦
            frameLoader.start();
            // 通知界面重绘自己, 就是把当前帧给先画出来
            invalidateSelf();
        }
    }
```

至此, 我们触发了 `frameLoader.start()` , 并且界面上目前也因为 `invalidateSelf()` 而绘制上了第一帧
再来看看第三步: 循环加载帧, 并渲染到界面上

**`GifFrameLoader` 依赖 `GifDecoder` 加载完成下一帧通知 `GifDrawable` 刷新视图**

```java
// GifFrameLoader

    public void start() {
        if (isRunning) {
            return;
        }
        isRunning = true;
        isCleared = false;

        // 上面都是些状态信息, 跳过不看, 这个函数看名字就知道跑去加载下一帧了
        loadNextFrame();
    }

    private void loadNextFrame() {
        if (!isRunning || isLoadPending) {
            return;
        }
        isLoadPending = true;

        // 这行是移动 gifDecoder 的解析位置, 跳到下一帧的位置
        gifDecoder.advance();
        // 获取下一帧的延时时间(gif 每帧之间都有个时间间隔)
        long targetTime = SystemClock.uptimeMillis() + gifDecoder.getNextDelay();
        DelayTarget next = new DelayTarget(handler, gifDecoder.getCurrentFrameIndex(), targetTime);
        // 开始异步加载, 即不在主线程执行加载程序
        requestBuilder
                .signature(new FrameSignature())
                .into(next);
    }
```

直接触发 `loadNextFrame()` 去加载下一帧, 真正的代码则是
* `gifDecoder.advance()` 可以粗略理解成跳到下一帧头部位置
* `gifDecoder.getNextDelay()` 获得下一帧的间隔时间
* `requestBuilder.into()` 异步解析下一帧, 解析完成会回调 `DelayTarget`

下面看看 `DelayTarget` 里面的回调

```java
// DelayTarget

    public void onResourceReady(Bitmap resource, GlideAnimation<? super Bitmap> glideAnimation) {
            this.resource = resource;
            // 解析完成了, resource就存储着下一帧的图
            Message msg = handler.obtainMessage(FrameLoaderCallback.MSG_DELAY, this);
            // 这里 MSG_DELAY 明显是处理帧的时间间隔, 现在要异步切回主线程处理刷新问题了
            handler.sendMessageAtTime(msg, targetTime);
    }

    private class FrameLoaderCallback implements Handler.Callback {
        public static final int MSG_DELAY = 1;
        public static final int MSG_CLEAR = 2;

        @Override
        public boolean handleMessage(Message msg) {
            if (msg.what == MSG_DELAY) {
                GifFrameLoader.DelayTarget target = (DelayTarget) msg.obj;
                // 继续追踪
                onFrameReady(target);
                return true;
            } else if (msg.what == MSG_CLEAR) {
                GifFrameLoader.DelayTarget target = (DelayTarget) msg.obj;
                Glide.clear(target);
            }
            return false;
        }
    }


// GifFrameLoader

    void onFrameReady(DelayTarget delayTarget) {
        if (isCleared) {
            handler.obtainMessage(FrameLoaderCallback.MSG_CLEAR, delayTarget).sendToTarget();
            return;
        }

        DelayTarget previous = current;
        current = delayTarget;
        // callback 是一个 GlideDrawable, 告诉它我帮你把下一帧加载出来了, 下面来看看 GlideDrawable 是如何做的
        callback.onFrameReady(delayTarget.index);

        if (previous != null) {
            handler.obtainMessage(FrameLoaderCallback.MSG_CLEAR, previous).sendToTarget();
        }

        isLoadPending = false;
        loadNextFrame();
    }
```

* `DelayTarget.onResourceReady()` 加载下一帧完成了;
* `handler.sendMessageAtTime` 切回主线程, 顺便还把 帧的时间间隔问题解决了
* `callback.onFrameReady()` 通知 callback 也就是 GlideDrawable 加载好了

下面来看看 GlideDrawable 在被通知加载好了之后做了些啥

```java
// GlideDrawable

    public void onFrameReady(int frameIndex) {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.HONEYCOMB && getCallback() == null) {
            stop();
            reset();
            return;
        }

        // 刷新自己, 触发 draw() 方法
        invalidateSelf();

        // 如果满一个来回, 则loop 循环数量加一
        if (frameIndex == decoder.getFrameCount() - 1) {
            loopCount++;
        }

        // 循环次数够了,跳出循环
        if (maxLoopCount != LOOP_FOREVER && loopCount >= maxLoopCount) {
            stop();
        }
    }

    public void draw(Canvas canvas) {
        if (isRecycled) {
            return;
        }

        if (applyGravity) {
            Gravity.apply(GifState.GRAVITY, getIntrinsicWidth(), getIntrinsicHeight(), getBounds(), destRect);
            applyGravity = false;
        }

        // 从 frameLoader 拿出当前帧(也就是之前委托它加载的下一帧)
        Bitmap currentFrame = frameLoader.getCurrentFrame();
        Bitmap toDraw = currentFrame != null ? currentFrame : state.firstFrame;
        // 直接往画布上绘制
        canvas.drawBitmap(toDraw, null, destRect, paint);
    }
```

* 下一帧加载好了之后, GlideDrawable 触发自己的 `draw()` 方法开始绘制
* `frameLoader.getCurrentFrame()` 取出下一帧
* `canvas.drawBitmap()` 直接往界面上绘制

至此, glide gif的加载实现已讲解完毕! 感谢观看