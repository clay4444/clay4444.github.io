---
title: ArrayList源码赏析
tags:
  - 源码解析系列
abbrlink: 5b758090
date: 2018-01-08 11:28:43
---

# ArrayList

- ArrayList是List类的一个典型的实现，是基于数组实现的List类，因此，ArrayList封装了一个动态的、可变长度的Object[]数组。ArrayList是通过==initialCapacity==参数来设置数组长度的，当向ArrayList添加的数据超出了ArrayList的长度之后，initialCapacity会自动增加。

```java
//elementData存储ArrayList内的元素，size表示它包含的元素的数量。
private transient Object[] elementData;
private int size;
```

<br/>

- 关键字：transient，Java的serialization提供了一种持久化对象实例的机制。当持久化对象时，可能有一个特殊的对象数据成员，我们不想用serialization机制来保存它。为了在一个特定对象的一个域上关闭serialization，可以在这个域前加上关键字transient。

```java
public class UserInfo implements Serializable {  
  private static final long serialVersionUID = 996890129747019948L;  
  private String name;  
  private transient String psw;  

  public UserInfo(String name, String psw) {  
    this.name = name;  
    this.psw = psw;  
  }  

  public String toString() {  
    return "name=" + name + ", psw=" + psw;  
  }  
}  

public class TestTransient {  
  public static void main(String[] args) {  
    UserInfo userInfo = new UserInfo("张三", "123456");  
    System.out.println(userInfo);  
    try {  
      // 序列化，被设置为transient的属性没有被序列化  
      ObjectOutputStream o = new ObjectOutputStream(new FileOutputStream(  
        "UserInfo.out"));  
      o.writeObject(userInfo);  
      o.close();  
    } catch (Exception e) {  
      // TODO: handle exception  
      e.printStackTrace();  
    }  
    try {  
      // 重新读取内容  
      ObjectInputStream in = new ObjectInputStream(new FileInputStream(  
        "UserInfo.out"));  
      UserInfo readUserInfo = (UserInfo) in.readObject();  
      //读取后psw的内容为null  
      System.out.println(readUserInfo.toString());  
    } catch (Exception e) {  
      // TODO: handle exception  
      e.printStackTrace();  
    }  
  }  
}
```

- ==被标记为transient的属性在对象被序列化的时候不会被保存==。

<br/>

## 构造方法

<br/>

- ArrayList提供了三种方式的构造器。

```java
// ArrayList带容量大小的构造函数。
public ArrayList(int initialCapacity) {
  super();
  if (initialCapacity < 0)
    throw new IllegalArgumentException("Illegal Capacity: "+
                                       initialCapacity);
  this.elementData = new Object[initialCapacity];
}

//ArrayList无参数构造参数，默认容量0
public ArrayList() {
  super();
  this.elementData = EMPTY_ELEMENTDATA;
}

// 创建一个包含collection的ArrayList   
public ArrayList(Collection<? extends E> c) {
  elementData = c.toArray(); //调用toArray()方法把collection转换成数组 
  size = elementData.length; //把数组的长度赋值给ArrayList的size属性
  // c.toArray might (incorrectly) not return Object[] (see 6260652)
  if (elementData.getClass() != Object[].class)
    elementData = Arrays.copyOf(elementData, size, Object[].class);
}
```

<br/>

- 注意点：在JDK1.6中无参数的构造方法是这么写的：

```java
// ArrayList无参构造函数。默认容量是10。    

public ArrayList() {    

  this(10);    
}  
```

<br/>

- 在1.7前，会默认在内存中直接分配10个空间，但是在1.7有了改变，会先在内存中分配一个对象的内存空间，但是这个对象是没有长度的。但是在你进行添加的时候，默认的会去拿对象的默认大小来作比较。

<br/>

- ArrayList的实现中大量地调用了Arrays.copyof()和System.arraycopy()方法。我们有必要对这两个方法的实现做下深入的了解。

<br/>

- 首先来看Arrays.copyof()方法。它有很多个重载的方法，但实现思路都是一样的，我们来看泛型版本的源码：

```java
public static <T> T[] copyOf(T[] original, int newLength) {  
  return (T[]) copyOf(original, newLength, original.getClass());  
}
```

- 很明显调用了另一个copyof方法，该方法有三个参数，最后一个参数指明要转换的数据的类型，其源码如下：

```java
public static <T,U> T[] copyOf(U[] original, int newLength, Class<? extends T[]> newType) {  
  T[] copy = ((Object)newType == (Object)Object[].class)  
    ? (T[]) new Object[newLength]  
    : (T[]) Array.newInstance(newType.getComponentType(), newLength);  
  System.arraycopy(original, 0, copy, 0,  
                   Math.min(original.length, newLength));  
  return copy;  
}
```

- 这里可以很明显地看出，该方法实际上是在其内部又创建了一个长度为newlength的数组，调用System.arraycopy()方法，将原来数组中的元素复制到了新的数组中。

<br/>

- 下面来看System.arraycopy()方法。该方法被标记了native，调用了系统的C/C++代码，在JDK中是看不到的，但在openJDK中可以看到其源码。该函数实际上最终调用了c语言的memmove()函数，因此它可以保证同一个数组内元素的正确复制和移动，比一般的复制方法的实现效率要高很多，很适合用来批量处理数组。Java强烈推荐在复制大量数组元素时用该方法，以取得更高的效率。
- 函数原型是：

```java
public static void arraycopy(Object src,
                             int srcPos,
                             Object dest,
                             int destPos,
                             int length)
```

- src:源数组；	srcPos:源数组要复制的起始位置；
  - dest:目的数组；destPos:目的数组放置的起始位置；	length:复制的长度。
- 注意：src and dest都必须是同类型或者可以进行转换类型的数组．
  <br/>
- 有趣的是这个函数可以实现自己到自己复制，比如：

```java
int[] fun ={0,1,2,3,4,5,6}; 
System.arraycopy(fun,0,fun,3,3);
```

- 则结果为：{0,1,2,0,1,2,6};

<br/>

- 实现过程是这样的，先生成一个长度为length的临时数组,将fun数组中srcPos到srcPos+length-1之间的数据拷贝到临时数组中，再执行System.arraycopy(临时数组,0,fun,3,3).

<br/>

- 所以Arrays.copyof()函数的作用就是一防止若不是Object[]将调用Arrays.copyOf方法将其转为Object[]）

<br/>

## ArrayList的动态扩容（核心）

- 当ArrayList进行add操作的时候，如果添加的元素超出了数组的长度，怎么办？

```java
public boolean add(E e) {
  ensureCapacityInternal(size + 1);  // Increments modCount!!
  elementData[size++] = e;
  return true;
}
```

<br/>

- add方法会去调用下面的方法，根据传入的最小需要容量minCapacity来和数组的容量长度对比，若minCapactity大于或等于数组容量，则需要进行扩容。

```java
private void ensureCapacityInternal(int minCapacity) {
  if (elementData == EMPTY_ELEMENTDATA) {                     //如果此时还是空数组
    minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);  
    //在最小需要容量和DEFAULT_CAPACITY(10)之间取最大值，也就是说最小初始化10个。
  }

  ensureExplicitCapacity(minCapacity);
}

private void ensureExplicitCapacity(int minCapacity) {
  modCount++;                     //快速失败行为

  //超出了数组可容纳的长度，需要进行动态扩展
  if (minCapacity - elementData.length > 0)
    grow(minCapacity);
}
```

<br/>

- 扩容的时候会去调用grow()方法来进行动态扩容，在grow中采用了位运算，我们知道位运算的速度远远快于整除运算：

```java
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

//这才是动态扩展的精髓，看到这个方法，ArrayList瞬间被打回原形
private void grow(int minCapacity) {
  int oldCapacity = elementData.length;
  //首先得到数组的旧容量，然后进行oldCapacity + (oldCapacity >> 1)，将oldCapacity 右移一位，其效果相当于oldCapacity /2，整句的结果就是设置新数组的容量扩展为原来数组的1.5倍
  int newCapacity = oldCapacity + (oldCapacity >> 1);
  //再判断一下新数组的容量够不够，够了就直接使用这个长度创建新数组， 
  //不够就将数组长度设置为需要的长度
  if (newCapacity - minCapacity < 0)
    newCapacity = minCapacity;
  //判断有没超过最大限制，如果超出限制则调用hugeCapacity
  if (newCapacity - MAX_ARRAY_SIZE > 0)
    newCapacity = hugeCapacity(minCapacity);
  //将原来数组的值copy新数组中去， ArrayList的引用指向新数组
  //这儿会新创建数组，如果数据量很大，重复的创建的数组，那么还是会影响效率，
  //因此鼓励在合适的时候通过构造方法指定默认的capaticy大小
  elementData = Arrays.copyOf(elementData, newCapacity);
}
```

```java
private static int hugeCapacity(int minCapacity) {
  if (minCapacity < 0) // overflow
    throw new OutOfMemoryError();
  return (minCapacity > MAX_ARRAY_SIZE) ?
    Integer.MAX_VALUE :
  MAX_ARRAY_SIZE;
}
```

<br/>

- 有一点需要注意的是，容量拓展，是创建一个新的数组，然后将旧数组上的数组copy到新数组，这是一个很大的消耗，所以在我们使用ArrayList时，最好能预计数据的大小，在第一次创建时就申请够内存。
  <br/>
- 看一下JDK1.6的动态扩容的实现原理：
- 从代码上，我们可以看出区别： 
  <br/>

1. 第一：在容量进行扩展的时候，其实例如整除运算将容量扩展为原来的1.5倍加1，而jdk1.7是利用位运算，从效率上，jdk1.7就要快于jdk1.6。 

<br/>

1. 第二：在算出newCapacity时，其没有和ArrayList所定义的MAX_ARRAY_SIZE作比较，为什么没有进行比较呢，原因是jdk1.6没有定义这个MAX_ARRAY_SIZE最大容量，也就是说，其没有最大容量限制的，但是jdk1.7做了一个改进，进行了容量限制。

<br/>

- 注意ArrayList的两个转化为静态数组的toArray方法。 
  <br/>

1. 第一个，Object[] toArray()方法。该方法有可能会抛出java.lang.ClassCastException异常，如果直接用向下转型的方法，将整个ArrayList集合转变为指定类型的Array数组，便会抛出该异常，而如果转化为Array数组时不向下转型，而是将每个元素向下转型，则不会抛出该异常，显然对数组中的元素一个个进行向下转型，效率不高，且不太方便。

```java
public Object[] toArray() {
  return Arrays.copyOf(elementData, size);
}
```

<br/>

1. 第二个，<T> T[] toArray(T[]a)方法。__该方法可以直接将ArrayList转换得到的Array进行整体向下转型__（转型其实是在该方法的源码中实现的），且从该方法的源码中可以看出，参数a的大小不足时，内部会调用Arrays.copyOf方法，该方法内部创建一个新的数组返回，因此对该方法的常用形式如下：

```java
public static Integer[] vectorToArray2(ArrayList<Integer> v) {    
  Integer[] newText = (Integer[])v.toArray(new Integer[0]);    
  return newText;    
}
```

```java
//源码
public <T> T[] toArray(T[] a) {
  if (a.length < size)
    // Make a new array of a's runtime type, but my contents:
    return (T[]) Arrays.copyOf(elementData, size, a.getClass());
  System.arraycopy(elementData, 0, a, 0, size);
  if (a.length > size)
    a[size] = null;
  return a;
}
```

<br/>

## remove - 按索引移除元素

### **总结：**

<br/>

1. 移除索引不能大于最大元素数量
2. 每次操作都需要对移除后的数据进行移动操作：在删除较多的场景里面使用该方法影响性能

```java
//按索引移除该元素，并返回被移除的元素值
public E remove(int index) {
  // 检查索引的有效性，索引大于size则数组越界
  rangeCheck(index);

  modCount++; //快速失败思想，增加修改次数
  E oldValue = elementData(index);

  //计算需要移动的元素个数
  int numMoved = size - index - 1;
  if (numMoved > 0)
    //把源数组的删除索引之后的元素都往前copy覆盖
    System.arraycopy(elementData, index+1, elementData, index,
                     numMoved);
  //置空最后一个元素，并把size减少1
  //这里置空元素，只是为了让gc尽快的回收掉移除的元素，小技巧用得很对
  elementData[--size] = null; // clear to let GC do its work

  return oldValue;
}
// 检查指定下标是否在范围内
private void rangeCheck(int index) {
  //这里可以看到只检查了是否大于等于siez，如果为负数的话，则没有检查，那么在插入的时候Native Method会抛出数据越界的
  if (index >= size)
    throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
```

<br/>

## remove - 移除指定的元素

### **总结：**

1. 按value删除元素，只会删除第一次出现的元素
2. 按value删除元素，在内部会进行for循环匹配元素：所以对于删除来说，list性能稍低

```java
// 移除指定的元素，如果这个元素存在的话，并且移除的是第一次出现的元素
// 移除成功或则返回true，未搜索找元素则返回false
public boolean remove(Object o) {
  //因为list支持 null 元素，所以对null进行特殊处理
  if (o == null) {
    for (int index = 0; index < size; index++)
      if (elementData[index] == null) {
        fastRemove(index);
        return true;
      }
  } else {
    //使用元素下标进行遍历判断
    for (int index = 0; index < size; index++)
      if (o.equals(elementData[index])) {

        fastRemove(index);
        return true;
      }
  }
  return false;
}
/*
     * Private remove method that skips bounds checking and does not
     * return the value removed.
     * 私有的删除元素方法，不进行边界检查和是否存在的检查，直接按照指定的索引进行删除
     * （貌似又学到一个规则：公开的的方法必须校验边界规则什么的，私有的方法就不用了，方便公用       * 吗？）
     * 该方法的代码和public remove(index) 中的代码部分主要删除逻辑一致
     */
private void fastRemove(int index) {
  modCount++;
  int numMoved = size - index - 1;
  if (numMoved > 0)
    //
    System.arraycopy(elementData, index+1, elementData, index,
                     numMoved);
  elementData[--size] = null; // clear to let GC do its work
}
```

<br/>

## set - 用指定的元素替代此列表中指定位置上的元素。

**总结：**

1. 直接根据下标替换掉原有元素

```java
//用指定的元素替代此列表中指定位置上的元素。
public E set(int index, E element) {
  //在remove中也使用了该函数，检查 index>=siez
  rangeCheck(index);
  //记录替换之前的元素
  E oldValue = elementData(index);
  //直接更新
  elementData[index] = element;
  return oldValue;
}
//获得指定下标的 元素
@SuppressWarnings("unchecked")
E elementData(int index) {
  return (E) elementData[index];
}
```

<br/>

## get - 返回此列表中指定位置上的元素。

```java
public E get(int index) {
  //检查下标是否越界size
  rangeCheck(index);
  //直接通过下标拿到了元素
  return elementData(index);
}
```

<br/>

## Iterator - 遍历

<br/>

### 总结一

**要让一个对象拥有foreach的目标，则需要有以下两个要求：**

<br/>

1. 目标对象实现 Iterable 接口
2. 实现Iterator，对数据进行常用操作访问，实现接口的方法会返回一个 Iterator 类型的对象，所以得自己实现对自己数据结构的访问等操作，foreach 是调用了 Iterator 的方法实现的。

<br/>

**所以问题就来了：**

```java
for(i=0;i<size;i++) 这种语法是不是和迭代器无关了，从这个看来，for(;;)只是对i进行了一个自增的操作，和具体的对象不是绑定状态，但是由于ArrayList能使用下标访问数据，所以就能多一种遍历方式了
```

<br/>

### 总结二

1. 遍历获取元素 是通过下标直接获取

<br/>

1. 但是删除元素的话是委托了 原来的删除方式，很原来的删除原理一致，但是保证了当前元素下标的正确性，弥补了for(;;)删除导致容器size变化和元素下标位置发生变化 从而无法获取正确的元素的 问题

<br/>

1. Iterator 可以使用while来遍历

```java
// 返回一个在一组 T 类型的元素上进行迭代的迭代器。
// 这个跌迭代器是 快速失败的，也就是会检查modCount，发现不对就抛出异常
public Iterator<E> iterator() {
  return new Itr();
}
/**
     * An optimized version of AbstractList.Itr
     * 对父类的优化。- 这个抽象层次真好
     */
private class Itr implements Iterator<E> {
  // 返回下一步元素的索引
  int cursor;       // index of next element to return
  //返回最后一次操作的元素索引，-1 表示没有操作过元素，用于删除操作使用
  int lastRet = -1; // index of last element returned; -1 if no such
  //快速失败检查，保留一个当前迭代器所持有的数据版本；
  int expectedModCount = modCount;

  //是否有下一个元素
  public boolean hasNext() {
    // 当前游标不等于size的话，就表示还有下一个元素
    return cursor != size;
  }

  @SuppressWarnings("unchecked")
  public E next() {
    checkForComodification();
    int i = cursor; //在开头获取下一步的值，且以0开始，则i表示当前操作的元素下标
    if (i >= size) /当前元素索引如果大于了size
      throw new NoSuchElementException();
    Object[] elementData = ArrayList.this.elementData;
    // 判断 当前下标是否大于了数组的长度（什么情况下会出现此问题呢？）
    if (i >= elementData.length)
      throw new ConcurrentModificationException();
    cursor = i + 1; //记录下一步要操作的元素
    return (E) elementData[lastRet = i];
  }

  public void remove() {
    // 如果还没有操作过元素，比如，没有调用next方法，就调用该方法就表示状态不正确
    if (lastRet < 0)
      throw new IllegalStateException();
    checkForComodification(); //每次操作都需要检查是否快速失败

    try {
      //调用原来的移除方法进行删除元素，当前也会使modCount次数增加
      ArrayList.this.remove(lastRet);
      cursor = lastRet; //因为删除了元素，删除下标后面的元素都会往前移动
      lastRet = -1; //变成-1 状态，如果连续调用该方法则抛出异常（这里设计得真的好巧妙，保证了删除操作）
      //更改当前迭代器所持有的数据版本，否则就会导致快速失败异常了
      expectedModCount = modCount;
    } catch (IndexOutOfBoundsException ex) {
      //对于数据越界操作，都定义为 当方法检测到对象的并发修改，但不允许这种修改时，抛出此异常
      throw new ConcurrentModificationException();
    }
  }
  //检查当前迭代器的版本是否与 容器列表中的数据版本是否一致，如果不一致，那么当前迭代数据就会发生下标越界等异常，所以需要抛出异常（比如在多线程中，或则在迭代中使用list.remove() 或则add等方法 都会导致抛出异常）
  final void checkForComodification() {
    if (modCount != expectedModCount)
      throw new ConcurrentModificationException();
  }
}
```

<br/>

## contains - 如果包含指定的元素返回true

**总结：**

<br/>

1. 查找是否包含同样是从头到尾的循环遍历匹配
2. 是否匹配 使用equals方法比对，所以自己可以覆盖equals方法来查找引用型自定义对象

```java
public boolean contains(Object o) {
  return indexOf(o) >= 0;
}
// 在数组中循环遍历查找指定的元素，如果存在则返回该元素下标，是返回首次出现的元素哦
public int indexOf(Object o) {
  if (o == null) {
    for (int i = 0; i < size; i++)
      if (elementData[i]==null)
        return i;
  } else {
    for (int i = 0; i < size; i++)
      //使用的是equals方法
      if (o.equals(elementData[i]))
        return i;
  }
  return -1;
}
```

