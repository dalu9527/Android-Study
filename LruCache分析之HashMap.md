# LruCache之HashMap分析

LruCache是Android的一个内部类，提供了基于内存实现的缓存，打算分析一下其实现的原理，不过发现其内部是基于LinkedHashMap，而这个又是基于HashMap，所以，从头开始学习咯。

在讨论HashMap前，有必要先谈谈数组和链表这两种常用数据结构。

数组在内存中开辟的空间是连续的，如果要插入或者删除一个node，那么这个node之后的所有数据都要整体move，但数组的查询快（二分查找）。其特点是：寻址容易，增删困难

链表在内存中离散存储，插入和删除十分轻松，但查询较慢，每次都要从头到尾遍历一遍。其特点是：寻址困难，增删容易

哈希表的存在就是取其精华，去其糟粕。

哈希表有多种不同的实现方案，本文接下来介绍的是最常用的一种（也是JDK中HashMap的实现方案）—— 拉链法，我们可以理解为“数组+链表” 。如图：

![](http://img.blog.csdn.net/20151220011610891)

（以下是以 Android SDK 里面的方法，和 Java JDK 里面的些许不同，谷歌程序猿：乌拉）

HashMap内部其实就是一个Entry数组(table)

	/**
     * The hash table. If this hash map contains a mapping for null, it is
     * not represented this hash table.
     */
    transient HashMapEntry<K, V>[] table;

数组中每个元素都可以看成一个桶，每一个桶又构成了一条链表。

## 源码 ##

### 构造方法 ###

	/**
     * Constructs a new empty {@code HashMap} instance.
     */
    @SuppressWarnings("unchecked")
    public HashMap() {
        table = (HashMapEntry<K, V>[]) EMPTY_TABLE;
        threshold = -1; // Forces first put invocation to replace EMPTY_TABLE
    }

看构造方法，会涉及到HashMapEntry这个数据结构

	static class HashMapEntry<K, V> implements Entry<K, V> {
        final K key;// 键
        V value;// 值
        final int hash;// 哈希值
        HashMapEntry<K, V> next;// 指向下一个节点

        HashMapEntry(K key, V value, int hash, HashMapEntry<K, V> next) {
            this.key = key;
            this.value = value;
            this.hash = hash;
            this.next = next;
        }

        public final K getKey() {
            return key;
        }

        public final V getValue() {
            return value;
        }

        public final V setValue(V value) {
            V oldValue = this.value;
            this.value = value;
            return oldValue;
        }
		// 判断两个Entry是否相等
		
        @Override public final boolean equals(Object o) {
            if (!(o instanceof Entry)) {// 如果Object o不是Entry类型，直接返回false
                return false;
            }
            Entry<?, ?> e = (Entry<?, ?>) o;
            return Objects.equal(e.getKey(), key)
                    && Objects.equal(e.getValue(), value);// 若两个Entry的“key”和“value”都相等，则返回true。
        }

        @Override public final int hashCode() {// 实现hashCode()，判断存储在哪个数组里面
            return (key == null ? 0 : key.hashCode()) ^
                    (value == null ? 0 : value.hashCode());
        }

        @Override public final String toString() {
            return key + "=" + value;
        }
    }

这是一个静态内部类 HashMapEntry，实现的是Entry接口，是HashMap链式存储对应的链表（数组加链表形式）

继续看，涉及到了 EMPTY_TABLE 定义如下

	/**
     * An empty table shared by all zero-capacity maps (typically from default
     * constructor). It is never written to, and replaced on first put. Its size
     * is set to half the minimum, so that the first resize will create a
     * minimum-sized table.
     */
    private static final Entry[] EMPTY_TABLE
            = new HashMapEntry[MINIMUM_CAPACITY >>> 1];

 	/**
     * Min capacity (other than zero) for a HashMap. Must be a power of two
     * greater than 1 (and less than 1 << 30).
     */
    private static final int MINIMUM_CAPACITY = 4;

MINIMUM_CAPACITY往右移动一位，大小变为2了

所以上面的构造函数就是在HashMap中有一个table，保存的是一个HashMapEntry类型的数组，数组的大小为2（区别与OracleJDK，其默认大小是16）

    /**
     * Constructs a new {@code HashMap} instance with the specified capacity.
     *
     * @param capacity
     *            the initial capacity of this hash map.
     * @throws IllegalArgumentException
     *                when the capacity is less than zero.
     */
    public HashMap(int capacity) {
        if (capacity < 0) {//容量小于领，抛异常
            throw new IllegalArgumentException("Capacity: " + capacity);
        }

        if (capacity == 0) {//容量为零，替换成默认的EMPTY_TABLE
            @SuppressWarnings("unchecked")
            HashMapEntry<K, V>[] tab = (HashMapEntry<K, V>[]) EMPTY_TABLE;
            table = tab;
            threshold = -1; // Forces first put() to replace EMPTY_TABLE
            return;
        }

        if (capacity < MINIMUM_CAPACITY) {//不能小于规定的最小值，也就是4
            capacity = MINIMUM_CAPACITY;
        } else if (capacity > MAXIMUM_CAPACITY) {不能大于规定的最大值，也就是 1 << 30
            capacity = MAXIMUM_CAPACITY;
        } else {//寻找最合适的
            capacity = Collections.roundUpToPowerOfTwo(capacity);
        }
        makeTable(capacity);
    }

Collections.roundUpToPowerOfTwo()

	/**
     * Returns the smallest power of two >= its argument, with several caveats:
     * If the argument is negative but not Integer.MIN_VALUE, the method returns
     * zero. If the argument is > 2^30 or equal to Integer.MIN_VALUE, the method
     * returns Integer.MIN_VALUE. If the argument is zero, the method returns
     * zero.
     * @hide
     */
    public static int roundUpToPowerOfTwo(int i) {
        i--; // If input is a power of two, shift its high-order bit right.

        // "Smear" the high-order bit all the way to the right.
        i |= i >>>  1;
        i |= i >>>  2;
        i |= i >>>  4;
        i |= i >>>  8;
        i |= i >>> 16;

        return i + 1;
    }

看注释应该知道这个方法的功能就是返回一个比指定值i，也就是上面的 capacity 大的2的n次方的最小值

例如输入i = 17，二进制表示为00010001


i -- 后，　　　　	二进制表示为 00010000，十进制i = 16
i >>> 1 后，　　	二进制表示为 00001000，十进制i = 8
i |= 后，　　　　	二进制表示为 00011000，十进制i = 24
i >>> 2 后，　　	二进制表示为 00000110，十进制i = 6
i |= 后，　　　　	二进制表示为 00011110，十进制i = 30
i >>> 4 后，　　	二进制表示为 00000001，十进制i = 1
i |= 后，　　　　	二进制表示为 00011111，十进制i = 31
i >>> 8 后，　	　二进制表示为 00011111，十进制i = 31
i |= 后，　　　　	二进制表示为 00011111，十进制i = 31
i >>> 16 后，　　二进制表示为 00011111，十进制i = 31
i |= 后，　　　　	二进制表示为 00011111，十进制i = 31
i + 1 后，　　　	二进制表示为 00100000，十进制i = 32

所以，比i = 17大的最小的2的次方应该是2的5次方

例如输入i = 16，二进制表示为0001000

i -- 后，　　　　	二进制表示为 00001111，十进制i = 15
i >>> 1 后，　　	二进制表示为 00000111，十进制i = 7
i |= 后，　　　　	二进制表示为 00001111，十进制i = 15
i >>> 2 后，	　　	二进制表示为 00000011，十进制i = 3
i |= 后，　　　　	二进制表示为 00001111，十进制i = 15
i >>> 4 后，	　　	二进制表示为 00000000，十进制i = 0
i |= 后，　　　　	二进制表示为 00001111，十进制i = 15
i >>> 8 后，　　	二进制表示为 00001111，十进制i = 15
i |= 后，　　　　	二进制表示为 00001111，十进制i = 15
i >>> 16 后，　　二进制表示为 00001111，十进制i = 15
i |= 后，　　　　	二进制表示为 00001111，十进制i = 15
i + 1 后，	　　　	二进制表示为 00010000，十进制i = 16

所以，比i = 16大的最小的2的次方应该是2的4次方，就是其本身


言归正传，继续看makeTable方法

	/**
     * Allocate a table of the given capacity and set the threshold accordingly.
     * @param newCapacity must be a power of two
     */
    private HashMapEntry<K, V>[] makeTable(int newCapacity) {
        @SuppressWarnings("unchecked") HashMapEntry<K, V>[] newTable
                = (HashMapEntry<K, V>[]) new HashMapEntry[newCapacity];
        table = newTable;
        threshold = (newCapacity >> 1) + (newCapacity >> 2); // 3/4 capacity 
        return newTable;
    }

根据代码可知，其初始化了一个HashMapEntry类型的数组table，用来存放HashMapEntry，而threshold如它的翻译般，就是一个阙值，这个值是将来扩容的参考，是容量的3/4(新阈值 = 新容量/2 +  新容量/4，相当于乘以容量的 3/4，not care 加载因子)

    /**
     * Constructs a new {@code HashMap} instance with the specified capacity and
     * load factor.
     *
     * @param capacity
     *            the initial capacity of this hash map.
     * @param loadFactor
     *            the initial load factor.
     * @throws IllegalArgumentException
     *                when the capacity is less than zero or the load factor is
     *                less or equal to zero or NaN.
     */
    public HashMap(int capacity, float loadFactor) {
        this(capacity);

        if (loadFactor <= 0 || Float.isNaN(loadFactor)) {
            throw new IllegalArgumentException("Load factor: " + loadFactor);
        }

        /*
         * Note that this implementation ignores loadFactor; it always uses
         * a load factor of 3/4. This simplifies the code and generally
         * improves performance.
         */
    }

上面这个个构造方法最终会调用这个构造方法 HashMap(int capacity)，看注释，里面说明了装载因子为0.75。

注意：OracleJDK中的阈值计算公式是：当前Entry数组长度*加载因子，其默认的加载因子是0.75，加载因子也可以通过构造器来设置。AndroidJDK的加载因子也是0.75，不同的是，AndroidJDK不支持其他数值的加载因子

    /**
     * Constructs a new {@code HashMap} instance containing the mappings from
     * the specified map.
     *
     * @param map
     *            the mappings to add.
     */
    public HashMap(Map<? extends K, ? extends V> map) {
        this(capacityForInitSize(map.size()));
        constructorPutAll(map);
    }

这个构造方法传入的参数是个 map，根据 map 的大小初始化。

HashMap 的构造方法讲完后，一般我们使用时，都是先往里面放数据，下面就看看 put() 方法

### put() ###


	/**
     * Maps the specified key to the specified value.
     *
     * @param key
     *            the key.
     * @param value
     *            the value.
     * @return the value of any previous mapping with the specified key or
     *         {@code null} if there was no such mapping.
     */
    @Override public V put(K key, V value) {
        if (key == null) {// 若key为null，则将该键值对添加到table[0]中。
            return putValueForNullKey(value);
        }
		// 若key不为null，则计算该key的哈希值，然后将其添加到该哈希值对应的链表中。
        int hash = Collections.secondaryHash(key);// Collections.secondaryHash能够使得hash过后的值的分布更加均匀，尽可能地避免冲突
        HashMapEntry<K, V>[] tab = table;
        int index = hash & (tab.length - 1);// 根据hash值计算存储位置 index的值永远都是0到2的n次方减1之间，可以保证结果不大于tab.length
		
        for (HashMapEntry<K, V> e = tab[index]; e != null; e = e.next) {// 循环遍历Entry数组,若key对应的键值对已经存在，则用新的value取代旧的value。然后返回原来的 oldValue
            if (e.hash == hash && key.equals(e.key)) {//根据hash值是否相等以及 key值是否一样进行判断
                preModify(e);
                V oldValue = e.value;
                e.value = value;
                return oldValue;
            }
        }

        // No entry for (non-null) key is present; create one
        modCount++;// 修改次数+1
        if (size++ > threshold) {// 如何hashmap的大小超过了阙值，进行扩容
            tab = doubleCapacity();
            index = hash & (tab.length - 1);
        }
        addNewEntry(key, value, hash, index);// 添加新的Entry
        return null;
    }

里面涉及到了，Collections.secondaryHash(key)，该函数的原理在[这里](http://burtleburtle.net/bob/hash/integer.html)

从源码可以看出，put 方法是有返回值的（可以理解为put也是包含了get方法的精髓），根据返回值不同，可以有其他作用。

另外，我们构造 HashMap 的构造方法时， 阙值 threshold = -1 ，是满足二倍扩容的，也就是说，在 AndroidJDK 的 HashMap 中使用无参构造方法后，第一次 put 数据就会触发哈希表的二倍扩容，因为扩容后数组的长度发生了变化，所以数据入桶的位置也会发生变化，这个时候需要新构建 Hash 表。而另外，HashMap 是允许 Key 为null，看看 putValueForNullKey 方法

	private V putValueForNullKey(V value) {
        HashMapEntry<K, V> entry = entryForNullKey;
        if (entry == null) {
            addNewEntryForNullKey(value);
            size++;// hashmap大小 + 1
            modCount++;// 修改次数 + 1
            return null;
        } else {
            preModify(entry);
            V oldValue = entry.value;
            entry.value = value;
            return oldValue;
        }
    }

entryForNullKey的定义如下

 	/**
     * The entry representing the null key, or null if there's no such mapping.
     */
    transient HashMapEntry<K, V> entryForNullKey;

还是个HashMapEntry

当entry为空时，此时调用 addNewEntryForNullKey 方法，如下

	/**
     * Creates a new entry for the null key, and the given value and
     * inserts it into the hash table. This method is called by put
     * (and indirectly, putAll), and overridden by LinkedHashMap.
     */
    void addNewEntryForNullKey(V value) {
        entryForNullKey = new HashMapEntry<K, V>(null, value, 0, null);
    }

会新构造一个新的HashMapEntry，传入value，其他为null or 0。

当entry不为空时，调用 preModify 方法，如下

	/**
     * Give LinkedHashMap a chance to take action when we modify an existing
     * entry.
     *
     * @param e the entry we're about to modify.
     */
    void preModify(HashMapEntry<K, V> e) { }// LinkedHashMap实现

好吧，就一个空方法，接下来就是返回 oldValue

当key 不为空时，主要过程已经标注在了代码中，这里看一下扩容方法 doubleCapacity()

	/**
     * Doubles the capacity of the hash table. Existing entries are placed in
     * the correct bucket on the enlarged table. If the current capacity is,
     * MAXIMUM_CAPACITY, this method is a no-op. Returns the table, which
     * will be new unless we were already at MAXIMUM_CAPACITY.
     */
    private HashMapEntry<K, V>[] doubleCapacity() {
        HashMapEntry<K, V>[] oldTable = table;// 原table 标记为 oldTable
        int oldCapacity = oldTable.length;// 旧容量
        if (oldCapacity == MAXIMUM_CAPACITY) {// 就容量已经是最大值了，就不用扩容了（也扩不了）
            return oldTable;
        }
        int newCapacity = oldCapacity * 2;// 新容量是旧容量的2倍
        HashMapEntry<K, V>[] newTable = makeTable(newCapacity);// 根据新容量重新创建一个
        if (size == 0) {// 如果原来HashMap的size为0，则直接返回  
            return newTable;
        }

        for (int j = 0; j < oldCapacity; j++) {
            /*
             * Rehash the bucket using the minimum number of field writes.
             * This is the most subtle and delicate code in the class.
             */
			// 代码注释都说下面的代码是非常精妙的了，那就看看咯
            HashMapEntry<K, V> e = oldTable[j];
            if (e == null) {
                continue;
            }

			// 下面这三行，忒抽象，举例说明好了
			//oldCapacity假设为16(00010000)，int highBit = e.hash & oldCapacity能够得到高位的值，因为经过与操作过后，低位一定是0
            int highBit = e.hash & oldCapacity;
            HashMapEntry<K, V> broken = null;
			// J 在这里是index，J 与 高位的值进行或操作过后，就能得到在扩容后的新的index值。
			// 设想一下，理论上我们得到的新的值应该是 newValue = hash & (newCapacity - 1) 
			// 与 oldValue = hash & (oldCapacity - 1) 的区别仅在于高位上。 
			// 因此我们用 J | highBit 就可以得到新的index值。
            newTable[j | highBit] = e;
			// 下面的操作就是如果当前元素下面挂载的还有元素就重新排放
            for (HashMapEntry<K, V> n = e.next; n != null; e = n, n = n.next) {
				//跟上面的类似，以下一个高位作为排放的依据
                int nextHighBit = n.hash & oldCapacity;
                if (nextHighBit != highBit) {// 说明位于不同的位置，just存放就可以
                    if (broken == null)
                        newTable[j | nextHighBit] = n;
                    else
                        broken.next = n;
                    broken = e;
                    highBit = nextHighBit;
                }// 如果相等说明这两个元素肯定还位于数组的同一位置以链表的形式存在，not care
            }
            if (broken != null)
                broken.next = null;
        }
        return newTable;
    }

扩容方法讲完后，继续put方法的addNewEntry(key, value, hash, index);

	/**
     * Creates a new entry for the given key, value, hash, and index and
     * inserts it into the hash table. This method is called by put
     * (and indirectly, putAll), and overridden by LinkedHashMap. The hash
     * must incorporate the secondary hash function.
     */
    void addNewEntry(K key, V value, int hash, int index) {
        table[index] = new HashMapEntry<K, V>(key, value, hash, table[index]);
    }

这个方法就是将老Entry作为新建Entry对象的next节点返回给当前数组元素（物理空间上其实是在链表头部添加新节点）

put 方法后，就看看 get() 方法

### get() ###

	/**
     * Returns the value of the mapping with the specified key.
     *
     * @param key
     *            the key.
     * @return the value of the mapping with the specified key, or {@code null}
     *         if no mapping for the specified key is found.
     */
    public V get(Object key) {
        if (key == null) {//取空
            HashMapEntry<K, V> e = entryForNullKey;
            return e == null ? null : e.value;
        }

        int hash = Collections.secondaryHash(key);
        HashMapEntry<K, V>[] tab = table;
        for (HashMapEntry<K, V> e = tab[hash & (tab.length - 1)];
                e != null; e = e.next) {
            K eKey = e.key;
            if (eKey == key || (e.hash == hash && key.equals(eKey))) {
                return e.value;
            }
        }
        return null;
    }

这个方法，看代码就应该可以理解，和 put 的操作相反，怎么存的再怎么取出来就ok。

下面看看 containsKey 方法

### containsKey() ###

	/**
     * Returns whether this map contains the specified key.
     *
     * @param key
     *            the key to search for.
     * @return {@code true} if this map contains the specified key,
     *         {@code false} otherwise.
     */
    @Override public boolean containsKey(Object key) {
        if (key == null) {
            return entryForNullKey != null;
        }

        int hash = Collections.secondaryHash(key);
        HashMapEntry<K, V>[] tab = table;
        for (HashMapEntry<K, V> e = tab[hash & (tab.length - 1)];
                e != null; e = e.next) {
            K eKey = e.key;
            if (eKey == key || (e.hash == hash && key.equals(eKey))) {
                return true;
            }
        }
        return false;
    }

和 get 方法类似，返回方法不同而已。

看看孪生兄弟 containsValue 

### containsValue ###

	/**
     * Returns whether this map contains the specified value.
     *
     * @param value
     *            the value to search for.
     * @return {@code true} if this map contains the specified value,
     *         {@code false} otherwise.
     */
    @Override public boolean containsValue(Object value) {
        HashMapEntry[] tab = table;
        int len = tab.length;
        if (value == null) {
            for (int i = 0; i < len; i++) {
                for (HashMapEntry e = tab[i]; e != null; e = e.next) {
                    if (e.value == null) {
                        return true;
                    }
                }
            }
            return entryForNullKey != null && entryForNullKey.value == null;
        }

        // value is non-null
        for (int i = 0; i < len; i++) {
            for (HashMapEntry e = tab[i]; e != null; e = e.next) {
                if (value.equals(e.value)) {
                    return true;
                }
            }
        }
        return entryForNullKey != null && value.equals(entryForNullKey.value);
    }

这个是查找是否含有value，和上面查找是否含有key类似，不过这个的循环次数就比寻找key要多，数组和链表都要查找一遍（没有办法啦，谁让 hashMap 是根据 key 来存，偏偏要取 value ，不得不耗时咯）。

前面有一个构造方法，没有仔细看

  	/**
     * Constructs a new {@code HashMap} instance containing the mappings from
     * the specified map.
     *
     * @param map
     *            the mappings to add.
     */
    public HashMap(Map<? extends K, ? extends V> map) {
        this(capacityForInitSize(map.size()));
        constructorPutAll(map);
    }
 
首先看看  capacityForInitSize 方法

### capacityForInitSize() ###

	/**
     * Returns an appropriate capacity for the specified initial size. Does
     * not round the result up to a power of two; the caller must do this!
     * The returned value will be between 0 and MAXIMUM_CAPACITY (inclusive).
     */
    static int capacityForInitSize(int size) {
        int result = (size >> 1) + size; // Multiply by 3/2 to allow for growth

        // boolean expr is equivalent to result >= 0 && result<MAXIMUM_CAPACITY
        return (result & ~(MAXIMUM_CAPACITY-1))==0 ? result : MAXIMUM_CAPACITY;
    }

这个方法是根据 map 的 size 进行合理的扩容，扩容的大小就是 size 的 1.5 倍，最后是根据扩容的大小判断返回值，如果扩容的大小大于 1<<30 则返回 1<<30 (MAXIMUM_CAPACITY),否则就返回扩容后的大小。

下面就是 constructorPutAll 方法

### constructorPutAll() ###

	/**
     * Inserts all of the elements of map into this HashMap in a manner
     * suitable for use by constructors and pseudo-constructors (i.e., clone,
     * readObject). Also used by LinkedHashMap.
     */
    final void constructorPutAll(Map<? extends K, ? extends V> map) {
        if (table == EMPTY_TABLE) {
            doubleCapacity(); // Don't do unchecked puts to a shared table.
        }
        for (Entry<? extends K, ? extends V> e : map.entrySet()) {
            constructorPut(e.getKey(), e.getValue());
        }
    }

该方法会把所有的 key 和 value 存储起来，利用 constructorPut 方法

### constructorPut() ###

	/**
     * This method is just like put, except that it doesn't do things that
     * are inappropriate or unnecessary for constructors and pseudo-constructors
     * (i.e., clone, readObject). In particular, this method does not check to
     * ensure that capacity is sufficient, and does not increment modCount.
     */
    private void constructorPut(K key, V value) {
        if (key == null) {
            HashMapEntry<K, V> entry = entryForNullKey;
            if (entry == null) {
                entryForNullKey = constructorNewEntry(null, value, 0, null);
                size++;
            } else {
                entry.value = value;
            }
            return;
        }

        int hash = Collections.secondaryHash(key);
        HashMapEntry<K, V>[] tab = table;
        int index = hash & (tab.length - 1);
        HashMapEntry<K, V> first = tab[index];
        for (HashMapEntry<K, V> e = first; e != null; e = e.next) {
            if (e.hash == hash && key.equals(e.key)) {
                e.value = value;
                return;
            }
        }

        // No entry for (non-null) key is present; create one
        tab[index] = constructorNewEntry(key, value, hash, first);
        size++;
    }

该方法和 put 方法很类似，但是注释也讲明了二者的区别。

再看 putAll 方法

### putAll() ###

	/**
     * Copies all the mappings in the specified map to this map. These mappings
     * will replace all mappings that this map had for any of the keys currently
     * in the given map.
     *
     * @param map
     *            the map to copy mappings from.
     */
    @Override public void putAll(Map<? extends K, ? extends V> map) {
        ensureCapacity(map.size());
        super.putAll(map);
    }

调用了 ensureCapacity 方法

### ensureCapacity() ###

	/**
     * Ensures that the hash table has sufficient capacity to store the
     * specified number of mappings, with room to grow. If not, it increases the
     * capacity as appropriate. Like doubleCapacity, this method moves existing
     * entries to new buckets as appropriate. Unlike doubleCapacity, this method
     * can grow the table by factors of 2^n for n > 1. Hopefully, a single call
     * to this method will be faster than multiple calls to doubleCapacity.
     *
     *  <p>This method is called only by putAll.
     */
    private void ensureCapacity(int numMappings) {
        int newCapacity = Collections.roundUpToPowerOfTwo(capacityForInitSize(numMappings));// 上面讲过
        HashMapEntry<K, V>[] oldTable = table;
        int oldCapacity = oldTable.length;
        if (newCapacity <= oldCapacity) {
            return;
        }
        if (newCapacity == oldCapacity * 2) {// 初始化的空间大小是原来大小2倍
            doubleCapacity();
            return;
        }

        // We're growing by at least 4x, rehash in the obvious way
        HashMapEntry<K, V>[] newTable = makeTable(newCapacity);// 根据newCapacity初始化新数组
        if (size != 0) {// 重新挂载
            int newMask = newCapacity - 1;
            for (int i = 0; i < oldCapacity; i++) {
                for (HashMapEntry<K, V> e = oldTable[i]; e != null;) {
                    HashMapEntry<K, V> oldNext = e.next;
                    int newIndex = e.hash & newMask;
                    HashMapEntry<K, V> newNext = newTable[newIndex];
                    newTable[newIndex] = e;
                    e.next = newNext;
                    e = oldNext;
                }
            }
        }
    }

接下来就是 remove 方法

### remove() ###

	/**
     * Removes the mapping with the specified key from this map.
     *
     * @param key
     *            the key of the mapping to remove.
     * @return the value of the removed mapping or {@code null} if no mapping
     *         for the specified key was found.
     */
    @Override public V remove(Object key) {
        if (key == null) {// key 为空就调用下面方法去除
            return removeNullKey();
        }
        int hash = Collections.secondaryHash(key);
        HashMapEntry<K, V>[] tab = table;
        int index = hash & (tab.length - 1);
        for (HashMapEntry<K, V> e = tab[index], prev = null;
                e != null; prev = e, e = e.next) {// 对链表的操作
            if (e.hash == hash && key.equals(e.key)) {
                if (prev == null) {
                    tab[index] = e.next;
                } else {
                    prev.next = e.next;
                }
                modCount++;// 修改次数 + 1
                size--;// 大小 - 1
                postRemove(e);// 标记去除的e
                return e.value;
            }
        }
        return null;
    }

    private V removeNullKey() {// 针对key 为null 进行处理
        HashMapEntry<K, V> e = entryForNullKey;
        if (e == null) {
            return null;
        }
        entryForNullKey = null;
        modCount++;
        size--;
        postRemove(e);
        return e.value;
    }

    /**
     * Subclass overrides this method to unlink entry.
     */
    void postRemove(HashMapEntry<K, V> e) { }// LinkedHashMap实现

休息一下

还记得 HashMap 的类声明吗，

	public class HashMap<K, V> extends AbstractMap<K, V> implements Cloneable, Serializable 

实现了 Cloneable 和 Serializable接口,都是两个空接口，一个实现浅拷贝，一个实现序列化

### clone() ###

	/**
     * Returns a shallow copy of this map.
     *
     * @return a shallow copy of this map.
     */
    @SuppressWarnings("unchecked")
    @Override public Object clone() {
        /*
         * This could be made more efficient. It unnecessarily hashes all of
         * the elements in the map.
         */
        HashMap<K, V> result;
        try {
            result = (HashMap<K, V>) super.clone();// here
        } catch (CloneNotSupportedException e) {
            throw new AssertionError(e);
        }

        // Restore clone to empty state, retaining our capacity and threshold
        result.makeTable(table.length);
        result.entryForNullKey = null;
        result.size = 0;
        result.keySet = null;
        result.entrySet = null;
        result.values = null;

        result.init(); // Give subclass a chance to initialize itself
        result.constructorPutAll(this); // Calls method overridden in subclass!!
        return result;
    }

### HashMap 的迭代器 ###

 	private abstract class HashIterator {
        int nextIndex;
        HashMapEntry<K, V> nextEntry = entryForNullKey;
        HashMapEntry<K, V> lastEntryReturned;
        int expectedModCount = modCount;

        HashIterator() {
            if (nextEntry == null) {
                HashMapEntry<K, V>[] tab = table;
                HashMapEntry<K, V> next = null;
                while (next == null && nextIndex < tab.length) {
                    next = tab[nextIndex++];
                }
                nextEntry = next;
            }
        }

        public boolean hasNext() {
            return nextEntry != null;
        }

        HashMapEntry<K, V> nextEntry() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            if (nextEntry == null)
                throw new NoSuchElementException();

            HashMapEntry<K, V> entryToReturn = nextEntry;
            HashMapEntry<K, V>[] tab = table;
            HashMapEntry<K, V> next = entryToReturn.next;
            while (next == null && nextIndex < tab.length) {
                next = tab[nextIndex++];
            }
            nextEntry = next;
            return lastEntryReturned = entryToReturn;
        }

        public void remove() {
            if (lastEntryReturned == null)
                throw new IllegalStateException();
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            HashMap.this.remove(lastEntryReturned.key);
            lastEntryReturned = null;
            expectedModCount = modCount;
        }
    }


上面的是定义在 HashMap 内部的迭代器类，在迭代的时候，外部可以通过调用 put 和 remove 的方法，来改变正在迭代的对象。但从设计之处，HashMap自身就不是线程安全的，因此HashMap在迭代的时候使用了一种Fast—Fail的实现方式，在 HashIterator 里面维持了一个 expectedModCount 的变量，在每次调用的时候如果发现 ModCount != expectedModCount，则抛出 ConcurrentModificationException 异常。但本身这种检验不能保证在发生错误的情况下，一定能抛出异常，所以我们需要在使用HashMap的时候，心里知道这是「非线程安全」的。

### HashMap 的序列化 ###

  private void writeObject(ObjectOutputStream stream) throws IOException {
        // Emulate loadFactor field for other implementations to read
        ObjectOutputStream.PutField fields = stream.putFields();
        fields.put("loadFactor", DEFAULT_LOAD_FACTOR);
        stream.writeFields();
			
        stream.writeInt(table.length); // Capacity 写入容量
        stream.writeInt(size);// 写入数量
        for (Entry<K, V> e : entrySet()) {// 迭代写入key 和value
            stream.writeObject(e.getKey());
            stream.writeObject(e.getValue());
        }
    }

    private void readObject(ObjectInputStream stream) throws IOException,
            ClassNotFoundException {
        stream.defaultReadObject();
        int capacity = stream.readInt();
		/**下面的代码和上面的很类似*/
        if (capacity < 0) {
            throw new InvalidObjectException("Capacity: " + capacity);
        }
        if (capacity < MINIMUM_CAPACITY) {
            capacity = MINIMUM_CAPACITY;
        } else if (capacity > MAXIMUM_CAPACITY) {
            capacity = MAXIMUM_CAPACITY;
        } else {
            capacity = Collections.roundUpToPowerOfTwo(capacity);
        }

        makeTable(capacity);// 根据容量创建

        int size = stream.readInt();// 获得size大小
        if (size < 0) {
            throw new InvalidObjectException("Size: " + size);
        }

        init(); // Give subclass (LinkedHashMap) a chance to initialize itself
        for (int i = 0; i < size; i++) {
            @SuppressWarnings("unchecked") K key = (K) stream.readObject();
            @SuppressWarnings("unchecked") V val = (V) stream.readObject();
            constructorPut(key, val); // 前面讲到的另一个类似put的方法
        }
    }

涉及序列化，需要了解一个 Java 的关键字 [transient](http://www.blogjava.net/fhtdy2004/archive/2009/06/20/286112.html) ，该关键字用来表示一个域不是该对象串行化的一部分。当一个对象被串行化的时候，transient 型变量的值不包括在串行化的表示中，然而非 transient 型的变量是被包括进去的。  

## 总结 ##

至此，HashMap 算是告一段落，可以看出谷歌处理 HashMap 时和 Java JDK 里面的方法的不同。虽然谷歌工程师大牛，但是也存在一些问题

- HashMap的每一次扩容都会重新构建一个length是原来两倍的Entry表，这个二倍扩容的策略很容易造成空间浪费。试想一下，假如我们总共有100万条数据要存放，当我put到第75万条时达到阈值，Hash表会重新构建一个200万大小的数组，但是我们最后只放了100万数据，剩下的100万个空间将被浪费。
- HashMap在存储这些数据的过程中需要不断扩容，不断的构建Entry表，不断的做hash运算，会很慢。
- 此外，HashMap获取数据是通过遍历Entry链表来实现的，在数据量很大时候会慢上加慢。

在 Android 项目中使用 HashMap，主要针对小数据量的任务比较ok。


## 参考链接 ##

[HashMap源码分析](http://blog.leanote.com/post/snzang/c5ac5124a7cc)

[Android HashMap源码详解](http://blog.csdn.net/abcdef314159/article/details/51165630)

[Java HashMap 源码解析](http://woaitqs.github.io/program/2015/04/14/read-source-code-about-hashmap)