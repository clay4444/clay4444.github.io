---
title: CAS算法原子变量底层到底是如何工作的
abbrlink: 2822593d
date: 2017-08-25 14:36:19
categories:
  - 多线程编程
---



### 原子变量 CAS算法

- i++ 的原子性问题：i++ 的操作实际上分为三个步骤“读-改-写”

~~~java
int i = 10;
i = i++; //10

int temp = i;
i = i + 1;
i = temp;
~~~

- 原子变量：在 java.util.concurrent.atomic 包下提供了一些原子变量。

1. volatile 保证内存可见性

2. CAS（Compare-And-Swap） 算法保证数据变量的原子性

   CAS 算法是硬件对于并发操作的支持

   CAS 包含了三个操作数：

   ​	①内存值  V

   ​	②预估值  A

   ​	③更新值  B

   ​	当且仅当 V == A 时， V = B; 否则，不会执行任何操作。

- 测试代码

~~~java
public class TestAtomicDemo {

  public static void main(String[] args) {
    AtomicDemo ad = new AtomicDemo();	
    for (int i = 0; i < 10; i++) {
      new Thread(ad).start();
    }
  }
}

class AtomicDemo implements Runnable{
  //	private volatile int serialNumber = 0;

  private AtomicInteger serialNumber = new AtomicInteger(0);

  @Override
  public void run() {
    try {
      Thread.sleep(200);
    } catch (InterruptedException e) {
    }
    System.out.println(getSerialNumber());
  }

  public int getSerialNumber(){
    return serialNumber.getAndIncrement();
  }
}
~~~

- 图示


{% asset_img clipboard.png %}

左边是第一个线程进行的操作，每个方框中的操作都是同步的。

{% asset_img clipboard1.png %}

右边是第二个线程进行的操作，每个方框中的操作都是同步的。

所以无论有多少个线程修改它的值，它的值总是唯一的。也就是具有原子性的。



### 手动模拟CAS算法：

~~~java
/*
 * 模拟 CAS 算法
 *  具体的细节在于：每次在线程执行之前，都会自己get一遍value的值。
 *  然后在compareAndSwap方法执行的时候，还会再get一遍value的值，
 *  expectedValue也就是每个线程自己get的原来的value值。
 *  然后在compareAndSwap方法中，会自己在get一遍最新的值，并且和expectedValue比较。
 *  这样做的原因就是防止在两个操作中间，有其他线程修改了主存中变量的值。
 * 如果真的其他线程修改成功了，意味着其他所有修改此变量的线程全部会失败，因为compareAndSwap在自己get一 * 遍最新的值的时候，不可能和expectedValue相等，因为compareAndSwap是线程同步的。
 */
public class TestCompareAndSwap {

  public static void main(String[] args) {
    final CompareAndSwap cas = new CompareAndSwap();

    for (int i = 0; i < 10; i++) {
      new Thread(new Runnable() {	
        @Override
        public void run() {
          int expectedValue = cas.get();
          boolean b = cas.compareAndSet(expectedValue, (int)(Math.random() * 101));
          System.out.println(b);
        }
      }).start();
    }	
  }
}
class CompareAndSwap{
  private int value;

  //获取内存值
  public synchronized int get(){
    return value;
  }
  //比较
  public synchronized int compareAndSwap(int expectedValue, int newValue){
    int oldValue = value;

    if(oldValue == expectedValue){
      this.value = newValue;
    }
    return oldValue;
  }
  //设置
  public synchronized boolean compareAndSet(int expectedValue, int newValue){
    return expectedValue == compareAndSwap(expectedValue, newValue);
  }
}
~~~

