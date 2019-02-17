---
title: LinkedList源码赏析
abbrlink: df7893e2
date: 2018-01-13 12:12:55
tags:
  - 源码解析系列
---

# LinkedList

## 从功能阅读-简单函数开始-List的Api

一般在使用中最常用的就是，数据的存、取、删、循环遍历，就从这几个点分析源码

#### 1 - boolean add(E e)

将指定元素添加到此列表的结尾。此方法等效于 addLast(E)。

```java
public boolean add(E e) {
  linkLast(e);
  return true;
}
/**
     * Links e as last element.
     */
void linkLast(E e) {
  //先取得此队列的最后一个元素
  final Node<E> l = last; 
  //构造 节点，此节点的前一个节点是：最后一个节点，下一个节点：因为当前是把元素追加到列表末尾，所以下一个节点是空
  final Node<E> newNode = new Node<>(l, e, null);
  last = newNode; //更改全局变量，让最后一个节点等于新的节点
  if (l == null) 
    first = newNode; //最后一个节点为null则第一个节点就是当前节点，假设是首次使用：first和last都为null; 新增加一个节点，last和first就是当前增加的的节点
  else
    l.next = newNode; //如果最后一个节点有值，则 最后一个节点的下一个节点 就是当前节点， 向一个链条一样，把新增的节点链接了起来
  size++;
  modCount++; //又是快速失败 版本号
}
```

#### 2 - Node - 链表实现节点类

定义很简单

```java
private static class Node<E> {
  E item;  //当前元素
  Node<E> next; //下一个元素
  Node<E> prev; //上一个元素
  Node(Node<E> prev, E element, Node<E> next) {
    this.item = element;
    this.next = next;
    this.prev = prev;
  }
}
```

#### 3 - void add(int index, E element)

在此列表中指定的位置插入指定的元素。

**总结**

1. 随机写入，也就是使用下标，都要进行循环链表找到对应的链表节点
2. 找到节点之后，写入很快，（不用向数组那样还需要移动n个元素的下标）

```java
//其实总结一下就三点：1.新建一个节点，前指向被占用位置元素的前一个节点，后指向当前被占用位置的元素
			//2. 把被占用位置的元素的前一个节点置成新节点，
             //3.把前一个节点的指向改成新的节点，
             //只是需要判断一下要插入的节点是否是头结点，如果是头结点。省略第三步，

//在指定位置插入元素，如果该位置已经有了元素，则改变这个元素的前/后索引
    public void add(int index, E element) {
        checkPositionIndex(index); // 同样检查
        if (index == size) //如果下标等于此列表的大小，则相当于往列表末尾添加元素
            linkLast(element);
        else
            linkBefore(element, node(index));
    }
    /**
     * Inserts element e before non-null Node succ.
     */
     // 把占用的节点往后移动
    void linkBefore(E e, Node<E> succ) {
        // assert succ != null;
        // 仔细看看后面的图 核心函数 E unlink(Node<E> x)
        // 改变受影响的 ： prev.next  和 next.prev 还有first（为什么没有last了呢，因为添加到列表末尾走了正常的添加流程）
        final Node<E> pred = succ.prev;				//succ是要被占用的元素，pred是它的上一个元素
        final Node<E> newNode = new Node<>(pred, e, succ);//①只是创建了一个前后都有指针，还有数据的node
        succ.prev = newNode;//②被占用位置的元素的前一个节点变成新节点
    //判断被占用位置的元素的前一个节点是否是空，如果是空，说明要把数据插入到第一个元素的位置。其实此时链表已经设置完成了，只需要把全局变量first再设置一下新节点就行了，如果不是空，就需要把前一个节点指向的下一个节点改成新节点，
        if (pred == null)//说明succ就是第一个元素，要把数据插入到第一个元素的位置。
            first = newNode;
        else
            pred.next = newNode;//③被占用位置的元素的前一个节点的下一个节点以前是被占用位置的元素，现在变成新节点，根据①②，链表就连接完成。
        size++;
        modCount++;
    }
```

#### 4 - boolean remove(Object o)

从此列表中移除首次出现的指定元素（如果存在）。

**总结：**

1. 删除指定元素，也是需要从头遍历匹配的

2. ArrayList和LinkedList的本方法都是通过循环遍历匹配首个出现的值；

   ​

   区别：

   1. ArrayList 删除元素需要删除元素后面所有的元素移动下标
   2. LinkedList直接改变受影响的 前后（首尾）的prev/next指向，就完成了 删除，相对来说，而受影响的元素最多也只有（first/last变量、被删除元素的前/后元素），相对来说比list删除性能高，

```java
public boolean remove(Object o) {
  if (o == null) {
    for (Node<E> x = first; x != null; x = x.next) {
      if (x.item == null) {
        unlink(x);
        return true;
      }
    }
  } else {
    //从头开始遍历链表，直到下一个元素为null则表示链表到末尾了
    for (Node<E> x = first; x != null; x = x.next) {
      //找到与指定元素匹配的链表节点
      if (o.equals(x.item)) {
        unlink(x); //处理链表的链接：因为删除了一个节点，这个节点的前后节点需要链接起来
        return true;
      }
    }
  }
  return false;
}
```

#### 5- 核心函数 - E unlink(Node x)

```java
/**
     * Unlinks non-null node x.
     * 断开当前元素的链接，把指定元素从链表中去掉，并处理断开之后的链接操作
     */
E unlink(Node<E> x) {
  // assert x != null; x绝对不等于null，记得这里是链表节点，并不是元素值，所以永远都不为null（需要在外部判断）； 原来自己写的实现 可以这样规定的
  final E element = x.item; //获得元素值
  final Node<E> next = x.next; //下一个元素
  final Node<E> prev = x.prev; //上一个元素
  // 下面的逻辑 初学者的话（比如像我这样的）看不懂，难懂是什么逻辑，直接看下面的图解链接删除元素，你就懂了
  if (prev == null) { //上一个节点为null，一般出现在，当前节点就是first
    first = next;   //意思就是想删除第一个节点，此时直接把第二个节点置成第一个节点。
  } else {
    prev.next = next;
    x.prev = null;
  }
  if (next == null) { //下一个节点为null,一般是：当前节点就是最后一个节点last
    last = prev;	//意思就是想删除最后一个节点，此时直接把倒数第二个元素(prev)设置为last
  } else {
    next.prev = prev;
    x.next = null;
  }
  x.item = null; // 手动置为null，GC，x的前节点，后节点，还有中间的元素，都要删除。
  size--;
  modCount++;
  return element;
}
```

{% asset_img 核心函数unlink.png) %}

#### 6 - E remove(int index)

移除此列表中指定位置处的元素。

**总结：**

1. 把链表当成有下标的形式进行操作，但是需要循环来确定index所对应的链表节点 - 我自己觉得很巧妙
2. 在循环过程中，用了一个小小（部分）的二分查找进行优化
3. 相同的含义的移除方法，相对于ArrayList来说，LinkedList的随机访问弱于 ArrayList（因为ArrayList是通过下标直接定位的）

```java
// 按指定索引删除元素
    // 链表按道理来说没有下标的，但是总有元素个数吧，而且还是有序的，从头遍历到尾巴，就相当于是下标了
    public E remove(int index) {
        //检查下标是否越界
        checkElementIndex(index);
        //这里调用了核心函数，断开此元素在链接，但是要先根据下标找到对应的链表节点
        return unlink(node(index));
    }
    private void checkElementIndex(int index) {
        if (!isElementIndex(index))
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }
    /**
     * Tells if the argument is the index of an existing element.
     */
    private boolean isElementIndex(int index) {
        //这里检查下标越界 比较完美一点，ArrayList里面都不检查是否为负数呢，不知道是为什么
        return index >= 0 && index < size;
    }
```



#### 7 - 核心函数 - Node node(int index)

```java
 /**     根据指定下标返回 对应的链表节点
     * Returns the (non-null) Node at the specified element index.
     * 只要 下标通过了检查，那么就必定能返回一个链表节点
     */
    Node<E> node(int index) {
        // 这个是断言的句子，在开发的时候是不是就应该打开呢，编写完该工具类，测试能上线之后，就注释掉该语句了？ 应该是这样的吧
        // assert isElementIndex(index);
        // 下面的查找还优化了一点，不是在任何情况下都从头开始查找
        // 有点类似部分二分查找法，先二分，然后判定是从头开始还是从尾巴开始
        if (index < (size >> 1)) {
            Node<E> x = first;
            // 从头开始遍历，当i = index的时候，那么也就找到了 下标对应的链表节点
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }
```



#### 8 - set - 将此列表中指定位置的元素替换为指定的元素。

**总结：**

1. 在LinkedList中通过下标操作的话，都需要遍历表拿到对应的链表节点

```java
public E set(int index, E element) {
        checkElementIndex(index); //又检查了下标
        Node<E> x = node(index); //找到此下标对应的链表节点
        E oldVal = x.item;
        x.item = element; 
        return oldVal;
    }
```



#### 9 - E get(int index)

返回此列表中指定位置处的元素。

**总结：**

1. 同样调用了node(index)循环遍历找到对应的链表节点

```java
public E get(int index) {
        checkElementIndex(index);
        // 使用核心函数找到对应的链表节点
        return node(index).item;
    }
```



#### 10 - Iterator iterator()

转调的父类的方法，这个就比较抽象了，我就不分析了。

```java
public Iterator<E> iterator() {
        return listIterator(); //AbstractSequentialList 父类的方法
    }
    ========================= 以下又是AbstractList的方法实现    =========================
    public ListIterator<E> listIterator() {
        return listIterator(0);
    }
    public ListIterator<E> listIterator(final int index) {
        rangeCheckForAdd(index);
        return new ListItr(index);
    }
     private class ListItr extends Itr implements ListIterator<E> {
        ListItr(int index) {
            cursor = index;
        }
        public boolean hasPrevious() {
            return cursor != 0;
        }
        public E previous() {
            checkForComodification();
            try {
                int i = cursor - 1;
                E previous = get(i);
                lastRet = cursor = i;
                return previous;
            } catch (IndexOutOfBoundsException e) {
                checkForComodification();
                throw new NoSuchElementException();
            }
        }
        public int nextIndex() {
            return cursor;
        }
        public int previousIndex() {
            return cursor-1;
        }
        public void set(E e) {
            if (lastRet < 0)
                throw new IllegalStateException();
            checkForComodification();
            try {
                AbstractList.this.set(lastRet, e);
                expectedModCount = modCount;
            } catch (IndexOutOfBoundsException ex) {
                throw new ConcurrentModificationException();
            }
        }
        public void add(E e) {
            checkForComodification();
            try {
                int i = cursor;
                AbstractList.this.add(i, e);
                lastRet = -1;
                cursor = i + 1;
                expectedModCount = modCount;
            } catch (IndexOutOfBoundsException ex) {
                throw new ConcurrentModificationException();
            }
        }
    }
```



#### 11 - boolean contains(Object o)

如果此列表包含指定元素，则返回 true。

```java
public boolean contains(Object o) {
        return indexOf(o) != -1;
    }
    // 和ArrayList 的 同名方法实现差不多。都是通过遍历查找
    public int indexOf(Object o) {
        int index = 0;
        if (o == null) {
            for (Node<E> x = first; x != null; x = x.next) {
                if (x.item == null)
                    return index;
                index++;
            }
        } else {
            for (Node<E> x = first; x != null; x = x.next) {
                if (o.equals(x.item))
                    return index;
                index++;
            }
        }
        return -1;
    }
```

# Deque(双向链表)

> 将元素推入此列表所表示的堆栈。换句话说，将该元素插入此列表的开头 

**总结：** 实现很简单

1. 直接修改first和受影响的节点指向

```java
 public void push(E e) {
        addFirst(e); //直接转调了addFirst
 }
 //将指定元素插入此列表的开头。 
 public void addFirst(E e) {
        linkFirst(e);
 }
    /**
     * Links e as first element. 链接第一个元素
     */
    private void linkFirst(E e) {
        final Node<E> f = first;
        final Node<E> newNode = new Node<>(null, e, f);
        first = newNode;
        if (f == null) //只有第一次增加数据的时候 第一个节点才会为null，所以最后一个节点也就是当前节点了
            last = newNode;
        else
            f.prev = newNode;
        size++;
        modCount++;
    }
```

## 1 - E pop()

> 从此列表所表示的堆栈处弹出一个元素。换句话说，移除并返回此列表的第一个元素。

```java
public E pop() {
  return removeFirst();
}
public E removeFirst() {
  final Node<E> f = first;
  if (f == null)
    throw new NoSuchElementException();
  return unlinkFirst(f);
}
```

### 2 - 核心函数 - E unlinkFirst(Node f)

```java
  /**
     * Unlinks non-null first node f.
     * 该函数和 unlink 函数的处理类似，只不过unlink处理更全面，该函数只针对first做处理
     * 思路是直接把第二个节点的prev置为null，
     * first置为第二个节点。
     * 要注意把第一个节点的prev，item，next都置为空，方便GC回收。
     * 一直都要考虑初始的情况，即一个数据都没有的情况。
     */
    private E unlinkFirst(Node<E> f) {
        // assert f == first && f != null;
        final E element = f.item;
        final Node<E> next = f.next;
        f.item = null;
        f.next = null; // help GC 弹出first元素的话，里面包含了next元素，也置为null，gc确保被回收
        first = next;
        if (next == null)
            last = null;//next没有数据，这个(第一个)也要删除，所以这个操作后一个数据都没有了。
        else
            next.prev = null; //因为是first所以前一个是null
        size--;
        modCount++;
        return element;
    }
```

## 3 - E poll()

> 获取并移除此列表的头（第一个元素）

```java
public E poll() {
  final Node<E> f = first;
  return (f == null) ? null : unlinkFirst(f);
}
```

## 4 - E remove()

> 获取并移除此列表的头（第一个元素）。

```java
    public E remove() {
        return removeFirst();
    }
    public E removeFirst() {
        final Node<E> f = first;
        if (f == null)
            throw new NoSuchElementException();
        return unlinkFirst(f);
    }
```

#### 小结 poll 和 pop 和 remove的区别

功能都是移除first并返回移除的元素，唯一的区别就是

1. poll 如果first为null 则返回null
2. pop 则抛出异常
3. remove 也是抛出异常

## 5 - E peek()

> 获取但不移除此列表的头（第一个元素）。

```java
    public E element() {
        return getFirst();
    }
    public E getFirst() {
        final Node<E> f = first;
        if (f == null)
            throw new NoSuchElementException();
        return f.item;
    }
```

### 6 - E element()

> 获取但不移除此列表的头（第一个元素）。

```java
    public E element() {
        return getFirst();
    }
    public E getFirst() {
        final Node<E> f = first;
        if (f == null)
            throw new NoSuchElementException();
        return f.item;
    }
```

### 小结：peek 与 element的区别

两个功能都是一样的，唯一的区别就是：element如果first节点为null则抛出异常

### 7 - boolean offer(E e)

> 将指定元素添加到此列表的末尾（最后一个元素）。

```java
    public boolean offer(E e) {
        //直接委托了 list的api添加到列表的尾部，也就是直接对 last做指向变更操作
        return add(e);
    }
```





# 简单总结

1. 底层使用链表实现
2. list接口相关的api：大部分都需要first往后遍历操作
3. deque接口相关的api：没有随机访问的能力，都是对first和last操作，
4. 由于LinkedList是list和deque的结合，所以在队列的功能上，还增加了随机访问的能力
5. 随机访问能力在增删多的情况下，也比list性能高，因为链表不需要移动n个元素
6. 随机访问的能力E get(int index)是通过核心函数 - Node node(int index)实现的。里面采用了类似二分查找的算法。