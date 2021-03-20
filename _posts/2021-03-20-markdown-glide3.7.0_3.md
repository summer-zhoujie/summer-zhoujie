---
layout:     post
title:      "Android Glide 3.7.0 源码解析(三), 生命周期绑定"
subtitle:   ""
date:       2021-03-20 15:16:58
author:     "zhoujie"
catalog: true # 是否开启目录
tags:
    - glide3.7.0
    - 源码
---

# 一、流程图解
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210314175314605.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5,size_16,color_FFFFFF,t_70)
> 注意:
> * 一个 Fragment / Activity 会对应生成一个 RequestManager
> * 一个 Application 对应一个 applicationManager , 这是一个全局唯一的 RequestManager
> * 每个 RequestManager 会有一个 Lifecycle 和 一个 RequestTracker
> * 每个 RequestTracker 有个 List< Request >
>![在这里插入图片描述](https://img-blog.csdnimg.cn/20210314182433256.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5,size_16,color_FFFFFF,t_70)


1. 如果是主线程就注册创建一个**无界面的 Fragment** 加到 Fragment / Activity , 依赖这个Fragment 来监听生命周期
2. 如果是非主线程 , 就创建一个 Application 级别的 Lifecycle , 模拟生命周期
3. 在 1. 中创建的 Fragment 可以反馈 **内存** 和 **界面** 的 生命周期 , 这就完成了对内存和界面的监听
4. 可以根据 1. 中 Fragment , 来决定是否监控 **网络状态** ( 如果界面都没了, 那自然也就没必要监控网络状态了 )

# 二、源码分析

## 生命周期创建
![在这里插入图片描述](https://img-blog.csdnimg.cn/2021031418002898.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5,size_16,color_FFFFFF,t_70)
Glide 的 5 个 with 方法最终都对应的是 RequestManagerRetriever 的 5 个 get()
![在这里插入图片描述](https://img-blog.csdnimg.cn/2021031418023553.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5,size_16,color_FFFFFF,t_70)
而 RequestManagerRetriever 的 5 个 get() 最终对应 RequestManagerRetriever 的三个方法
* **非主线程时** : get(activity.getApplicationContext())
* **主线程 + Activity 时** : fragmentGet(activity, activity.getFragmentManager())
* **主线程 + Fragment 时** : supportFragmentGet(fragment.getActivity(), fragment.getChildFragmentManager())

先来看看简单点的, **非主线时**
```java
// RequestManagerRetriever
	public RequestManager get(Context context) {
        if (context == null) {
            throw new IllegalArgumentException("You cannot start a load on a null Context");
        } else if (Util.isOnMainThread() && !(context instanceof Application)) {
            if (context instanceof FragmentActivity) {
                return get((FragmentActivity) context);
            } else if (context instanceof Activity) {
                return get((Activity) context);
            } else if (context instanceof ContextWrapper) {
                return get(((ContextWrapper) context).getBaseContext());
            }
        }

		// 最终走到这里
        return getApplicationManager(context);
    }

	private RequestManager getApplicationManager(Context context) {
        if (applicationManager == null) {
            synchronized (this) {
                if (applicationManager == null) {
                    applicationManager = new RequestManager(
                    		context.getApplicationContext(),
                            new ApplicationLifecycle(),
                            new EmptyRequestManagerTreeNode());
                }
            }
        }
        // applicationManager 全局唯一
        return applicationManager;
    }

// ApplicationLifecycle
	class ApplicationLifecycle implements Lifecycle {
    	@Override
    	public void addListener(LifecycleListener listener) {
    		// 这里模拟了一个假的生命周期, 只有start
        	listener.onStart();
    	}
	}
```
小结:
* 创建了一个 **ApplicationLifecycle** ( 模拟生命周期 / 全局生命周期 )
* 放进一个全局唯一 **applicationManager** ( RequestManager 实例 ) 中

**主线程 + Activity 时**

```java
// RequestManagerRetriever
	RequestManager fragmentGet(Context context, android.app.FragmentManager fm) {
		// 这里应该就是生成空白 Fragment 的步骤
        RequestManagerFragment current = getRequestManagerFragment(fm);
        ...
        requestManager = new RequestManager(context,
        						// 这里就是创建生命周期的地方了
        						current.getLifecycle(),
        						current.getRequestManagerTreeNode());
        current.setRequestManager(requestManager);
        ...
        return requestManager;
    }

	RequestManagerFragment getRequestManagerFragment(final android.app.FragmentManager fm) {

		// 一系列验证当前界面是否绑定过 Glide 的空白 Fragment
        RequestManagerFragment current = (RequestManagerFragment) fm.findFragmentByTag(FRAGMENT_TAG);
        if (current == null) {
            current = pendingRequestManagerFragments.get(fm);
            if (current == null) {

            	// 未绑定过 , 创建一个新的
                current = new RequestManagerFragment();
                pendingRequestManagerFragments.put(fm, current);
                fm.beginTransaction().add(current,FRAGMENT_TAG)
                					 .commitAllowingStateLoss();
                handler.obtainMessage(ID_REMOVE_FRAGMENT_MANAGER, fm).sendToTarget();
            }
        }
        return current;
    }

// RequestManagerFragment
	ActivityFragmentLifecycle getLifecycle() {
		// 根据上面的代码, 应该是在构造函数里创建了, 跟进去看看
        return lifecycle;
    }

	public RequestManagerFragment() {
		// 最终创建了一个 ActivityFragmentLifecycle 实例 , 来看看它是如何工作的
        this(new ActivityFragmentLifecycle());
    }

	RequestManagerFragment(ActivityFragmentLifecycle lifecycle) {
        this.lifecycle = lifecycle;
    }

	@Override
    public void onStart() {
        super.onStart();
        lifecycle.onStart();
    }

    @Override
    public void onStop() {
        super.onStop();
        lifecycle.onStop();
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        lifecycle.onDestroy();
    }

// ActivityFragmentLifecycle
	void onStart() {
        isStarted = true;
        for (LifecycleListener lifecycleListener : Util.getSnapshot(lifecycleListeners)) {
            lifecycleListener.onStart();
        }
    }

    void onStop() {
        isStarted = false;
        for (LifecycleListener lifecycleListener : Util.getSnapshot(lifecycleListeners)) {
            lifecycleListener.onStop();
        }
    }

    void onDestroy() {
        isDestroyed = true;
        for (LifecycleListener lifecycleListener : Util.getSnapshot(lifecycleListeners)) {
            lifecycleListener.onDestroy();
        }
    }
```
很简单,
* 往 **Activity** 插入一个没界面的 **RequestManagerFragment**
* 创建了一个 **ActivityFragmentLifecycle 实例**, 在 **RequestManagerFragment** 生命周期函数被调用时调用

**主线程 + Fragment 时**

我们就不看了, 结果差不多
* 往 **Fragment** 插入一个没界面的 **SupportRequestManagerFragment**
* 创建了一个 **ActivityFragmentLifecycle 实例**, 在 **SupportRequestManagerFragment** 生命周期函数被调用时调用

## 内存监听
* 在 SupportRequestManagerFragment 的 onLowMemory()
* 在 RequestManagerFragment  的 onTrimMemory() / onLowMemory()

```java
// SupportRequestManagerFragment
	@Override
    public void onLowMemory() {
        ...
        requestManager.onLowMemory();
    }

// RequestManagerFragment
	@Override
    public void onTrimMemory(int level) {
        ...
        requestManager.onTrimMemory(level);
    }

    @Override
    public void onLowMemory() {
        ...
        requestManager.onLowMemory();
    }

// requestManager 是啥? 还记得 RequestManagerFragment 创建的代码 ?
// RequestManagerRetriever
	RequestManager fragmentGet(Context context, android.app.FragmentManager fm) {
        RequestManagerFragment current = getRequestManagerFragment(fm);
        ...
        requestManager = new RequestManager(context,
        						current.getLifecycle(),
        						current.getRequestManagerTreeNode());
        // 这里这里这里
        current.setRequestManager(requestManager);
        ...
        return requestManager;
    }

// RequestManager
	public void onTrimMemory(int level) {
        glide.trimMemory(level);
    }

    public void onLowMemory() {
        glide.clearMemory();
    }

// Glide 单例
	public void clearMemory() {
		...
        memoryCache.clearMemory();
        bitmapPool.clearMemory();
    }

    public void trimMemory(int level) {
        ...
        memoryCache.trimMemory(level);
        bitmapPool.trimMemory(level);
    }
```
小结:
* 空白 Fragment 返回存储吃紧的时候, 交由 Glide 处理去了
* memoryCache 清理内存缓存
* bitmapPool 清理内存缓存
> memoryCache 很容易就能猜出来是 Glide 的用来做内存缓存的;
> 那么 bitmapPool 是个啥? [Android Glide 3.7.0 源码解析(四), BitmapPool作用及原理](/2021/03/20/markdown-glide3.7.0_4/index.html)
> 1. 在 Android 中图片的显示实体是一个 Bitmap 对象, 每一次图片显示都会先将图片资源构建成一个 Bitmap 对象, 而创建和销毁 Bitmap 的过程比较耗系统资源, 严重时还会引起GC频繁, 界面卡顿
> 2. 举个例子: 列表显示头像, 一页10个头像展示, 假定GC的阈值就是10张图
>  普通方案: 创建10个 Bitmap 再释放, 再创建10个 Bitmap 用来展示下一页, 这样没滑动一页就是触发一次GC
>  Glide 方案: 创建一个 BitmapPool 参照线程池理解, 创建好10个 bitmap 不释放, 下一页的10个图像, 借用已有的 Bitmap 的内存空间, 不论滑动多少页, 都不会触发 GC 了

## 请求任务监听
来看看 RequestManager 的构造函数
```java
// RequestManager
	public RequestManager(Context context, Lifecycle lifecycle, RequestManagerTreeNode treeNode) {
        this(context, lifecycle, treeNode, new RequestTracker(), new ConnectivityMonitorFactory());
    }

	RequestManager(Context context, final Lifecycle lifecycle, RequestManagerTreeNode treeNode,
            RequestTracker requestTracker, ConnectivityMonitorFactory factory) {
        this.context = context.getApplicationContext();
        this.lifecycle = lifecycle;
        this.treeNode = treeNode;
        this.requestTracker = requestTracker;
        this.glide = Glide.get(context);
        this.optionsApplier = new OptionsApplier();

        ConnectivityMonitor connectivityMonitor = factory.build(context,
                new RequestManagerConnectivityListener(requestTracker));

        if (Util.isOnBackgroundThread()) {
            new Handler(Looper.getMainLooper()).post(new Runnable() {
                @Override
                public void run() {
                    lifecycle.addListener(RequestManager.this);
                }
            });
        } else {
        	// 把 RequestManager 自身注册进入 lifecycle
            lifecycle.addListener(this);
        }
        // 这里是网络状态的生命周期注册, 下一小节讲
        lifecycle.addListener(connectivityMonitor);
    }

// RequestManager 的 LifecycleListener 实现
	@Override
    public void onStart() {
        // onStart might not be called because this object may be created after the fragment/activity's onStart method.
        resumeRequests();
    }

	public void resumeRequests() {
        Util.assertMainThread();
        requestTracker.resumeRequests();
    }

    /**
     * Lifecycle callback that unregisters for connectivity events (if the android.permission.ACCESS_NETWORK_STATE
     * permission is present) and pauses in progress loads.
     */
    @Override
    public void onStop() {
        pauseRequests();
    }

	public void pauseRequests() {
        Util.assertMainThread();
        requestTracker.pauseRequests();
    }
    /**
     * Lifecycle callback that cancels all in progress requests and clears and recycles resources for all completed
     * requests.
     */
    @Override
    public void onDestroy() {
        requestTracker.clearRequests();
    }
```
1. RequestManager 在初始化的时候, 把自己注册进入 Lifecycle
2. 而通过 RequestManager 对 LifecycleListener 的 实现可得, 最终都是调用 **RequestManager.requestTracker** 来实现功能

现在我们来追踪 requestTracker 看看
```java
// RequestManager
	public RequestManager(Context context, Lifecycle lifecycle, RequestManagerTreeNode treeNode) {
		// 这里直接 new 了一个对象
        this(context, lifecycle, treeNode, new RequestTracker(), new ConnectivityMonitorFactory());
    }

// RequestTracker

	// LifecycleListener.onStart
	public void resumeRequests() {
        isPaused = false;
        for (Request request : Util.getSnapshot(requests)) {
            if (!request.isComplete() && !request.isCancelled() && !request.isRunning()) {
                request.begin();
            }
        }
        pendingRequests.clear();
    }

	// LifecycleListener.onStop
	public void pauseRequests() {
        isPaused = true;
        for (Request request : Util.getSnapshot(requests)) {
            if (request.isRunning()) {
                request.pause();
                pendingRequests.add(request);
            }
        }
    }

	// LifecycleListener.onDestroy
	public void clearRequests() {
        for (Request request : Util.getSnapshot(requests)) {
            request.clear();
        }
        pendingRequests.clear();
    }
```
1. 可以看到这里最终调用的是 Request(真实) 的 begin() , pause() , clear()
2. 这里的 Request(真实) 是一个 GenericRequest 对象 (详细参考: [Android Glide 3.7.0 源码解析(二), 从一次图片加载流程看源码](https://blog.csdn.net/qq_25778369/article/details/114577763))

继续追踪 GenericRequest
```java
// GenericRequest

	// LifecycleListener.onStart
    public void begin() {
        startTime = LogTime.getLogTime();
        if (model == null) {
            onException(null);
            return;
        }

        status = Status.WAITING_FOR_SIZE;
        if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
        	// 就是调用这个方法开始新一次的下载任务
            onSizeReady(overrideWidth, overrideHeight);
        } else {
            target.getSize(this);
        }

        if (!isComplete() && !isFailed() && canNotifyStatusChanged()) {
            target.onLoadStarted(getPlaceholderDrawable());
        }
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            logV("finished run method in " + LogTime.getElapsedMillis(startTime));
        }
    }

	// LifecycleListener.onStop
	public void pause() {
		// 看来 onStop 和 onDestroy  是调用的相同的逻辑
        clear();
        status = Status.PAUSED;
    }

	// LifecycleListener.onDestroy
	public void clear() {
        Util.assertMainThread();
        if (status == Status.CLEARED) {
            return;
        }
        // 任务的停止都是在这边控制
        cancel();
        // Resource must be released before canNotifyStatusChanged is called.
        if (resource != null) {
        	// 此处就是一些资源的释放, 过滤不看
            releaseResource(resource);
        }
        if (canNotifyStatusChanged()) {
            target.onLoadCleared(getPlaceholderDrawable());
        }
        // Must be after cancel().
        status = Status.CLEARED;
    }

	void cancel() {
        status = Status.CANCELLED;
        if (loadStatus != null) {
            loadStatus.cancel();
            loadStatus = null;
        }
    }

// LoadStatus
	public static class LoadStatus {
        private final EngineJob engineJob;
        private final ResourceCallback cb;

        public LoadStatus(ResourceCallback cb, EngineJob engineJob) {
            this.cb = cb;
            this.engineJob = engineJob;
        }

        public void cancel() {
        	// 最终调用这个 EngineJob 来实现任务的取消的
            engineJob.removeCallback(cb);
        }
    }
```
1. 生命周期方法 onStart() 最终通过 Request 的 begin() 来发起一个请求
2. 而 onStop() 和 onDestroy() 则是通过 EngineJob 的 removeCallback() 来实现
3. 这个 EngineJob 其实是管理下载时 Request 的线程调度的(具体参见: [Android Glide 3.7.0 源码解析(二), 从一次图片加载流程看源码](https://editor.csdn.net/md/?articleId=114577763))
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210315105844940.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5,size_16,color_FFFFFF,t_70)

继续追踪 EngineJob 来看看
```java
// EngineJob
	public void removeCallback(ResourceCallback cb) {
        Util.assertMainThread();
        if (hasResource || hasException) {
            addIgnoredCallback(cb);
        } else {
        	// 先移除监听回调
            cbs.remove(cb);
            if (cbs.isEmpty()) {
            	// 都移除完毕之后, 看看它做了啥
                cancel();
            }
        }
    }

	void cancel() {
        if (hasException || hasResource || isCancelled) {
            return;
        }
        engineRunnable.cancel();
        Future currentFuture = future;
        if (currentFuture != null) {
        	// 这里的 future 是一个 ExecutorService.submit 返回的最终到这里终止任务
            currentFuture.cancel(true);
        }
        isCancelled = true;
        listener.onEngineJobCancelled(this, key);
    }
```
说明:
1. Glide 单例在实例化的时候, 会创建一个 diskCacheService ( ExecutorService 类型 ) 和 Engine 对象, 并把 diskCacheService 封装进 Engine
2.  调用 Engine.load() 来执行一个 Request(真实) 的任务
3. Engine.load() 会 创建一个 EngineJob 实例, 并把 diskCacheService 传递进去
4. EngineJob.start() 来开始一个任务, 其实就是调用 diskCacheService.submit()
5. 所以, 上面的 cancel() 其实就是 Future.cancel()
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210315111813772.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5,size_16,color_FFFFFF,t_70)
6. 每个 Request(真实) 对应一个 EngineJob 实例

还剩下最后一个没有看了, 网络状态变化的监听

## 网络状态变化的监听
还记得 RequestManager 的构造函数?
```java
// RequestManager

	public RequestManager(Context context, Lifecycle lifecycle, RequestManagerTreeNode treeNode) {
        this(context, lifecycle, treeNode, new RequestTracker(), new ConnectivityMonitorFactory());
    }

    RequestManager(Context context, final Lifecycle lifecycle, RequestManagerTreeNode treeNode,
            RequestTracker requestTracker, ConnectivityMonitorFactory factory) {
        this.context = context.getApplicationContext();
        this.lifecycle = lifecycle;
        this.treeNode = treeNode;
        this.requestTracker = requestTracker;
        this.glide = Glide.get(context);
        this.optionsApplier = new OptionsApplier();

        ConnectivityMonitor connectivityMonitor = factory.build(context,
                new RequestManagerConnectivityListener(requestTracker));

        // If we're the application level request manager, we may be created on a background thread. In that case we
        // cannot risk synchronously pausing or resuming requests, so we hack around the issue by delaying adding
        // ourselves as a lifecycle listener by posting to the main thread. This should be entirely safe.
        if (Util.isOnBackgroundThread()) {
            new Handler(Looper.getMainLooper()).post(new Runnable() {
                @Override
                public void run() {
                    lifecycle.addListener(RequestManager.this);
                }
            });
        } else {
            lifecycle.addListener(this);
        }
        // 就是这里 创建了一个 connectivityMonitor 实例, 并注册进监听
        lifecycle.addListener(connectivityMonitor);
    }

//DefaultConnectivityMonitor ConnectivityMonitor 子类
	@Override
    public void onStart() {
        register();
    }

    @Override
    public void onStop() {
        unregister();
    }

	private void register() {
        if (isRegistered) {
            return;
        }

        isConnected = isConnected(context);
        context.registerReceiver(connectivityReceiver, new IntentFilter(ConnectivityManager.CONNECTIVITY_ACTION));
        isRegistered = true;
    }

    private void unregister() {
        if (!isRegistered) {
            return;
        }

        context.unregisterReceiver(connectivityReceiver);
        isRegistered = false;
    }
```
很简单 ,  就是界面活着的时候去监听网络状态变换, 界面销毁的时候, 解注册对网络状态变化的监测


