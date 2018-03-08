---
layout: post  
title:  "mysql死锁还原和分析"  
date:   2018-03-08 01:20  
categories: mysql  
permalink: /Priest/mysql-deadLock

---

## 前言  
mysql的死锁是生产环境相对常遇到，在开发、测试环境却很难重现的一个问题，以前线上出现死锁的时候，运维都是直接暴力重启mysql来解决。所以很多时候开发很难找到到底这个死锁是怎么发生的，是哪些SQL语句导致死锁的，今天花了点时间测试下，大致给一个思路来还原业务代码如何导致死锁的发生以及发生死锁后，怎么去定位是哪几条SQL导致的。

##什么是死锁
当多个事务在执行过程，形成相互等待对方释放锁资源的情景。  
举个例子：  
```
      事务1                 事务2
    update data01       
                          updatedata02
    update data02    
                          update data01
```
这样执行过程就会形成相互等待对方释放锁资源的情况，就造成了死锁。  

##还原死锁如何发生
为了还原死锁是如何发生的，我尝试在java代码里还原，  
原理大致如下：  
> 一个事务里执行2次更新操作，然后将2次更新操作间隔开来，比如隔5S  
> 同时开2个线程，一起访问这个方法，根据特定参数 调整2次更新的间隔时间，来实现上面的情况

贴下代码：  
```
    @Test
    public void test() throws InterruptedException {
         RecommendFeed feed1 = RecommendFeed.builder().recommendId(1) .plateId("123").build();

         RecommendFeed feed2 = RecommendFeed.builder().recommendId(3).plateId("123").build();

         new Thread(new DeadLockTask(feed1, feed2 ,recommendFeedService)).start();
         new Thread(new DeadLockTask(feed2, feed1 ,recommendFeedService)).start();

        Thread.sleep(30000L);
    }

    class DeadLockTask implements Runnable {
            private RecommendFeed k1;
            private RecommendFeed k2;
            private RecommendFeedService recommendFeedService;

            public DeadLockTask(RecommendFeed k1, RecommendFeed k2, RecommendFeedService recommendFeedService) {
                this.k1 = k1;
                this.k2 = k2;
                this.recommendFeedService = recommendFeedService;
            }

            @Override
            public void run() {
                try {
                    recommendFeedService.testDeadLock(k1, k2);
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }

```
事务操作方法  
```
    @Transactional(readOnly = false,rollbackFor = Exception.class)
    public void testDeadLock(RecommendFeed f1, RecommendFeed f2) throws Exception {
        updateFeed(f1);
        if (f1.getRecommendId() == 1) {
            System.out.println("线程1更新第一条");
            Thread.sleep(5000L);
        } else {
            System.out.println("线程2更新第一条");
            Thread.sleep(8000L);
        }
        updateFeed(f2);
        Thread.sleep(5000L);
    }
```
执行的结果如下  
```
Cause: com.mysql.jdbc.exceptions.jdbc4.MySQLTransactionRollbackException: Deadlock found when trying to get lock; try restarting transaction
; SQL []; Deadlock found when trying to get lock; try restarting transaction; nested exception is com.mysql.jdbc.exceptions.jdbc4.MySQLTransactionRollbackException: Deadlock found when trying to get lock; try restarting transaction
	at org.springframework.jdbc.support.SQLErrorCodeSQLExceptionTranslator.doTranslate(SQLErrorCodeSQLExceptionTranslator.java:263)

```
所以大致的逻辑能够还原mysql的死锁的形成原因，接下来是定位具体的SQL  

##定位
登录mysql的服务器，查看下最新的死锁记录  
```
mysql> show engine innodb status \G;
```
可以看到一大堆的信息 ，但是聚焦在 **LATEST DETECTED DEADLOCK**，即最新的死锁日志，即可看出来是哪2个事务导致的死锁  
**事务1**
```
*** (1) TRANSACTION:
TRANSACTION 69997717, ACTIVE 7.870 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 3 lock struct(s), heap size 360, 2 row lock(s), undo log entries 1
LOCK BLOCKING MySQL thread id: 872488799 block 772754316
MySQL thread id 772754316, OS thread handle 0x7fd4ab4b2700, query id 2797181861 183.60.191.100 priority_high updating
update t_recommend_feed set title='11',link='222',pics='1',publish_time='2018-03-08 17:17:51',recommend_status=1,update_time='2018-03-08 17:17:57' where recommend_id=3

     ===等待锁 space id是5755===
(1) WAITING FOR THIS LOCK TO BE GRANTED:
    RECORD LOCKS space id 5755
```
**事务2**
```
(2) TRANSACTION:
TRANSACTION 69997714, ACTIVE 8.093 sec starting index read
mysql tables in use 1, locked 1
3 lock struct(s), heap size 360, 2 row lock(s), undo log entries 1
MySQL thread id 872488799, OS thread handle 0x7fd493871700, query id 2797181928 183.60.191.100 priority_high updating
update t_recommend_feed set title='11',link='222',pics='1',publish_time='2018-03-08 17:17:51',recommend_status=1,update_time='2018-03-08 17:18:00' where recommend_id=1

===拥有这个锁====
(2) HOLDS THE LOCK(S):
RECORD LOCKS space id 5755 page
```
大致可以判断出这两个事务在这两条记录的更新上形成了死锁。找到对应的代码进行优化，在业务上将SQL执行顺序进行调整，这个也是比较常见的解决方式。
