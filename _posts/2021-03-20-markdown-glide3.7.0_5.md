---
layout:     post
title:      "Android Glide 3.7.0 源码解析(五) , 如何获得ImageView的宽高"
subtitle:   ""
date:       2021-03-20 15:19:42
author:     "zhoujie"
catalog: true # 是否开启目录
tags:
    - glide3.7.0
    - 源码
---

# 前言
通过前面的 [Android Glide 3.7.0 源码解析 (二) , 从一次图片加载流程看源码](https://blog.csdn.net/qq_25778369/article/details/114577763) 我们知道
Request(真实) 只有在图片组件的大小准备好了才会开始真正的加载
```java
// GenericRequest
	public void begin() {
        startTime = LogTime.getLogTime();
        if (model == null) {
            onException(null);
            return;
        }

        status = Status.WAITING_FOR_SIZE;
        if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
        	// 指定了特定的 size
            onSizeReady(overrideWidth, overrideHeight);
        } else {
        	// 未指定, 则去获取 size , 得到后调用 onSizeReady()...
            target.getSize(this);
        }

        if (!isComplete() && !isFailed() && canNotifyStatusChanged()) {
            target.onLoadStarted(getPlaceholderDrawable());
        }
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            logV("finished run method in " + LogTime.getElapsedMillis(startTime));
        }
    }
```
我们参考 [Android Glide 3.7.0 源码解析 (二) , 从一次图片加载流程看源码](https://blog.csdn.net/qq_25778369/article/details/114577763) 一文中介绍的流程, 可得 target 是一个 GlideDrawableImageViewTarget 类型, 下面就来看看是如何获取 size 的

# 如何获取 size

```java
// GlideDrawableImageViewTarget

	private final SizeDeterminer sizeDeterminer;
	@Override
    public void getSize(SizeReadyCallback cb) {
        sizeDeterminer.getSize(cb);
    }

// SizeDeterminer
	public void getSize(SizeReadyCallback cb) {
			// 计算当前View宽高的代码
            int currentWidth = getViewWidthOrParam();
            int currentHeight = getViewHeightOrParam();
            // 直接开始计算View的宽高, 如果>0或者是WRAP_CONTENT(-2)就代表有效
            if (isSizeValid(currentWidth) && isSizeValid(currentHeight)) {
            	// View 已经加载完, 能算出宽高了
                cb.onSizeReady(currentWidth, currentHeight);
            } else {
                if (!cbs.contains(cb)) {
                    cbs.add(cb);
                }
                if (layoutListener == null) {
                	// 给View的刷新注册监听, 希望View加载完, SizeDeterminerLayoutListener 能收到通知事件
                    final ViewTreeObserver observer = view.getViewTreeObserver();
                    layoutListener = new SizeDeterminerLayoutListener(this);
                    observer.addOnPreDrawListener(layoutListener);
                }
            }
        }

	private boolean isSizeValid(int size) {
            return size > 0 || size == LayoutParams.WRAP_CONTENT;
        }
```
> 关于 isSizeValid >0 有效可以理解, **为啥  size == LayoutParams.WRAP_CONTENT 也会被判定有效???**
> 其实在分析完后, 发现这条判断条件好像也用不上

需要弄清楚两个问题:
1. **getViewWidthOrParam()** 和 **getViewHeightOrParam()** 方法是如何计算宽高的
2. **SizeDeterminerLayoutListener** 监听 View 的变化之后, 又做了些啥

## 计算Size的基本函数
我们来看看如何计算 **当前状态下(可能没加载完全)** 的View的宽高

```java
// SizeDeterminer
	private static final int PENDING_SIZE = 0;
	private int getViewHeightOrParam() {
            final LayoutParams layoutParams = view.getLayoutParams();
            if (isSizeValid(view.getHeight())) {
            	// 直接 getHeight 如果 >0 的话
                return view.getHeight();
            } else if (layoutParams != null) {
            	// 其他情况
                return getSizeForParam(layoutParams.height, true /*isHeight*/);
            } else {
            	// 默认返回计算失败 0
                return PENDING_SIZE;
            }
        }

        private int getViewWidthOrParam() {
            final LayoutParams layoutParams = view.getLayoutParams();
            if (isSizeValid(view.getWidth())) {
            	// 直接 getWidth 如果 >0 的话
                return view.getWidth();
            } else if (layoutParams != null) {
            	// 其他情况
                return getSizeForParam(layoutParams.width, false /*isHeight*/);
            } else {
            	// 默认返回计算失败 0
                return PENDING_SIZE;
            }
        }

		// WRAP_CONTETNT 的情况计算
        private int getSizeForParam(int param, boolean isHeight) {
            if (param == LayoutParams.WRAP_CONTENT) {
            	// 计算 WRAP_CONTETNT 的核心代码
                Point displayDimens = getDisplayDimens();
                return isHeight ? displayDimens.y : displayDimens.x;
            } else {
                return param;
            }
        }

	private Point getDisplayDimens() {
            if (displayDimens != null) {
                return displayDimens;
            }
            WindowManager windowManager = (WindowManager) view.getContext().getSystemService(Context.WINDOW_SERVICE);
            // 很简单, 就是获取 View 的展示区域
            Display display = windowManager.getDefaultDisplay();
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.HONEYCOMB_MR2) {
                displayDimens = new Point();
                display.getSize(displayDimens);
            } else {
                displayDimens = new Point(display.getWidth(), display.getHeight());
            }
            return displayDimens;
        }
```
至此, 我们看完了**计算 View 大小的函数** ( getViewWidthOrParam() / getViewHeightOrParam() ) , 原理很简单
1. getHeight / getWidth 能计算出来 >0 的值, 就以它为准
2. LayoutParams.WRAP_CONTENT 或者 LayoutParams.MATCH_PARENT 时, **windowManager.getDefaultDisplay()** 来获得 View 的显示区域

简单画了一个流程图
![windowManager.getDefaultDisplay()](https://img-blog.csdnimg.cn/20210320150020953.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5,size_16,color_FFFFFF,t_70)
3. 一种是返回 0 或者 MATCH_PARENT , 表示我没计算出来
4. 一种是返回 getHeight / getWidth 或者 windowManager.getDefaultDisplay() , 表示我计算出来了

分析完基本的计算宽高的函数, 我们再来看看, **SizeDeterminerLayoutListener** 监听 View 刷新之后都做了啥

## 监听 View 刷新

```java
// SizeDeterminerLayoutListener
	private static class SizeDeterminerLayoutListener implements ViewTreeObserver.OnPreDrawListener {
            private final WeakReference<SizeDeterminer> sizeDeterminerRef;

            public SizeDeterminerLayoutListener(SizeDeterminer sizeDeterminer) {
                sizeDeterminerRef = new WeakReference<SizeDeterminer>(sizeDeterminer);
            }

            @Override
            public boolean onPreDraw() {
                // View 刷新的回调
                SizeDeterminer sizeDeterminer = sizeDeterminerRef.get();
                if (sizeDeterminer != null) {
                	// SizeDeterminer 去检查当前 View 的宽高是否有效
                    sizeDeterminer.checkCurrentDimens();
                }
                return true;
            }
        }

// SizeDeterminer
	private void checkCurrentDimens() {
            if (cbs.isEmpty()) {
                return;
            }

			// 重新再计算宽高
            int currentWidth = getViewWidthOrParam();
            int currentHeight = getViewHeightOrParam();
            if (!isSizeValid(currentWidth) || !isSizeValid(currentHeight)) {
                return;
            }

			// 宽高有效, 通知 GenericRequest
            notifyCbs(currentWidth, currentHeight);

            // 移除对 View 刷新的监听
            ViewTreeObserver observer = view.getViewTreeObserver();
            if (observer.isAlive()) {
                observer.removeOnPreDrawListener(layoutListener);
            }
            layoutListener = null;
        }

	private void notifyCbs(int width, int height) {
            for (SizeReadyCallback cb : cbs) {
            	// 这里的 cb 就是 GenericRequest 实例了
                cb.onSizeReady(width, height);
            }
            cbs.clear();
        }
```
逻辑很简单
1. View 刷新了
2. 再次计算宽高, 有效 -> GenericRequest.onSizeReady
() 开始加载图片; 移除刷新监听
3. 再次计算宽高, 无效 -> 啥也不做, 继续傻等

# 小结
说了那么多, 总结一下流程
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210320151209911.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5,size_16,color_FFFFFF,t_70)
