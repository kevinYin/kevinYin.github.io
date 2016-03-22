---
layout: post
title:  "JVM调优实践后篇"
date:   2016-03-20 18:16
categories: JVM
permalink: /Priest/JVM-tuning-final

---



**前言：**前两篇把调优的目的以及可用于JDK定位、分析的工具，那么本次我们将制造一个场景来进行模拟线上操作，我们厂目前JVM暂时没遇到过JVM调优问题，所以只能模拟线上环境情况来测试优化过程。

场景
====

模拟一个请求调用的一个方法，不断地制造大量的对象存在一个map里，让触发full  GC，导致系统访问比较卡顿。

```java
 private static ConcurrentHashMap<String,byte[]> bigMap = new ConcurrentHashMap<>();

    @RequestMapping("gc")
    public  void testGC(HttpServletRequest request) {
        byte[] bytes = new byte[1024 * 1024 * 3];
        while (true) {
            bigMap.put(String.valueOf(Math.random()),bytes);
        }
    }
```

分析
===

遇到线上卡顿状态后，不是马上dump一个文件进行分析，因为jmap dump 会导致中断JVM（jmap dump 会触发 full GC）。如果是线上，可以按照以下顺序进行：

1.先获取进程号，使用jps获取tomcat的进程号。

>kevinYin.github.io git:master # jps

>59571 RemoteMavenServer

>74327 Launcher

2.用` jstat -gcutil 74327 10000 90` 每10秒输出一次 GC次数以及当前各区空间使用情况，以及时间长度

> S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT

>0.00  10.55  11.95  31.08  97.51  94.88     49    4.559     5    3.697    8.256

>0.00  10.55  96.47  31.08  97.51  94.88     49    4.559     5    3.697    8.256

>100.00   0.00   0.00  93.70  97.63  94.70     52    7.248     7    7.967   15.215

> 99.99   0.00   0.00  93.95  97.63  94.70     54    9.683     8   12.729   22.412

young GC和full  GC在不断地增加，从第二和第三次看，full GC的数据，在10秒内增加了2次full GC，耗时 4.3秒，第三和第四次中full GC增加了1次，但是耗时接近4秒。所以full GC是最主要的耗时所在。

3. 前面2步基本不能定位程序问题具体所在，最终的还是得靠`jmap dump`来导出dump文件进行分析。
`注意：`dump文件的路径只能是在一个不存在的文件进行才可写入，否则失败。

>kevinYin.github.io git:master # jmap -dump:format=b,file=/data/1.dump 84283

>Dumping heap to /data/1.dump ...

>Heap dump file created

生成dump文件后，直接导出用MAT打开分析。
![dump file](http://7xrmyq.com1.z0.glb.clouddn.com/dump2.png)

然后可以看出对象的分布情况：
![dumpFile2](http://7xrmyq.com1.z0.glb.clouddn.com/dump12.png)
提示看出靠前的对象 以及它的来源，根据这个就可以找到问题所在。

分析完后，进行查看，点击 工具栏第三个图标（树状图），查看当前哪些对象最大,在这里就看到是ConcurrentHashMap，占了73.8%。

![duqwe2](http://7xrmyq.com1.z0.glb.clouddn.com/6.png)

优化
=====

1. 先查看JVM的系统设置参数.

>kevinYin.github.io git:master # jmap -heap 84283


>   MinHeapFreeRatio         = 40

>   MaxHeapFreeRatio         = 70

>   MaxHeapSize              = 704643072 (672.0MB)

>   NewSize                  = 209715200 (200.0MB)

>   MaxNewSize               = 209715200 (200.0MB)

>   OldSize                  = 494927872 (472.0MB)

>   NewRatio                 = 2

>   SurvivorRatio            = 8

2.然后就是加堆内存，如果 Xmx 和  Xms 都没有设置的话，设置下，并且让两者的值一样。
但是貌似 `jmap -heap 84283` 并没有输出系统设置的JVM的参数，可通过这个命令输出已设置的参数
`java -server -XX:+UnlockExperimentalVMOptions -XX:+UnlockDiagnosticVMOptions -XX:+PrintFlagsFinal Benchmark | grep ":"`

3.设置优化参数，以tomcat为例：
tomcat是在bin目录下 的 Catalina.sh进行设置，比如的设置参数：

>CATALINA_OPTS='-server -Xms672m -Xmx672m -Xmn200m -Xss300k

>-XX:+HeapDumpOnOutOfMemoryError

>-XX:HeapDumpPath=/Library/tomcat/dumpFile/dump.hprof  

>-verbose:gc -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintTenuring    Distribution

>-Xloggc:/Library/tomcat/apache-tomcat-7.0.68/logs/gc_log.out'

JVM调优不会为了调优而调优，而是为了解决线上卡顿问题而进行，一般如果进行加大堆内存，如果没有问题，那就可以暂时使用，我们厂就是这样。
如果不行，则需要进行几种调优方案的设置，然后分别放到配置一样的机器上进行测试，然后收集GC信息来判断哪个方案是可行，最后应用到
所有机器上。
