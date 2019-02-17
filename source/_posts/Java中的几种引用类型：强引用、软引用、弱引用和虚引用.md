---
title: Java中的几种引用类型：强引用、软引用、弱引用和虚引用
categories:
  - Java基础
tags:
  - GC
abbrlink: eeeb2ca
date: 2018-03-16 14:42:34
---

Java虽然有内存管理机制，但仍应该警惕内存泄露的问题。例如对象池、缓存中的过期对象都有可能引发内存泄露的问题。

<br/>

从JDK1.2版本开始，加入了对象的几种引用级别，从而使程序能够更好的控制对象的生命周期，帮助开发者能够更好的缓解和处理内存泄露的问题。

<br/>

这几种引用级别由高到低分别为：强引用、软引用、弱引用和虚引用。

<br/>

### **强引用**

平时我们编程的时候例如：Object object=new Object（）；那object就是一个强引用了。如果一个对象具有强引用，那就类似于必不可少的生活用品，垃圾回收器绝不会回收它。当内存空 间不足，Java虚拟机宁愿抛出OutOfMemoryError错误，使程序异常终止，也不会靠随意回收具有强引用的对象来解决内存不足问题。

<br/>

### 软引用（SoftReference）

如果一个对象只具有软引用，那就类似于可有可物的生活用品。如果内存空间足够，垃圾回收器就不会回收它，如果内存 空间不足了，就会回收这些对象的内存。只要垃圾回收器没有回收它，该对象就可以被程序使用。软引用可用来实现内存敏感的高速缓存。 软引用可以和一个引用队列（ReferenceQueue）联合使用，如果软引用所引用的对象被垃圾回收，Java虚拟机就会把这个软引用加入到与之关联 的引用队列中。

<br/>

### 弱引用（WeakReference）

如果一个对象只具有弱引用，那就类似于可有可物的生活用品。弱引用与软引用的区别在于：只具有弱引用的对象拥有更 短暂的生命周期。在垃圾回收器线程扫描它 所管辖的内存区域的过程中，一旦发现了只具有弱引用的对象，不管当前内存空间足够与否，都会回收它的内存。不过，由于垃圾回收器是一个优先级很低的线程， 因此不一定会很快发现那些只具有弱引用的对象。 弱引用可以和一个引用队列（ReferenceQueue）联合使用，如果弱引用所引用的对象被垃圾回收，Java虚拟机就会把这个弱引用加入到与之关联 的引用队列中。

<br/>

### 虚引用（PhantomReference）

如果一个对象只具有弱引用，那就类似于可有可物的生活用品。弱引用与软引用的区别在于：只具有弱引用的对象拥有更 短暂的生命周期。在垃圾回收器线程扫描它 所管辖的内存区域的过程中，一旦发现了只具有弱引用的对象，不管当前内存空间足够与否，都会回收它的内存。不过，由于垃圾回收器是一个优先级很低的线程， 因此不一定会很快发现那些只具有弱引用的对象。 弱引用可以和一个引用队列（ReferenceQueue）联合使用，如果弱引用所引用的对象被垃圾回收，Java虚拟机就会把这个弱引用加入到与之关联 的引用队列中。

<br/>

### 虚引用（PhantomReference）

“虚引用”顾名思义，就是形同虚设，与其他几种引用都不同，虚引用并不会决定对象的生命周期。如果一个对象 仅持有虚引用，那么它就和没有任何引用一样，在任何时候都可能被垃圾回收。 虚引用主要用来跟踪对象被垃圾回收的活动。虚引用与软引用和弱引用的一个区别在于：虚引用必须和引用队列 （ReferenceQueue）联合使用。当垃圾回收器准备回收一个对象时，如果发现它还有虚引用，就会在回收对象的内存之前，把这个虚引用加入到与之 关联的引用队列中。程序可以通过判断引用队列中是否已经加入了虚引用，来了解被引用的对象是否将要被垃圾回收。程序如果发现某个虚引用已经被加入到引用队 列，那么就可以在所引用的对象的内存被回收之前采取必要的行动。

<br/>

### 引用队列

在java.lang.ref包中提供了三个类：SoftReference类、WeakReference类和PhantomReference类，它们分别代表软引用、弱引用和虚引用。ReferenceQueue类表示引用队列，它可以和这三种引用类联合使用，以便跟踪Java虚拟机回收所引用的对象的活动。

<br/>

以下程序创建了一个String对象、ReferenceQueue对象和WeakReference对象：

```java
//创建一个强引用
String str = new String("hello");
//创建引用队列, <String>为范型标记，表明队列中存放String对象的引用
ReferenceQueue<String> rq = new ReferenceQueue<String>();
//创建一个弱引用，它引用"hello"对象，并且与rq引用队列关联
//<String>为范型标记，表明WeakReference会弱引用String对象
WeakReference<String> wf = new WeakReference<String>(str, rq);
```

<br/>

以上程序代码执行完毕，内存中引用与对象的关系如图所示，"hello"对象同时具有强引用和弱引用：

{% asset_img t1.png %}

带实线的箭头表示强引用，带虚线的箭头表示弱引用。从图中可以看出，此时"hello"对象被str强引用，并且被一个WeakReference对象弱引用，因此"hello"对象不会被垃圾回收。

<br/>

在以下程序代码中，把引用"hello"对象的str变量置为null，然后再通过WeakReference弱引用的get()方法获得"hello"对象的引用：

```java
String str = new String("hello"); //① 
ReferenceQueue<String> rq = new ReferenceQueue<String>(); //② 
WeakReference<String> wf = new WeakReference<String>(str, rq); //③
str=null; //④取消"hello"对象的强引用
String str1=wf.get(); //⑤假如"hello"对象没有被回收，str1引用"hello"对象
//假如"hello"对象没有被回收，rq.poll()返回null
Reference<? extends String> ref=rq.poll(); //⑥
```

执行完以上第④行后，内存中引用与对象的关系下图所示：

{% asset_img t2.png %}

此时"hello"对象仅仅具有弱引用，因此它有可能被垃圾回收。假如它还没有被垃圾回收，那么接下来在第⑤行执行wf.get()方法会返回 "hello"对象的引用，并且使得这个对象被str1强引用。再接下来在第⑥行执行rq.poll()方法会返回null，因为此时引用队列中没有任何 引用。ReferenceQueue的poll()方法用于返回队列中的引用，如果没有则返回null。

<br/>

在以下程序代码中，执行完第④行后，"hello"对象仅仅具有弱引用。接下来两次调用System.gc()方法，催促垃圾回收器工作，从而提高 "hello"对象被回收的可能性。假如"hello"对象被回收，那么WeakReference对象的引用被加入到ReferenceQueue中， 接下来wf.get()方法返回null，并且rq.poll()方法返回WeakReference对象的引用。

```java
String str = new String("hello"); //①
ReferenceQueue<String> rq = new ReferenceQueue<String>(); //② 
WeakReference<String> wf = new WeakReference<String>(str, rq); //③
str=null; //④
//两次催促垃圾回收器工作，提高"hello"对象被回收的可能性
System.gc(); //⑤
System.gc(); //⑥
String str1=wf.get(); //⑦ 假如"hello"对象被回收，str1为null
Reference<? extends String> ref=rq.poll(); //⑧
```

<br/>

在以下例程的References类中，依次创建了10个软引用、10个弱引用和10个虚引用，它们各自引用一个Grocery对象。从程序运行时的打印结果可以看出，虚引用形同虚设，它所引用的对象随时可能被垃圾回收，具有弱引用的对象拥有稍微长的生命周期，当垃圾回收器执行回收操作时，有可能被垃圾回收，具有软引用的对象拥有较长的生命周期，但在Java虚拟机认为内存不足的情况下，也会被垃圾回收。

```java
import java.lang.ref.*;
import java.util.*;
class Grocery {
  private static final int SIZE = 10000;
  // 属性d使得每个Grocery对象占用较多内存，有80K左右
  private double[] d = new double[SIZE];
  private String id;
  public Grocery(String id) {
    this.id = id;
  }
  public String toString() {
    return id;
  }
  public void finalize() {
    System.out.println("Finalizing " + id);
  }
}
public class References {
  private static ReferenceQueue<Grocery> rq = new ReferenceQueue<Grocery>();
  public static void checkQueue() {
    Reference<? extends Grocery> inq = rq.poll(); // 从队列中取出一个引用
    if (inq != null)
      System.out.println("In queue: " + inq + " : " + inq.get());
  }
  public static void main(String[] args) {
    final int size = 10;
    // 创建10个Grocery对象以及10个软引用
    Set<SoftReference<Grocery>> sa = new HashSet<SoftReference<Grocery>>();
    for (int i = 0; i < size; i++) {
      SoftReference<Grocery> ref = new SoftReference<Grocery>(
        new Grocery("Soft " + i), rq);
      System.out.println("Just created: " + ref.get());
      sa.add(ref);
    }
    System.gc();
    checkQueue();
    // 创建10个Grocery对象以及10个弱引用
    Set<WeakReference<Grocery>> wa = new HashSet<WeakReference<Grocery>>();
    for (int i = 0; i < size; i++) {
      WeakReference<Grocery> ref = new WeakReference<Grocery>(
        new Grocery("Weak " + i), rq);
      System.out.println("Just created: " + ref.get());
      wa.add(ref);
    }
    System.gc();
    checkQueue();
    // 创建10个Grocery对象以及10个虚引用
    Set<PhantomReference<Grocery>> pa = new HashSet<PhantomReference<Grocery>>();
    for (int i = 0; i < size; i++) {
      PhantomReference<Grocery> ref = new PhantomReference<Grocery>(
        new Grocery("Phantom " + i), rq);
      System.out.println("Just created: " + ref.get());
      pa.add(ref);
    }
    System.gc();
    checkQueue();
  }
}
```