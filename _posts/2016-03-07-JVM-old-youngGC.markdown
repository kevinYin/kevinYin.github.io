---
layout: post
title:  "年轻代和老年代的垃圾回收"
date:   2016-03-06 12:16
categories: JVM
permalink: /Priest/old-younGC

---


**背景：**现代商业虚拟机都是采用了分代收集算法进行垃圾回收，java堆依此将java堆拆分为年轻代和年老代。

<h2>年轻代的垃圾回收过程</h2>

**描述：**年轻代分为3个区，Eden区，from survival ，to survival，三者的大小比例是8：1：1.年轻代的对象大多数都是短命鬼（IBM统计过年轻代98%的对象都是会被回收的），其GC的过程名称为Minor GC。

**Minor GC过程：**

>1.创建对象伊始，大部分对象都被分配在Eden区，一次GC后，大部分对象都会被回收，没有被回收的幸存的对象，将会从 Eden区和from survival区 复制到to survival区;复制完后，from survival区被清空，它将成为下一次GC的to survival区，而本次的 to survival区将会成为 from survival区。

>2.如果复制到 to survival区的对象由于to survival区内存不足，存活的对象将会移到老年代。

>3.如果年轻代部分对象在多次minor GC没有被回收，那么达到一定的年龄后 会被转移到老年代。JVM默认的最高年龄是15，一次Minor GC会给对象的年龄+1，达到系统设置的年龄后，会被转移到 老年代。

**设置常用的参数：**
	
	1.-XX:InitialTenuringThreshold:设定老年代阀值的初始值
	
	2.-XX:MaxTenuringThreshold:设定老年代阀值的初始值
	
	3.-XX:TargetSurvivorRatio : 设定幸存区的目标使用率

例子：XX:MaxTenuringThreshold=10 -XX:TargetSurvivorRatio=90 设定老年代阀值的上限为10,幸存区空间目标使用率为90%。

**优化原则：**希望最小化短命对象晋升到老年代的数量，同时也希望最小化新生代GC 的次数和持续时间。

<h2>年老代的垃圾回收过程</h2>

**描述：**老年代回收的过程以及效率有依赖于JVM设置何种垃圾收集器，常规的回收过程是：扫描并标记所有需要回收的对象，标记完成之后，让存活的对象向一端移动，然后将端边界以外的内存进行清理，GC过程称之为 full GC 。
**收集器：**

1.Serial GC ：针对单核机器设计，采用单线程进行GC。过程：将老年代所有存活的对象进行标记，保持位置不变，然后将所有不存活的对象的空间释放，最后把存活的对象移到对空间前面以保持空间的连续性，最终结果就是分为有对象和无对象的空间（空间压缩）。

2.Parallel GC ：采用多线程进行GC，提高GC效率，适合多核机器，被称为“高吞吐GC”。

3.Parallel Old GC：在JDK 5中被引入，与Parallel GC相比唯一的区别在于Parallel的GC算法是为老年代设计的。它的执行过程分为三步：标记(mark)–总结(summary)–压缩(compaction)。其中summary步骤会会分别为存活的对象在已执行过GC的空间上标出位置。

4.CMS GC：中断JVM（STW）时间非常短，在垃圾回收执行过程中，其他线程依然在执行，也被称为低延迟GC，适用于所有应用对响应时间要求比较严格的场景

5.G1 GC ： G1最大的改进在于其性能表现，它比以上任何一种GC都更快速。但是不稳定，其目标是在将来取代CMS GC，成为最高效的收集器。

**说明：**：java8 默认的垃圾回收器是Parallel GC ，经过测试，java8中 Parallel GC的效率最高，详情看 [这里](http://www.importnew.com/16533.html)。

**设置常用的参数：**
	
	1.-XX:+UseParallelGC 设置收集器
	
	2.-XX:NewRatio	新生代与老年代的比例


**优化原则：**：降低老年代的GC次数，最好是能够0次，因为GC过程会中断JVM （STW），如果触发full GC，希望中断JVM的时间要很短。
