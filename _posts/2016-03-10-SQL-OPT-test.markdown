---
layout: post
title:  "JVM调优实践 中篇"
date:   2016-03-10 00:16
categories: JVM
permalink: /Priest/JVM-tunin-2

---


<h2>JDK工具</h2>
 
`前言：` JDK自带了很多工具用于监控以及处理故障（jps jmap jstat 等），同时目前也有其他专业分析JVM异常的工具，比如eclipse的MAT（Memory Analyse Tool），都可以用于分析JVM的故障。

<h3>JPS:查看虚拟机进程</h3>

jps就是用于查看所有正在运行的虚拟机进程，它是后面其他工具分析JVM 问题的基础，用法如下。

**1.获取进程号以及主类名称**

>kevinYin.github.io git:master # jps   
                                                                    
>6086 Bootstrap
>7018 Launcher
>14459 Jps 


**2.jps -v 获取启动时的JVM参数**

>kevinYin.github.io git:master # jps -v

>6086 Bootstrap -Djava.util.logging.config.file=/Library/tomcat/>apache-tomcat-7.0.68//conf/logging.properties -
······

>-Xms672m -Xmx672m -Xmn200m -Xss300k -XX:+UseConcMarkSweepGC -XX:+UseParNewGC -XX:+CMSClassUnloadingEnabled -XX:+DisableExplicitGC -verbose:gc -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintTenuringDistribution -Xloggc:/Library/tomcat/apache-tomcat-7.0.68/logs/gc_log.out

JPS 还有几个参数可以进行使用 -q -m -l，但都没很大用处，就不照搬教科书了，记住这个就差不多了。


<h3>jstat:虚拟机统计信息监视工具，是定位JVM性能问题首选工具</h3>

jstat 主要用来获取当前java 堆信息，比如 Eden区和2个幸存区，老区，元空间，young GC 和 full  GC的次数 以及时间。

>kevinYin.github.io git:master # jstat -gc 6086                                                     
>
 S0C    S1C    S0U    S1U      EC       EU        OC         OU       MC     MU    CCSC   CCSU   YGC     YGCT    FGC    FGCT     GCT
 
>20480.0  20480.0   0.0    1643.2  163840.0  130577.7   483328.0   76608.0    56820.0  55175.3  6188.0  5816.1    157    1.691   4      0.106    1.797


`jstat -gc pid `输出的是具体的大小，而 另一个常用的 `jstat -gcutil pid`则输出的是比例

>kevinYin.github.io git:master # jstat -gcutil 6086                                               

>  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT

>  9.49   0.00  24.90  15.85  97.11  93.99    162    1.720     4    
0.106    1.826

解读下：
>S0:幸存区0：使用9.49%；  E:eden区:使用率24.90 YGC：young GC次数 162次，YGCT的总耗时：1.72S  ，FGC ： full GC的次数 4次，总耗时 0.106S  总的GC耗时（GCt）：1.826

<h3>jinfo:查看参数值</h3>

没什么好说的，就主要用来查看参数值

>kevinYin.github.io git:master # jinfo -flag SurvivorRatio 6086                                     

>-XX:SurvivorRatio=8

<h3>jmap:生成堆转储快照（dump文件）</h3>

jmap主要就是用于生成java堆转储快照 以及查看java堆得详细信息。jmap有好几个参数，但是常用的就那一两个：

**1.jmap -heap pid  (显示堆信息，使用何种回收器，分代情况)**

>kevinYin.github.io git:master # jmap -heap 6086                                                  
> ·····
>using parallel threads in the new generation.
using thread-local object allocation.
Concurrent Mark-Sweep GC

>Heap Configuration:
   MinHeapFreeRatio         = 40
   MaxHeapFreeRatio         = 70
   MaxHeapSize              = 704643072 (672.0MB)
   NewSize                  = 209715200 (200.0MB)
   MaxNewSize               = 209715200 (200.0MB)
   OldSize                  = 494927872 (472.0MB)
   NewRatio                 = 2
   SurvivorRatio            = 8
   MetaspaceSize            = 21807104 (20.796875MB)
   CompressedClassSpaceSize = 1073741824 (1024.0MB)
   MaxMetaspaceSize         = 17592186044415 MB
   G1HeapRegionSize         = 0 (0.0MB)

**2.jmap -histo pid : 显示 类、实例数量以及容量等**

> num       #instances         #bytes  class name

>   1:        152960       22569120  [C

>   2:         14101       17528280  [B

>   3:        150165        3603960  java.lang.String

>   4:         25780        2268640  java.lang.reflect.Method

>   5:         58164        1861248  java.util.HashMap$Node
 
**3.生成dump文件**

>kevinYin.github.io git:master # jmap -dump:format=b,file=jvm.dump 6086 
                        
>Dumping heap to /code/github/kevinYin.github.io/jvm.dump ...

>Heap dump file created

生成dump文件后，就可以进行分析了，可以用 jhat 或者 MAT.

`备注`：jmap是分析JVM 问题的一个很重要的工具，一般的处理流程，先`-heap`查看下堆信息，然后`-histo:live`查看下是哪一个对象占用了很大的内存空间,最后dump 下获取`-dump`文件进行分析。

<h3>jhat:分析JVM堆转储快照工具</h3>
分析dump文件的：

>kevinYin.github.io git:master # jhat heapDump                                                    

>Reading from heapDump...
Dump file created Sat Mar 19 16:09:25 CST 2016
Snapshot read, resolving...
Resolving 822548 objects...
Chasing references, expect 164 dots.........................
Eliminating duplicate references............................
Snapshot resolved.
Started HTTP server on port 7000
Server is ready.

然后访问 http://localhost:7000/ ，查看所有的类，感觉没多大作用，然后访问下http://localhost:7000/histo/ 查看所有的类型的数量以及占用空间，跟jmap -histo 的效果一样。 总体感觉没很大用处。

<h3>jstack:用于生成java虚拟机当前时刻的线程快照</h3>
生成线程快照的主要目的是定位线程出现长时间停顿的原因，如线程间死锁、死循环、请求外部资源导致的长时间等待等。线程出现停顿的时候通过jstack来查看各个线程的调用堆栈，就可以知道没有响应的线程到底在后台做什么事情，或者等待什么资源。

//显示除堆栈信息外加锁的信息
>kevinYin.github.io git:master # jstack -l  6086

具体的线程快照信息相对复杂，在这里暂不做分析。

`总结:`JDK自带这么多个工具，实际上使用比较实用的，jps是基础，jstat是首要的定位工具，详细分析用jmap。

接下来就试试用这些工具进行 实践分析，调优，请看后篇。 
  