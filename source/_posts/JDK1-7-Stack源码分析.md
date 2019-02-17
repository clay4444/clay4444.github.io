---
title: JDK1.7-Stack源码分析
tags:
  - 源码解析系列
abbrlink: 1c652596
date: 2018-01-17 10:58:31
---

### 继承关系

>  在类的继承关系上，只继承Vector，，Vector继承自AbstractList，实现了List，底层用数组实现。

### 源代码结构分析

#### 1.构造器

```java
public Stack() {
}
```

#### 2. 压栈

```java
public E push(E item) {
  addElement(item);//调用Vector中的addElement方法。

  return item;
}
```

```java
public synchronized void addElement(E obj) {
  modCount++;
  ensureCapacityHelper(elementCount + 1);
  elementData[elementCount++] = obj;	//elementCount是实际的数量，先赋值，后++
}
```

```java
private void ensureCapacityHelper(int minCapacity) {
  if (minCapacity - elementData.length > 0)//需要的最小容量超过了数组长度
    grow(minCapacity);		//核心函数
}
```

#### 3. 核心函数

```java
	private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;//允许的最大空间

	//核心函数
    private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;  //之前的旧容量
        int newCapacity = oldCapacity + ((capacityIncrement > 0) ?
                                         capacityIncrement : oldCapacity
      //如果增长因子大于0，就用旧容量加上增长因子
      //否则，就把容量翻倍。
        if (newCapacity - minCapacity < 0)//如果增加之后的新容量仍然小于需要的最小容量
            newCapacity = minCapacity;   //就把需要的最小容量设置为新容量
        if (newCapacity - MAX_ARRAY_SIZE > 0) //如果新容量超过了限制，就调用设置超大容量的方法。
            newCapacity = hugeCapacity(minCapacity);
        elementData = Arrays.copyOf(elementData, newCapacity);//返回一个新数组
    }
```

```java
Integer.MAX_VALUE	private static int hugeCapacity(int minCapacity) {
  if (minCapacity < 0) // overflow
    throw new OutOfMemoryError();
  return (minCapacity > MAX_ARRAY_SIZE) ?  
    //如果真的大于MAX_ARRAY_SIZE，就返回Integer.MAX_VALUE
    Integer.MAX_VALUE :
  //否则返回定义的MAX_ARRAY_SIZE
  MAX_ARRAY_SIZE;
}
```

#### 4. 出栈peek()方法。

```java
public synchronized E peek() {
  int     len = size();

  if (len == 0)
    throw new EmptyStackException();
  return elementAt(len - 1);//调用vector中的elementAt()方法
}
```

```java
public synchronized E elementAt(int index) {
  if (index >= elementCount) {
    throw new ArrayIndexOutOfBoundsException(index + " >= " + elementCount);
  }

  return elementData(index);  //调用自己的elementData方法。
}		
```

```java
E elementData(int index) {
  return (E) elementData[index];
}
```

#### 5. 出栈并删除元素之pop()方法

```java
public synchronized E pop() {
  E       obj;
  int     len = size();

  obj = peek(); //实际上是调用了peek方法
  removeElementAt(len - 1);//然后在调用vector的removeElementAt方法把元素删除

  return obj;
}
```

```java
//删除对应索引的元素
public synchronized void removeElementAt(int index) {
  modCount++;  //快速失败行为
  if (index >= elementCount) {
    throw new ArrayIndexOutOfBoundsException(index + " >= " +
                                             elementCount);
  }
  else if (index < 0) {
    throw new ArrayIndexOutOfBoundsException(index);
  }
  int j = elementCount - index - 1;//元素数量-要删除的索引-1 = 要复制的长度
  if (j > 0) {
    //源数组，从哪里开始复制，目的数组，从哪里开始粘贴，要复制的长度，
    //相当于往前移动了一步
    System.arraycopy(elementData, index + 1, elementData, index, j);
  }
  elementCount--;//元素个数-1
  elementData[elementCount] = null; /* to let gc do its work *///删除一个元素，最后一位置空，使其被回收
}
```

