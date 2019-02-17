---
title: ArrayList添加一个元素add方法的核心
tags:
  - 源码解析系列
abbrlink: 55c3a3c7
date: 2018-01-11 11:10:09
---

### ArrayList添加一个元素add方法的核心

<br/>

### add(E e)

-确定需要确定内部容量ensureCapacityInternal(int minCapacity)，参数为当前长度+1

-把e添加进去

<br/>

### 确定内部容量ensureCapacityInternal(int minCapacity)

-判断是否是空数组，如果是空数组，就在需要的最小容量和DEFAULT_CAPACITY之间取最大值，也就是说最小初始化10个

-也就是说确定内部容量ensureCapacityInternal这个方法的目的就是确定初始化容量不能小于10个

-如果不是空数组，就开始确定明确需要的最小容量ensureCapacityInternal(int minCapacity)

<br/>

### 确定明确需要的最小容量ensureCapacityInternal(int minCapacity)

-先把操作数modcount+1

-再判断需要的最小容量是否大于数组长度length

-如果小于，什么都不干，如果大于，开始增长

<br/>

### grow函数

-先记录当前的旧的容量，

-直接把新的容量设置为旧容量的1.5倍

-再判断是否足够，如果不够，直接设置需要的最小容量为新的容量

-紧接着判断长度是否超过MAX_ARRAY_SIZE，MAX_ARRAY_SIZE定义为Integer的最大值-8，如果超过了，调用hugeCapacity方法设置超大容量，设置为Integer的最大值。

-最后调用Arrays.copyOf方法复制一个新的数组返回，