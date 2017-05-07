---
layout: post  
title:  "本地缓存的实现方案"  
date:   2017-05-07 17:36  
categories: 分布式缓存  
permalink: /Priest/guava-cache

---

## 背景
近期有个项目需要用到本地缓存，出了几种方案，那时候时间比较急，没有考虑很多，所以没有做的很好，后来发现了guava cache，是一个相当好用的现成的实现方案。所以特意去了解下，的确是比我们自己写的好。  
## 本地缓存和分布式缓存  
**本地缓存 **

> 1.效应速度非常快，因为没有网络开销 (优)  
> 2.不支持集群共享，与应用耦合一起  
> 3.在分布式情况下，本地缓存在每个应用都占用内存，比分布式缓存相对来说会有资源浪费  
> 4.缓存更新都每个应用都需要维护自己的本地缓存，不能集中式处理   


**分布式缓存 **  
最大的好处就是解耦合，多个应用可以共享  

## 实现本地缓存所需要的功能
1.清除策略
2.大小限制
3.更新缓存

## 本地缓存的实现方案  
1. guava cache
2. apache commone collection 的 LRUMap
3. 堆外内存

** LRUMap**  
LRUMap是apache 提供的一个带有lru特性的map，可以设置最大size，基本符合缓存的基本功能的要求，但是有个不好的地方，LRUMap本身是非线程安全的，所以需要自己手动处理线程安全的情况。  
```
Map lruMap = Collections.synchronizedMap(new LRUMap(1000));

```
**guava cache**  
不严格地讲，你可以理解guava cache就是带有缓存特性的ConcurrentHashMap,他是参考了ConcurrentHashMap的设计，然后在这个基础上添加了很多的缓存特性的功能，比如大小限制，更新处理，清除策略等。直接通过代码来理解guava cache 的基本使用：  
```java
package com.priest.cache.guava;

import com.google.common.cache.*;
import org.apache.commons.collections.map.LRUMap;
import org.junit.Assert;
import org.junit.Before;
import org.junit.Test;

import java.util.Collections;
import java.util.Map;
import java.util.Set;
import java.util.concurrent.Callable;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.TimeUnit;

/**
 * 详情 : guava 的本地缓存
 * <p>
 * 详细 :
 *
 * @author kevin 17/5/6
 */
public class GuavaLocalCache {

    private static LoadingCache<String, Integer> localCache;

    @Before
    public void initCache() {
        RemovalListener<String, Integer> removalListener = new RemovalListener<String, Integer>() {
            public void onRemoval(RemovalNotification<String, Integer> removal) {
                System.out.println(removal.getCause());
            }
        };

        localCache = CacheBuilder.newBuilder()
                //设置并发级别为8，并发级别是指可以同时写缓存的线程数
                .concurrencyLevel(1)
                // 设置最大
                .maximumSize(21)
                //缓存项在给定时间内没有被写访问（创建或覆盖），则回收
                .expireAfterWrite(6, TimeUnit.SECONDS)
                //删除的时候监听处理
                .removalListener(removalListener)
                //设置要统计缓存的命中率
                .recordStats()
                .build(
                        // 在本地找不到缓存 可以去哪里找
                        new CacheLoader<String, Integer>() {
                            public Integer load(String key) throws Exception {
                                System.out.println("找不到 ，就去加载");
                                return loadNewValue(key);
                            }
                        });
        for (int i = 0; i < 20; i++) {
            localCache.put(String.valueOf(i), initValue(i));
        }
    }

    private Integer loadNewValue(String key) {
        return 3000 + Integer.valueOf(key);
    }

    private Integer initValue(int key) {
        return 1000 + key;
    }

    @Test
    public void testLoadCache() throws ExecutionException {
        Integer value1 = localCache.get("2");
        Assert.assertTrue(value1.intValue() == initValue(2));
        Integer notInValue = localCache.get("21");
        Assert.assertTrue(notInValue.intValue() == loadNewValue("21"));
    }

    @Test
    public void testExpire() throws Exception {
        Integer value1 = localCache.get("2");
        Assert.assertTrue(value1.intValue() == initValue(2));
        Thread.sleep(6000L);
        Integer notInValue = localCache.get("3");
        Assert.assertTrue(notInValue.intValue() == loadNewValue("3"));
    }

    @Test
    public void testRemovalListener() throws Exception {
        Integer value1 = localCache.get("1");
        Assert.assertTrue(value1.intValue() == initValue(1));
        // 删除
        localCache.invalidate("1");
        Integer notInValue = localCache.get("1");
        Assert.assertTrue(notInValue.intValue() == loadNewValue("1"));
    }

    @Test
    public void testRefresh() throws Exception {
        Integer value1 = localCache.get("1");
        Assert.assertTrue(value1.intValue() == initValue(1));
        localCache.refresh("1");
        Integer notInValue = localCache.get("1");
        Assert.assertTrue(notInValue.intValue() == loadNewValue("1"));

    }

    /**
     * 默认采用LRU的清空策略
     * @throws Exception
     */
    @Test
    public void testLRU() throws Exception {
        localCache.get("0");
        localCache.put("22", 22);
        localCache.put("23", 22);
        localCache.put("24", 22);
        Set<String> keys = localCache.asMap().keySet();
        Assert.assertFalse(keys.contains("1"));
        Assert.assertFalse(keys.contains("2"));
    }

    /**
     * 相比初始化缓存多的时候直接指定获取方式，这种会更灵活些，每次都可以指定数据源
     * @throws Exception
     */
    @Test
    public void testCallableGet() throws Exception {
        Integer value = localCache.get("55", new Callable<Integer>() {
            public Integer call() throws Exception {
                Thread.sleep(3000);
                System.out.println("获取");
                return 9999;
            }
        });
        Assert.assertTrue(value == 9999);
        System.out.println("完成");
        Map lruMap = Collections.synchronizedMap(new LRUMap(1000));

    }

}

```
