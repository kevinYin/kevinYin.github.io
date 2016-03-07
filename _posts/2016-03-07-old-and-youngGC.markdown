---
layout: post
title:  "年轻代和老年代的垃圾回收"
date:   2016-03-06 00:16
categories: JVM
permalink: /Priest/old-younGC
---

#年轻代和老年代的垃圾回收
---
**背景：**现代商业虚拟机都是采用了分代收集算法进行垃圾回收，java堆依此将java堆拆分为年轻代和年老代。
##年轻代的垃圾回收过程
**描述：**年轻代分为3个区，Eden区，from survival ，to survival，三者的大小比例是8：1：1.年轻代的对象大多数都是短命鬼（IBM统计过年轻代98%的对象都是会被回收的），其GC的过程名称为Minor GC。

**Minor GC过程：**
>1.创建对象伊始，大部分对象都被分配在Eden区，一次GC后，大部分对象都会被回收，没有被回收的幸存的对象，将会从 Eden区和from survival区 复制到to survival区;复制完后，from survival区被清空，它将成为下一次GC的to survival区，而本次的 to survival区将会成为 from survival区。

>2.如果复制到 to survival区的对象由于to survival区内存不足，存活的对象将会移到老年代。

>3.如果年轻代部分对象在多次minor GC没有被回收，那么达到一定的年龄后 会被转移到老年代。JVM默认的最高年龄是15，一次Minor GC会给对象的年龄+1，达到系统设置的年龄后，会被转移到 老年代。

**设置常用的参数：** 

-XX:InitialTenuringThreshold:设定老年代阀值的初始值

-XX:MaxTenuringThreshold:设定老年代阀值的初始值

-XX:TargetSurvivorRatio : 设定幸存区的目标使用率

例子：XX:MaxTenuringThreshold=10 -XX:TargetSurvivorRatio=90 设定老年代阀值的上限为10,幸存区空间目标使用率为90%。

**优化原则：**希望最小化短命对象晋升到老年代的数量，同时也希望最小化新生代GC 的次数和持续时间。

##年老代的垃圾回收过程

				