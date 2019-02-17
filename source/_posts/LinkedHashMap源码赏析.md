---
title: LinkedHashMap源码赏析
abbrlink: 21482a37
date: 2018-01-16 14:13:47
tags:
  - 源码解析系列
---

### 两种排序方式

1. 根据时间(存储)排序，即先插入的是最老的，要被删除的，
2. 根据访问时间排序，每访问一次，就放到队列尾部，将来移除的时候，都是移除队列头部的元素。

### 基础介绍

 LinkedHashMap类似于HashMap，但是迭代遍历它时，取得“键值对”的顺序是插入次序，或者是最近最少使用（LRU）的次序。只比HashMap慢一点；而在迭代访问时反而更快，因为它使用链表维护内部次序

```java
public class LinkedHashMap<K,V> extends HashMap<K,V> implements Map<K,V>
```

### 两个重要属性

```java
	/**
     * The head of the doubly linked list.
     * 双向链表的头节点
     */
    private transient Entry<K,V> header;
    /**
     * The iteration ordering method for this linked hash map: true
     * for access-order, false for insertion-order.
     * true表示最近最少使用次序，false表示插入顺序
     */
    private final boolean accessOrder;
```

### 构造方法

```java
// 构造方法1，构造一个指定初始容量和负载因子的、按照插入顺序的LinkedList
public LinkedHashMap(int initialCapacity, float loadFactor) {
    super(initialCapacity, loadFactor);
    accessOrder = false;
}
// 构造方法2，构造一个指定初始容量的LinkedHashMap，取得键值对的顺序是插入顺序
public LinkedHashMap(int initialCapacity) {
    super(initialCapacity);
    accessOrder = false;
}
// 构造方法3，用默认的初始化容量和负载因子创建一个LinkedHashMap，取得键值对的顺序是插入顺序
public LinkedHashMap() {
    super();
    accessOrder = false;
}
// 构造方法4，通过传入的map创建一个LinkedHashMap，容量为默认容量（16）和(map.zise()/DEFAULT_LOAD_FACTORY)+1的较大者，装载因子为默认值
public LinkedHashMap(Map<? extends K, ? extends V> m) {
    super(m);
    accessOrder = false;
}
// 构造方法5，根据指定容量、装载因子和键值对保持顺序创建一个LinkedHashMap
public LinkedHashMap(int initialCapacity,
             float loadFactor,
                         boolean accessOrder) {
    super(initialCapacity, loadFactor);
    this.accessOrder = accessOrder;
}
```

- 从构造方法中可以看出，默认都采用插入顺序来维持取出键值对的次序。所有构造方法都是通过调用父类的构造方法来创建对象的。
- LinkedHashMap是基于双向链表的，而且属性中定了一个header节点，为什么构造方法都没有对其进行初始化呢？
- 注意LinkedHashMap中有一个init()方法， HashMap的构造方法都调用了init()方法，这里LinkedHashMap的构造方法在调用父类构造方法后将从父类构造方法中调用init()方法（这也解释了为什么HashMap中会有一个没有内容的init()方法）。

```java
void init() {
    header = new Entry<K,V>(-1, null, null, null);
   	header.before = header.after = header;//此处说明了linkedHashMap是由双向链表实现的。
 }
```

- 看init()方法，的确是对header进行了初始化，并构造成一个双向循环链表
- 有人会说linkedList也是双向循环链表实现的，1.6之前可能是，但是1.7之后，不再是循环链表，而是采用一个first头指针和last尾指针。
- ​     transfer(HashMap.Entry[] newTable)方法和init()方法一样也在HashTable中被调用。transfer(HashMap.Entry[] newTable)方法在HashMap调用resize(int newCapacity)方法的时候被调用。

```java
void transfer(HashMap.Entry[] newTable) {
    int newCapacity = newTable.length;
    for (Entry<K,V> e = header.after; e != header; e = e.after) {
        int index = indexFor(e.hash, newCapacity);
        e.next = newTable[index];
        newTable[index] = e;
    }
}
```

- 根据链表节点e的哈希值计算e在新容量的table数组中的索引，并将e插入到计算出的索引所引用的链表

  ### containsValue(Object value)

```java
public boolean containsValue(Object value) {
    // Overridden to take advantage of faster iterator
    if (value==null) {
        for (Entry e = header.after; e != header; e = e.after)
            if (e.value==null)
                return true;
    } else {
        for (Entry e = header.after; e != header; e = e.after)
                if (value.equals(e.value))
                    return true;
    }
    return false;
}
```

重写父类的containsValue(Object value)方法，直接通过header遍历链表判断是否有值和value相等，而不用查询table数组。

### **get(Object key)**

```java
public V get(Object key) {
    Entry<K,V> e = (Entry<K,V>)getEntry(key);
    if (e == null)
        return null;
    e.recordAccess(this);
    return e.value;
}
```

​	get(Object key)方法通过HashMap的getEntry(Object key)方法获取节点，并返回该节点的value值，获取节点如果为null则返回null。recordAccess(HashMap<K,V> m)是LinkedHashMap的内部类Entry的一个方法，将在介绍Entry的时候进行详细的介绍。

### **clear()**

```java
1 public void clear() {
2     super.clear();
3     header.before = header.after = header;
4 }
```

​	 clear()方法先调用父类的方法clear()方法，之后将链表的header节点的before和after引用都指向header自身，即header节点就是一个双向循环链表。这样就无法访问到原链表中剩余的其他节点，他们都将被GC回收。

### LinkedHashMap的内部类Entry<K,V>

```java
// 这是一个私有的、静态的内部类，继承自HashMap的Entry。
private static class Entry<K,V> extends HashMap.Entry<K,V> {
        // 对前后节点的引用
        Entry<K,V> before, after;
    // 构造方法直接调用父类的构造方法
    Entry(int hash, K key, V value, HashMap.Entry<K,V> next) {
        super(hash, key, value, next);
    }

    // 移除该节点，只需修改前一节点的after引用和后一节点的before引用
private void remove() {
    // 修改后该节点服务再被访问，会被GC回收
  	//按照HashMap源码解析中的图示理解
        before.after = after;
        after.before = before;
    }

    // 在指定节点之前插入当前节点（双向链表插入节点的过程）
private void addBefore(Entry<K,V> existingEntry) {
    // 将当前节点的after引用指向existingEntry
        after  = existingEntry;
        // 将before的引用指向existingEntry节点的前一节点
before = existingEntry.before;
        // 将原先existingEntry节点的前一节点的after引用指向当前节点
before.after = this; 
        // 将原先existingEntry节点的后一节点的before引用指向当前节点
        after.before = this;
    }

// 当调用此类的get方法或put方法（put方法将调用到父类HashMap.Entry的put
// 方法）都将调用到recordAccess(HashMap<K,V> m)方法
// 如果accessOrder为true，即使用的是最近最少使用的次序，则将当前被修改的
// 节点移动到header节点之前，即链表的尾部。
// 这也是为什么在HashMap.Entry中有一个空的
// recordAccess(HashMap<K,V> m)方法的原因
    void recordAccess(HashMap<K,V> m) {
        LinkedHashMap<K,V> lm = (LinkedHashMap<K,V>)m;
        if (lm.accessOrder) {
            lm.modCount++;
            remove();
            addBefore(lm.header);
        }
    }
    // 和recordAccess(HashMap<K.V> m)方法一样，在HashMap.Entry中同样有一个对应的空方法。当进行删除（remove）操作的时候会被调用
    void recordRemoval(HashMap<K,V> m) {
        remove();
    }
}
```

{% asset_img LinkedHashMap在指定节点之前插入当前节点.png %}

双向链表插入节点的过程图解



### **createEntry(int hash,K key,V value,int bucketIndex)**

```java
void createEntry(int hash, K key, V value, int bucketIndex) {
    HashMap.Entry<K,V> old = table[bucketIndex];
    Entry<K,V> e = new Entry<K,V>(hash, key, value, old);
    table[bucketIndex] = e;//意思就是把e插入到table[bucketIndex]这个链表之前
    e.addBefore(header);
    size++;
}
```

​	createEntry(int hash,K key,V value,int bucketIndex)方法覆盖了父类HashMap中的方法。这个方法不会拓展table数组的大小。该方法首先保留table中bucketIndex处的节点，然后调用Entry的构造方法（将调用到父类HashMap.Entry的构造方法）添加一个节点，即将当前节点的next引用指向table[bucketIndex] 的节点，之后调用的e.addBefore(header)是修改链表，将e节点添加到header节点之前。

​     该方法同时在table[bucketIndex]的链表中添加了节点，也在LinkedHashMap自身的链表中添加了节点。



### **addEntry(int hash, K key, V value, int bucketIndex)**

```java
void addEntry(int hash, K key, V value, int bucketIndex) {
    createEntry(hash, key, value, bucketIndex);
    Entry<K,V> eldest = header.after;
    if (removeEldestEntry(eldest)) {
        removeEntryForKey(eldest.key);
    } else {
        if (size >= threshold)
            resize(2 * table.length);
    }
}
```

 首先调用createEntry(int hash,K key,V value,int bucketIndex)方法，之后获取LinkedHashMap中“最老”（最近最少使用）的节点，接着涉及到了removeEldestEntry(Entry<K,V> eldest)方法，来看一下：

```java
1 protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
2     return false;
3 }
```

**为什么这个方法始终返回false？** 

 结合上面的addEntry(int hash,K key,V value,int bucketIndex)方法，这样设计可以使LinkedHashMap成为一个正常的Map，不会去移除“最老”的节点。 

  **为什么不在代码中直接去除这部分逻辑而是设计成这样呢？** 

这为开发者提供了方便，若希望将Map当做Cache来使用，并且限制大小，只需继承LinkedHashMap并重写removeEldestEntry(Entry<K,V> eldest)方法，像这样： 

```java
1 private static final int MAX_ENTRIES = 100;
2 protected boolean removeEldestEntry(Map.Entry eldest) {
3      return size() > MAX_ENTRIES;
4 }
```



### 迭代器处理

​	LinkedHashMap除了以上内容外还有和迭代相关的三个方法及三个内部类以及一个抽象内部类，分别是：newKeyIterator()、newValueIterator()、newEntryIterator()和KeyIterator类、ValueIterator类、EntryIterator类以及LinkedHashIterator类。

​     三个new方法分别返回对应的三个类的实例。而三个类都继承自抽象类LinkedHashIterator。下面看迭代相关的三个类。

```java
private class KeyIterator extends LinkedHashIterator<K> {
    public K next() { return nextEntry().getKey(); }
}
private class ValueIterator extends LinkedHashIterator<V> {
    public V next() { return nextEntry().value; }
}
private class EntryIterator extends LinkedHashIterator<Map.Entry<K,V>> {
    public Map.Entry<K,V> next() { return nextEntry(); }
}
```

​	从上面可以看出这三个类都很简单，只有一个next()方法，next()方法也只是去调用LinkedHashIterator类中相应的方法。 和KeyIterator类、ValueIterator类、EntryIterator类以及LinkedHashIterator类。

下面是LinkedHashIterator类的内容。

```java
private abstract class LinkedHashIterator<T> implements Iterator<T> {
    Entry<K,V> nextEntry    = header.after;
    Entry<K,V> lastReturned = null;
    // 和LinkedList中ListItr类定义了expectedModCount用途一致
    int expectedModCount = modCount;
    // 下一个节点如果是header节点说明当前节点是链表的最后一个节点，即已经遍历完链表了，没有下一个节点了
    public boolean hasNext() {
        return nextEntry != header;
    }
    //移除上一次被返回的节点lastReturned 
    public void remove() {
        if (lastReturned == null)
            throw new IllegalStateException();
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
        LinkedHashMap.this.remove(lastReturned.key);
        lastReturned = null;
        expectedModCount = modCount;
    }
    // 返回下一个节点
    Entry<K,V> nextEntry() {
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
        if (nextEntry == header)
            throw new NoSuchElementException();
        // 获取并记录返回的节点
        Entry<K,V> e = lastReturned = nextEntry;
        // 保存对下一个节点的引用
        nextEntry = e.after;
        return e;
    }
}
```

LinkedHashMap本应和HashMap及LinkedList一起分析，比较他们的异同。为了弥补，这里简单的总结一些他们之间的异同：

​     HashMap使用哈希表来存储数据，并用拉链法来处理冲突。LinkedHashMap继承自HashMap，同时自身有一个链表，使用链表存储数据，不存在冲突。LinkedList和LinkedHashMap一样使用一个双向循环链表，但存储的是简单的数据，并不是“键值对”。所以HashMap和LinkedHashMap是Map，而LinkedList是一个List，这是他们本质的区别。LinkedList和LinkedHashMap都可以维护内容的顺序，但HashMap不维护顺序。