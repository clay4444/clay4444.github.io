---
title: HashMap源码赏析
abbrlink: 526318cc
date: 2018-01-14 11:13:32
tags:
  - 源码解析系列
---

### 什么是HashMap

基于哈希表的一个Map接口实现，存储的对象是一个键值对对象(Entry<K,V>)；

### HashMap补充说明

基于数组和链表实现，内部维护着一个数组table，该数组保存着每个链表的表头结点；查找时，先通过hash函数计算key的hash值，再根据key的hash值计算数组索引（取余法），然后根据索引找到链表表头结点，然后遍历查找该链表；

### HashMap数据结构

画了个示意图，如下，左边的数组索引是根据key的hash值计算得到，不同hash值有可能产生一样的索引，即哈希冲突，此时采用链地址法处理哈希冲突，即将所有索引一致的节点构成一个单链表；

{% asset_img HashMap.png %}

### HashMap源码分析，大部分都加了注释

```java
package java.util;
import java.io.*;

public class HashMap<K,V>
    extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable
{

    /**
     * 默认初始容量，默认为2的4次方 = 16，2的n次方是为了加快hash计算速度，；；减少hash冲突，，，h & (length-1)，，1111111
     */
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

    /**
     * 最大容量，默认为2的30次方，
     */
    static final int MAXIMUM_CAPACITY = 1 << 30;

    /**
     * 默认负载因子，默认为0.75，也就是到容量的3/4的时候，就开始扩容。
     */
    static final float DEFAULT_LOAD_FACTOR = 0.75f;

    /**
     * 当数组表还没扩容的时候，一个共享的空表对象
     */
    static final Entry<?,?>[] EMPTY_TABLE = {};

    /**
     * 数组表，大小可以改变，且大小必须为2的幂
     */
    transient Entry<K,V>[] table = (Entry<K,V>[]) EMPTY_TABLE;

    /**
     * 当前Map中key-value映射的个数
     */
    transient int size;

    /**
     * 下次扩容阈值，当size > capacity * load factor时，开始扩容
     */
    int threshold;

    /**
     * 负载因子
     */
    final float loadFactor;

    /**
     * Hash表结构性修改次数，用于实现迭代器快速失败行为
     */
    transient int modCount;

    /**
     * 容量阈值，默认大小为Integer.MAX_VALUE
     */
    static final int ALTERNATIVE_HASHING_THRESHOLD_DEFAULT = Integer.MAX_VALUE;

    /**
     * 静态内部类Holder，存放一些只能在虚拟机启动后才能初始化的值
     */
    private static class Holder {

        /**
         * 容量阈值，初始化hashSeed的时候会用到该值
         */
        static final int ALTERNATIVE_HASHING_THRESHOLD;

        static {
            //获取系统变量jdk.map.althashing.threshold
            String altThreshold = java.security.AccessController.doPrivileged(
                new sun.security.action.GetPropertyAction(
                    "jdk.map.althashing.threshold"));

            int threshold;
            try {
                threshold = (null != altThreshold)
                        ? Integer.parseInt(altThreshold)
                        : ALTERNATIVE_HASHING_THRESHOLD_DEFAULT;

                // jdk.map.althashing.threshold系统变量默认为-1，如果为-1，则将阈值设为Integer.MAX_VALUE
                if (threshold == -1) {
                    threshold = Integer.MAX_VALUE;
                }
                //阈值需要为正数
                if (threshold < 0) {
                    throw new IllegalArgumentException("value must be positive integer.");
                }
            } catch(IllegalArgumentException failed) {
                throw new Error("Illegal value for 'jdk.map.althashing.threshold'", failed);
            }

            ALTERNATIVE_HASHING_THRESHOLD = threshold;
        }
    }

    /**
     * 计算hash值的时候需要用到
     */
    transient int hashSeed = 0;

    /**
     * 生成一个空的HashMap,并指定其容量大小和负载因子
     *
     */
    public HashMap(int initialCapacity, float loadFactor) {
        //保证初始容量大于等于0
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        //保证初始容量不大于最大容量MAXIMUM_CAPACITY
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        
        //loadFactor小于0或为无效数字
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        //负载因子
        this.loadFactor = loadFactor;
        //下次扩容大小
        threshold = initialCapacity;
        init();
    }

    /**
     * 生成一个空的HashMap,并指定其容量大小，负载因子使用默认的0.75
     *
     */
    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }

    /**
     * 生成一个空的HashMap,容量大小使用默认值16，负载因子使用默认值0.75
     */
    public HashMap() {
        this(DEFAULT_INITIAL_CAPACITY, DEFAULT_LOAD_FACTOR);
    }

    /**
     * 根据指定的map生成一个新的HashMap,负载因子使用默认值，初始容量大小为Math.max((int) (m.size() / DEFAULT_LOAD_FACTOR) + 1,DEFAULT_INITIAL_CAPACITY)
     */
    public HashMap(Map<? extends K, ? extends V> m) {
        this(Math.max((int) (m.size() / DEFAULT_LOAD_FACTOR) + 1,
                      DEFAULT_INITIAL_CAPACITY), DEFAULT_LOAD_FACTOR);
        inflateTable(threshold);

        putAllForCreate(m);
    }

    //返回>=number的最小2的n次方值，如number=5，则返回8
    private static int roundUpToPowerOf2(int number) {
        // assert number >= 0 : "number must be non-negative";
        return number >= MAXIMUM_CAPACITY
                ? MAXIMUM_CAPACITY
                : (number > 1) ? Integer.highestOneBit((number - 1) << 1) : 1;
    }

    /**
     * 对table扩容
     */
    private void inflateTable(int toSize) {
        // Find a power of 2 >= toSize
        //找一个值（2的n次方，且>=toSize）
        int capacity = roundUpToPowerOf2(toSize);

        //下次扩容阈值
        threshold = (int) Math.min(capacity * loadFactor, MAXIMUM_CAPACITY + 1);
        
        table = new Entry[capacity];
        initHashSeedAsNeeded(capacity);
    }

    // internal utilities

    void init() {
    }

    /**
     * 初始化hashSeed
     */
    final boolean initHashSeedAsNeeded(int capacity) {
        boolean currentAltHashing = hashSeed != 0;
        boolean useAltHashing = sun.misc.VM.isBooted() &&
                (capacity >= Holder.ALTERNATIVE_HASHING_THRESHOLD);
        boolean switching = currentAltHashing ^ useAltHashing;
        if (switching) {
            hashSeed = useAltHashing
                ? sun.misc.Hashing.randomHashSeed(this)
                : 0;
        }
        return switching;
    }

    /**
     * 生成hash值
     */
    final int hash(Object k) {
        int h = hashSeed;
        
        //如果key是字符串，调用un.misc.Hashing.stringHash32生成hash值
        //Oracle表示能生成更好的hash分布，不过这在jdk8中已删除
        if (0 != h && k instanceof String) {
            return sun.misc.Hashing.stringHash32((String) k);
        }
        //一次散列，调用k的hashCode方法，与hashSeed做异或操作
        h ^= k.hashCode();

        // This function ensures that hashCodes that differ only by
        // constant multiples at each bit position have a bounded
        // number of collisions (approximately 8 at default load factor).
        //二次散列，
        h ^= (h >>> 20) ^ (h >>> 12);
        return h ^ (h >>> 7) ^ (h >>> 4);
    }

    /**
     * 返回hash值的索引，采用除模取余法，h & (length-1)操作 等价于 hash % length操作， 但&操作性能更优
     */
    static int indexFor(int h, int length) {
        // assert Integer.bitCount(length) == 1 : "length must be a non-zero power of 2";
        return h & (length-1);
    }

    /**
     * 返回key-value映射个数
     */
    public int size() {
        return size;
    }

    /**
     * 判断map是否为空
     */
    public boolean isEmpty() {
        return size == 0;
    }

    /**
     * 返回指定key对应的value
     */
    public V get(Object key) {
        //key为null情况
        if (key == null)
            return getForNullKey();
        
        //根据key查找节点
        Entry<K,V> entry = getEntry(key);

        //返回key对应的值
        return null == entry ? null : entry.getValue();
    }

    /**
     * 查找key为null的value，注意如果key为null，则其hash值为0，默认是放在table[0]里的
     */
    private V getForNullKey() {
        if (size == 0) {
            return null;
        }
        //在table[0]的链表上查找key为null的键值对，因为null默认是存在table[0]的桶里
        for (Entry<K,V> e = table[0]; e != null; e = e.next) {
            if (e.key == null)
                return e.value;
        }
        return null;
    }

    /**
     *判断是否包含指定的key
     */
    public boolean containsKey(Object key) {
        return getEntry(key) != null;
    }

    /**
     * 根据key查找键值对，找不到返回null
     */
    final Entry<K,V> getEntry(Object key) {
        if (size == 0) {
            return null;
        }
        //如果key为null，hash值为0，否则调用hash方法，对key生成hash值
        int hash = (key == null) ? 0 : hash(key);
        
        //调用indexFor方法生成hash值的索引，遍历该索引下的链表，查找key“相等”的键值对
        for (Entry<K,V> e = table[indexFor(hash, table.length)];
             e != null;
             e = e.next) {
            Object k;
            if (e.hash == hash &&
                ((k = e.key) == key || (key != null && key.equals(k))))
                return e;
        }
        return null;
    }

    /**
     * 向map存入一个键值对，如果key已存在，则覆盖
     */
    public V put(K key, V value) {
        //数组为空，对数组扩容
        if (table == EMPTY_TABLE) {
            inflateTable(threshold);
        }
        
        //对key为null的键值对调用putForNullKey处理
        if (key == null)
            return putForNullKey(value);
        
        //生成hash值
        int hash = hash(key);
        
        //生成hash值索引
        int i = indexFor(hash, table.length);
        
        //查找是否有key“相等”的键值对，有的话覆盖
        for (Entry<K,V> e = table[i]; e != null; e = e.next) {
            Object k;
            if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this);
                return oldValue;
            }
        }

        //操作次数加一，用于迭代器快速失败行为
        modCount++;
        
        //在指定hash值索引处的链表上增加该键值对
        addEntry(hash, key, value, i);
        return null;
    }

    /**
     * 存放key为null的键值对，存放在索引为0的链表上，已存在的话，替换
     */
    private V putForNullKey(V value) {
        for (Entry<K,V> e = table[0]; e != null; e = e.next) {
            //已存在key为null，则替换
            if (e.key == null) {
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this);
                return oldValue;
            }
        }
        //操作次数加一，用于迭代器快速失败行为
        modCount++;
        //在指定hash值索引处的链表上增加该键值对
        addEntry(0, null, value, 0);
        return null;
    }

    /**
     * 添加键值对
     */
    private void putForCreate(K key, V value) {
        //生成hash值
        int hash = null == key ? 0 : hash(key);
        
        //生成hash值索引，
        int i = indexFor(hash, table.length);

        /**
         * key“相等”，则替换
         */
        for (Entry<K,V> e = table[i]; e != null; e = e.next) {
            Object k;
            if (e.hash == hash &&
                ((k = e.key) == key || (key != null && key.equals(k)))) {
                e.value = value;
                return;
            }
        }
        //在指定索引处的链表上创建该键值对
        createEntry(hash, key, value, i);
    }
    
    //将制定map的键值对添加到map中
    private void putAllForCreate(Map<? extends K, ? extends V> m) {
        for (Map.Entry<? extends K, ? extends V> e : m.entrySet())
            putForCreate(e.getKey(), e.getValue());
    }

    /**
     * 对数组扩容
     */
    void resize(int newCapacity) {
        Entry[] oldTable = table;
        int oldCapacity = oldTable.length;
        
        if (oldCapacity == MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return;
        }
        
        //创建一个指定大小的数组
        Entry[] newTable = new Entry[newCapacity];
        
        transfer(newTable, initHashSeedAsNeeded(newCapacity));
        
        //table索引替换成新数组
        table = newTable;
        
        //重新计算阈值
        threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
    }

    /**
     * 拷贝旧的键值对到新的哈希表中
     */
    void transfer(Entry[] newTable, boolean rehash) {
        int newCapacity = newTable.length;
        //遍历旧的数组
        for (Entry<K,V> e : table) {
            while(null != e) {
                Entry<K,V> next = e.next;
                if (rehash) {
                    e.hash = null == e.key ? 0 : hash(e.key);
                }
                //根据新的数组长度，重新计算索引，
                int i = indexFor(e.hash, newCapacity);
                
                //插入到链表表头
                e.next = newTable[i];
                
                //将e放到索引为i处
                newTable[i] = e;
                
                //将e设置成下个节点
                e = next;
            }
        }
    }

    /**
     * 将制定map的键值对put到本map，key“相等”的直接覆盖
     */
    public void putAll(Map<? extends K, ? extends V> m) {
        int numKeysToBeAdded = m.size();
        if (numKeysToBeAdded == 0)
            return;

        //空map，扩容
        if (table == EMPTY_TABLE) {
            inflateTable((int) Math.max(numKeysToBeAdded * loadFactor, threshold));
        }

        /*
         * 判断是否需要扩容
         */
        if (numKeysToBeAdded > threshold) {
            int targetCapacity = (int)(numKeysToBeAdded / loadFactor + 1);
            if (targetCapacity > MAXIMUM_CAPACITY)
                targetCapacity = MAXIMUM_CAPACITY;
            int newCapacity = table.length;
            while (newCapacity < targetCapacity)
                newCapacity <<= 1;
            if (newCapacity > table.length)
                resize(newCapacity);
        }

        //依次遍历键值对，并put
        for (Map.Entry<? extends K, ? extends V> e : m.entrySet())
            put(e.getKey(), e.getValue());
    }

    /**
     * 移除指定key的键值对
     */
    public V remove(Object key) {
        Entry<K,V> e = removeEntryForKey(key);
        return (e == null ? null : e.value);
    }

    /**
     * 移除指定key的键值对
     */
    final Entry<K,V> removeEntryForKey(Object key) {
        if (size == 0) {
            return null;
        }
        //计算hash值及索引
        int hash = (key == null) ? 0 : hash(key);
        int i = indexFor(hash, table.length);
        
        Entry<K,V> prev = table[i];
        Entry<K,V> e = prev;

        //头节点为table[i]的单链表上执行删除节点操作
        while (e != null) {
            Entry<K,V> next = e.next;
            Object k;
            //找到要删除的节点
            if (e.hash == hash &&
                ((k = e.key) == key || (key != null && key.equals(k)))) {
                modCount++;
                size--;
                if (prev == e)
                    table[i] = next;
                else
                    prev.next = next;
                e.recordRemoval(this);
                return e;
            }
            prev = e;
            e = next;
        }

        return e;
    }

    /**
     * 删除指定键值对对象(Entry对象)
     */
    final Entry<K,V> removeMapping(Object o) {
        if (size == 0 || !(o instanceof Map.Entry))
            return null;

        Map.Entry<K,V> entry = (Map.Entry<K,V>) o;
        Object key = entry.getKey();
        int hash = (key == null) ? 0 : hash(key);
        //得到数组索引
        int i = indexFor(hash, table.length);
        Entry<K,V> prev = table[i];
        Entry<K,V> e = prev;
        //开始遍历该单链表
        while (e != null) {
            Entry<K,V> next = e.next;
            //找到节点
            if (e.hash == hash && e.equals(entry)) {
                modCount++;
                size--;
                if (prev == e)
                    table[i] = next;
                else
                    prev.next = next;
                e.recordRemoval(this);
                return e;
            }
            prev = e;
            e = next;
        }

        return e;
    }

    /**
     * 清空map，将table数组所有元素设为null
     */
    public void clear() {
        modCount++;
        Arrays.fill(table, null);
        size = 0;
    }

    /**
     * 判断是否含有指定value的键值对
     */
    public boolean containsValue(Object value) {
        if (value == null)
            return containsNullValue();

        Entry[] tab = table;
        //遍历table数组
        for (int i = 0; i < tab.length ; i++)
            //遍历每条单链表
            for (Entry e = tab[i] ; e != null ; e = e.next)
                if (value.equals(e.value))
                    return true;
        return false;
    }

    /**
     * 判断是否含有value为null的键值对
     */
    private boolean containsNullValue() {
        Entry[] tab = table;
        for (int i = 0; i < tab.length ; i++)
            for (Entry e = tab[i] ; e != null ; e = e.next)
                if (e.value == null)
                    return true;
        return false;
    }

    /**
     * 浅拷贝，键值对不复制
     */
    public Object clone() {
        HashMap<K,V> result = null;
        try {
            result = (HashMap<K,V>)super.clone();
        } catch (CloneNotSupportedException e) {
            // assert false;
        }
        if (result.table != EMPTY_TABLE) {
            result.inflateTable(Math.min(
                (int) Math.min(
                    size * Math.min(1 / loadFactor, 4.0f),
                    // we have limits...
                    HashMap.MAXIMUM_CAPACITY),
               table.length));
        }
        result.entrySet = null;
        result.modCount = 0;
        result.size = 0;
        result.init();
        result.putAllForCreate(this);

        return result;
    }

    //内部类，节点对象，每个节点包含下个节点的引用
    static class Entry<K,V> implements Map.Entry<K,V> {
        final K key;
        V value;
        Entry<K,V> next;
        int hash;

        /**
         * 创建节点
         */
        Entry(int h, K k, V v, Entry<K,V> n) {
            value = v;
            next = n;
            key = k;
            hash = h;
        }
        //获取节点的key
        public final K getKey() {
            return key;
        }
        //获取节点的value
        public final V getValue() {
            return value;
        }
        
        //设置新value，并返回旧的value
        public final V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }

        //判断key和value是否相同,两个都“相等”，返回true
        public final boolean equals(Object o) {
            if (!(o instanceof Map.Entry))
                return false;
            Map.Entry e = (Map.Entry)o;
            Object k1 = getKey();
            Object k2 = e.getKey();
            if (k1 == k2 || (k1 != null && k1.equals(k2))) {
                Object v1 = getValue();
                Object v2 = e.getValue();
                if (v1 == v2 || (v1 != null && v1.equals(v2)))
                    return true;
            }
            return false;
        }

        public final int hashCode() {
            return Objects.hashCode(getKey()) ^ Objects.hashCode(getValue());
        }

        public final String toString() {
            return getKey() + "=" + getValue();
        }

        /**
         * This method is invoked whenever the value in an entry is
         * overwritten by an invocation of put(k,v) for a key k that's already
         * in the HashMap.
         */
        void recordAccess(HashMap<K,V> m) {
        }

        /**
         * This method is invoked whenever the entry is
         * removed from the table.
         */
        void recordRemoval(HashMap<K,V> m) {
        }
    }

    /**
     * 添加新节点，如有必要，执行扩容操作
     */
    void addEntry(int hash, K key, V value, int bucketIndex) {
        if ((size >= threshold) && (null != table[bucketIndex])) {
            resize(2 * table.length);
            hash = (null != key) ? hash(key) : 0;
            bucketIndex = indexFor(hash, table.length);
        }

        createEntry(hash, key, value, bucketIndex);
    }

    /**
     * 插入单链表表头
     */
    void createEntry(int hash, K key, V value, int bucketIndex) {
        Entry<K,V> e = table[bucketIndex];
        table[bucketIndex] = new Entry<>(hash, key, value, e);
        size++;
    }

    //hashmap迭代器
    private abstract class HashIterator<E> implements Iterator<E> {
        Entry<K,V> next;        // 下个键值对索引
        int expectedModCount;   // 用于判断快速失败行为
        int index;              // current slot
        Entry<K,V> current;     // current entry

        HashIterator() {
            expectedModCount = modCount;
            if (size > 0) { // advance to first entry
                Entry[] t = table;
                while (index < t.length && (next = t[index++]) == null)
                    ;
            }
        }

        public final boolean hasNext() {
            return next != null;
        }

        final Entry<K,V> nextEntry() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            Entry<K,V> e = next;
            if (e == null)
                throw new NoSuchElementException();

            if ((next = e.next) == null) {
                Entry[] t = table;
                while (index < t.length && (next = t[index++]) == null)
                    ;
            }
            current = e;
            return e;
        }

        public void remove() {
            if (current == null)
                throw new IllegalStateException();
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            Object k = current.key;
            current = null;
            HashMap.this.removeEntryForKey(k);
            expectedModCount = modCount;
        }
    }

    //ValueIterator迭代器
    private final class ValueIterator extends HashIterator<V> {
        public V next() {
            return nextEntry().value;
        }
    }
    //KeyIterator迭代器
    private final class KeyIterator extends HashIterator<K> {
        public K next() {
            return nextEntry().getKey();
        }
    }
    ////KeyIterator迭代器
    private final class EntryIterator extends HashIterator<Map.Entry<K,V>> {
        public Map.Entry<K,V> next() {
            return nextEntry();
        }
    }

    // 返回迭代器方法
    Iterator<K> newKeyIterator()   {
        return new KeyIterator();
    }
    Iterator<V> newValueIterator()   {
        return new ValueIterator();
    }
    Iterator<Map.Entry<K,V>> newEntryIterator()   {
        return new EntryIterator();
    }


    // Views

    private transient Set<Map.Entry<K,V>> entrySet = null;

    /**
     * 返回一个set集合，包含key
     */
    public Set<K> keySet() {
        Set<K> ks = keySet;
        return (ks != null ? ks : (keySet = new KeySet()));
    }

    private final class KeySet extends AbstractSet<K> {
        public Iterator<K> iterator() {
            return newKeyIterator();
        }
        public int size() {
            return size;
        }
        public boolean contains(Object o) {
            return containsKey(o);
        }
        public boolean remove(Object o) {
            return HashMap.this.removeEntryForKey(o) != null;
        }
        public void clear() {
            HashMap.this.clear();
        }
    }

    /**
     * 返回一个value集合，包含value
     */
    public Collection<V> values() {
        Collection<V> vs = values;
        return (vs != null ? vs : (values = new Values()));
    }

    private final class Values extends AbstractCollection<V> {
        public Iterator<V> iterator() {
            return newValueIterator();
        }
        public int size() {
            return size;
        }
        public boolean contains(Object o) {
            return containsValue(o);
        }
        public void clear() {
            HashMap.this.clear();
        }
    }

    /**
     * 返回一个键值对集合
     */
    public Set<Map.Entry<K,V>> entrySet() {
        return entrySet0();
    }

    private Set<Map.Entry<K,V>> entrySet0() {
        Set<Map.Entry<K,V>> es = entrySet;
        return es != null ? es : (entrySet = new EntrySet());
    }

    private final class EntrySet extends AbstractSet<Map.Entry<K,V>> {
        public Iterator<Map.Entry<K,V>> iterator() {
            return newEntryIterator();
        }
        public boolean contains(Object o) {
            if (!(o instanceof Map.Entry))
                return false;
            Map.Entry<K,V> e = (Map.Entry<K,V>) o;
            Entry<K,V> candidate = getEntry(e.getKey());
            return candidate != null && candidate.equals(e);
        }
        public boolean remove(Object o) {
            return removeMapping(o) != null;
        }
        public int size() {
            return size;
        }
        public void clear() {
            HashMap.this.clear();
        }
    }

    /**
     * map序列化,可实现深拷贝
     */
    private void writeObject(java.io.ObjectOutputStream s)
        throws IOException
    {
        // Write out the threshold, loadfactor, and any hidden stuff
        s.defaultWriteObject();

        // Write out number of buckets
        if (table==EMPTY_TABLE) {
            s.writeInt(roundUpToPowerOf2(threshold));
        } else {
           s.writeInt(table.length);
        }

        // Write out size (number of Mappings)
        s.writeInt(size);

        // Write out keys and values (alternating)
        if (size > 0) {
            for(Map.Entry<K,V> e : entrySet0()) {
                s.writeObject(e.getKey());
                s.writeObject(e.getValue());
            }
        }
    }

    private static final long serialVersionUID = 362498820763181265L;

    /**
     * 反序列化，读取字节码转为对象
     */
    private void readObject(java.io.ObjectInputStream s)
         throws IOException, ClassNotFoundException
    {
        // Read in the threshold (ignored), loadfactor, and any hidden stuff
        s.defaultReadObject();
        if (loadFactor <= 0 || Float.isNaN(loadFactor)) {
            throw new InvalidObjectException("Illegal load factor: " +
                                               loadFactor);
        }

        // set other fields that need values
        table = (Entry<K,V>[]) EMPTY_TABLE;

        // Read in number of buckets
        s.readInt(); // ignored.

        // Read number of mappings
        int mappings = s.readInt();
        if (mappings < 0)
            throw new InvalidObjectException("Illegal mappings count: " +
                                               mappings);

        // capacity chosen by number of mappings and desired load (if >= 0.25)
        int capacity = (int) Math.min(
                    mappings * Math.min(1 / loadFactor, 4.0f),
                    // we have limits...
                    HashMap.MAXIMUM_CAPACITY);

        // allocate the bucket array;
        if (mappings > 0) {
            inflateTable(capacity);
        } else {
            threshold = capacity;
        }

        init();  // Give subclass a chance to do its thing.

        // Read the keys and values, and put the mappings in the HashMap
        for (int i = 0; i < mappings; i++) {
            K key = (K) s.readObject();
            V value = (V) s.readObject();
            putForCreate(key, value);
        }
    }

    // These methods are used when serializing HashSets
    int   capacity()     { return table.length; }
    float loadFactor()   { return loadFactor;   }
}
}
```



### 单独分析

```java
  	//put方法
  	public V put(K key, V value) {
        //数组为空，对数组扩容
        if (table == EMPTY_TABLE) {
            inflateTable(threshold);//threshold是下次扩容阈值
        }
        
        //对key为null的键值对调用putForNullKey处理
        if (key == null)
            return putForNullKey(value);
        
        //生成hash值
        int hash = hash(key);
        
        //生成hash值索引
        int i = indexFor(hash, table.length);
        
        //查找是否有key“相等”的键值对，有的话覆盖
        for (Entry<K,V> e = table[i]; e != null; e = e.next) {
            Object k;
            if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this);
                return oldValue;
            }
        }

        //操作次数加一，用于迭代器快速失败行为
        modCount++;
        
        //在指定hash值索引处的链表上增加该键值对
        addEntry(hash, key, value, i);//i是hash值索引
        return null;
    }
```

```java
    /**
     * 对table扩容
     */
    private void inflateTable(int toSize) {
        // Find a power of 2 >= toSize
        //找一个值（2的n次方，且>=toSize）
        int capacity = roundUpToPowerOf2(toSize);

        //下次扩容阈值
        threshold = (int) Math.min(capacity * loadFactor, MAXIMUM_CAPACITY + 1);
        
        table = new Entry[capacity];
        initHashSeedAsNeeded(capacity);
    }
```

```java
	//返回>=number的最小2的n次方值，如number=5，则返回8
    private static int roundUpToPowerOf2(int number) {
        // assert number >= 0 : "number must be non-negative";
        return number >= MAXIMUM_CAPACITY
                ? MAXIMUM_CAPACITY
                : (number > 1) ? Integer.highestOneBit((number - 1) << 1) : 1;
    }
```

```java
	/**
     * 初始化hashSeed
     */
    final boolean initHashSeedAsNeeded(int capacity) {
        boolean currentAltHashing = hashSeed != 0;
        boolean useAltHashing = sun.misc.VM.isBooted() &&
                (capacity >= Holder.ALTERNATIVE_HASHING_THRESHOLD);
        boolean switching = currentAltHashing ^ useAltHashing;
        if (switching) {
            hashSeed = useAltHashing
                ? sun.misc.Hashing.randomHashSeed(this)
                : 0;
        }
        return switching;
    }
```

```java
  	/**	
     * 生成hash值
     */
    final int hash(Object k) {
        int h = hashSeed;
        
        //如果key是字符串，调用un.misc.Hashing.stringHash32生成hash值
        //Oracle表示能生成更好的hash分布，不过这在jdk8中已删除
        if (0 != h && k instanceof String) {
            return sun.misc.Hashing.stringHash32((String) k);
        }
        //一次散列，调用k的hashCode方法，与hashSeed做异或操作
        h ^= k.hashCode();

        // This function ensures that hashCodes that differ only by
        // constant multiples at each bit position have a bounded
        // number of collisions (approximately 8 at default load factor).
        //二次散列，
        h ^= (h >>> 20) ^ (h >>> 12);
        return h ^ (h >>> 7) ^ (h >>> 4);
    }
```

```java
	/**
     * 返回hash值的索引，采用除模取余法，h & (length-1)操作 等价于 hash % length操作， 但&操作性能更			优
     */
    static int indexFor(int h, int length) {
        // assert Integer.bitCount(length) == 1 : "length must be a non-zero power of 2";
        return h & (length-1);
    }
```

```java
	/**
     * 添加新节点，如有必要，执行扩容操作
     */
    void addEntry(int hash, K key, V value, int bucketIndex) {
        if ((size >= threshold) && (null != table[bucketIndex])) {
            resize(2 * table.length);//扩容成两倍
            hash = (null != key) ? hash(key) : 0;
            bucketIndex = indexFor(hash, table.length);
        }

        createEntry(hash, key, value, bucketIndex);
    }
```

#### 核心方法：扩容

```java
	/**
     * 对数组扩容
     */
    void resize(int newCapacity) {
        Entry[] oldTable = table;
        int oldCapacity = oldTable.length;
        
        if (oldCapacity == MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return;
        }
        
        //创建一个指定大小的数组
        Entry[] newTable = new Entry[newCapacity];
        
        transfer(newTable, initHashSeedAsNeeded(newCapacity));
        
        //table索引替换成新数组
        table = newTable;
        
        //重新计算阈值
        threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
    }	
```

```java
	/**
     * 初始化hashSeed
     */
    final boolean initHashSeedAsNeeded(int capacity) {
        boolean currentAltHashing = hashSeed != 0;
        boolean useAltHashing = sun.misc.VM.isBooted() &&
                (capacity >= Holder.ALTERNATIVE_HASHING_THRESHOLD);
        boolean switching = currentAltHashing ^ useAltHashing;
        if (switching) {
            hashSeed = useAltHashing
                ? sun.misc.Hashing.randomHashSeed(this)
                : 0;
        }
        return switching;
    }
```

```java
	/**
     * 拷贝旧的键值对到新的哈希表中
     */
    void transfer(Entry[] newTable, boolean rehash) {
        int newCapacity = newTable.length;
        //遍历旧的数组
        for (Entry<K,V> e : table) {
            while(null != e) {
                Entry<K,V> next = e.next;
                if (rehash) {
                    e.hash = null == e.key ? 0 : hash(e.key);
                }
                //根据新的数组长度，重新计算索引，
                int i = indexFor(e.hash, newCapacity);
                
                //插入到链表表头，这也写的太难理解了，第一次执行的时候没用，
                //意思就是从前边插入。
                e.next = newTable[i];
                
                //将e放到索引为i处
                newTable[i] = e;
                
                //将e设置成下个节点，这样就相当于在一直循环遍历了
                e = next;
            }
        }
    }
```

```java
	/**
     * 插入单链表表头
     */
    void createEntry(int hash, K key, V value, int bucketIndex) {
        Entry<K,V> e = table[bucketIndex];
	    //意思就是直接让要插入的元素插在表头，插入的时候都是往表头插。	
        table[bucketIndex] = new Entry<>(hash, key, value, e);
        size++;
    }
```

- 该方法说明，每次put键值对的时候，总是将新的该键值对放在table[bucketIndex]处（即头结点处）。

### get方法

```java
	/**
     * 返回指定key对应的value
     */
    public V get(Object key) {
        //key为null情况
        if (key == null)
            return getForNullKey();
        
        //根据key查找节点
        Entry<K,V> entry = getEntry(key);

        //返回key对应的值
        return null == entry ? null : entry.getValue();
    }
```

```java
	/**
     * 查找key为null的value，注意如果key为null，则其hash值为0，默认是放在以table[0]为头结点的链表中
     * 不一定是存放在头结点table[0]中
     */
    private V getForNullKey() {
        if (size == 0) {
            return null;
        }
        //在table[0]的链表上查找key为null的键值对，因为null默认是存在table[0]的桶里
        for (Entry<K,V> e = table[0]; e != null; e = e.next) {
            if (e.key == null)
                return e.value;
        }
        return null;
    }
```

```java
	 /**
     * 根据key查找键值对，找不到返回null
     */
    final Entry<K,V> getEntry(Object key) {
        if (size == 0) {
            return null;
        }
        //如果key为null，hash值为0，否则调用hash方法，对key生成hash值
        int hash = (key == null) ? 0 : hash(key);
        
        //调用indexFor方法生成hash值的索引，遍历该索引下的链表，查找key“相等”的键值对
        for (Entry<K,V> e = table[indexFor(hash, table.length)];
             e != null;
             e = e.next) {
            Object k;
            if (e.hash == hash &&
                ((k = e.key) == key || (key != null && key.equals(k))))
                return e;
        }
        return null;
    }
```

### 数据结构

```java
// Entry是单向链表。    
// 它是 “HashMap链式存储法”对应的链表。    
// 它实现了Map.Entry 接口，即实现getKey(), getValue(), setValue(V value), equals(Object o), hashCode()这些函数    
static class Entry<K,V> implements Map.Entry<K,V> {    
    final K key;    
    V value;    
    // 指向下一个节点    
    Entry<K,V> next;    
    final int hash;    
  
    // 构造函数。    
    // 输入参数包括"哈希值(h)", "键(k)", "值(v)", "下一节点(n)"    
    Entry(int h, K k, V v, Entry<K,V> n) {    
        value = v;    
        next = n;    
        key = k;    
        hash = h;    
    }    
  
    public final K getKey() {    
        return key;    
    }    
  
    public final V getValue() {    
        return value;    
    }    
  
    public final V setValue(V newValue) {    
        V oldValue = value;    
        value = newValue;    
        return oldValue;    
    }    
  
    // 判断两个Entry是否相等    
    // 若两个Entry的“key”和“value”都相等，则返回true。    
    // 否则，返回false    
    public final boolean equals(Object o) {    
        if (!(o instanceof Map.Entry))    
            return false;    
        Map.Entry e = (Map.Entry)o;    
        Object k1 = getKey();    
        Object k2 = e.getKey();    
        if (k1 == k2 || (k1 != null && k1.equals(k2))) {    
            Object v1 = getValue();    
            Object v2 = e.getValue();    
            if (v1 == v2 || (v1 != null && v1.equals(v2)))    
                return true;    
        }    
        return false;    
    }    
  
    // 实现hashCode()    
    public final int hashCode() {    
        return (key==null   ? 0 : key.hashCode()) ^    
               (value==null ? 0 : value.hashCode());    
    }    
  
    public final String toString() {    
        return getKey() + "=" + getValue();    
    }    
  
    // 当向HashMap中添加元素时，绘调用recordAccess()。    
    // 这里不做任何处理    
    void recordAccess(HashMap<K,V> m) {    
    }    
  
    // 当从HashMap中删除元素时，绘调用recordRemoval()。    
    // 这里不做任何处理    
    void recordRemoval(HashMap<K,V> m) {    
    }    
}    
```

- 它的结构元素除了key、value、hash外，还有next，next指向下一个节点。另外，这里覆写了equals和hashCode方法来保证键值对的独一无二。
- HashMap共有四个构造方法。构造方法中提到了两个很重要的参数：初始容量和加载因子。这两个参数是影响HashMap性能的重要参数，其中容量表示哈希表中槽的数量（即哈希数组的长度），初始容量是创建哈希表时的容量（从构造函数中可以看出，如果不指明，则默认为16），加载因子是哈希表在其容量自动增加之前可以达到多满的一种尺度，当哈希表中的条目数超出了加载因子与当前容量的乘积时，则要对该哈希表进行 resize 操作（即扩容）。
- 下面说下加载因子，如果加载因子越大，对空间的利用更充分，但是查找效率会降低（链表长度会越来越长）；如果加载因子太小，那么表中的数据将过于稀疏（很多空间还没用，就开始扩容了），对空间造成严重浪费。如果我们在构造方法中不指定，则系统默认加载因子为0.75，这是一个比较理想的值，一般情况下我们是无需修改的。
- 另外，无论我们指定的容量为多少，构造方法都会将实际容量设为不小于指定容量的2的次方的一个数，且最大值不能超过2的30次方
- HashMap中key和value都允许为null。

## 为什么哈希表的容量一定要是2的整数次幂？

首先，length为2的整数次幂的话，h&(length-1)就相当于对length取模，这样便保证了散列的均匀，同时也提升了效率；其次，length为2的整数次幂的话，为偶数，这样length-1为奇数，奇数的最后一位是1，这样便保证了h&(length-1)的最后一位可能为0，也可能为1（这取决于h的值），即与后的结果可能为偶数，也可能为奇数，这样便可以保证散列的均匀性，而如果length为奇数的话，很明显length-1为偶数，它的最后一位是0，这样h&(length-1)的最后一位肯定为0，即只能为偶数，这样任何hash值都只会被散列到数组的偶数下标位置上，这便浪费了近一半的空间，因此，length取2的整数次幂，是为了使不同hash值发生碰撞的概率较小，这样就能使元素在哈希表中均匀地散列。