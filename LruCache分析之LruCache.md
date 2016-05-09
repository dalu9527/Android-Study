# LruCache 分析

LruCache 是 Android 的一个内部类，提供了基于内存实现的缓存

用法

 		//获取系统分配给每个应用程序的最大内存，每个应用系统分配32M  
        int maxMemory = (int) Runtime.getRuntime().maxMemory();    
        int mCacheSize = maxMemory / 8;  
        //给LruCache分配1/8 4M  
        mMemoryCache = new LruCache<String, Bitmap>(mCacheSize){  
  
            //必须重写此方法，来测量Bitmap的大小  
            @Override  
            protected int sizeOf(String key, Bitmap value) {  
                return value.getRowBytes() * value.getHeight();  
            }  
              
        };  


## 源码 ##

LRUCache 的源码不是很长，我们从上到下逐个分析，先看官方对这个类的注释（注释往往很有用）

### 注释 ###

	/**
 	* BEGIN LAYOUTLIB CHANGE
 	* This is a custom version that doesn't use the non standard LinkedHashMap#eldest.
 	* END LAYOUTLIB CHANGE
 	*
 	* A cache that holds strong references to a limited number of values. Each time
 	* a value is accessed, it is moved to the head of a queue. When a value is
 	* added to a full cache, the value at the end of that queue is evicted and may
 	* become eligible for garbage collection.
 	*
 	* <p>If your cached values hold resources that need to be explicitly released,
 	* override {@link #entryRemoved}.
 	*
 	* <p>If a cache miss should be computed on demand for the corresponding keys,
 	* override {@link #create}. This simplifies the calling code, allowing it to
 	* assume a value will always be returned, even when there's a cache miss.
 	*
 	* <p>By default, the cache size is measured in the number of entries. Override
 	* {@link #sizeOf} to size the cache in different units. For example, this cache
 	* is limited to 4MiB of bitmaps:
 	* <pre>   {@code
 	*   int cacheSize = 4 * 1024 * 1024; // 4MiB
 	*   LruCache<String, Bitmap> bitmapCache = new LruCache<String, Bitmap>(cacheSize) {
 	*       protected int sizeOf(String key, Bitmap value) {
 	*           return value.getByteCount();
 	*       }
 	*   }}</pre>
 	*
 	* <p>This class is thread-safe. Perform multiple cache operations atomically by
 	* synchronizing on the cache: <pre>   {@code
 	*   synchronized (cache) {
 	*     if (cache.get(key) == null) {
 	*         cache.put(key, value);
 	*     }
 	*   }}</pre>
 	*
 	* <p>This class does not allow null to be used as a key or value. A return
 	* value of null from {@link #get}, {@link #put} or {@link #remove} is
 	* unambiguous: the key was not in the cache.
 	*
 	* <p>This class appeared in Android 3.1 (Honeycomb MR1); it's available as part
 	* of <a href="http://developer.android.com/sdk/compatibility-library.html">Android's
 	* Support Package</a> for earlier releases.
 	*/

注释比较长，不过提供了几个关键信息

- 说明了LRU的工作原理，最近使用的会放进队列的头部，最久未使用的放进队列的尾部，会首先删除队尾元素
- 如果你cache的某个值需要明确释放，重写 entryRemoved 方法
- 如果 key 相对应的 item 丢掉，重写create()，这简化了调用代码，即使丢失了也总会返回。
- 默认的，我们需要重写 sizeOf 方法
- 该类是线程安全的
- 该类不允许空值和空key
- 该类出现在 Android 3.1系统及其以后，但是会向下兼容

下面看其定义的变量

### 变量 ###

 	private final LinkedHashMap<K, V> map;// 以 LinkedHashMap 进行存储

    /** Size of this cache in units. Not necessarily the number of elements. */
    private int size;// 当前大小
    private int maxSize;// 最大容量

    private int putCount;// put次数
    private int createCount;// 创建次数
    private int evictionCount;// 回收次数
    private int hitCount;// 命中次数
    private int missCount;// 未命中次数

再看构造方法

### 构造方法 ###

	/**
     * @param maxSize for caches that do not override {@link #sizeOf}, this is
     *     the maximum number of entries in the cache. For all other caches,
     *     this is the maximum sum of the sizes of the entries in this cache.
     */
    public LruCache(int maxSize) {
        if (maxSize <= 0) {// 必须大于 0 ，看上面的用发就知道了
            throw new IllegalArgumentException("maxSize <= 0");
        }
        this.maxSize = maxSize;
        this.map = new LinkedHashMap<K, V>(0, 0.75f, true);// 将LinkedHashMap的accessOrder 设置为 true 来实现 LRU
    }

构造方法没啥好说的，主要就是 将 LinkedHashMap 的 accessOrder 设置为 true 来实现 LRU，利用访问顺序而不是插入顺序。

接下来是 resize 方法

### resize() ###

	/**
     * Sets the size of the cache.
     * @param maxSize The new maximum size.
     *
     * @hide
     */
    public void resize(int maxSize) {
        if (maxSize <= 0) {
            throw new IllegalArgumentException("maxSize <= 0");
        }

        synchronized (this) {
            this.maxSize = maxSize;
        }
        trimToSize(maxSize);
    }

重新计算缓存的大小，里面涉及到了 trimToSize 方法。另外，注意 synchronized

### trimToSize() ###

	/**
     * @param maxSize the maximum size of the cache before returning. May be -1
     *     to evict even 0-sized elements.
     */
    private void trimToSize(int maxSize) {
        while (true) {// 死循环
            K key;
            V value;
            synchronized (this) {// 线程安全  
                if (size < 0 || (map.isEmpty() && size != 0)) {
                    throw new IllegalStateException(getClass().getName()
                            + ".sizeOf() is reporting inconsistent results!");
                }

                if (size <= maxSize) {
                    break;
                }

                // BEGIN LAYOUTLIB CHANGE
                // get the last item in the linked list.
                // This is not efficient, the goal here is to minimize the changes
                // compared to the platform version.
                Map.Entry<K, V> toEvict = null;
                for (Map.Entry<K, V> entry : map.entrySet()) {
                    toEvict = entry;
                }
                // END LAYOUTLIB CHANGE

                if (toEvict == null) {
                    break;
                }

                key = toEvict.getKey();
                value = toEvict.getValue();
                map.remove(key);
                size -= safeSizeOf(key, value);
                evictionCount++;
            }

            entryRemoved(true, key, value, null);
        }
    }

该方法根据 maxSize 来调整内存 cache 的大小，如果 maxSize 传入 -1，则清空缓存中的所有对象。该源码可知，该内部是一个死循环，靠满足相应的条件达到退出的目的

- 条件1，当当前大小 size 小于 最大容量时，退出
- 当需要删除的 entry 为空时，会退出。这里 请看 LAYOUTLIB CHANGE 之间的代码，这里好像删除的是 Map 的队尾元素，但是，我们知道针对 LinkedHashMap ，队尾则是最新使用的元素，这里把最新的删掉了，和原意相违背。注释里面讲解到：This is not efficient, the goal here is to minimize the changes compared to the platform version.

上面的代码是 Android API 22 Platform 里面的

而在 Android API 23 Platform 里面新代码改变了

 	Map.Entry<K, V> toEvict = map.eldest();
    
这里可以看出，是取的最久未使用的，将最久未使用的删除了。（谷歌的工程师也在不断的调整）

里面涉及到 safeSizeOf 方法

### safeSizeOf() ###

	private int safeSizeOf(K key, V value) {
        int result = sizeOf(key, value);
        if (result < 0) {
            throw new IllegalStateException("Negative size: " + key + "=" + value);
        }
        return result;
    }

里面涉及到了我们需要复写的方法 sizeOf 

最后调用 entryRemoved 方法

### entryRemoved() ###

	/**
     * Called for entries that have been evicted or removed. This method is
     * invoked when a value is evicted to make space, removed by a call to
     * {@link #remove}, or replaced by a call to {@link #put}. The default
     * implementation does nothing.
     *
     * <p>The method is called without synchronization: other threads may
     * access the cache while this method is executing.
     *
     * @param evicted true if the entry is being removed to make space, false
     *     if the removal was caused by a {@link #put} or {@link #remove}.
     * @param newValue the new value for {@code key}, if it exists. If non-null,
     *     this removal was caused by a {@link #put}. Otherwise it was caused by
     *     an eviction or a {@link #remove}.
     */
    protected void entryRemoved(boolean evicted, K key, V oldValue, V newValue) {}

该方法是空方法，看注释可知：当 item 被回收或者删掉时调用。该方法当 value 被回收释放存储空间时被 remove 调用，或者替换 item 值时 put 调用，默认实现什么都没做。

每次回收对象就调用该函数，这里参数为 true --为释放空间被删除；false --get、put 或 remove 导致。前面类注释可知，需要用户考量进行重写

接下来看 get 方法

### get() ###

	/**
     * Returns the value for {@code key} if it exists in the cache or can be
     * created by {@code #create}. If a value was returned, it is moved to the
     * head of the queue. This returns null if a value is not cached and cannot
     * be created.
     */
    public final V get(K key) {
        if (key == null) {
            throw new NullPointerException("key == null");
        }

        V mapValue;
        synchronized (this) {
            mapValue = map.get(key);
            if (mapValue != null) {
                hitCount++;// 命中
                return mapValue;
            }
            missCount++;// 未命中
        }

        /*
         * Attempt to create a value. This may take a long time, and the map
         * may be different when create() returns. If a conflicting value was
         * added to the map while create() was working, we leave that value in
         * the map and release the created value.
         */
		// 如果丢失了就试图创建一个item
        V createdValue = create(key);
        if (createdValue == null) {
            return null;
        }
		// 接下来是如果用户重写了create方法后，可能会执行到
        synchronized (this) {
            createCount++;// 创建数增加  
            mapValue = map.put(key, createdValue);// 将刚刚创建的值放入map中，返回的值是在map中与key相对应的旧值（就是在放入new value前的old value）  
			// 如果有人实现了create方法，需要注意create方法的注释
			/**
		     * Called after a cache miss to compute a value for the corresponding key.
		     * Returns the computed value or null if no value can be computed. The
		     * default implementation returns null.
		     *
		     * <p>The method is called without synchronization: other threads may
		     * access the cache while this method is executing.
		     *
		     * <p>If a value for {@code key} exists in the cache when this method
		     * returns, the created value will be released with {@link #entryRemoved}
		     * and discarded. This can occur when multiple threads request the same key
		     * at the same time (causing multiple values to be created), or when one
		     * thread calls {@link #put} while another is creating a value for the same
		     * key.
		     */
			// 涉及到了多线程会造成冲突的问题
            if (mapValue != null) {
                // There was a conflict so undo that last put
                map.put(key, mapValue);// 如果之前位置上已经有元素了，就还把原来的放回去，等于size没变  
            } else {
                size += safeSizeOf(key, createdValue);// 如果之前的位置上没有元素，说明createdValue是新加上去的，所以要加上createdValue的size 
            }
        }
		/* 
         * 刚刚如果检测到旧值，因为最后旧值还是在map中，但是中途被回收了，所以还是要通知别人这个对象被回收过。 
         * 所以就调用了entryRemoved
         */  
        if (mapValue != null) {
            entryRemoved(false, key, createdValue, mapValue);
            return mapValue;
        } else {
			/* 
             * 如果刚刚没有检测到旧值，将新值放入map。 
             * 那么需要重新检测是否size是否超出了maxSize，所以就调用了trimToSize,并返回新值 
             */  
            trimToSize(maxSize);
            return createdValue;
        }
    }

开头注释说明：通过 key 返回相应的 item，或者创建返回相应的 item。相应的 item 会移动到队列的头部，如果 item 的 value 没有被 cache 或者不能被创建，则返回 null。

这里面涉及到了 create 方法

### create() ###

	/**
     * Called after a cache miss to compute a value for the corresponding key.
     * Returns the computed value or null if no value can be computed. The
     * default implementation returns null.
     *
     * <p>The method is called without synchronization: other threads may
     * access the cache while this method is executing.
     *
     * <p>If a value for {@code key} exists in the cache when this method
     * returns, the created value will be released with {@link #entryRemoved}
     * and discarded. This can occur when multiple threads request the same key
     * at the same time (causing multiple values to be created), or when one
     * thread calls {@link #put} while another is creating a value for the same
     * key.
     */
    protected V create(K key) {
        return null;
    }

create 函数是根据 key 来创建相应的 item，但是在 LruCache 中默认返回的是null。因为 LruCache 未记录被回收的数据，这里读者可以重写该 create 函数，为 key 创建相应的 item，这里是需要读者自行设计。请注意，多线程会导致冲突

下面看一下 put 方法

### put() ###

	/**
     * Caches {@code value} for {@code key}. The value is moved to the head of
     * the queue.
     *
     * @return the previous value mapped by {@code key}.
     */
    public final V put(K key, V value) {
        if (key == null || value == null) {
            throw new NullPointerException("key == null || value == null");
        }

        V previous;
        synchronized (this) {
            putCount++;// put次数增加
            size += safeSizeOf(key, value);// 计算size
            previous = map.put(key, value);// 这里其实是放到了队尾
            if (previous != null) {// 不为空，说明之前有数据，所以要把size减去
                size -= safeSizeOf(key, previous);
            }
        }

        if (previous != null) {
            entryRemoved(false, key, previous, value);
        }

        trimToSize(maxSize);
        return previous;
    }

接下来看看 remove 方法

### remove() ###

	/**
     * Removes the entry for {@code key} if it exists.
     *
     * @return the previous value mapped by {@code key}.
     */
    public final V remove(K key) {
        if (key == null) {
            throw new NullPointerException("key == null");
        }

        V previous;
        synchronized (this) {
            previous = map.remove(key);// 删除key，并返回旧值  
            if (previous != null) {
                size -= safeSizeOf(key, previous);// 如果旧值不为空，则为size减去其旧值大小  
            }
        }

        if (previous != null) {
            entryRemoved(false, key, previous, null);
        }

        return previous;
    }

最后看看其他方法

### 其他方法 ###

 	/**
     * Returns the size of the entry for {@code key} and {@code value} in
     * user-defined units.  The default implementation returns 1 so that size
     * is the number of entries and max size is the maximum number of entries.
     * 返回用户定义的item的大小，默认返回1代表item的数量，最大size就是最大item值
     * <p>An entry's size must not change while it is in the cache.
     */
    protected int sizeOf(K key, V value) {
        return 1;
    }

    /**
     * Clear the cache, calling {@link #entryRemoved} on each removed entry.
     */
    public final void evictAll() {
        trimToSize(-1); // -1 will evict 0-sized elements 清除map中全部的数据 
    }

    /**
     * For caches that do not override {@link #sizeOf}, this returns the number
     * of entries in the cache. For all other caches, this returns the sum of
     * the sizes of the entries in this cache.
     */
    public synchronized final int size() {
        return size;
    }

    /**
     * For caches that do not override {@link #sizeOf}, this returns the maximum
     * number of entries in the cache. For all other caches, this returns the
     * maximum sum of the sizes of the entries in this cache.
     */
    public synchronized final int maxSize() {
        return maxSize;
    }

    /**
     * Returns the number of times {@link #get} returned a value that was
     * already present in the cache.
     */
    public synchronized final int hitCount() {
        return hitCount;
    }

    /**
     * Returns the number of times {@link #get} returned null or required a new
     * value to be created.
     */
    public synchronized final int missCount() {
        return missCount;
    }

    /**
     * Returns the number of times {@link #create(Object)} returned a value.
     */
    public synchronized final int createCount() {
        return createCount;
    }

    /**
     * Returns the number of times {@link #put} was called.
     */
    public synchronized final int putCount() {
        return putCount;
    }

    /**
     * Returns the number of values that have been evicted.
     */
    public synchronized final int evictionCount() {
        return evictionCount;
    }

    /**
     * Returns a copy of the current contents of the cache, ordered from least
     * recently accessed to most recently accessed.
     */
    public synchronized final Map<K, V> snapshot() {
        return new LinkedHashMap<K, V>(map);// 返回当前缓存中所有的条目集合
    }

    @Override public synchronized final String toString() {
        int accesses = hitCount + missCount;
        int hitPercent = accesses != 0 ? (100 * hitCount / accesses) : 0;
        return String.format("LruCache[maxSize=%d,hits=%d,misses=%d,hitRate=%d%%]",
                maxSize, hitCount, missCount, hitPercent);
    }

看相关注释，不难理解

## 总结 ##

- LruCache 封装了 LinkedHashMap，提供了 LRU 缓存的功能；
- LruCache 通过 trimToSize 方法自动删除最近最少访问的键值对；
- LruCache 不允许空键值；
- LruCache 线程安全；
- LruCache 的源码在不同版本中不一样，需要区分
- 继承 LruCache 时，必须要复写 sizeOf 方法，用于计算每个条目的大小。

## 参考链接 ##

[LruCache 源码解析](https://github.com/LittleFriendsGroup/AndroidSdkSourceAnalysis/blob/master/article/LruCache%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.md)

[Android中LruCache的源码分析](http://blog.csdn.net/lintcgirl/article/details/47777321)

[Android LruCache源码详解](http://blog.csdn.net/abcdef314159/article/details/51190661)