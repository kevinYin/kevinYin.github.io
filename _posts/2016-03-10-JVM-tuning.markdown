---
layout: post
title:  "JVM调优实践 前篇"
date:   2016-03-12 18:16
categories: JVM
permalink: /Priest/JVM-tuning-in-action

---


<h2>为何要进行JVM调优</h2>
	
基于没有修改过默认JVM参数的情况下，大多数情况下是我们是不需要进行JVM调优的。如果没有出现问题的时候，是不需要进行优化的，需要进行JVM优化的情况一般有以下情况：

>1.系统产生过多的超时的日志，即出现过多的系统超时的问题

>2.没有设置过-Xms (JVM堆初始化大小)和 -Xmx（最大堆内存） 的内存大小

>3.使用了 -server 选项（设置JVM的一些优化参数）

**总结：**`JVM调优是不得已时的选择`

<h2>调优的目标以及优化参数</h2>

<h3>调优的目标</h3>

在之前学习JVM垃圾回收时，学到JVM堆内存按年代分为年轻代和年老代，最终影响到系统性能的是老年代的垃圾回收过程，因为老年代的垃圾回收会产生中断JVM （STW）的问题，导致系统出现卡顿或者超时的问题，所以调优的目的就是调优老年代的垃圾回收。可以从2个方面进行调优老年代的垃圾回收，分别是：

>1.降低触发full GC的次数

>2.降低full GC的时间长度。

**降低触发full GC的次数：**   可以通过降低年轻代到年老代对象的数量以及扩大老年代内存来降低GC的次数。

**降低full GC的时间长度：**   可以降低老年代的内存大小来降低full GC的时间长度或者降低 full GC触发的老年代内存比例。

可以看到两点优化入口的方式，设置内存大小是有冲突的，所以内存带下过大以及过小都不好，需要设置适合的内存大小空间，所以是没有固定的优化方法，只能是从两者之间寻求一种平衡，最终的优化的方案都是在不断地试验中得到的。

<h3>优化参数</h3>

对调优的目标会有针对性的参数进行设置，JVM有很多参数可以进行优化，但是常用的参数就几个：

>1.`-Xms` (JVM堆初始化大小)和 `-Xmx`（最大堆内存）, 一般设置为两者参数值一样，目的是为了避免堆自动扩展

>2.`-XX:NewRatio` 设置年轻代与年老代的比例

>3.`-XX:NewSize` 设置年轻代的大小

>4.`-XX:SurvivorRatio`  设置Eden区与survival区的比例   


以上是网上NHN公司性能实验室工程师 Sangmin Lee的实践后总结常用到的，但是我个人觉得还有个参数也是可以同样有参考作用.

>5.`-XX:MaxTenuringThreshold`  设定老年代阀值的上限(年轻代的对象会在多次GC后如果还存留在幸存区，但不会一直保留，有规定的年限（默认15），超过年限后就会直接进入老年代)


<h2>理解GC日志</h2>

为了进一步了解GC的信息，其中最基本的就是GC的日志了，于是写个程序在运行过程中打印其中的GC日志信息，来理解GC的过程。

```java
public class TestGC {

	private static final  int _1MB = 1024*1024;
		
	public static void main(String[] args) {
	    byte[] a1,a2,a3;
	    String testString = "";
	    for (int i = 0; i < 10; i++) {
	        a1 = new byte[_1MB * 4];
	        a2 = new byte[5 * _1MB];
	    }
	}
}
```
 
其 JVM的参数设置为：

	-verbose:gc -Xms20M -Xmx20M -Xmn5M
	-XX:+PrintGCDetails
	-XX:SurvivorRatio=8
	-XX:MaxTenuringThreshold=15

打印出的日志如下：

	[GC (Allocation Failure) [PSYoungGen: 5611K->512K(9216K)] 14827K->9736K(19456K), 0.0011273 secs] [Times: user=0.01 sys=0.00, real=0.00 secs] 
	[GC (Allocation Failure) [PSYoungGen: 5632K->496K(9216K)] 14856K->9720K(19456K), 0.0010376 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
	[GC (Allocation Failure) --[PSYoungGen: 4755K->4755K(9216K)] 13979K->13979K(19456K), 0.0019501 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
	[Full GC (Ergonomics) [PSYoungGen: 4755K->0K(9216K)] [ParOldGen: 9224K->4509K(10240K)] 13979K->4509K(19456K), [Metaspace: 3019K->3019K(1056768K)], 0.0056390 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
	......
	Heap
	 PSYoungGen      total 9216K, used 5202K [0x00000007bf600000, 	0x00000007c0000000, 0x00000007c0000000)
 	  eden space 8192K, 63% used 		[0x00000007bf600000,0x00000007bfb14930,0x00000007bfe00000)
 	 from space 1024K, 0% used [0x00000007bfe00000,0x00000007bfe00000,0x00000007bff00000)
 	  to   space 1024K, 0% used [0x00000007bff00000,0x00000007bff00000,0x00000007c0000000)
	 ParOldGen       total 10240K, used 8704K [0x00000007bec00000, 0x00000007bf600000, 0x00000007bf600000)
	  object space 10240K, 85% used [0x00000007bec00000,0x00000007bf480020,0x00000007bf600000)
	 Metaspace       used 3025K, capacity 4494K, committed 4864K, reserved 1056768K
	  class space    used 333K, capacity 386K, committed 512K, reserved 1048576K

`解读:`
(1)`[GC (Allocation Failure) [PSYoungGen: 5611K->512K(9216K)] 14827K->9736K(19456K), 0.0011273 secs] [Times: user=0.01 sys=0.00, real=0.00 secs]`
翻译：young GC：【年轻代空间：GC前该空间使用大小是5611K->GC后该空间使用512K大小（该区域总的大小是9216K）】 当前的java堆使用大小是14827K -> GC 后还占用9736K(java堆得大小是19456K)，本次GC耗时0.0011273S

（2）`[Full GC (Ergonomics) [PSYoungGen: 4755K->0K(9216K)] [ParOldGen: 9224K->4509K(10240K)] 13979K->4509K(19456K), [Metaspace: 3019K->3019K(1056768K)], 0.0056390 secs]`
翻译：full GC：【年轻代空间：GC前该空间使用大小是4755K->GC后该空间使用0大小（该区域总的大小是9216K）】 【老年代空间：GC前该空间使用大小是9224K->GC后该空间使用4509K大小（该区域总的大小是10240K）】当前的java堆使用大小是13979K -> GC 后还占用4509K(java堆得大小是19456K)，【元空间：GC前该空间使用大小是3019K->GC后该空间使用3019K大小（该区域总的大小是1056768K】，本次GC耗时0.0056390

备注：Metaspace 是 java8 替代永久代的一个内存空间，之前的InternedStrings 也从PermanentSpace转移到MetaSpace中，MetaSpace可以自动扩展。

<h2>JVM参数的获取</h2>

调优前，需要进行JVM目前已经设置的参数来做参考，目前JVM有提供以下参数进行JVM参数的获取。

`java -server -XX:+UnlockExperimentalVMOptions -XX:+UnlockDiagnosticVMOptions -XX:+PrintFlagsFinal Benchmark` 用来获取所有的JVM的参数，一大坨。

加上“：” 过滤出已经设置的JVM的参数，例如：
`java -server -XX:+UnlockExperimentalVMOptions -XX:+UnlockDiagnosticVMOptions -XX:+PrintFlagsFinal Benchmark | grep ":"`  打印出的结果如是：

>     intx CICompilerCount                         := 3                                   {product}
    uintx InitialHeapSize                          := 134217728                           {product}
    uintx MaxHeapSize                              := 2147483648                          {product}
    uintx MaxNewSize                               := 715653120                           {product}
    uintx MinHeapDeltaBytes                        := 524288                              {product}
    uintx NewSize                                  := 44564480                            {product}
    uintx OldSize                                  := 89653248                            {product}
     bool PrintFlagsFinal                          := true                                {product}
     bool UnlockDiagnosticVMOptions                := true                                {diagnostic}
     bool UnlockExperimentalVMOptions              := true                                {experimental}
     bool UseCompressedClassPointers               := true                                {lp64_product}
     bool UseCompressedOops                        := true                                {lp64_product}
     bool UseFastUnorderedTimeStamps               := true                                {experimental}
     bool UseParallelGC                            := true                                {product}
 

 可以看到JVM的目前的设置参数，可以看到 java8 默认的收集器就是 `UseParallelGC `....balala。
 
 本篇到此结束，讲完了基本调优的目的 ，关联参数 以及 查看理解 GC日志和获取当前JVM的参数，接下来，就讲下几个JVM 自带的工具来定位 GC问题 ，最后再酿造场景进行调优分析。












