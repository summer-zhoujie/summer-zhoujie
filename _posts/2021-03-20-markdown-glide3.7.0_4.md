---
layout:     post
title:      "Android Glide 3.7.0 源码解析(四), BitmapPool作用及原理"
subtitle:   ""
date:       2021-03-20 15:18:48
author:     "zhoujie"
catalog: true # 是否开启目录
tags:
    - glide3.7.0
    - 源码
---

# 一、作用
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210317194500875.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5,size_16,color_FFFFFF,t_70)
1.  Android 中图片显示的实体其实是一个 Bitmap 对象, 每次图片显示时, 都会构建一个 Bitmap 对象, 不用时再销毁
2. 假设: 一个长列表每项都有个图片需要显示, 我们在快速滑动长列表的时候, 会产生什么?
  	Bitmap 对象被频繁的创建和释放, 导致 GC 频繁
3. 如何解决上述问题?
	BitmapPool , 一个 Bitmap 的对象池, 让一个新的图片资源复用在旧的 Bitmap对象上, 假设, 长列表一页有 20 个图片资源, BitmapPool 大小也刚好是 20 , 那么当滑动列表的时候, 我们一直在沿用最开始创建的 20 个 Bitmap 对象, 也就解决了频繁 GC 的问题(频繁的 GC 会导致卡顿)


# 二、原理
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210317200040202.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5,size_16,color_FFFFFF,t_70)

## 框架结构
如上图:
1. BitmapPool 一共有两种实现 **BitmapPoolAdapter** 和 **LruBitmapPool**
2. 其中, **BitmapPoolAdapter** 是**空实现**
3. **LruBitmapPool** 字面意思可猜: **一个 LRU 算法实现的 Bitmap 缓存池**
4. LruBitmapPool 的 **LRU 算法**实现依赖于 **LruPoolStrategy 提供功能支持**
5. 又 , LruPoolStrategy 有三种实现方案 **AttributeStrategy** , **SizeConfigStrategy** 和 **SizeStrategy**

## 接口抽象
先来看看 BitmapPool 接口是如何抽象的
```java
/**
 * 定义了一个允许复用 Bitmap 对象的池子的接口
 */
public interface BitmapPool {

    /**
     * 返回当前 BitmapPool 的最大容量，单位是 byte 字节
     */
    int getMaxSize();

    /**
     * 可以通过此方法设置一个因子，此因子会乘以最开始设置的最大容量，将结果作为新的最大容量
     *
     * Note1: 上面计算完成以后，如果 BitmapPool 的当前实际容量比当前最大容量大，则会清除 BitampPool 中的对象，直到当前实际容量小于当前最大容量
     * Note2: 开发者可以通过此方法动态地、同步地调整 BitmapPool 最大容量
     */
    void setSizeMultiplier(float sizeMultiplier);

    /**
     * 加入一个新的不用的 bitmap 实例到池子中, 如果加入失败返回 false 且 , 需要调用者自行执行 bitmap.recycle() 来释放资源
     *
     * 可能返回失败的情况:
     * 1. bitmap.isMutable() 返回false
     * 2. bitmap 加入之后就超过了池子的最大容量
     */
    boolean put(Bitmap bitmap);

    /**
     * 根据传入的条件返回合适的 bitmap 实例, 如果没有合适的则返回null
     * 该方法会默认擦除原 bitmap 实例中存储的像素值, 赋值成透明
     * 速度比 getDirty() 要慢些
     */
    Bitmap get(int width, int height, Bitmap.Config config);

    /**
     * 和 get() 方法语义一致, 区别是, 不会擦除旧的像素数据
     * 速度比 get() 要快
     */
    Bitmap getDirty(int width, int height, Bitmap.Config config);

    /**
     * 清空池子中的缓存数据
     */
    void clearMemory();

    /**
     * 做指定级别的缓存数据清空 ( 意思是清除多少数据由 level 决定 )
     */
    void trimMemory(int level);
}
```
看完上面的接口抽象, 大家应该对 BitmapPool 要实现的功能有所了解了, 下面来看看 BitmapPool 的具体实现
1. **BitmapPoolAdapter**  空实现
2.  **LruBitmapPool** 遵循 LRU(least recently used) 算法的实现

## BitmapPoolAdapter 空实现
```java
public class BitmapPoolAdapter implements BitmapPool {
    @Override
    public int getMaxSize() {
        return 0;
    }

    @Override
    public void setSizeMultiplier(float sizeMultiplier) {
        // Do nothing.
    }

    @Override
    public boolean put(Bitmap bitmap) {
    	// 空实现, 谁来都返回 false
        return false;
    }

    @Override
    public Bitmap get(int width, int height, Bitmap.Config config) {
        return null;
    }

    @Override
    public Bitmap getDirty(int width, int height, Bitmap.Config config) {
        return null;
    }

    @Override
    public void clearMemory() {
        // Do nothing.
    }

    @Override
    public void trimMemory(int level) {
        // Do nothing.
    }
}
```
可以直接看 put() 方法, 谁来都返回 false , 池子里不可能有数据, 所以很容易得出, 是个**假实现**
* **问题: 为啥需要这个假实现 ???**
追踪 Glide 中 BitmapPool 初始化的地方
```java
// GlideBuilder
	Glide createGlide() {
        ...
        if (bitmapPool == null) {
        	// Build.VERSION_CODES.HONEYCOMB ( api: 11 , android 3.0)
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.HONEYCOMB) {
                int size = calculator.getBitmapPoolSize();
                bitmapPool = new LruBitmapPool(size);
            } else {
            	// android 3.0 以下需要这个空实现
                bitmapPool = new BitmapPoolAdapter();
            }
        }
		...
    }
```
因为在3.0以前 Bitmap 的数据是存在 native 区域，3.0以后存在 Dalvik 内存区域，API11 后 系统提供了 Bitmap 复用的 API, [官方详细传送门](https://developer.android.com/topic/performance/graphics/manage-memory.html)
分析完 BitmapPoolAdapter 空实现存在的意义, 赶紧来看看正经实现长啥样

## LruBitmapPool LRU算法实现

### LruBitmapPool 外壳
先看构造函数
```java
// LruBitmapPool
	public LruBitmapPool(int maxSize) {
        this(maxSize, getDefaultStrategy(), getDefaultAllowedConfigs());
    }

	LruBitmapPool(int maxSize, LruPoolStrategy strategy, Set<Bitmap.Config> allowedConfigs) {
        this.initialMaxSize = maxSize;
        this.maxSize = maxSize;
        this.strategy = strategy;
        this.allowedConfigs = allowedConfigs;
        this.tracker = new NullBitmapTracker();
    }
```
>有几个问题需要弄明白:
>1. size 传的多少, 在哪边计算的?
>默认4个屏幕大小, 在 MemorySizeCalculator 中可查找答案
>2. strategy 是干嘛的?
>真正实现 LRU 策略的地方
>3. allowedConfigs 又是干嘛的?
>约束缓存的 Bitmap 实例的 config, 要求必须是 allowedConfigs 中的一种

池子最主要是看 get() 和 put() 方法
```java
public class LruBitmapPool implements BitmapPool {
	@Override
    public synchronized boolean put(Bitmap bitmap) {
        if (bitmap == null) {
            throw new NullPointerException("Bitmap must not be null");
        }
        if (!bitmap.isMutable() || strategy.getSize(bitmap) > maxSize || !allowedConfigs.contains(bitmap.getConfig())) {
            // bitmap.isMutable() == false;
            // 加入这个 bitmap 实例, 会超出池子的最大存储容量
            // 这个 bitmap 的 config 不在池子允许的范畴内
            return false;
        }

        final int size = strategy.getSize(bitmap);
        // 加入真正的 LRU 缓存中
        strategy.put(bitmap);
        // 修改当前池子的大小
        currentSize += size;
		// 检查当前池子大小, 如果大于最大容量, 就往外丢 bitmap 实例, 直到不超容量
        evict();
        return true;
    }

    @Override
    public synchronized Bitmap get(int width, int height, Bitmap.Config config) {
        Bitmap result = getDirty(width, height, config);
        if (result != null) {
            // 擦除原来 bitmap 实例的像素数据
            result.eraseColor(Color.TRANSPARENT);
        }

        return result;
    }

	public synchronized Bitmap getDirty(int width, int height, Bitmap.Config config) {
        // 从 LRU 缓存中拿出一个符合条件的 bitmap 实例
        final Bitmap result = strategy.get(width, height, config != null ? config : DEFAULT_CONFIG);
        if (result == null) {
            // 拿出失败, 返回 null
        } else {
            // 拿出成功, 重新计算池子大小
            currentSize -= strategy.getSize(result);
        }
        return result;
    }
}
```
通过上面的代码可以看出:
1. 核心的**缓存功能**都是 **LruPoolStrategy** 在实现
2. LruBitmapPool 主要负责 **入口的条件判断** ( put()时 ) 和 **池子的大小管理**

### LruPoolStrategy 核心
下面追踪 LruPoolStrategy 的实现
```java
// LruBitmapPool
	public LruBitmapPool(int maxSize) {
        this(maxSize, getDefaultStrategy(), getDefaultAllowedConfigs());
    }

    private static LruPoolStrategy getDefaultStrategy() {
        final LruPoolStrategy strategy;
        // KITKAT ( api: 19 , android 4.4 )
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {
            strategy = new SizeConfigStrategy();
        } else {
            strategy = new AttributeStrategy();
        }
        return strategy;
    }
```
>问题:
>为啥 KITKAT 前后使用不同的实现方案? 这地方暂时还说明不了, 等到文章快结束时进行说明

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210317210215215.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5,size_10,color_FFFFFF,t_10)
这张图大家还有印象吧, 我们先来看看其中一个 **AttributeStrategy 的数据结构**, 其他的使用类推, 就会简单很多了
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210320135643832.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5,size_16,color_FFFFFF,t_70)

下面来对各个属性进行说明
1. **Key** 是**查找条件**, 用来唯一识别 **一组Bitmaps**
2. **KeyPool** 是用来**缓存并复用 Key 实例**的,  想想看: 上述案例, 长列表滑动, 频繁的从 BitmapPool 里面 get() put() 势必会频繁的创建和释放 Key 实例, 这里用个 Pool 把 Key 实例们缓存起来, 不必频繁创建了( 颇有套娃嫌疑! )
3. **GroupedLinkedMap**里面封存了LRU缓存策略'
4. **HashMap结构**是用来**降低**查找符合 Key 条件的**算法时间复杂度的**, 如果在**LRU 单向循环列表**里面执行查找, 是 O(n) , 而在 HashMap 中查找是 O(1) ~ O(n) ( [算法的时间复杂度/空间复杂度详解,暂未实现,标记一下]() ), 这里是一个典型的**以空间换时间**的案例
6. **LRU 单向循环列表** 是真正的 **LRU 缓存**
    >每个节点都是一个 LinkedEntry 实例;
    >每个 LinkedEntry 实例又存储了**符合 Key 条件的 一组 Bitmap**;
    >当往 BitmapPool put() 的时候: 1. 找到符合 Key 条件的 Entry, 插入Entry 的 values 队尾; 2. 找不到, 永远插入队尾, 代表着最近不常使用
    >当往 BitmapPool get() 的时候: 1. 找到符合 Key 条件的 Entry, 移除 Entry 中 values 最后一条给外部使用; 2.  3. 找不到, 创建一个新的条件为 Key 的 Entry; 4.  移动 Entry 到队头, 代表最近最常使用

理论讲完来看看实践 ( 源码 )
```java
// AttributeStrategy
class AttributeStrategy implements LruPoolStrategy {
    private final KeyPool keyPool = new KeyPool();
    private final GroupedLinkedMap<Key, Bitmap> groupedMap = new GroupedLinkedMap<Key, Bitmap>();

    public void put(Bitmap bitmap) {
    	// 从 keyPool 缓存中拿取 Key 实例进行复用
        final Key key = keyPool.get(bitmap.getWidth(), bitmap.getHeight(), bitmap.getConfig());

		// 调用 GroupedLinkedMap 来进行 LRU 缓存
        groupedMap.put(key, bitmap);
    }

    @Override
    public Bitmap get(int width, int height, Bitmap.Config config) {
    	// 从 keyPool 缓存中拿取 Key 实例进行复用
        final Key key = keyPool.get(width, height, config);

		// 调用 GroupedLinkedMap 来进行 LRU 缓存的获取
        return groupedMap.get(key);
    }
}

// AttributeStrategy$KeyPool
    static class KeyPool extends BaseKeyPool<Key> {
        public Key get(int width, int height, Bitmap.Config config) {
            Key result = get();
            result.init(width, height, config);
            return result;
        }

        @Override
        protected Key create() {
            return new Key(this);
        }
    }

// BaseKeyPool
abstract class BaseKeyPool<T extends Poolable> {
    private static final int MAX_SIZE = 20;
    // 就是个简单的队列实现, 最大存储容量是20
    private final Queue<T> keyPool = Util.createQueue(MAX_SIZE);

    protected T get() {
        T result = keyPool.poll();
        if (result == null) {
            result = create();
        }
        return result;
    }

    public void offer(T key) {
        if (keyPool.size() < MAX_SIZE) {
            keyPool.offer(key);
        }
    }

    protected abstract T create();
}
```
可以看到
1. KeyPool, 就是一个容量为 20 的队列, 很简单,
2. put() 和 get() 还是通过 **GroupedLinkedMap**来实现功能

追踪进去 GroupedLinkedMap
```java
class GroupedLinkedMap<K extends Poolable, V> {
	// 单向循环链表
    private final LinkedEntry<K, V> head = new LinkedEntry<K, V>();
    // 空间换时间的 HashMap
    private final Map<K, LinkedEntry<K, V>> keyToEntry = new HashMap<K, LinkedEntry<K, V>>();

    public void put(K key, V value) {
        LinkedEntry<K, V> entry = keyToEntry.get(key);

        if (entry == null) {
            entry = new LinkedEntry<K, V>(key);
            // 插入新的, 直接插入到循环链表队尾, 代表最近不常使用
            makeTail(entry);
            keyToEntry.put(key, entry);
        } else {
            key.offer();
        }

		// 插入旧的(匹配到Key条件), 直接插入
        entry.add(value);
    }

    public V get(K key) {
        LinkedEntry<K, V> entry = keyToEntry.get(key);
        if (entry == null) {
            entry = new LinkedEntry<K, V>(key);
            // 找不到符合 Key 条件的, 创建一个符合 Key 条件的 Entry
            keyToEntry.put(key, entry);
        } else {
            key.offer();
        }

		// 移动 Entry 到队头, 代表最近最常使用
        makeHead(entry);

		// 移除 Entry 的 values 里的一个 Bitmap 对象供外部使用
        return entry.removeLast();
    }

	// 内存不足的时候需要清理了
	public V removeLast() {
		// 从队尾开始清除, 清除最近不常使用
        LinkedEntry<K, V> last = head.prev;

        while (!last.equals(head)) {
            V removed = last.removeLast();
            if (removed != null) {
                return removed;
            } else {
                removeEntry(last);
                keyToEntry.remove(last.key);
                last.key.offer();
            }

            last = last.prev;
        }

        return null;
    }
}
```
以上 AttributeStrategy 的原理讲完了, 下面来看看另外一个实现 SizeConfigStrategy,
1. 它和 AttributeStrategy 的区别在于存储的Key值条件不一样, **SizeConfigStrategy 的 Key 是 size (w*h) 和 config 组成**, 之所以能这么做完全依赖于 Bitmap 的一个方法 reconfigure()
2. 还记得前文提到 KITKAT 前后 LruPoolStrategy 的不同实现, **reconfigure()** 方法就是在 KITKAT 之后才支持的
3. **reconfigure()** 作用很简单就是修改已存在的 Bitmap 的大小配置, 这样才能让缓存以比较模糊的 Size 维度进行, 而不是 width 和 height 的强制匹配

下面来看看 **SizeConfigStrategy** 与 **AttributeStrategy** 不同的地方
```java
// SizeConfigStrategy

	@Override
    public Bitmap get(int width, int height, Bitmap.Config config) {
        int size = Util.getBitmapByteSize(width, height, config);
        Key targetKey = keyPool.get(size, config);
        // 关键代码在这里, 找到一个合适的 size 的 Bitmap 缓存实例
        Key bestKey = findBestKey(targetKey, size, config);

        Bitmap result = groupedMap.get(bestKey);
        if (result != null) {
            // Decrement must be called before reconfigure.
            decrementBitmapOfSize(Util.getBitmapByteSize(result), result.getConfig());
            // 将缓存的 Bitmap 宽高和配置重行修改, 返回给下张图用
            result.reconfigure(width, height,
                    result.getConfig() != null ? result.getConfig() : Bitmap.Config.ARGB_8888);
        }
        return result;
    }

    private Key findBestKey(Key key, int size, Bitmap.Config config) {
        Key result = key;
        for (Bitmap.Config possibleConfig : getInConfigs(config)) {
        	// 找到符合 config 的一组 size
            NavigableMap<Integer, Integer> sizesForPossibleConfig = getSizesForConfig(possibleConfig);
            // 从 size 里面比当前需要的大的所有size中, 找到最小的那个 返回给外面
            Integer possibleSize = sizesForPossibleConfig.ceilingKey(size);
            if (possibleSize != null && possibleSize <= size * MAX_SIZE_MULTIPLE) {
                if (possibleSize != size
                        || (possibleConfig == null ? config != null : !possibleConfig.equals(config))) {
                    keyPool.offer(key);
                    result = keyPool.get(possibleSize, possibleConfig);
                }
                break;
            }
        }
        return result;
    }

	private NavigableMap<Integer, Integer> getSizesForConfig(Bitmap.Config config) {
		// 找到符合 config 的一组 size
        NavigableMap<Integer, Integer> sizes = sortedSizes.get(config);
        if (sizes == null) {
            sizes = new TreeMap<Integer, Integer>();
            sortedSizes.put(config, sizes);
        }
        return sizes;
    }

	// 缓存, 按照 config 为 Key 的方式存储一组 Bitmap 的 size
	private final Map<Bitmap.Config, NavigableMap<Integer, Integer>> sortedSizes =
            new HashMap<Bitmap.Config, NavigableMap<Integer, Integer>>();
```
1. 关于 sortedSizes 还是为了 **空间换时间** , 主要赚取一个遍历的时间成本差价
2. get 方法会先找到 config 相关的一组size; 然后找出这一组 size 中比当前需要 size 大的全部 size; 最后, 在筛选之后的 size 组中, 找到一个最小的, 返回给外面使用 ( 总之: **找一个比需要 size 大的,并且最贴近需要 size 的 Bitmap 缓存实例** )

## 回顾
至此, BitmapPool 相关的就讲完了, 下面回顾下这张图
![在这里插入图片描述](https://img-blog.csdnimg.cn/2021032013563065.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5,size_16,color_FFFFFF,t_70)

核心原理都在这里了, 就是一个**遵循 LRU 算法的 对象池子**, 或者再精确些, 就是一个**单向循环链表**, 列表的每个元素都存着一组 Bitmap 缓存实例, 以备外面使用