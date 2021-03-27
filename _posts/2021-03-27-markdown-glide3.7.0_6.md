---
layout:     post
title:      "Android Glide 3.7.0 源码解析(六) , 缓存结构详述"
subtitle:   ""
date:       2021-03-27 20:38:32
author:     "zhoujie"
catalog: true # 是否开启目录
tags:
    - glide3.7.0
    - 源码
---

# 结构总览
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210325222328644.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5,size_16,color_FFFFFF,t_10)

1. 内存缓存是由 LruResourceCache 和 activeResources 组成, 缓存的是 **EngineResource** 类型
**第一级缓存**: **LruResourceCache** 是一个最终是一个 LinkedHashMap 来实现 Lru , 存储的是没有被界面使用的缓存资源, 并由LRU控制缓存大小
**第二级缓存**: **activeResources** 是由一个 Map<Key, WeakReference<EngineResource<?>>> 构成, 存储的是正在被界面使用的资源弱引用缓存, 不同于内存缓存用 Lru 算法策略的是，该缓存是无容量大小限制的，内部用引用计数来确定是否被外界所正在使用，当引用为0时会从该缓存中移除，加入到内存缓存
2. DiskLruCacheWrapper(data) DiskLruCacheWrapper(source) 组成了磁盘缓存, DiskLruCacheWrapper 内部是由一个 DiskLruCache 构建, 提供了文件级别的LRU策略缓存
**第三级缓存**: **DiskLruCacheWrapper(data)**, 缓存的是图形转换后的资源
**第四级缓存**: **DiskLruCacheWrapper(source)**, 缓存的是未经图形转换的原始资源, 未处理是指没有经过图形变换(裁剪, 缩放等)
3. **第三级缓存和第四级缓存公用一个 Lru 缓存 DiskLruCacheWrapper, 只是因为这个缓存中存储的数据类型不一样所以本文中称之为 DiskLruCacheWrapper(data) 和 DiskLruCacheWrapper(source)**
>查找时: 是由 Engine.load 时候产生Key(由10多个参数决定), 来进行缓存匹配, 其中当获取原始未处理资源时会调用 Key.getOriginalKey 来产生一个参数较少的 Key, 用来匹配未被处理的原始资源

>还记得[Android Glide 3.7.0 源码解析(二), 从一次图片加载流程看源码](/2021/03/14/markdown-glide3.7.0_2/index.html) 文章中提到的
>![在这里插入图片描述](https://img-blog.csdnimg.cn/20210325223704783.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5,size_16,color_FFFFFF,t_70)

* **缓存**的初始化是在 Glide 单例**创建**时进行的
* **内存缓存匹配**是在 Engine.load方法中用传入的 Key 进行**匹配**, 先匹配 **LruResourceCache(第一级)** 再匹配 **activeResources(第二级)**

>  **LruResourceCache(第一级)** 和  **activeResources(第二级)** 的区别:
> 	* **LruResourceCache(第一级)** 中存储的是不被界面使用的资源实例
> 	* **activeResources(第二级)** 中存储的是正在被界面组件使用的资源实例(可以使多个界面组件,使用同一个活跃缓存)
>
> **activeResources(第二级)** 存在的意义:
>* 不同于 **LruResourceCache(第一级)** 用 Lru 算法策略的是，该缓存是无容量大小限制的，内部用引用计数来确定是否被外界所正在使用，当引用为0时会从该缓存中移除，加入到内存缓存, 该缓存的作用是保护正在使用的资源并复用，试想一下如果没有这级缓存，只有 **LruResourceCache(第一级)** 那么当 **LruResourceCache(第一级)** 缓存达到容量上限时有可能移除掉正在使用的图片资源，当应用中另外一个地方需要同时显示同样的图片资源时Glide将在内存中找不到这个资源对象则又会重建一个新的资源对象。
>
>**Glide 4.x行为变化**
>* Glide 4.x.x版本, 会先从 **activeResources(第二级)** 中匹配, 再从  **LruResourceCache(第一级)** 中匹配
* **磁盘缓存匹配**是在 EngineRunable.run 开始 decode 时, **先匹配**有没有图形转化后的数据, 即 **DiskLruCacheWrapper(data 第三级)** , **再匹配** Source (图片的原数据) 即 **DiskLruCacheWrapper(Source 第四级)**, 为什么不放在 Engine.load 一起匹配, 因为存储在磁盘中的缓存格式是未经转码或者未经图形转换的数据, 需要在 EngineRunable.run 开辟的 **非主线程(转码/图形转换耗时)** 中用 EngineJob 的 decode() 方法来对缓存数据进行转码/图形转换, 而内存缓存则没有这种问题, 内存缓存缓存的就是处理(转码/图形转换)之后的数据, 即 **EngineResource**
* **磁盘缓存存入** 是在 EngineRunable.run 方法中 EngineJob 真正 decode(从网络下载) 完毕开始的, 在解析 SourceData 也就是图片原始数据时, cacheAndDecodeSourceData() 方法来执行**加入 DiskLruCacheWrapper(Source 第四级)** 缓存, 在 writeTransformedToCache() 方法中把图形变换后的图形数据(非原始) **加入 DiskLruCacheWrapper(data 第三级)** 缓存
* **内存缓存存入** 在 Engine.onEngineJobComplete 方法中缓存到 **activeResources(第二级)** 表示正在被界面使用, 在 EngineResource release 的时候, 判断引用计数是否到 0 (是否匹配 **activeResources(第二级)** ), 如果不匹配, 则尝试放入 **LruResourceCache(第一级)** 中

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210327190719464.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5,size_16,color_FFFFFF,t_70)

以上是一个缓存使用的流程图, 解释一下
1. Engine.load 检查 **LruResourceCache(第一级)** 和 **activeResources(第二级)** 缓存
2. 切换线程, 到执行下载的地方, 开始检查 **DiskLruCacheWrapper(data 第三级)** 和 **DiskLruCacheWrapper(Source 第四级)** 缓存
3. 都没有缓存, 执行下载, 下载完成保存到 **DiskLruCacheWrapper(Source 第四级)** , 处理过后(解码/图形变换) 保存到 **DiskLruCacheWrapper(data 第三级)**
4. 往界面回调时, 保存到 **activeResources(第二级)**
5. 当界面被释放时, 回调 Request(真实) 的 clear函数, 查看 **activeResources(第二级)** 有没有计数为0的缓存, 保存到 **LruResourceCache(第一级)**
> [Android Glide 3.7.0 源码解析(三), 生命周期绑定](/2021/03/20/markdown-glide3.7.0_3/index.html) 一文中有提到界面被释放时, Request(真实) 的行为


框架逻辑已经有概念了, 接下来看源码就不会迷路了

# 缓存创建
**LruResourceCache(第一级) 创建**, 是在 Glide 单例初始化的时候 GlideBuilder.createGlide()
```java
// GlideBuilder
	Glide createGlide() {
		...
        if (memoryCache == null) {
        	// 创建了一个 LruResourceCache 实例
            memoryCache = new LruResourceCache(calculator.getMemoryCacheSize());
        }
        ...
    }
```
**activeResources(第二级) 缓存创建**,  也是在 Glide 单例初始化的时候 GlideBuilder.createGlide(), 创建了一个 Engine 实例, 并在这个实例的构造函数中初始化了 activeResources 弱引用数组
> activeResources 是一个 Map<Key, WeakReference<EngineResource<?>>> 类型
> Engine 实例是 Glide 单例的一个成员变量, 即, 也是全局唯一的

```java
// GlideBuilder
	Glide createGlide() {
		...
        if (engine == null) {
            engine = new Engine(memoryCache, diskCacheFactory, diskCacheService, sourceService);
        }
        ...
    }

// Engine
	public Engine(MemoryCache memoryCache, DiskCache.Factory diskCacheFactory, ExecutorService diskCacheService,
            ExecutorService sourceService) {
        this(memoryCache, diskCacheFactory, diskCacheService, sourceService, null, null, null, null, null);
    }

	Engine(MemoryCache cache, DiskCache.Factory diskCacheFactory, ExecutorService diskCacheService,
            ExecutorService sourceService, Map<Key, EngineJob> jobs, EngineKeyFactory keyFactory,
            Map<Key, WeakReference<EngineResource<?>>> activeResources, EngineJobFactory engineJobFactory,
            ResourceRecycler resourceRecycler) {
        ...

        if (activeResources == null) {
        	// 最终创建了一个 HashMap
            activeResources = new HashMap<Key, WeakReference<EngineResource<?>>>();
        }
        this.activeResources = activeResources;

        ...
    }
```
 **DiskLruCacheWrapper(data 第三级)** 和 **DiskLruCacheWrapper(Source 第四级)** 缓存的创建

```java
// GlideBuilder
	Glide createGlide() {
        ...
        if (diskCacheFactory == null) {
            diskCacheFactory = new InternalCacheDiskCacheFactory(context);
        }
		...
    }

// InternalCacheDiskCacheFactory
	public final class InternalCacheDiskCacheFactory extends DiskLruCacheFactory {}

// DiskLruCacheFactory
	@Override
    public DiskCache build() {
        ...
        return DiskLruCacheWrapper.get(cacheDir, diskCacheSize);
    }

// DiskLruCacheWrapper
	public static synchronized DiskCache get(File directory, int maxSize) {
        ...
        if (wrapper == null) {
        	// 最终创建了一个 DiskLruCacheWrapper 实例
            wrapper = new DiskLruCacheWrapper(directory, maxSize);
        }
        return wrapper;
    }
```

# 缓存的使用
根据上文的框架流程图, 直接看 Engine.load
```java
// Engine
	public <T, Z, R> LoadStatus load(Key signature, int width, int height, DataFetcher<T> fetcher,
            DataLoadProvider<T, Z> loadProvider, Transformation<Z> transformation, ResourceTranscoder<Z, R> transcoder,
            Priority priority, boolean isMemoryCacheable, DiskCacheStrategy diskCacheStrategy, ResourceCallback cb) {
        Util.assertMainThread();
        long startTime = LogTime.getLogTime();

        final String id = fetcher.getId();
        // 可以看到在这里打包了一个 Key 值, 用来唯一标识内存中的一个缓存实例
        EngineKey key = keyFactory.buildKey(id, signature, width, height, loadProvider.getCacheDecoder(),
                loadProvider.getSourceDecoder(), transformation, loadProvider.getEncoder(),
                transcoder, loadProvider.getSourceEncoder());

		// 先从 【LruResourceCache(第一级)】 中搜索
        EngineResource<?> cached = loadFromCache(key, isMemoryCacheable);
        if (cached != null) {
        	// 匹配中数据直接回调上抛
            cb.onResourceReady(cached);
            return null;
        }
		// 再从 【activeResources(第二级)】 中搜索
        EngineResource<?> active = loadFromActiveResources(key, isMemoryCacheable);
        if (active != null) {
            cb.onResourceReady(active);
            return null;
        }

       	...
		// 如果都匹配不到就执行下载
        ...
        engineJob.start(runnable);
		...
    }

	private EngineResource<?> loadFromCache(Key key, boolean isMemoryCacheable) {
        if (!isMemoryCacheable) {
            return null;
        }

		// 尝试从【LruResourceCache(第一级)】中获取/移除
        EngineResource<?> cached = getEngineResourceFromCache(key);
        if (cached != null) {
            cached.acquire();
            // 成功了, 加入到【activeResources(第二级)】中去
            activeResources.put(key, new ResourceWeakReference(key, cached, getReferenceQueue()));
        }
        return cached;
    }

	private EngineResource<?> loadFromActiveResources(Key key, boolean isMemoryCacheable) {
        if (!isMemoryCacheable) {
            return null;
        }

        EngineResource<?> active = null;
        // 尝试从【activeResources(第二级)】中匹配
        WeakReference<EngineResource<?>> activeRef = activeResources.get(key);
        if (activeRef != null) {
            active = activeRef.get();
            if (active != null) {
            	// 匹配到,则增加引用计数, 表示界面上正有多少元素在使用它
                active.acquire();
            } else {
            	// 匹配为空, 可能被释放了, 移除被释放资源的弱引用
                activeResources.remove(key);
            }
        }

        return active;
    }
```

* 先从 **LruResourceCache(第一级)** 中匹配, 再从 **activeResources(第二级)** 中匹配
* 不行就执行下载

下面跟进去看下载的流程

```java
// EngineRunnable
	public void run() {
        ...
        // 执行下载
        resource = decode();
        ...
        // 下载成功回调
        onLoadComplete(resource);
        ...
    }

	private Resource<?> decode() throws Exception {
        if (isDecodingFromCache()) {
        	// 先匹配【三/四级缓存】
            return decodeFromCache();
        } else {
        	// 匹配不到就真的去下载
            return decodeFromSource();
        }
    }

	private Resource<?> decodeFromCache() throws Exception {
        Resource<?> result = null;
        try {
        	// 匹配【DiskLruCacheWrapper(data 第三级)】
            result = decodeJob.decodeResultFromCache();
        } catch (Exception e) {
            if (Log.isLoggable(TAG, Log.DEBUG)) {
                Log.d(TAG, "Exception decoding result from cache: " + e);
            }
        }

        if (result == null) {
        	// 匹配【DiskLruCacheWrapper(Source 第四级)】
            result = decodeJob.decodeSourceFromCache();
        }
        return result;
    }
```
* 看到在线程切换之后开始匹配  **DiskLruCacheWrapper(data 第三级)** 和 **DiskLruCacheWrapper(Source 第四级)**

匹配不到, 下载完成之后又做了些啥?

```java
// EngineRunnable
	private void onLoadComplete(Resource resource) {
        manager.onResourceReady(resource);
    }

// EngineJob
	public void onResourceReady(final Resource<?> resource) {
        this.resource = resource;
        MAIN_THREAD_HANDLER.obtainMessage(MSG_COMPLETE, this).sendToTarget();
    }

	private static class MainThreadCallback implements Handler.Callback {

        @Override
        public boolean handleMessage(Message message) {
            if (MSG_COMPLETE == message.what || MSG_EXCEPTION == message.what) {
                EngineJob job = (EngineJob) message.obj;
                if (MSG_COMPLETE == message.what) {
                	// 走这里, 开始处理下载完成, 上抛图片数据到界面的逻辑了
                    job.handleResultOnMainThread();
                } else {
                    job.handleExceptionOnMainThread();
                }
                return true;
            }

            return false;
        }
    }

	private void handleResultOnMainThread() {
        ...
        // listener是Engine实例
        listener.onEngineJobComplete(key, engineResource);
		// cb 是在 Engine.load中传入的 GenericRequest 实例
        for (ResourceCallback cb : cbs) {
            if (!isInIgnoredCallbacks(cb)) {
                engineResource.acquire();
                cb.onResourceReady(engineResource);
            }
        }
        ...
    }

// Engine
	public void onEngineJobComplete(Key key, EngineResource<?> resource) {
        ...
        // 这里加入资源到【activeResources(第二级)】
        activeResources.put(key, new ResourceWeakReference(key, resource, getReferenceQueue()));
        ...
    }
```
可以看到在上抛图片数据到界面的过程中, 将其缓存到  **activeResources(第二级)**
最后来看看何时缓存到 **LruResourceCache(第一级)**
通过 [Android Glide 3.7.0 源码解析(三), 生命周期绑定](/2021/03/20/markdown-glide3.7.0_2/index.html)一文, 我们知道, 界面释放时, 会触发 GenericRequest 的 clear 方法

```java
// GenericRequest
	public void clear() {
        Util.assertMainThread();
        if (status == Status.CLEARED) {
            return;
        }
        cancel();
        // Resource must be released before canNotifyStatusChanged is called.
        if (resource != null) {
        	// 重点在这里, 需要释放界面正在使用的 【activeResources(第二级)】
            releaseResource(resource);
        }
        if (canNotifyStatusChanged()) {
            target.onLoadCleared(getPlaceholderDrawable());
        }
        // Must be after cancel().
        status = Status.CLEARED;
    }

	private void releaseResource(Resource resource) {
		// 进入 Engine 实例看看
        engine.release(resource);
        this.resource = null;
    }

// Engine
	public void release(Resource resource) {
        ...
        ((EngineResource) resource).release();
        ...
    }

// EngineResource
	void release() {
        if (acquired <= 0) {
            throw new IllegalStateException("Cannot release a recycled or not yet acquired resource");
        }
        if (!Looper.getMainLooper().equals(Looper.myLooper())) {
            throw new IllegalThreadStateException("Must call release on the main thread");
        }
        if (--acquired == 0) {
        	// 缩减引用计数, 当达到0时触发listener去处理(这里listener是Engine实例)
            listener.onResourceReleased(key, this);
        }
    }

// Engine
	public void onResourceReleased(Key cacheKey, EngineResource resource) {
        Util.assertMainThread();
        // 从 【activeResources(第二级)】中移除
        activeResources.remove(cacheKey);
        if (resource.isCacheable()) {
        	// 存入到 【DiskLruCacheWrapper(data 第三级)】
            cache.put(cacheKey, resource);
        } else {
            resourceRecycler.recycle(resource);
        }
    }
```
界面释放资源的时候, 如果 **activeResources(第二级)** 的资源引用计数归零, 则将资源从 **activeResources(第二级)** 移除, 并尝试将移除的资源放置到 **LruResourceCache(第一级)** 至此, 源代码分析完毕!

