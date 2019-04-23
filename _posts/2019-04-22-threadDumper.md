---
layout: post  
title:  "ThreadDumper协助定位线程阻塞问题"  
date:   2019-04-22 00:30  
categories: java  
permalink: /Priest/ThreadDumper

---

## 背景  
419大促值班的时候，遇到一个生产的OSP（VIP的RPC框架）的框架异常，意思大致是某个RPC服务接口出现方法级线程池爆满，而通过定位，发现该接口是定时任务每隔半个小时触发一次。初步判断是这个方法可能是某个地方组塞住了，然后每隔半个小时创建新线程执行该方法的时候，由于之前的线程都没有运行完，所以线程一直在阻塞着，导致该方法的线程池爆满。  
## 定位
### log
出问题之后，第一时间就是看log，从异常栈看到是RPC的某个方法，出现这个方法级线程池爆满。而日志里面让我耳目一新的是 **Thread dump by ThreadDumpper for rejected by method level business ** ，然后接下来的是一些异常栈信息，就是把当前存活的线程的线程栈打出来，把每个存活的线程都状态，线程栈信息全打出来，类似  
```java
"thread-dump-0" Id=13 WAITING
	at java.lang.Object.wait(Native Method)
	at java.util.concurrent.ForkJoinTask.externalAwaitDone(ForkJoinTask.java:334)
	at java.util.concurrent.ForkJoinTask.doInvoke(ForkJoinTask.java:405)
	at java.util.concurrent.ForkJoinTask.invoke(ForkJoinTask.java:734)
	at java.util.stream.ReduceOps$ReduceOp.evaluateParallel(ReduceOps.java:714)
	at java.util.stream.AbstractPipeline.evaluate(AbstractPipeline.java:233)
	at java.util.stream.ReferencePipeline.collect(ReferencePipeline.java:499)
	at com.kevin.vjtool.StaticWithLmdUtils.<clinit>(StaticWithLmdUtils.java:20)
	...
```

也就是在这个ThreadDumpper做的thread dump让我们能快速定位到生产的这个问题。为了说明下这个问题，我写了测试代码还原线上问题。  

### 还原线上问题
这个问题其实很简单，就是建立个线程池，然后开满线程去跑一个会引起线程阻塞的方法，等到线程池用满之后，再来一个线程就会触发reject抛异常，抛异常的时候再将当前存活的线程的线程栈打出来。  
#### 会引起阻塞/死锁的操作
**是的，这份代码会触发死锁，后面再解释**
```java
public class StaticWithLmdUtils {

    static {
        System.out.println("begin init");
        List<Integer> nums = Lists.newArrayList(1, 2, 3);
        List<Object> collect = nums.parallelStream().map(e -> e + 100).collect(Collectors.toList());
        System.out.println("end init");
    }

    public static void test() {
        System.out.println("hello");
    }
}
```
#### 模拟多个线程耗尽线程池
```java
public class ThreadDumpMain {

    public static void main(String[] args) {
        ThreadFactory push2CDNFactory = new ThreadFactoryBuilder().setNameFormat("thread-dump-%d").build();
        ExecutorService push2CDNPool = new ThreadPoolExecutor(72, 72, 0L, TimeUnit.MILLISECONDS,
                new LinkedBlockingQueue<>(1), push2CDNFactory, new ThreadPoolExecutor.AbortPolicy());
        try {
            for (int i = 0; i < 160; i++) {
                push2CDNPool.submit(new Thread(() -> {
                    StaticWithLmdUtils.test();
                }));
            }
        } catch (Exception e) {
        	 // com.vip.vjtools.vjkit.concurrent. ThreadDumpper  from vjtool
            ThreadDumpper dumpper = new ThreadDumpper(20, 1000 * 60 * 10);
            dumpper.tryThreadDump();
        }
    }
}
```
解释：采用一个固定大小的线程池，然后创建多个线程扔到线程池跑，到满了之后，触发vjtool的ThreadDumpper进行将当前存活的线程栈打出来，通过观察线程栈信息来定位问题。  

#### 运行结果
省略了一部分，把重要的抽取出来，实际上使用需要自己要仔细观察所有的线程栈  
```java

"thread-dump-41" Id=98 RUNNABLE
	at com.kevin.vjtool.ThreadDumpMain.lambda$main$0(ThreadDumpMain.java:24)
	at com.kevin.vjtool.ThreadDumpMain$$Lambda$1/75457651.run(Unknown Source)
	at java.lang.Thread.run(Thread.java:745)
	at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511)
	at java.util.concurrent.FutureTask.run$$$capture(FutureTask.java:266)
	at java.util.concurrent.FutureTask.run(FutureTask.java)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
	...

"ForkJoinPool.commonPool-worker-3" Id=84 WAITING
	at sun.misc.Unsafe.park(Native Method)
	at java.util.concurrent.ForkJoinPool.awaitWork(ForkJoinPool.java:1824)
	at java.util.concurrent.ForkJoinPool.runWorker(ForkJoinPool.java:1693)
	at java.util.concurrent.ForkJoinWorkerThread.run(ForkJoinWorkerThread.java:157)
	...
	
"thread-dump-0" Id=13 WAITING
	at java.lang.Object.wait(Native Method)
	at java.util.concurrent.ForkJoinTask.externalAwaitDone(ForkJoinTask.java:334)
	at java.util.concurrent.ForkJoinTask.doInvoke(ForkJoinTask.java:405)
	at java.util.concurrent.ForkJoinTask.invoke(ForkJoinTask.java:734)
	at java.util.stream.ReduceOps$ReduceOp.evaluateParallel(ReduceOps.java:714)
	at java.util.stream.AbstractPipeline.evaluate(AbstractPipeline.java:233)
	at java.util.stream.ReferencePipeline.collect(ReferencePipeline.java:499)
	at com.kevin.vjtool.StaticWithLmdUtils.<clinit>(StaticWithLmdUtils.java:20)
	...

"thread-dump-29" Id=71 RUNNABLE
	at com.kevin.vjtool.ThreadDumpMain.lambda$main$0(ThreadDumpMain.java:24)
	at com.kevin.vjtool.ThreadDumpMain$$Lambda$1/75457651.run(Unknown Source)
	at java.lang.Thread.run(Thread.java:745)
	at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511)
	at java.util.concurrent.FutureTask.run$$$capture(FutureTask.java:266)
	at java.util.concurrent.FutureTask.run(FutureTask.java)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
	...
```
#### 分析
这里我们看到一个非常奇怪的地方就是WAITING状态，名字Wiethread-dump-0线程的线程栈，在**StaticWithLmdUtils.java:20**，类的第20行的代码非常简单  
```java
List<Object> collect = nums.parallelStream().map(e -> e + 100).collect(Collectors.toList());
```
为什么这个会一直在WAITING，一个很普通的操作。反复运行几次还是一样的问题，于是上Stack Overflow查了下，发现是java的一个不算bug的bug，详情见 **https://stackoverflow.com/questions/34820066/why-does-parallel-stream-with-lambda-in-static-initializer-cause-a-deadlock** ，简单讲就是在静态代码块初始化的时候使用到parallelStream会引起deadlock，然后别的线程调用到该类的方法时就会阻塞住。深层次的原因，我们组几个小伙伴看了大半天官方的说明还是没了解，有兴趣的可以深入看看（个人理解为是parallelStream会发起多个线程并行执行与当前线程有资源竞争导致）。至于解决方案就比较简单了，改为使用strem，不用parallelStream带来的死锁。  

### ThreadDumper 
在这次问题的定位中，ThreadDumper发挥了极大的作用，特别是在线程池爆满这种异常情况发生时，将当时的线程快照记录起来，快速帮助定位问题。  
说下ThreadDumper使用  
ThreadDumper的源码中有个有参构造，第一个参数是打印出栈的层次，第二个是间隔时间，层次默认是8，为了避免过多的输出，我个人觉得在不是很损耗性能情况下，把层次打得多点，有利于查看源头。至于间隔时间**timeIntervalLimiter**，为了避免过度频繁打印带来JVM的停顿而作的保护，其实现是通过一个AtomicLong来记录上一次触发的时间。而当前存活的线程的线程栈是通过**Map<Thread, StackTraceElement[]> threads = Thread.getAllStackTraces();**来获取。  
