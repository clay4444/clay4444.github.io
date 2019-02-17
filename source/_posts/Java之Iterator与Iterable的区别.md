---
title: Java之Iterator与Iterable的区别
tags:
  - 源码解析系列
abbrlink: 786266a6
date: 2018-01-19 10:04:03
---

### Iterable的定义：

[Java](http://lib.csdn.net/base/java).lang包

```java
/**
 * Implementing this interface allows an object to be the target of
 * the "foreach" statement.
 *
 * @param <T> the type of elements returned by the iterator
 *
 * @since 1.5
 */
public interface Iterable<T> {

  /**
     * Returns an iterator over a set of elements of type T.
     *
     * @return an Iterator.
     */
  Iterator<T> iterator();
}
```

### Iterator的定义：

java.util包：

```java
[java] view plain copy print?
  public interface Iterator<E> {  
  boolean hasNext();  

  E next();  

  void remove();  
}  
```

Iterator是迭代器类，而Iterable是为了只要实现该接口就可以使用foreach，进行迭代。

Iterable中封装了Iterator接口，只要实现了Iterable接口的类，就可以使用Iterator迭代器了。

集合Collection、List、Set都是Iterable的实现类，所以他们及其他们的子类都可以使用foreach进行迭代。

那为什么这些集合类不直接实现Iterator呢？

Iterator中和核心的方法next(),hasnext(),remove(),都是依赖当前位置，如果这些集合直接实现Iterator，则必须包括当前迭代位置的指针。当集合在方法间进行传递的时候，由于当前位置不可知，所以next()之后的值，也不可知。而当实现Iterable则不然，每次调用都返回一个从头开始的迭代器，**各个迭代器之间互不影响**。

<br/>

### 总结一下就是

**如果在两个不同的方法里面调用itor的next会得到不同的值。而返回迭代器就不一样了，你只能每次都从头开始取**