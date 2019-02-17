---
title: ArrayList删除特定元素的方法
tags:
  - 源码解析系列
abbrlink: d36c0402
date: 2018-01-12 11:11:05
---

## Java ArrayList删除特定元素的方法

<br/>

#### **最朴实的方法，使用下标的方式：**

```java
ArrayList al = new ArrayList(); 

al.add("a"); 

al.add("b"); 

for (int i = 0; i < al.size(); i++) { 

  if (al.get(i) == "b") { 

    al.remove(i); 

    i--; 
  } 
```

**在代码中，删除元素后，需要把下标减一。这是因为在每次删除元素后，ArrayList会将后面部分的元素依次往上挪一个位置(就是copy)，所以，下一个需要访问的下标还是当前下标，所以必须得减一才能把所有元素都遍历**

<br/>

**这里还有一点需要注意的就是增强for循环会使用Iterator迭代，但是for (int i = 0; i < al.size(); i++) 是不会调用的，而且ConcurrentModificationException时出现在迭代器next和remove方法中的时候的异常，remove(index)和remove(object)是不会检查expectedModCount = modCount的，所以不会出现ConcurrentModificationException异常**

<br/>

#### 还有另外一种方式：

```java
ArrayList al = new ArrayList(); 

al.add("a"); 
al.add("b"); 
al.add("b"); 
al.add("c"); 
al.add("d"); 

for (String s : al) { 
  if (s.equals("a")) { 

    al.remove(s); 

  } 
} 
```

<br/>

**此处使用元素遍历的方式，当获取到的当前元素与特定元素相同时，即删除元素。从表面上看，代码没有问题，可是运行时却报异常：**

```java
Exception in thread "main" java.util.ConcurrentModificationException 
 
at java.util.ArrayList$Itr.checkForComodification(ArrayList.java:886) 
 
at java.util.ArrayList$Itr.next(ArrayList.java:836) 
 
at com.mine.collection.TestArrayList.main(TestArrayList.java:17) 
```

<br/>

**从异常堆栈可以看出，是ArrayList的迭代器报出的异常，说明通过元素遍历集合时，实际上是使用迭代器进行访问的。可为什么会产生这个异常呢?打断点单步调试进去发现，是这行代码抛出的异常：**

```java
final void checkForComodification() { 

  if (modCount != expectedModCount) 

    throw new ConcurrentModificationException(); 

} 
```

<br/>

**modCount是集合修改的次数，当remove元素的时候就会加1，初始值为集合的大小。迭代器每次取得下一个元素的时候，都会进行判断，比较集合修改的次数和期望修改的次数是否一样，如果不一样，则抛出异常。查看集合的remove方法：**

```java
private void fastRemove(int index) { 

  modCount++; 

  int numMoved = size - index - 1; 

  if (numMoved > 0) 

    System.arraycopy(elementData, index+1, elementData, index, 

                     numMoved); 

  elementData[--size] = null; // clear to let GC do its work 

} 
```

<br/>

**可以看到，删除元素时modCount已经加一，但是expectModCount并没有增加。所以在使用迭代器遍历下一个元素的时候，会抛出异常。那怎么解决这个问题呢?其实使用迭代器自身的删除方法就没有问题了**

```java
ArrayList al = new ArrayList(); 

al.add("a"); 
al.add("b"); 	
al.add("b"); 
al.add("c");
al.add("d"); 

Iterator iter = al.iterator(); 

while (iter.hasNext()) { 

  if (iter.next().equals("a")) { 

    iter.remove(); 
  } 

} 
```

<br/>

**查看迭代器自身的删除方法，果不其然，每次删除之后都会修改expectedModCount为modCount。这样的话就不会抛出异常**

```java
public void remove() { 

  if (lastRet < 0) 

    throw new IllegalStateException(); 

  checkForComodification(); 

  try { 

    ArrayList.this.remove(lastRet); 

    cursor = lastRet; 

    lastRet = -1; 

    expectedModCount = modCount; 

  } catch (IndexOutOfBoundsException ex) { 

    throw new ConcurrentModificationException(); 

  } 

} 
```

<br/>

**建议以后操作集合类的元素时，尽量使用迭代器。可是还有一个地方不明白，modCount和expectedModCount这两个变量究竟是干什么用的?为什么集合在进行操作时需要修改它?为什么迭代器在获取下一个元素的时候需要判断它们是否一样?它们存在总是有道理的吧**

<br/>

**其实从异常的类型应该是能想到原因：ConcurrentModificationException.同时修改异常。看下面一个例子**

```java
List list = new ArrayList(); 

// Insert some sample values. 

list.add("Value1"); 

list.add("Value2"); 

list.add("Value3"); 

// Get two iterators. 

Iterator ite = list.iterator(); 

Iterator ite2 = list.iterator(); 

// Point to the first object of the list and then, remove it. 

ite.next(); 

ite.remove(); 

/* The second iterator tries to remove the first object as well. The object does not exist and thus, a ConcurrentModificationException is thrown. */ 

ite2.next(); 

ite2.remove(); 
```

<br/>

**同样的，也会报出ConcurrentModificationException。**