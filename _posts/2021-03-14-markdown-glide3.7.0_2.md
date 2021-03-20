---
layout:     post
title:      "Android Glide 3.7.0 源码解析 (二) , 从一次图片加载流程看源码"
subtitle:   ""
date:       2021-03-14 15:02:35
author:     "zhoujie"
catalog: true # 是否开启目录
tags:
    - glide3.7.0
    - 源码
---

# 一、加载图片代码
```java
Glide.with(activity).load(url).into(imageView);
```
# 二、流程图
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210310215407519.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5,size_16,color_FFFFFF,t_70)
> 1. Glide.with 方法, 创建 RequestManager 实例
> 2. RequestManager.load 方法, 创建 GenericRequestBuilder 实例, 并打包编/解码, 转码, 图形转换, 下载等工具
> 解码: File, InputStream 转换成 Bitmap, Drawable
> 编码: 将数据写入缓存区
>
> 3. GenericRequestBuilder.into 方法, 使用 load 构建的参数构建一个 Request 实例
> 4. Request 实例, 执行下载, 解码, 图形变换, 数据转码, 生成 Resource 图片资源
> 5. onSourceReady 方法, 将处理好的 Resource 回调到 Target 并显示出来

# 三、源码执行过程
## [3.1] with()
> * 特别注意: 源码较多, 为了精简不会贴全, 省略部分会以 ... 来表示

```java
// Glide

    public static RequestManager with(Activity activity) {
        RequestManagerRetriever retriever = RequestManagerRetriever.get();
        return retriever.get(activity);
    }
```
进入 RequestManagerRetriever.get
```java
// RequestManagerRetriever

    public RequestManager get(Activity activity) {
        if (Util.isOnBackgroundThread() ...) {
            return ...;
        } else {
            android.app.FragmentManager fm = activity.getFragmentManager();
            return fragmentGet(activity, fm);
        }
    }
```
两个分支, 假定, 在主线程调用, 则, 进入 fragmentGet
```java
// RequestManagerRetriever

    RequestManager fragmentGet(Context context, android.app.FragmentManager fm) {
        RequestManagerFragment current = getRequestManagerFragment(fm);
        ...
        requestManager = new RequestManager(context, current.getLifecycle(), current.getRequestManagerTreeNode());
        current.setRequestManager(requestManager);
        ...
        return requestManager;
    }
```
> * RequestManagerFragment 是用来监测生命周期, 内存的, 另外网络状态 ConnectivityMonitor 也会将自己绑入生命周期, 不是主线这里不做赘述, 详细原理可以点击查看这篇文章, [Android Glide 3.7.0 源码解析(三), 生命周期绑定](/2021/03/20/markdown-glide3.7.0_3/index.html)

创建了一个 RequestManager 实例, 传入**生命周期 ( ActivityFragmentLifecycle ) **和**生命里面的Fragment树结构 ( RequestManagerTreeNode )**
### [3.1.1] RequestManager.glide
```java
// RequestManager

	RequestManager(Context context, final Lifecycle lifecycle, RequestManagerTreeNode treeNode,
            RequestTracker requestTracker, ConnectivityMonitorFactory factory) {
        ...
        this.glide = Glide.get(context);
        ...
    }
```
**这里创建了一个 Glide 单例**, 留作后用
## [3.2] load()
```java
// RequestManager

    public DrawableTypeRequest<String> load(String string) {
        return (DrawableTypeRequest<String>) fromString().load(string);
    }
```
> * **创建了一个 DrawableTypeRequest , 它是个啥?**
> 继承自 GenericRequestBuilder ( [UML 类图参考](https://img-blog.csdnimg.cn/20210311195313868.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5) ) , **集合了编/解码, 图形变换, 转码, 下载功能** ( 下面的流程中我们需要注意这些功能都是怎么集合进去的 )
> * 在流程中的位置和功能:
> ![在这里插入图片描述](https://img-blog.csdnimg.cn/20210311193627438.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5,size_16,color_FFFFFF,t_70)
> 功能: **调用 GenericRequestBuilder.into 生成真正的 Request 实例**, GenericRequestBuilder 可以理解为一个用户态的 Request

1 行代码, 2 个分支, 第一 fromString, 第二 load, 先看第 1 个分支
```java
// RequestManager

	public DrawableTypeRequest<String> fromString() {
        return loadGeneric(String.class);
    }

	private <T> DrawableTypeRequest<T> loadGeneric(Class<T> modelClass) {
        ModelLoader<T, InputStream> streamModelLoader = Glide.buildStreamModelLoader(modelClass, context);
        ModelLoader<T, ParcelFileDescriptor> fileDescriptorModelLoader =
                Glide.buildFileDescriptorModelLoader(modelClass, context);
        ...

        return optionsApplier.apply(
                new DrawableTypeRequest<T>(modelClass, streamModelLoader, fileDescriptorModelLoader, context,
                        glide, requestTracker, lifecycle, optionsApplier));
    }
```
> 留意一下这里的 modelClass == String.class
> StreamStringLoader 和 FileDescriptorStringLoader 都属于 ModelLoader ( [UML 类图参考](https://img-blog.csdnimg.cn/20210311202331373.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5
) ) 类型, 功用: **数据下载**
1.  构建 StreamStringLoader 和 FileDescriptorStringLoader 实例, 属于**下载模块**
2.  以 1 中实例 , 构建 DrawableTypeRequest 实例, **集合了编/解码, 图形变换, 转码, 下载功能**

下面来看看这些个功能模块是怎么初始化, 被打包进 DrawableTypeRequest 的

### [3.2.1] 创建Request(用户态)
创建Request(用户态) DrawableTypeRequest
```java
// RequestManager
	private <T> DrawableTypeRequest<T> loadGeneric(Class<T> modelClass) {
        ModelLoader<T, InputStream> streamModelLoader = Glide.buildStreamModelLoader(modelClass, context);
        ModelLoader<T, ParcelFileDescriptor> fileDescriptorModelLoader =
                Glide.buildFileDescriptorModelLoader(modelClass, context);
        ...

        return optionsApplier.apply(
                new DrawableTypeRequest<T>(modelClass, streamModelLoader, fileDescriptorModelLoader, context,
                        glide, requestTracker, lifecycle, optionsApplier));
    }

// DrawableTypeRequest

	DrawableTypeRequest(Class<ModelType> modelClass, ModelLoader<ModelType, InputStream> streamModelLoader,
            ModelLoader<ModelType, ParcelFileDescriptor> fileDescriptorModelLoader, Context context, Glide glide,
            RequestTracker requestTracker, Lifecycle lifecycle, RequestManager.OptionsApplier optionsApplier) {
        super(context, modelClass,
        		// 构建一个 FixedLoadProvider
                buildProvider(glide, streamModelLoader, fileDescriptorModelLoader, GifBitmapWrapper.class,
                        GlideDrawable.class, null),
                glide, requestTracker, lifecycle);
        // 2 个下载模块, 存储到自己的成员变量中
        this.streamModelLoader = streamModelLoader;
        this.fileDescriptorModelLoader = fileDescriptorModelLoader;
        ...
    }

	DrawableRequestBuilder(Context context, Class<ModelType> modelClass,
            LoadProvider<ModelType, ImageVideoWrapper, GifBitmapWrapper, GlideDrawable> loadProvider, Glide glide,
            RequestTracker requestTracker, Lifecycle lifecycle) {
        super(context, modelClass, loadProvider, GlideDrawable.class, glide, requestTracker, lifecycle);

        // 十字星消失动画工厂
        crossFade();
    }
	GenericRequestBuilder(Context context, Class<ModelType> modelClass,
            LoadProvider<ModelType, DataType, ResourceType, TranscodeType> loadProvider,
            Class<TranscodeType> transcodeClass, Glide glide, RequestTracker requestTracker, Lifecycle lifecycle) {
        this.context = context;
        // String.class
        this.modelClass = modelClass;
        // 转码类型
        this.transcodeClass = transcodeClass;
        // glide 单例
        this.glide = glide;
        this.requestTracker = requestTracker;
        // 生命周期
        this.lifecycle = lifecycle;
        // ChildLoadProvider
        this.loadProvider = loadProvider != null
                ? new ChildLoadProvider<ModelType, DataType, ResourceType, TranscodeType>(loadProvider) : null;

        ...
    }

	public final DrawableRequestBuilder<ModelType> crossFade() {
		// 十字星消失动画工厂
        super.animate(new DrawableCrossFadeFactory<GlideDrawable>());
        return this;
    }

	GenericRequestBuilder<ModelType, DataType, ResourceType, TranscodeType> animate(
            GlideAnimationFactory<TranscodeType> animationFactory) {
        this.animationFactory = animationFactory;

        return this;
    }
```
捋一下上面创建了哪些模块
* 2 个下载模块, StreamStringLoader 和 FileDescriptorStringLoader
* 十字星渐变动画工厂 DrawableCrossFadeFactory < GlideDrawable >
* modelClass , 类型是 String.class
* 转码类型 transcodeClass , 是 GlideDrawable.class
* 生命周期 lifecycle
* ChildLoadProvider

### [3.2.2] 创建下载模块1
创建下载模块1 StreamStringLoader
```java
// Glide

	public static <T> ModelLoader<T, InputStream> buildStreamModelLoader(Class<T> modelClass, Context context) {
        return buildModelLoader(modelClass, InputStream.class, context);
    }

    public static <T, Y> ModelLoader<T, Y> buildModelLoader(Class<T> modelClass, Class<Y> resourceClass,
            Context context) {
         ...
        return Glide.get(context).getLoaderFactory().buildModelLoader(modelClass, resourceClass);
    }

	private GenericLoaderFactory getLoaderFactory() {
        return loaderFactory;
    }
```
进入 GenericLoaderFactory.buildModelLoader 查看
> 留意一下这里的 modelClass == String.class , resourceClass == InputStream.class

```java
// GenericLoaderFactory

	public synchronized <T, Y> ModelLoader<T, Y> buildModelLoader(Class<T> modelClass, Class<Y> resourceClass) {
        ...
        final ModelLoaderFactory<T, Y> factory = getFactory(modelClass, resourceClass);
        ...
        result = factory.build(context, this);
        ...
        return result;
    }

	private <T, Y> ModelLoaderFactory<T, Y> getFactory(Class<T> modelClass, Class<Y> resourceClass) {
        ...
        	// 从 modelClassToResourceFactories map 里面读取 factory
            for (Class<? super T> registeredModelClass : modelClassToResourceFactories.keySet()) {
                ...
                if (registeredModelClass.isAssignableFrom(modelClass)) {
                    Map<Class/*Y*/, ModelLoaderFactory/*T, Y*/> currentResourceToFactories =
                            modelClassToResourceFactories.get(registeredModelClass);
                    if (currentResourceToFactories != null) {
                        result = currentResourceToFactories.get(resourceClass);
                        if (result != null) {
                            break;
                        }
                    }
                }
            }
        return result;
    }
```
找下 modelClassToResourceFactories.put 看看在哪赋的值
```java
// GenericLoaderFactory
	public synchronized <T, Y> ModelLoaderFactory<T, Y> register(Class<T> modelClass, Class<Y> resourceClass,
            ModelLoaderFactory<T, Y> factory) {
        ...
        modelClassToResourceFactories.put(modelClass, resourceToFactories);
        ...
        ModelLoaderFactory/*T, Y*/ previous = resourceToFactories.put(resourceClass, factory);
		...
        return previous;
    }

// Glide
	public <T, Y> void register(Class<T> modelClass, Class<Y> resourceClass, ModelLoaderFactory<T, Y> factory) {
        ModelLoaderFactory<T, Y> removed = loaderFactory.register(modelClass, resourceClass, factory);
        ...
    }

	Glide(Engine engine, MemoryCache memoryCache, BitmapPool bitmapPool, Context context, DecodeFormat decodeFormat) {
		register(File.class, ParcelFileDescriptor.class, new FileDescriptorFileLoader.Factory());
        register(File.class, InputStream.class, new StreamFileLoader.Factory());
        register(int.class, ParcelFileDescriptor.class, new FileDescriptorResourceLoader.Factory());
        register(int.class, InputStream.class, new StreamResourceLoader.Factory());
        register(Integer.class, ParcelFileDescriptor.class, new FileDescriptorResourceLoader.Factory());
        register(Integer.class, InputStream.class, new StreamResourceLoader.Factory());
        register(String.class, ParcelFileDescriptor.class, new FileDescriptorStringLoader.Factory());
        // 匹配到此 Factory
        register(String.class, InputStream.class, new StreamStringLoader.Factory());
        register(Uri.class, ParcelFileDescriptor.class, new FileDescriptorUriLoader.Factory());
        register(Uri.class, InputStream.class, new StreamUriLoader.Factory());
        register(URL.class, InputStream.class, new StreamUrlLoader.Factory());
        register(GlideUrl.class, InputStream.class, new HttpUrlGlideUrlLoader.Factory());
        register(byte[].class, InputStream.class, new StreamByteArrayLoader.Factory());
	}
```
> 留意一下这里的 modelClass == String.class , resourceClass == InputStream.class

可以看到在 Glide 单例构建的时候, 注册了一系列 Factory ,  根据上面我们传入的参数, 匹配到 **StreamStringLoader.Factory**, 回到上面我们开始找 Factory 的地方 ( *直接复制过来, 避免上下翻找文章* )
```java
// GenericLoaderFactory
	public synchronized <T, Y> ModelLoader<T, Y> buildModelLoader(Class<T> modelClass, Class<Y> resourceClass) {
        ...
        final ModelLoaderFactory<T, Y> factory = getFactory(modelClass, resourceClass);
        ...
        result = factory.build(context, this);
        ...
        return result;
    }

// StreamStringLoader.Factory
	public ModelLoader<String, InputStream> build(Context context, GenericLoaderFactory factories) {
            return new StreamStringLoader(factories.buildModelLoader(Uri.class, InputStream.class));
    }

	// 装饰者模式, 传入另一个 ModelLoader
	public StreamStringLoader(ModelLoader<Uri, InputStream> uriLoader) {
        super(uriLoader);
    }
```
* 查看 StreamStringLoader ( 也是 ModelLoader 类型 ) 的构造函数, 得, 装饰了另外一个 ModelLoader 类型 ( 功能: **数据下载** )
* 另外一个是谁 ? 追踪 GenericLoaderFactory.buildModelLoader(Uri.class, InputStream.class)
* 又回到上面找 Factory 的步骤, 根据 Glide 构造函数里面的注册代码得, Factory == new StreamUriLoader.Factory()

```java
// StreamUriLoader.Factory

	public ModelLoader<Uri, InputStream> build(Context context, GenericLoaderFactory factories) {
            return new StreamUriLoader(context, factories.buildModelLoader(GlideUrl.class, InputStream.class));
        }
```

* 又来了, 找 Factory , 根据 Glide 的注册代码得, Factory == HttpUrlGlideUrlLoader.Factory

```java
// HttpUrlGlideUrlLoader.Factory

        public ModelLoader<GlideUrl, InputStream> build(Context context, GenericLoaderFactory factories) {
            return new HttpUrlGlideUrlLoader(modelCache);
        }
```

* 找了 3 个 Factory, 梳理一下

```java
//Glide
	Glide.get(context).getLoaderFactory().buildModelLoader(String.class, InputStream.class)

// StreamStringLoader.Factory
    factories.buildModelLoader(Uri.class, InputStream.class)

// StreamUriLoader.Factory
    factories.buildModelLoader(GlideUrl.class, InputStream.class)

// HttpUrlGlideUrlLoader.Factory
	uriLoader = new HttpUrlGlideUrlLoader(modelCache)

// StreamStringLoader
	public StreamStringLoader(ModelLoader<Uri, InputStream> uriLoader) {
        super(uriLoader);
    }

// StringLoader
	private final ModelLoader<Uri, T> uriLoader;

    public StringLoader(ModelLoader<Uri, T> uriLoader) {
        this.uriLoader = uriLoader;
    }
```
* 构建了 HttpUrlGlideUrlLoader 和 StreamStringLoader 实例, 并将 HttpUrlGlideUrlLoader 存放在 StreamStringLoader 实例的 uriLoader 变量中

至此, **StreamStringLoader<String, InputStream> 创建完毕** 在其中保存了一个 HttpUrlGlideUrlLoader 实例
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210313135738439.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5,size_10,color_FFFFFF,t_70)


### [3.2.3] 创建下载模块2
FileDescriptorStringLoader , 这里回顾一下, 在 load 方法中有提到如下代码
```java
// RequestManager.load

	private <T> DrawableTypeRequest<T> loadGeneric(Class<T> modelClass) {
        ModelLoader<T, InputStream> streamModelLoader = Glide.buildStreamModelLoader(modelClass, context);
        ModelLoader<T, ParcelFileDescriptor> fileDescriptorModelLoader =
                Glide.buildFileDescriptorModelLoader(modelClass, context);
        ...

        return optionsApplier.apply(
                new DrawableTypeRequest<T>(modelClass, streamModelLoader, fileDescriptorModelLoader, context,
                        glide, requestTracker, lifecycle, optionsApplier));
    }
```
> 留意一下这里的 modelClass == String.class

下面我们就来查看 FileDescriptorStringLoader 的创建
```java
// Glide

	public static <T> ModelLoader<T, ParcelFileDescriptor> buildFileDescriptorModelLoader(Class<T> modelClass,
            Context context) {
        return buildModelLoader(modelClass, ParcelFileDescriptor.class, context);
    }

    public static <T, Y> ModelLoader<T, Y> buildModelLoader(Class<T> modelClass, Class<Y> resourceClass,
            Context context) {
         ...
         // modelClass == String.class , resourceClass == ParcelFileDescriptor.class
        return Glide.get(context).getLoaderFactory().buildModelLoader(modelClass, resourceClass);
    }

	Glide(Engine engine, MemoryCache memoryCache, BitmapPool bitmapPool, Context context, DecodeFormat decodeFormat) {
		register(File.class, ParcelFileDescriptor.class, new FileDescriptorFileLoader.Factory());
        register(File.class, InputStream.class, new StreamFileLoader.Factory());
        register(int.class, ParcelFileDescriptor.class, new FileDescriptorResourceLoader.Factory());
        register(int.class, InputStream.class, new StreamResourceLoader.Factory());
        register(Integer.class, ParcelFileDescriptor.class, new FileDescriptorResourceLoader.Factory());
        register(Integer.class, InputStream.class, new StreamResourceLoader.Factory());
        // 匹配到此 Factory
        register(String.class, ParcelFileDescriptor.class, new FileDescriptorStringLoader.Factory());
        register(String.class, InputStream.class, new StreamStringLoader.Factory());
        register(Uri.class, ParcelFileDescriptor.class, new FileDescriptorUriLoader.Factory());
        register(Uri.class, InputStream.class, new StreamUriLoader.Factory());
        register(URL.class, InputStream.class, new StreamUrlLoader.Factory());
        register(GlideUrl.class, InputStream.class, new HttpUrlGlideUrlLoader.Factory());
        register(byte[].class, InputStream.class, new StreamByteArrayLoader.Factory());
	}
```
有了上面 StreamStringLoader 创建分析, 找 Factory 相信大家都很熟练了, 直接上结果
```java
// FileDescriptorStringLoader.Factory
	public ModelLoader<String, ParcelFileDescriptor> build(Context context, GenericLoaderFactory factories) {
            return new FileDescriptorStringLoader(factories.buildModelLoader(Uri.class, ParcelFileDescriptor.class));
    }

// FileDescriptorUriLoader.Factory
	public ModelLoader<Uri, ParcelFileDescriptor> build(Context context, GenericLoaderFactory factories) {
            return new FileDescriptorUriLoader(context,
					// 无匹配项, 返回 null
					factories.buildModelLoader(GlideUrl.class,
                    ParcelFileDescriptor.class));

    }

// FileDescriptorStringLoader extends StringLoader
	public FileDescriptorStringLoader(ModelLoader<Uri, ParcelFileDescriptor> uriLoader) {
		// 这里 uriLoader == null
        super(uriLoader);
    }

// StringLoader
	private final ModelLoader<Uri, T> uriLoader;

    public StringLoader(ModelLoader<Uri, T> uriLoader) {
        this.uriLoader = uriLoader;
    }
```
* 至此, **FileDescriptorStringLoader<String, ParcelFileDescriptor>** 创建完毕,
* FileDescriptorStringLoader.uriLoader == FileDescriptorUriLoader
* FileDescriptorUriLoader.uriLoader == null
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210313161815572.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5,size_16,color_FFFFFF,t_70)
> FileDescriptorStringLoader 和 StreamStringLoader 是 ModelLoader ( [UML类图结构](https://img-blog.csdnimg.cn/20210311202331373.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5) )

2 个下载模块都创建完了, 我们继续跟进 DrawableTypeRequest 的创建

### [3.2.4] 创建工具集合1
创建工具集合  ChildLoadProvider , 接着 load() 的代码看

```java
// DrawableTypeRequest

	buildProvider(glide, streamModelLoader, fileDescriptorModelLoader, GifBitmapWrapper.class,
                        GlideDrawable.class, null)

	private static <A, Z, R> FixedLoadProvider<A, ImageVideoWrapper, Z, R> buildProvider(Glide glide,
            ModelLoader<A, InputStream> streamModelLoader,
            ModelLoader<A, ParcelFileDescriptor> fileDescriptorModelLoader, Class<Z> resourceClass,
            Class<R> transcodedClass,
            ResourceTranscoder<Z, R> transcoder) {
        ...

        if (transcoder == null) {
        	// 初始化转码工具
            transcoder = glide.buildTranscoder(resourceClass, transcodedClass);
        }
        DataLoadProvider<ImageVideoWrapper, Z> dataLoadProvider = glide.buildDataProvider(ImageVideoWrapper.class,
                resourceClass);
        ImageVideoModelLoader<A> modelLoader = new ImageVideoModelLoader<A>(streamModelLoader,
                fileDescriptorModelLoader);
        return new FixedLoadProvider<A, ImageVideoWrapper, Z, R>(modelLoader, transcoder, dataLoadProvider);
    }
```
> 注意
> resourceClass == GifBitmapWrapper.class , transcodedClass == GlideDrawable.class
> Z == GifBitmapWrapper.class , R == GlideDrawable.class
> A == String.class
>
* 构建了一个转码工具 GifBitmapWrapperDrawableTranscoder < GifBitmapWrapper,GlideDrawable >
* 构建 DataLoadProvider < ImageVideoWrapper , GifBitmapWrapper >
* 构建 ImageVideoModelLoader < String >
* 用上面的三个参数 **构建 FixedLoadProvider < String , ImageVideoWrapper , GifBitmapWrapper , GlideDrawable>**
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210313140321453.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5,size_20,color_FFFFFF,t_70)

#### 构建工具集合(转码1)
构建转码 ResourceTranscoder 实例

```java
// DrawableTypeRequest
	transcoder = glide.buildTranscoder(resourceClass, transcodedClass);

// Glide
	<Z, R> ResourceTranscoder<Z, R> buildTranscoder(Class<Z> decodedClass, Class<R> transcodedClass) {
		// 看看 transcoderRegistry 在哪边注册的
        return transcoderRegistry.get(decodedClass, transcodedClass);
    }

   	Glide(Engine engine, MemoryCache memoryCache, BitmapPool bitmapPool, Context context, DecodeFormat decodeFormat) {
		...
		// 和下载模块一样, 也是在 Glide 单例实例化的时候注册的
		transcoderRegistry.register(Bitmap.class, GlideBitmapDrawable.class,
                new GlideBitmapDrawableTranscoder(context.getResources(), bitmapPool));
        transcoderRegistry.register(GifBitmapWrapper.class, GlideDrawable.class,
                new GifBitmapWrapperDrawableTranscoder(
                        new GlideBitmapDrawableTranscoder(context.getResources(), bitmapPool)));
		...
	}

// GifBitmapWrapperDrawableTranscoder
	public GifBitmapWrapperDrawableTranscoder(
            ResourceTranscoder<Bitmap, GlideBitmapDrawable> bitmapDrawableResourceTranscoder) {
        this.bitmapDrawableResourceTranscoder = bitmapDrawableResourceTranscoder;
    }

// GlideBitmapDrawableTranscoder
	public GlideBitmapDrawableTranscoder(Resources resources, BitmapPool bitmapPool) {
        this.resources = resources;
        this.bitmapPool = bitmapPool;
    }
```
* 最后构建了一个 GifBitmapWrapperDrawableTranscoder < GifBitmapWrapper,GlideDrawable >实例;
* 并且包含一个 GlideBitmapDrawableTranscoder 实例, 作用是 GifBitmapWrapper 转换成 GlideDrawable
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210313140440482.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5,size_16,color_FFFFFF,t_70)

#### 构建工具集合(编/解码1)
构建工具集合(子类) ImageVideoGifDrawableLoadProvider < ImageVideoWrapper , GifBitmapWrapper >

```java
// DrawableTypeRequest

	/* resourceClass == GifBitmapWrapper.class */
	DataLoadProvider<ImageVideoWrapper, Z> dataLoadProvider = glide.buildDataProvider(ImageVideoWrapper.class,
                resourceClass);

// Glide
	<T, Z> DataLoadProvider<T, Z> buildDataProvider(Class<T> dataClass, Class<Z> decodedClass) {
		// 似曾相识? 没错也是在 Glide 单例初始化的时候注册的
        return dataLoadProviderRegistry.get(dataClass, decodedClass);
    }

	Glide(Engine engine, MemoryCache memoryCache, BitmapPool bitmapPool, Context context, DecodeFormat decodeFormat) {
      	...
        dataLoadProviderRegistry = new DataLoadProviderRegistry();

        StreamBitmapDataLoadProvider streamBitmapLoadProvider =
                new StreamBitmapDataLoadProvider(bitmapPool, decodeFormat);
        dataLoadProviderRegistry.register(InputStream.class, Bitmap.class, streamBitmapLoadProvider);

        FileDescriptorBitmapDataLoadProvider fileDescriptorLoadProvider =
                new FileDescriptorBitmapDataLoadProvider(bitmapPool, decodeFormat);
        dataLoadProviderRegistry.register(ParcelFileDescriptor.class, Bitmap.class, fileDescriptorLoadProvider);

        ImageVideoDataLoadProvider imageVideoDataLoadProvider =
                new ImageVideoDataLoadProvider(streamBitmapLoadProvider, fileDescriptorLoadProvider);
        dataLoadProviderRegistry.register(ImageVideoWrapper.class, Bitmap.class, imageVideoDataLoadProvider);

        GifDrawableLoadProvider gifDrawableLoadProvider =
                new GifDrawableLoadProvider(context, bitmapPool);
        dataLoadProviderRegistry.register(InputStream.class, GifDrawable.class, gifDrawableLoadProvider);

		// 匹配到这个
        dataLoadProviderRegistry.register(ImageVideoWrapper.class, GifBitmapWrapper.class,
                new ImageVideoGifDrawableLoadProvider(imageVideoDataLoadProvider, gifDrawableLoadProvider, bitmapPool));

        dataLoadProviderRegistry.register(InputStream.class, File.class, new StreamFileDataLoadProvider());
        ...
    }

// ImageVideoGifDrawableLoadProvider
	public ImageVideoGifDrawableLoadProvider(DataLoadProvider<ImageVideoWrapper, Bitmap> bitmapProvider,
            DataLoadProvider<InputStream, GifDrawable> gifProvider, BitmapPool bitmapPool) {

        final GifBitmapWrapperResourceDecoder decoder = new GifBitmapWrapperResourceDecoder(
                bitmapProvider.getSourceDecoder(),
                gifProvider.getSourceDecoder(),
                bitmapPool
        );
        // 解码工具
        cacheDecoder = new FileToStreamDecoder<GifBitmapWrapper>(new GifBitmapWrapperStreamResourceDecoder(decoder));
        sourceDecoder = decoder;
        // 编码工具
        encoder = new GifBitmapWrapperResourceEncoder(bitmapProvider.getEncoder(), gifProvider.getEncoder());

        //TODO: what about the gif provider?
        sourceEncoder = bitmapProvider.getSourceEncoder();
    }
```
> DataLoadProvider和我们最终需要创建的ChildLoadProvider是什么关系?  [UML类图参考](https://img-blog.csdnimg.cn/20210313124817824.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5)

1. 初始化解码工具 cacheDecoder: FileToStreamDecoder , sourceDecoder: GifBitmapWrapperResourceDecoder
2. 初始化编码工具 encoder: GifBitmapWrapperResourceEncoder , sourceEncoder: bitmapProvider.getSourceEncoder()

看看具体都是怎么初始化的
```java
// Glide 构造函数
	StreamBitmapDataLoadProvider streamBitmapLoadProvider =
                new StreamBitmapDataLoadProvider(bitmapPool, decodeFormat);
    FileDescriptorBitmapDataLoadProvider fileDescriptorLoadProvider =
                new FileDescriptorBitmapDataLoadProvider(bitmapPool, decodeFormat);
	ImageVideoDataLoadProvider imageVideoDataLoadProvider =
                new ImageVideoDataLoadProvider(streamBitmapLoadProvider, fileDescriptorLoadProvider);
    ...
    GifDrawableLoadProvider gifDrawableLoadProvider =
                new GifDrawableLoadProvider(context, bitmapPool);
    ...
	dataLoadProviderRegistry.register(ImageVideoWrapper.class, GifBitmapWrapper.class,
                new ImageVideoGifDrawableLoadProvider(imageVideoDataLoadProvider, gifDrawableLoadProvider, bitmapPool));

// StreamBitmapDataLoadProvider 构造
	public StreamBitmapDataLoadProvider(BitmapPool bitmapPool, DecodeFormat decodeFormat) {
        sourceEncoder = new StreamEncoder();
        decoder = new StreamBitmapDecoder(bitmapPool, decodeFormat);
        encoder = new BitmapEncoder();
        cacheDecoder = new FileToStreamDecoder<Bitmap>(decoder);
    }

// FileDescriptorBitmapDataLoadProvider 构造
	public FileDescriptorBitmapDataLoadProvider(BitmapPool bitmapPool, DecodeFormat decodeFormat) {
        cacheDecoder = new FileToStreamDecoder<Bitmap>(new StreamBitmapDecoder(bitmapPool, decodeFormat));
        sourceDecoder = new FileDescriptorBitmapDecoder(bitmapPool, decodeFormat);
        encoder = new BitmapEncoder();
        sourceEncoder = NullEncoder.get();
    }

// ImageVideoDataLoadProvider 构造
	public ImageVideoDataLoadProvider(DataLoadProvider<InputStream, Bitmap> streamBitmapProvider,
            DataLoadProvider<ParcelFileDescriptor, Bitmap> fileDescriptorBitmapProvider) {
        encoder = streamBitmapProvider.getEncoder();
        sourceEncoder = new ImageVideoWrapperEncoder(streamBitmapProvider.getSourceEncoder(),
                fileDescriptorBitmapProvider.getSourceEncoder());
        cacheDecoder = streamBitmapProvider.getCacheDecoder();
        sourceDecoder = new ImageVideoBitmapDecoder(streamBitmapProvider.getSourceDecoder(),
                fileDescriptorBitmapProvider.getSourceDecoder());
    }

// GifDrawableLoadProvider 构造
	public GifDrawableLoadProvider(Context context, BitmapPool bitmapPool) {
        decoder = new GifResourceDecoder(context, bitmapPool);
        cacheDecoder = new FileToStreamDecoder<GifDrawable>(decoder);
        encoder = new GifResourceEncoder(bitmapPool);
        sourceEncoder = new StreamEncoder();
    }
```
根据上面的代码我们来换算一下 ImageVideoGifDrawableLoadProvider 的构造
```java
// ImageVideoGifDrawableLoadProvider 构造函数

	/*
		bitmapProvider >>> imageVideoDataLoadProvider
		;
		gifProvider >>> gifDrawableLoadProvider
	*/
	public ImageVideoGifDrawableLoadProvider(DataLoadProvider<ImageVideoWrapper, Bitmap> imageVideoDataLoadProvider,
            DataLoadProvider<InputStream, GifDrawable> gifDrawableLoadProvider, BitmapPool bitmapPool) {

        final GifBitmapWrapperResourceDecoder decoder = new GifBitmapWrapperResourceDecoder(
                imageVideoDataLoadProvider.getSourceDecoder(),
                gifDrawableLoadProvider.getSourceDecoder(),
                bitmapPool
        );
        cacheDecoder = new FileToStreamDecoder<GifBitmapWrapper>(new GifBitmapWrapperStreamResourceDecoder(decoder));
        sourceDecoder = decoder;
        encoder = new GifBitmapWrapperResourceEncoder(imageVideoDataLoadProvider.getEncoder(), gifDrawableLoadProvider.getEncoder());
        sourceEncoder = imageVideoDataLoadProvider.getSourceEncoder();
    }

	/*
		imageVideoDataLoadProvider.getSourceDecoder
		>>>
		new ImageVideoBitmapDecoder(streamBitmapProvider.getSourceDecoder(),fileDescriptorBitmapProvider.getSourceDecoder())
        >>>
        new ImageVideoBitmapDecoder(new StreamBitmapDecoder(bitmapPool, decodeFormat), new FileDescriptorBitmapDecoder(bitmapPool, decodeFormat))
		;
		gifDrawableLoadProvider.getSourceDecoder() >>> new GifResourceDecoder(context, bitmapPool)
		;
		imageVideoDataLoadProvider.getEncoder() >>> streamBitmapProvider.getEncoder() >>> new BitmapEncoder()
		;
		gifDrawableLoadProvider.getEncoder() >>> new GifResourceEncoder(bitmapPool)
		;
		imageVideoDataLoadProvider.getSourceEncoder()
		>>>
		new ImageVideoWrapperEncoder(streamBitmapProvider.getSourceEncoder(), fileDescriptorBitmapProvider.getSourceEncoder())
        >>>
        new ImageVideoWrapperEncoder(new StreamEncoder(), NullEncoder.get())
	*/
	public ImageVideoGifDrawableLoadProvider(DataLoadProvider<ImageVideoWrapper, Bitmap> imageVideoDataLoadProvider,
            DataLoadProvider<InputStream, GifDrawable> gifDrawableLoadProvider, BitmapPool bitmapPool) {

        final GifBitmapWrapperResourceDecoder decoder = new GifBitmapWrapperResourceDecoder(
                new ImageVideoBitmapDecoder(new StreamBitmapDecoder(bitmapPool, decodeFormat), new FileDescriptorBitmapDecoder(bitmapPool, decodeFormat)),
                new GifResourceDecoder(context, bitmapPool),
                bitmapPool
        );
        cacheDecoder = new FileToStreamDecoder<GifBitmapWrapper>(new GifBitmapWrapperStreamResourceDecoder(decoder));
        sourceDecoder = decoder;
        encoder = new GifBitmapWrapperResourceEncoder(new BitmapEncoder(), new GifResourceEncoder(bitmapPool));
        sourceEncoder = new ImageVideoWrapperEncoder(new StreamEncoder(), NullEncoder.get());
    }
```
总结一下, 看看 ImageVideoGifDrawableLoadProvider 都初始化了哪些内容
* FileToStreamDecoder 存储在 cacheDecoder
* GifBitmapWrapperStreamResourceDecoder 存储在 sourceDecoder
* GifBitmapWrapperResourceEncoder 存储在 encoder
* ImageVideoWrapperEncoder 存储在  sourceEncoder
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210313164653242.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5,size_16,color_FFFFFF,t_70)


回到 **[3.2.4] ChildLoadProvider 工具集合的创建** , 还差个工具没有构建 ImageVideoModelLoader < String >
```java
// DrawableTypeRequest

	buildProvider(glide, streamModelLoader, fileDescriptorModelLoader, GifBitmapWrapper.class, GlideDrawable.class, null)

	private static <A, Z, R> FixedLoadProvider<A, ImageVideoWrapper, Z, R> buildProvider(Glide glide,
            ModelLoader<A, InputStream> streamModelLoader,
            ModelLoader<A, ParcelFileDescriptor> fileDescriptorModelLoader, Class<Z> resourceClass,
            Class<R> transcodedClass,
            ResourceTranscoder<Z, R> transcoder) {
        ...
        ImageVideoModelLoader<A> modelLoader = new ImageVideoModelLoader<A>(streamModelLoader,
                fileDescriptorModelLoader);
        ...
    }
```
> 可能之前的创建过程大家已经忘记了, 需要补充的是:
> * streamModelLoader 可以参考目录 **创建下载模块1**
> * fileDescriptorModelLoader 可以参考 **创建下载模块2**
> * A 是 **String.class**

#### 构建工具集合(下载3)

 ```java
 // ImageVideoModelLoader

 	public ImageVideoModelLoader(ModelLoader<A, InputStream> streamLoader,
            ModelLoader<A, ParcelFileDescriptor> fileDescriptorLoader) {
        ...
        this.streamLoader = streamLoader;
        this.fileDescriptorLoader = fileDescriptorLoader;
    }
 ```
所以最后创建的是 ImageVideoModelLoader < String > , 其下封装了两个之前创建的下载模块
* 下载模块1 StreamStringLoader
* 下载模块2 FileDescriptorStringLoader
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210313141039582.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5,size_16,color_FFFFFF,t_70)

## [3.3] load() 小结
load() 从头至尾只构建了 **Request(用户)** 这一个实例 , 并把 **一系列工具** 打包进这个实例
 1. 创建: Request(用户) 实例 , [GenericRequestBuilder 类型](https://img-blog.csdnimg.cn/20210311195313868.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5)
 2. 打包: 下载工具1 ( [ModelLoader 类型](https://img-blog.csdnimg.cn/20210311202331373.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5) )
 3. 打包: 下载工具2 ( [ModelLoader 类型](https://img-blog.csdnimg.cn/20210311202331373.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5) )
 4. 打包: 工具集合1 ( [DataLoadProvider 类型](https://img-blog.csdnimg.cn/20210313142352405.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5) )
		a. 打包: 转码1 ( GifBitmapWrapperDrawableTranscoder 类型 )
		b. 打包: 编/解码1 ( ImageVideoGifDrawableLoadProvider < ImageVideoWrapper , GifBitmapWrapper > 类型 )
		c. 打包: 下载3 ( ImageVideoModelLoader < String > 类型)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210313143350768.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5,size_16,color_FFFFFF,t_70)
现在再来看这张流程图解, 是不是清晰一点?
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210313143433730.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5,size_20,color_FFFFFF,t_70)

## [3.4] into()
**工具**都备齐了, 下面开始走 **构建Request(真实)>>下载 >>> 解码 >>> 转码 >>> 加载** 流程
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210313143806550.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5,size_16,color_FFFFFF,t_70)

### [3.4.1] 构建Request(真实)

```java
// GenericRequestBuilder (Request用户实例)
	public Target<TranscodeType> into(ImageView view) {
        ...
        // 前文创建 Request(用户) 提过 , GlideDrawable.class = transcodeClass
        return into(glide.buildImageViewTarget(view, transcodeClass));
    }

// Glide
	<R> Target<R> buildImageViewTarget(ImageView imageView, Class<R> transcodedClass) {
        return imageViewTargetFactory.buildTarget(imageView, transcodedClass);
    }

// ImageViewTargetFactory
	public <Z> Target<Z> buildTarget(ImageView view, Class<Z> clazz) {
        if (GlideDrawable.class.isAssignableFrom(clazz)) {
            return (Target<Z>) new GlideDrawableImageViewTarget(view);
        }
        ...
    }
```
创建了一个 GlideDrawableImageViewTarget < GlideDrawable > 实例 , 再往下跟进 into
```java
// GenericRequestBuilder (Request用户实例)

	// TranscodeType = GlideDrawable
	public <Y extends Target<TranscodeType>> Y into(Y target) {
        ...
        // 这里开始构建 Request(真实)!!!
        Request request = buildRequest(target);
        target.setRequest(request);
        lifecycle.addListener(target);
        requestTracker.runRequest(request);
        return target;
    }

	private Request buildRequest(Target<TranscodeType> target) {
        ...
        return buildRequestRecursive(target, null);
    }

	private Request buildRequestRecursive(Target<TranscodeType> target, ThumbnailRequestCoordinator parentCoordinator) {
        ...
        return obtainRequest(target, sizeMultiplier, priority, parentCoordinator);
    }

	private Request obtainRequest(Target<TranscodeType> target, float sizeMultiplier, Priority priority,
            RequestCoordinator requestCoordinator) {
        return GenericRequest.obtain(
                loadProvider,
                model,
                signature,
                context,
                priority,
                target,
                sizeMultiplier,
                placeholderDrawable,
                placeholderId,
                errorPlaceholder,
                errorId,
                fallbackDrawable,
                fallbackResource,
                requestListener,
                requestCoordinator,
                glide.getEngine(),
                transformation,
                transcodeClass,
                isCacheable,
                animationFactory,
                overrideWidth,
                overrideHeight,
                diskCacheStrategy);
    }
```
以上, Request(真实) ( [GenericRequest 类型](https://img-blog.csdnimg.cn/20210313145538902.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5) ) 构建完毕 , 可以看出它包含了所有 Glide 加载所需的工具 , 参数等
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210313145821672.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5,size_16,color_FFFFFF,t_70)
### [3.4.2] 下载
```java
// GenericRequestBuilder (Request用户实例)

	// TranscodeType = GlideDrawable
	public <Y extends Target<TranscodeType>> Y into(Y target) {
        ...
        // 这里开始构建 Request(真实)!!!
        Request request = buildRequest(target);
        target.setRequest(request);
        lifecycle.addListener(target);
        // 开始任务啦
        requestTracker.runRequest(request);
        return target;
    }

// RequestTracker
	public void runRequest(Request request) {
        ...
        request.begin();
        ...
    }
```
已知 Request 是 GenericRequest 类型
```java
// GenericRequest
	public void begin() {
        ...
        if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
        	// 最终走这个分支
            onSizeReady(overrideWidth, overrideHeight);
        } else {
        	// 异步获取 size , 最终回调 onSizeReady 方法
            target.getSize(this);
        }
		...
    }
```
> Glide 会按照 ImageView 控件的大小来计算所需要的图片的大小, 尽量减少内存开支
> 如果对于 异步计算界面 ImageView 组件大小感兴趣 , 可以看看这篇文章 [Android Glide 3.7.0 源码解析(五) , 如何获得ImageView的宽高](/2021/03/20/markdown-glide3.7.0_5/index.html)

下面进入 onSizeReady 继续
```java
// GenericRequest

	public void onSizeReady(int width, int height) {
        ...
        width = Math.round(sizeMultiplier * width);
        height = Math.round(sizeMultiplier * height);

        ModelLoader<A, T> modelLoader = loadProvider.getModelLoader();
        final DataFetcher<T> dataFetcher = modelLoader.getResourceFetcher(model, width, height);

        ResourceTranscoder<Z, R> transcoder = loadProvider.getTranscoder();

        loadedFromMemoryCache = true;
        // 这里
        loadStatus = engine.load(signature, width, height, dataFetcher, loadProvider, transformation, transcoder,
                priority, isMemoryCacheable, diskCacheStrategy, this);
        loadedFromMemoryCache = resource != null;

    }
```
可以翻看 **Request(真实)的构建**得, engine 是
```java
//
	private Request obtainRequest(Target<TranscodeType> target, float sizeMultiplier, Priority priority,
            RequestCoordinator requestCoordinator) {
        return GenericRequest.obtain(
        				...
						glide.getEngine()
						...
						);

// Glide
	Engine getEngine() {
        return engine;
    }

	Glide(Engine engine, MemoryCache memoryCache, BitmapPool bitmapPool, Context context, DecodeFormat decodeFormat) {
        this.engine = engine;
    }

// GlideBuilder
	Glide createGlide(){
		engine = new Engine(memoryCache, diskCacheFactory, diskCacheService, sourceService);
		return new Glide(engine, memoryCache, bitmapPool, context, decodeFormat);
	}
```
下面进入 Engine 的 load 方法
```java
// Engine

	public <T, Z, R> LoadStatus load(Key signature, int width, int height, DataFetcher<T> fetcher,
            DataLoadProvider<T, Z> loadProvider, Transformation<Z> transformation, ResourceTranscoder<Z, R> transcoder,
            Priority priority, boolean isMemoryCacheable, DiskCacheStrategy diskCacheStrategy, ResourceCallback cb) {
        ...
        EngineJob engineJob = engineJobFactory.build(key, isMemoryCacheable);
        DecodeJob<T, Z, R> decodeJob = new DecodeJob<T, Z, R>(key, width, height, fetcher, loadProvider, transformation,
                transcoder, diskCacheProvider, diskCacheStrategy, priority);
        EngineRunnable runnable = new EngineRunnable(engineJob, decodeJob, priority);
        jobs.put(key, engineJob);
        engineJob.addCallback(cb);
        // 调用 EngineRunnable.run
        engineJob.start(runnable);
		...
    }
```
> 根据 **构建的 Request ( 用户 )** FixedLoadProvider < String , ImageVideoWrapper , GifBitmapWrapper , GlideDrawable> 得
> * T : ImageVideoWrapper
> * Z : GifBitmapWrapper
>
> 根据 **创建的转码1** GifBitmapWrapperDrawableTranscoder < GifBitmapWrapper,GlideDrawable > 得
> * R : GlideDrawable

最终走到 EngineRunnable 的 run 方法
```java
// EngineRunnable

	public void run() {
        ...
        resource = decode();
        ...
    }

	private Resource<?> decode() throws Exception {
        ...
        return decodeFromSource();
        ...
    }

	private Resource<?> decodeFromSource() throws Exception {
        return decodeJob.decodeFromSource();
    }

// DecodeJob

	public Resource<Z> decodeFromSource() throws Exception {
	    // 分支1, 下载 && 解码
        Resource<T> decoded = decodeSource();
        // 分支2, 转码
        return transformEncodeAndTranscode(decoded);
    }

	// 此小节我们先分析 下载
	private Resource<T> decodeSource() throws Exception {
        ...
        final A data = fetcher.loadData(priority);
        decoded = decodeFromSourceData(data);
        fetcher.cleanup();
        ...
        return decoded;
    }
```
>注意: 线程已切换到子线程

来看看 fetcher 在哪边赋的值
```java
// GenericRequest
	public void onSizeReady(int width, int height) {
		ModelLoader<A, T> modelLoader = loadProvider.getModelLoader();
        final DataFetcher<T> dataFetcher = modelLoader.getResourceFetcher(model, width, height);
        // 这里
        loadStatus = engine.load(..., dataFetcher...);
    }

// Engine
	public <T, Z, R> LoadStatus load(Key signature, int width, int height, DataFetcher<T> fetcher,
            DataLoadProvider<T, Z> loadProvider, Transformation<Z> transformation, ResourceTranscoder<Z, R> transcoder,
            Priority priority, boolean isMemoryCacheable, DiskCacheStrategy diskCacheStrategy, ResourceCallback cb) {
        ...
        // 这里
        DecodeJob<T, Z, R> decodeJob = new DecodeJob<T, Z, R>(..., fetcher, ...);
		...
    }
```
根据 **创建的集合工具1中的下载3** ImageVideoModelLoader < String > 得, 调用的是ImageVideoModelLoader.getResourceFetcher
```java
// ImageVideoModelLoader

	public DataFetcher<ImageVideoWrapper> getResourceFetcher(A model, int width, int height) {
        ...
        return new ImageVideoFetcher(streamFetcher, fileDescriptorFetcher);
        ...
    }
```
* 根据**创建下载3**过程得知, streamFetcher 是 **下载1** , fileDescriptorFetcher 是 **下载2** ,
*  那么 fetcher 是 **ImageVideoFetcher**

现在再回到调用 fetcher 的部分
```java
// DecodeJob

	private Resource<T> decodeSource() throws Exception {
        ...
        //
        final A data = fetcher.loadData(priority);
        decoded = decodeFromSourceData(data);
        fetcher.cleanup();
        ...
        return decoded;
    }

// ImageVideoFetcher

	public ImageVideoWrapper loadData(Priority priority) throws Exception {
            InputStream is = null;
            ...
            // 调用下载1的getResourceFetcher.loadData
            is = streamFetcher.loadData(priority);
            ...
            ParcelFileDescriptor fileDescriptor = null;
            ...
            // 调用下载2的getResourceFetcher.loadData
            fileDescriptor = fileDescriptorFetcher.loadData(priority);
            ....
            return new ImageVideoWrapper(is, fileDescriptor);
        }
```
先看 **下载1**的 loadData
```java
// StreamStringLoader

	public DataFetcher<T> getResourceFetcher(String model, int width, int height) {
        Uri uri;
        if (TextUtils.isEmpty(model)) {
            return null;
        } else if (model.startsWith("/")) {
            uri = toFileUri(model);
        } else {
            uri = Uri.parse(model);
            final String scheme = uri.getScheme();
            if (scheme == null) {
                uri = toFileUri(model);
            }
        }

		// uriLoader 是一个 HttpUrlGlideUrlLoader 下载1构建的时候有提到
        return uriLoader.getResourceFetcher(uri, width, height);
    }

// HttpUrlGlideUrlLoader

	public DataFetcher<InputStream> getResourceFetcher(GlideUrl model, int width, int height) {
        ...
        return new HttpUrlFetcher(url);
    }

// HttpUrlFetcher
	public InputStream loadData(Priority priority) throws Exception {
        return loadDataWithRedirects(glideUrl.toURL(), 0 /*redirects*/, null /*lastUrl*/, glideUrl.getHeaders());
    }

	// 终于找到了, 一个标准的 HttpURLConnection 下载
	private InputStream loadDataWithRedirects(URL url, int redirects, URL lastUrl, Map<String, String> headers)
            throws IOException {
        ...
        urlConnection = connectionFactory.build(url);
        for (Map.Entry<String, String> headerEntry : headers.entrySet()) {
          urlConnection.addRequestProperty(headerEntry.getKey(), headerEntry.getValue());
        }
        urlConnection.setConnectTimeout(2500);
        urlConnection.setReadTimeout(2500);
        urlConnection.setUseCaches(false);
        urlConnection.setDoInput(true);

        // Connect explicitly to avoid errors in decoders if connection fails.
        urlConnection.connect();
        if (isCancelled) {
            return null;
        }
        final int statusCode = urlConnection.getResponseCode();
        if (statusCode / 100 == 2) {
        	// 读取流
            return getStreamForSuccessfulRequest(urlConnection);
        }
        ...
    }
	private InputStream getStreamForSuccessfulRequest(HttpURLConnection urlConnection)
            throws IOException {
        ...
        stream = urlConnection.getInputStream();
        ...
        return stream;
    }
```
* **下载1**的下载任务看完, 就是一个标准的 HttpURLConnection 下载
* 下面来看看 **下载2** 都干了啥?

**下载2**类型是 **FileDescriptorStringLoader<String, ParcelFileDescriptor>**
![在这里插入图片描述](https://img-blog.csdnimg.cn/2021031316185379.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5,size_16,color_FFFFFF,t_70)
```java
// FileDescriptorStringLoader
	public DataFetcher<T> getResourceFetcher(String model, int width, int height) {
        Uri uri;
        if (TextUtils.isEmpty(model)) {
            return null;
        } else if (model.startsWith("/")) {
            uri = toFileUri(model);
        } else {
            uri = Uri.parse(model);
            final String scheme = uri.getScheme();
            if (scheme == null) {
                uri = toFileUri(model);
            }
        }

        return uriLoader.getResourceFetcher(uri, width, height);
    }

// FileDescriptorUriLoader
	public final DataFetcher<T> getResourceFetcher(Uri model, int width, int height) {
        final String scheme = model.getScheme();

        DataFetcher<T> result = null;
        if (isLocalUri(scheme)) {
            if (AssetUriParser.isAssetUri(model)) {
                String path = AssetUriParser.toAssetPath(model);
                result = getAssetPathFetcher(context, path);
            } else {
                result = getLocalUriFetcher(context, model);
            }
        } else if (urlLoader != null && ("http".equals(scheme) || "https".equals(scheme))) {
            result = urlLoader.getResourceFetcher(new GlideUrl(model.toString()), width, height);
        }

        return result;
    }
```
* 得 **下载2**的 loaData 最后返回 null , 因为我们要下载的是一个网络资源
* 回到上面**下载3** 的下载

```java
// ImageVideoFetcher

	public ImageVideoWrapper loadData(Priority priority) throws Exception {
            InputStream is = null;
            ...
            // 调用下载1的getResourceFetcher.loadData
            is = streamFetcher.loadData(priority);
            ...
            ParcelFileDescriptor fileDescriptor = null;
            ...
            // 调用下载2的getResourceFetcher.loadData
            fileDescriptor = fileDescriptorFetcher.loadData(priority);
            ....
            return new ImageVideoWrapper(is, fileDescriptor);
        }
```
* **下载3**的下载结果为: new ImageVideoWrapper("下载1下载的流", null)
下载这一小节结束, 下一个流程: **解码**

### [3.4.3] 解码
让我们先回到上面刚刚开始切换线程的地方

```java
// DecodeJob

	public Resource<Z> decodeFromSource() throws Exception {
	    // 分支1, 下载 && 解码
        Resource<T> decoded = decodeSource();
        // 分支2, 转码
        return transformEncodeAndTranscode(decoded);
    }

	private Resource<T> decodeSource() throws Exception {
        Resource<T> decoded = null;
        ...
        final A data = fetcher.loadData(priority);
        ...
        decoded = decodeFromSourceData(data);
        ...
        fetcher.cleanup();
        return decoded;
    }
```
* 现在我们来看看**分支1 解码**

```java
// DecodeJob

	private Resource<T> decodeFromSourceData(A data) throws IOException {
        final Resource<T> decoded;
        ...
        decoded = loadProvider.getSourceDecoder().decode(data, width, height);
        ...
        return decoded;
    }
```
* 这里 **data** 类型是 **ImageVideoWrapper** (参考 [ 3.4.2 ] 下载)
* **loadProvider** 的类型是 **构建 FixedLoadProvider < String , ImageVideoWrapper , GifBitmapWrapper , GlideDrawable>** ( 参考 [ 3.2.4 ] 工具集合1 )

```java
// FixedLoadProvider
	public ResourceDecoder<T, Z> getSourceDecoder() {
        return dataLoadProvider.getSourceDecoder();
    }
```
* **dataLoadProvider** 类型是 **ImageVideoGifDrawableLoadProvider <ImageVideoWrapper ,GifBitmapWrapper > ** ( 参考编/解码1 )
* 其中 **SourceDecoder** 是 **GifBitmapWrapperResourceDecoder**

```java
// GifBitmapWrapperResourceDecoder
	public Resource<GifBitmapWrapper> decode(ImageVideoWrapper source, int width, int height) throws IOException {
        ...
        wrapper = decode(source, width, height, tempBytes);
        ...
        return wrapper != null ? new GifBitmapWrapperResource(wrapper) : null;
    }

	private GifBitmapWrapper decode(ImageVideoWrapper source, int width, int height, byte[] bytes) throws IOException {
        final GifBitmapWrapper result;
        if (source.getStream() != null) {
            result = decodeStream(source, width, height, bytes);
        } else {
            result = decodeBitmapWrapper(source, width, height);
        }
        return result;
    }
```
* 参考**下载**流程, 得, source = new ImageVideoWrapper("下载1下载的流", null)
* 走 decodeStream 函数

```java
// GifBitmapWrapperResourceDecoder

	private GifBitmapWrapper decodeStream(ImageVideoWrapper source, int width, int height, byte[] bytes)
            throws IOException {
        InputStream bis = streamFactory.build(source.getStream(), bytes);
        bis.mark(MARK_LIMIT_BYTES);
        ImageHeaderParser.ImageType type = parser.parse(bis);
        bis.reset();

        GifBitmapWrapper result = null;
        if (type == ImageHeaderParser.ImageType.GIF) {
            result = decodeGifWrapper(bis, width, height);
        }

        if (result == null) {
            ...
            ImageVideoWrapper forBitmapDecoder = new ImageVideoWrapper(bis, source.getFileDescriptor());
            // 显然我们不是个gif , 走这里
            result = decodeBitmapWrapper(forBitmapDecoder, width, height);
        }
        return result;
    }

	private GifBitmapWrapper decodeBitmapWrapper(ImageVideoWrapper toDecode, int width, int height) throws IOException {
        GifBitmapWrapper result = null;

		// 参考[编/解码1]可得 bitmapDecoder = ImageVideoBitmapDecoder
        Resource<Bitmap> bitmapResource = bitmapDecoder.decode(toDecode, width, height);
        if (bitmapResource != null) {
            result = new GifBitmapWrapper(bitmapResource, null);
        }

        return result;
    }

// ImageVideoBitmapDecoder
	public Resource<Bitmap> decode(ImageVideoWrapper source, int width, int height) throws IOException {
        Resource<Bitmap> result = null;
        InputStream is = source.getStream();
        if (is != null) {
            try {
            	// 我们是标准流, 应该走这里
            	// 参考[编/解码1]可得 streamDecoder = StreamBitmapDecoder
                result = streamDecoder.decode(is, width, height);
            } catch (IOException e) {
                if (Log.isLoggable(TAG, Log.VERBOSE)) {
                    Log.v(TAG, "Failed to load image from stream, trying FileDescriptor", e);
                }
            }
        }

        if (result == null) {
            ParcelFileDescriptor fileDescriptor = source.getFileDescriptor();
            if (fileDescriptor != null) {
                result = fileDescriptorDecoder.decode(fileDescriptor, width, height);
            }
        }
        return result;
    }

// StreamBitmapDecoder

	public Resource<Bitmap> decode(InputStream source, int width, int height) {
		// 层次太深, 这里就不做分析了, downsampler是一个工具类, 专门用作原始资源流转换成图片的
        Bitmap bitmap = downsampler.decode(source, bitmapPool, width, height, decodeFormat);
        return BitmapResource.obtain(bitmap, bitmapPool);
    }

// BitmapResource
	public static BitmapResource obtain(Bitmap bitmap, BitmapPool bitmapPool) {
        if (bitmap == null) {
            return null;
        } else {
            return new BitmapResource(bitmap, bitmapPool);
        }
    }
```
* 解码之后我们得到了一个 **BitmapResource**实例, 里面封存了一张 Bitmap, 是 Resource < Bitmap > 类型
* 最后返回给上面的时候是一个 **GifBitmapWrapper** 实例, 里面封存了一个 **BitmapResource** 实例 (参考 GifBitmapWrapperResourceDecoder.decodeBitmapWrapper 这个函数)

下面来看看转码做了些啥?

### [3.4.4] 转码
回到之前的 DecodeJob , 这次我们来看**分支2**

```java
// DecodeJob

	public Resource<Z> decodeFromSource() throws Exception {
	    // 分支1, 下载 && 解码
        Resource<T> decoded = decodeSource();
        // 分支2, 转码
        return transformEncodeAndTranscode(decoded);
    }

	private Resource<Z> transformEncodeAndTranscode(Resource<T> decoded) {
        long startTime = LogTime.getLogTime();
		// 这里的 T 是 GifBitmapWrapper , 这个函数是进行图形变换的, 和主线无关,跳过
        Resource<T> transformed = transform(decoded);

        // 转码
        Resource<Z> result = transcode(transformed);
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            logWithTimeAndKey("Transcoded transformed from source", startTime);
        }
        return result;
    }

	private Resource<Z> transcode(Resource<T> transformed) {
        if (transformed == null) {
            return null;
        }
        // 查看[转码1]工具, transcoder ==
        // GifBitmapWrapperDrawableTranscoder < GifBitmapWrapper,GlideDrawable >
        return transcoder.transcode(transformed);
    }

// GifBitmapWrapperDrawableTranscoder
	public Resource<GlideDrawable> transcode(Resource<GifBitmapWrapper> toTranscode) {
        GifBitmapWrapper gifBitmap = toTranscode.get();
        Resource<Bitmap> bitmapResource = gifBitmap.getBitmapResource();

        final Resource<? extends GlideDrawable> result;
        if (bitmapResource != null) {
        	// 上面我们解码出来一个bitmap, 所以走这里
        	// 查看[转码1]工具, bitmapDrawableResourceTranscoder ==
        	// GlideBitmapDrawableTranscoder
            result = bitmapDrawableResourceTranscoder.transcode(bitmapResource);
        } else {
            result = gifBitmap.getGifResource();
        }
        return (Resource<GlideDrawable>) result;
    }

// GlideBitmapDrawableTranscoder
	public Resource<GlideBitmapDrawable> transcode(Resource<Bitmap> toTranscode) {
		// bitmap 拿出来 封装成 GlideBitmapDrawable
        GlideBitmapDrawable drawable = new GlideBitmapDrawable(resources, toTranscode.get());
        // 再加一层装饰 GlideBitmapDrawableResource
        return new GlideBitmapDrawableResource(drawable, bitmapPool);
    }
```
* 最终转码过程为 **GifBitmapWrapper >>> GlideBitmapDrawableResource 的过程**
* 返回到外面的对象类型是 **Resource< GlideBitmapDrawable >**
* 其中 GlideBitmapDrawableResource 的结构参考如下图
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210313173100170.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5,size_16,color_FFFFFF,t_70)
最后一步, 加载图片到界面上显示出来

### [3.4.5] 加载
看看转码完成之后如何通知界面的, 下面的代码是一个逆序调用过程, 追溯下转码成功后的代码走向

```java
// DecodeJob

	public Resource<Z> decodeFromSource() throws Exception {
	    // 分支1, 下载 && 解码
        Resource<T> decoded = decodeSource();
        // 分支2, 转码
        return transformEncodeAndTranscode(decoded);
    }

// EngineRunnable
	private Resource<?> decodeFromSource() throws Exception {
        return decodeJob.decodeFromSource();
    }

	private Resource<?> decode() throws Exception {
        if (isDecodingFromCache()) {
            return decodeFromCache();
        } else {
            return decodeFromSource();
        }
    }

	public void run() {
        if (isCancelled) {
            return;
        }

        Exception exception = null;
        Resource<?> resource = null;
        try {
        	// 终于绕出来了
            resource = decode();
        } catch (Exception e) {
            if (Log.isLoggable(TAG, Log.VERBOSE)) {
                Log.v(TAG, "Exception decoding", e);
            }
            exception = e;
        }

        if (isCancelled) {
            if (resource != null) {
                resource.recycle();
            }
            return;
        }

        if (resource == null) {
            onLoadFailed(exception);
        } else {
        	// 这应该就是上抛结果的回调了
            onLoadComplete(resource);
        }
    }

	private void onLoadComplete(Resource resource) {
		// manager 是在 EngineRunnable 初始化时赋的值
        manager.onResourceReady(resource);
    }

	public EngineRunnable(EngineRunnableManager manager, DecodeJob<?, ?, ?> decodeJob, Priority priority) {
        this.manager = manager;
        this.decodeJob = decodeJob;
        this.stage = Stage.CACHE;
        this.priority = priority;
    }

// Engine
 public <T, Z, R> LoadStatus load(Key signature, int width, int height, DataFetcher<T> fetcher,
            DataLoadProvider<T, Z> loadProvider, Transformation<Z> transformation, ResourceTranscoder<Z, R> transcoder,
            Priority priority, boolean isMemoryCacheable, DiskCacheStrategy diskCacheStrategy, ResourceCallback cb) {
		...
        EngineJob engineJob = engineJobFactory.build(key, isMemoryCacheable);
        ...
        // 得 engineJob == manager , 此处代码也可在[3.4.2]下载流程中找到
        EngineRunnable runnable = new EngineRunnable(engineJob, decodeJob, priority);
        ...
    }
```
现在**转码结果**被上抛到 EngineJob , 跟进去看看
```java
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
                	// 走这
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
        // resource == Resource< GlideBitmapDrawable >
        // 这里又给resource 包了一层, 就不展开了, 最后是个 EngineResource
        engineResource = engineResourceFactory.build(resource, isCacheable);
        ...
        // 此处开始上抛结果, 此处的listener通过addCallback来赋值
        // 可在[3.4.2]下载流程中查到, 此赋值代码
        cb.onResourceReady(engineResource);
    }

// Engine
 public <T, Z, R> LoadStatus load(Key signature, int width, int height, DataFetcher<T> fetcher,
            DataLoadProvider<T, Z> loadProvider, Transformation<Z> transformation, ResourceTranscoder<Z, R> transcoder,
            Priority priority, boolean isMemoryCacheable, DiskCacheStrategy diskCacheStrategy, ResourceCallback cb) {
        ...
		EngineJob engineJob = engineJobFactory.build(key, isMemoryCacheable);
      	...
        engineJob.addCallback(cb);
        engineJob.start(runnable);
    }

// GenericRequest
	public void onSizeReady(int width, int height) {
        ...
        // 这里, 所以应追溯到 GenericRequest.onResourceReady
        loadStatus = engine.load(signature, width, height, dataFetcher, loadProvider, transformation, transcoder,
                priority, isMemoryCacheable, diskCacheStrategy, this);

    }

	public void onResourceReady(Resource<?> resource) {
		...
		// 前文提到 resource 是个 EngineResource, 里面包了一个Resource< GlideBitmapDrawable >
        Object received = resource.get();
        ...
        // 最后上抛了一个 received (new GlideBitmapDrawable(null, BitmapState))
        onResourceReady(resource, (R) received);
    }

// EngineResource
	public Z get() {
        return resource.get();
    }

// GlideBitmapDrawableResource
	public final T get() {
        return (T) drawable.getConstantState().newDrawable();
    }

// GlideBitmapDrawable
	public ConstantState getConstantState() {
        return state;
    }

// BitmapState
	public Drawable newDrawable() {
            return new GlideBitmapDrawable(null, this);
    }
```
继续看 GenericRequest 的上抛
```java
// GenericRequest

	private void onResourceReady(Resource<?> resource, R result) {
        ...
        // result == new GlideBitmapDrawable(null, BitmapState)
        // 还记得在[3.4.1]构建了一个 GlideDrawableImageViewTarget < GlideDrawable > ?
        target.onResourceReady(result, animation);
        ...
    }

// GlideDrawableImageViewTarget
	public void onResourceReady(GlideDrawable resource, GlideAnimation<? super GlideDrawable> animation) {
        super.onResourceReady(resource, animation);
    }

// GlideDrawableImageViewTarget.super
	public void onResourceReady(Z resource, GlideAnimation<? super Z> glideAnimation) {
        if (glideAnimation == null || !glideAnimation.animate(resource, this)) {
            setResource(resource);
        }
    }

// GlideDrawableImageViewTarget
	protected void setResource(GlideDrawable resource) {
        view.setImageDrawable(resource);
    }
```
最后调用 ImageView 的 setImageDrawable 方法, 完成图片资源的展示