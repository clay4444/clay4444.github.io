---
title: MR之mapreduce原理深入剖析
categories:
  - big-data
tags:
  - MapReduce
abbrlink: c1ee2109
date: 2017-09-29 16:03:26
---

### shuffle

shuffle的过程就是各种缓存，排序，分区的过程。

{% asset_img 11.png %}



### 详细解析

- 注意：环形缓冲区和   存储环形缓冲区溢出的数据放入内存，是分开的，只是它划在一起了而已，当内存中的数据到了80%之后，就开始通过spilder往文件里写，就把这80%写成一个文件，
- 再重复此步骤。就生成了很多文件。
- GroupingComparator的作用就是定义一个规则，让MR把复合这个规则的，定义为一个key。
- conbiner的作用和reduce阶段是一样的，只是reducer阶段是对全局所有maptask产生的一个分区。conbiner作用的知识当前的一个maptask，这个组件的效果就是大大提升了效率，如果有一万个 a,1 那么如果有conbiner组件，那么再单独的maptask阶段就产生一个 a,10000    就避免了后面传输10000个   key, value， 大大提升了效率。
- 但是使用conbiner组件一定要注意不能影响正确的业务逻辑。即：不能随便使用conbiner  例如

{% asset_img 22.png %}

- 图中用深红色框起来的，都是可以让用户自己定义的。
- 防止过多小文件产生过多maptask造成资源浪费的策略，(使用CombineTextInputFormat )  让它帮助我们自主合并小文件。