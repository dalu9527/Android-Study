# LruCache 之 LinkedHashMap 分析

LruCache是Android的一个内部类，提供了基于内存实现的缓存

双向链表

LinkedHashMap是key键有序的HashMap的一种实现。它除了使用哈希表这个数据结构，使用双向链表来保证key的顺序

![](http://ww4.sinaimg.cn/mw690/b254dc71jw1ew9u6xzpf6g20vg089jre.gif)

双向链表算是个很常见的数据结构，上图中的头节点的prev、尾节点的next指向null，双向链表还有一种变种，见下图

![](http://ww4.sinaimg.cn/mw690/b254dc71jw1ew9u6yj26xj20f905bmxf.jpg)

LinkedHashMap就是采用的这种方式

## 源码 ##

先看源码定义

	public class LinkedHashMap<K, V> extends HashMap<K, V> {

其是继承HashMap，那么就会拥有HashMap的大部分特色，例如允许null键和null值，默认容量是16，装载因子是0.75，非线程安全等等咯。另外，根据名字猜测，也会带有LinkList的特点。下面分析一下


定义的变量

	/**
     * A dummy entry in the circular linked list of entries in the map.
     * The first real entry is header.nxt, and the last is header.prv.
     * If the map is empty, header.nxt == header && header.prv == header.
     */
    transient LinkedEntry<K, V> header; // 内部双向链表的头结点  


首先，请先看LinkedEntry这个数据结构

### LinkedEntry ###

	/**
     * LinkedEntry adds nxt/prv double-links to plain HashMapEntry.
     */
    static class LinkedEntry<K, V> extends HashMapEntry<K, V> {
        LinkedEntry<K, V> nxt;
        LinkedEntry<K, V> prv;

        /** Create the header entry */
        LinkedEntry() {
            super(null, null, 0, null);
            nxt = prv = this;
        }

        /** Create a normal entry */
        LinkedEntry(K key, V value, int hash, HashMapEntry<K, V> next,
                    LinkedEntry<K, V> nxt, LinkedEntry<K, V> prv) {
            super(key, value, hash, next);
            this.nxt = nxt;
            this.prv = prv;
        }
    }

我们看到 LinkedEntry 继承了 HashMapEntry，并且在其基础上扩展了两个属性：nxt 表示链表的下一个元素，prv表示链表的前一个元素。


而这个header，大家看注释：The first real entry is header.nxt, and the last is header.prv.If the map is empty, header.nxt == header && header.prv == header.第一个真正的entry 是 header 的下一个，最后一个 entry 是 header 的前一个；如果为 null，则他的前一个和后一个都等于 header，因为他是个环形链表，所以 header 的下一个也是最先加入的，header 的前一个是最后一个加入的，下面的图可能大家就明了了

![](http://dl.iteye.com/upload/attachment/0067/9070/f1a24960-6587-38b7-9af0-d0e2cbc1530d.jpg)

    /**
     * True if access ordered, false if insertion ordered.
     */
    private final boolean accessOrder;// 是否按照访问顺序

accessOrder 代表的是是否按照访问顺序，可以分为：按插入顺序的链表，和按访问顺序的链表。true 表示按照访问顺序迭代，false 时表示按照插入顺序。而如果要实现 LRUCache 的 LRU 算法，就需要选择 true。

### 构造方法 ###

	/**
     * Constructs a new empty {@code LinkedHashMap} instance.
     */
    public LinkedHashMap() {
        init();
        accessOrder = false;// 默认为false，也就是插入顺序
    }

    /**
     * Constructs a new {@code LinkedHashMap} instance with the specified
     * capacity.
     *
     * @param initialCapacity
     *            the initial capacity of this map.
     * @throws IllegalArgumentException
     *                when the capacity is less than zero.
     */
    public LinkedHashMap(int initialCapacity) {//initialCapacity是初始化空间大小
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }

    /**
     * Constructs a new {@code LinkedHashMap} instance with the specified
     * capacity and load factor.
     *
     * @param initialCapacity
     *            the initial capacity of this map.
     * @param loadFactor
     *            the initial load factor.
     * @throws IllegalArgumentException
     *             when the capacity is less than zero or the load factor is
     *             less or equal to zero.
     */
    public LinkedHashMap(int initialCapacity, float loadFactor) {// loadFactor是加载因子，当他的size大于initialCapacity*loadFactor的时候就会扩容，他的默认值是0.75
        this(initialCapacity, loadFactor, false);
    }

    /**
     * Constructs a new {@code LinkedHashMap} instance with the specified
     * capacity, load factor and a flag specifying the ordering behavior.
     *
     * @param initialCapacity
     *            the initial capacity of this hash map.
     * @param loadFactor
     *            the initial load factor.
     * @param accessOrder
     *            {@code true} if the ordering should be done based on the last
     *            access (from least-recently accessed to most-recently
     *            accessed), and {@code false} if the ordering should be the
     *            order in which the entries were inserted.
     * @throws IllegalArgumentException
     *             when the capacity is less than zero or the load factor is
     *             less or equal to zero.
     */
    public LinkedHashMap(
            int initialCapacity, float loadFactor, boolean accessOrder) {
        super(initialCapacity, loadFactor);
        init();
        this.accessOrder = accessOrder;
    }

    /**
     * Constructs a new {@code LinkedHashMap} instance containing the mappings
     * from the specified map. The order of the elements is preserved.
     *
     * @param map
     *            the mappings to add.
     */
    public LinkedHashMap(Map<? extends K, ? extends V> map) {
        this(capacityForInitSize(map.size()));
        constructorPutAll(map);
    }
	

下面看看init方法

### init() ###

 	@Override void init() {
        header = new LinkedEntry<K, V>();
    }

这个方法初始化了一个header，也就是双向环形链表的头，并没有存放任何的数据。

在HashMap的源码中， init方法是一个空方法

 	/**
     * This method is called from the pseudo-constructors (clone and readObject)
     * prior to invoking constructorPut/constructorPutAll, which invoke the
     * overridden constructorNewEntry method. Normally it is a VERY bad idea to
     * invoke an overridden method from a pseudo-constructor (Effective Java
     * Item 17). In this case it is unavoidable, and the init method provides a
     * workaround.
     */
    void init() { }

LinkedHashMap 继承 HashMap 后重写了该方法。

初始变量和构造方法讲完后，下面按源文件顺序看看它定义的一些方法

### eldest() ###

 	/**
     * Returns the eldest entry in the map, or {@code null} if the map is empty.
     * @hide
     */
    public Entry<K, V> eldest() {
        LinkedEntry<K, V> eldest = header.nxt;
        return eldest != header ? eldest : null;
    }

这个方法看注释就明了：得到最老的元素，也是最先存放的元素或者如果 map 为空，就返回null。通过上面分析。我们知道最先存放的元素就是 header 的nxt。

### addNewEntry() ###

	/**
     * Evicts eldest entry if instructed, creates a new entry and links it in
     * as head of linked list. This method should call constructorNewEntry
     * (instead of duplicating code) if the performance of your VM permits.
     *
     * <p>It may seem strange that this method is tasked with adding the entry
     * to the hash table (which is properly the province of our superclass).
     * The alternative of passing the "next" link in to this method and
     * returning the newly created element does not work! If we remove an
     * (eldest) entry that happens to be the first entry in the same bucket
     * as the newly created entry, the "next" link would become invalid, and
     * the resulting hash table corrupt.
     */
    @Override void addNewEntry(K key, V value, int hash, int index) {
        LinkedEntry<K, V> header = this.header;

        // Remove eldest entry if instructed to do so.
        LinkedEntry<K, V> eldest = header.nxt;
        if (eldest != header && removeEldestEntry(eldest)) {
            remove(eldest.key);
        }

        // Create new entry, link it on to list, and put it into table
        LinkedEntry<K, V> oldTail = header.prv;
        LinkedEntry<K, V> newTail = new LinkedEntry<K,V>(
                key, value, hash, table[index], header, oldTail);
        table[index] = oldTail.nxt = header.prv = newTail;
    }

这个方法比 HashMap 的 addNewEntry 代码行数要多了。看代码可知，该方法是加入一个新的 entry，首先是找到 header，然后根据条件判断是否移除最老元素，默认的 removeEldestEntry() 方法返回 false

### removeEldestEntry ###

 	protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return false;
    }

如果其返回了 true，就会移除最老元素。

接下来，可以看到，首先是新建了一个新的 LinkedEntry，然后在加入到 table 中，最后一行中header.prv = newTail，这回大家可能明白了 header 的前一个元素就是新元素，也是最后一个加入的。

还有一个 addNewEntryForNullKey 方法，向 HashMap 看齐，是加入 null 的 entry

### addNewEntryForNullKey ###

	@Override void addNewEntryForNullKey(V value) {
        LinkedEntry<K, V> header = this.header;

        // Remove eldest entry if instructed to do so.
        LinkedEntry<K, V> eldest = header.nxt;
        if (eldest != header && removeEldestEntry(eldest)) {
            remove(eldest.key);
        }

        // Create new entry, link it on to list, and put it into table
        LinkedEntry<K, V> oldTail = header.prv;
        LinkedEntry<K, V> newTail = new LinkedEntry<K,V>(
                null, value, 0, null, header, oldTail);
        entryForNullKey = oldTail.nxt = header.prv = newTail;
    }

差别就是最后的一句话

继续看constructorNewEntry()方法

### constructorNewEntry() ###

 	/**
     * As above, but without eviction.
     */
    @Override HashMapEntry<K, V> constructorNewEntry(
            K key, V value, int hash, HashMapEntry<K, V> next) {
        LinkedEntry<K, V> header = this.header;
        LinkedEntry<K, V> oldTail = header.prv;
        LinkedEntry<K, V> newTail
                = new LinkedEntry<K,V>(key, value, hash, next, header, oldTail);
        return oldTail.nxt = header.prv = newTail;
    }

注释很明了：只不过他没有调用移除方法

下面就是get()方法

### get() ###


	/**
     * Returns the value of the mapping with the specified key.
     *
     * @param key
     *            the key.
     * @return the value of the mapping with the specified key, or {@code null}
     *         if no mapping for the specified key is found.
     */
    @Override public V get(Object key) {
        /*
         * This method is overridden to eliminate the need for a polymorphic
         * invocation in superclass at the expense of code duplication.
         */
        if (key == null) {
            HashMapEntry<K, V> e = entryForNullKey;
            if (e == null)
                return null;
            if (accessOrder)//这个方法和HashMap的get方法差不多，只不过他多了一个判断，因为LinkedHashMap多了一个链表的维护，所以他要判断是否是按照访问顺序来维护，
                makeTail((LinkedEntry<K, V>) e);
            return e.value;
        }

        int hash = Collections.secondaryHash(key);
        HashMapEntry<K, V>[] tab = table;
        for (HashMapEntry<K, V> e = tab[hash & (tab.length - 1)];
                e != null; e = e.next) {
            K eKey = e.key;
            if (eKey == key || (e.hash == hash && key.equals(eKey))) {
                if (accessOrder)
                    makeTail((LinkedEntry<K, V>) e);
                return e.value;
            }
        }
        return null;
    }

该方法复写了 HashMap 的 get 方法，并加入了自己的特色。

如果accessOrder为true（安装访问顺序维护），就会调用下面的方法

### makeTail() ###

	/**
     * Relinks the given entry to the tail of the list. Under access ordering,
     * this method is invoked whenever the value of a  pre-existing entry is
     * read by Map.get or modified by Map.put.
     */
    private void makeTail(LinkedEntry<K, V> e) {
        // Unlink e
        e.prv.nxt = e.nxt;
        e.nxt.prv = e.prv;

        // Relink e as tail
        LinkedEntry<K, V> header = this.header;
        LinkedEntry<K, V> oldTail = header.prv;
        e.nxt = header;
        e.prv = oldTail;
        oldTail.nxt = header.prv = e;
        modCount++;
    }

这个方法是实现 Lru 算法的关键，将元素插入到头表前一个元素(离头表最近的元素,也是最新的元素)。

头两行代码的意思就是把元素 e 从双向链表中删除，然后将 e 添加到 header 的前面（就是 tail），相当于把 e 又新添加到了双向链表中，最后的几行代码就是确保整个双向链表不能断掉。

get 方法的剩余部分类似 HashMap 的 get 方法，只不过在循环查找过程中，会再次判断访问类型。

### preModify() ###

	@Override void preModify(HashMapEntry<K, V> e) {
        if (accessOrder) {
            makeTail((LinkedEntry<K, V>) e);
        }
    }

该方法被 LinkedHashMap 重写，还记得 HashMap 的 put 方法（LinkedHashMap 并没有重写该方法，所以还是用的 HashMap 的 put 方法）吗，那里面涉及到了该方法。如果 accessOrder 为 true ，则 LinkedHashMap 的 put 效果如下

![](http://img.anywalks.com/s/1/Uploads/article_remote01/2015/11/27/132ee6853e5be3691bc0254ebdde386a.jpg)

看上面这张图咯，有没有啥问题呢？

如果 accessOrder 为 true ，那么，在 put<k1,v1> 时，就不应该放在 header.nxt，而应该是 entry_1 和 entry_0 颠倒一下就正确了。（为的就是保证每次 header 的 tail 是最近用到的，而 header.nxt 就是最久未使用的）

最新的元素总是属于 header 的 pre （也就是 the last）

### postRemove() ###

    @Override void postRemove(HashMapEntry<K, V> e) {
        LinkedEntry<K, V> le = (LinkedEntry<K, V>) e;
        le.prv.nxt = le.nxt;
        le.nxt.prv = le.prv;
        le.nxt = le.prv = null; // Help the GC (for performance)
    }

该方法就是去除结点的操作

### containsValue() ###

	/**
     * This override is done for LinkedHashMap performance: iteration is cheaper
     * via LinkedHashMap nxt links.
     */
    @Override public boolean containsValue(Object value) {
        if (value == null) {
            for (LinkedEntry<K, V> header = this.header, e = header.nxt;
                    e != header; e = e.nxt) {
                if (e.value == null) {
                    return true;
                }
            }
            return false;
        }

        // value is non-null
        for (LinkedEntry<K, V> header = this.header, e = header.nxt;
                e != header; e = e.nxt) {//迭代链表
            if (value.equals(e.value)) {
                return true;
            }
        }
        return false;
    }

LinkedHashMap 的 containsValue 方法是遍历双向链表，从链表的 header 开始查找，因为是环形的，如果查找一圈还没找到则返回false。 containsValue 从链表中查询，而 HashMap 从 table 数组中查询，进行该操作时，没有 HashMap 快（数组比链表迭代快）。

### clear() ###

	public void clear() {
        super.clear();

        // Clear all links to help GC
        LinkedEntry<K, V> header = this.header;
        for (LinkedEntry<K, V> e = header.nxt; e != header; ) {
            LinkedEntry<K, V> nxt = e.nxt;
            e.nxt = e.prv = null;
            e = nxt;
        }

        header.nxt = header.prv = header;
    }

清空链表咯

剩下的就是迭代器类了，就不贴了，感兴趣的可以自己看源码，注意：HashMap：是迭代数组，LinkedHashMap 迭代链表。

## 参考链接 ##

[LRUCache原理及HashMap LinkedHashMap内部实现原理](http://www.dlxedu.com/detail/20/471266.html)

[Android LinkedHashMap源码详解](http://blog.csdn.net/abcdef314159/article/details/51178860)

[理解LinkedHashMap](http://www.cnblogs.com/children/archive/2012/10/02/2710624.html)
