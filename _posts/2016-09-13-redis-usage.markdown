---
layout: post  
title:  "redis实践"  
date:   2016-09-13 23:16  
categories: 分布式缓存  
permalink: /Priest/redis-usage 

---

说说我厂各大系统中redis的实践.  
## 消息队列  

厂里前端的汽配商城有询价 报价业务,客户发布的询价经客服以及供应商报价之后,将会在汽配商城前端给客户推送报价消息.这里我们采用了的方案是
redis + websocket的方案.简单的来讲就是使用redis的 LPUSH 和 BRPOP 实现消息队列的发布和消费,具体是采用了jedis实现.示例代码如下:

```
@Before
    public void init() {
        jedis = new Jedis("localhost");
    }

    /**
     * 可以使用redis LPUSH 和 RPOP 来实现队列,但是RPOP在没有任务时,消费者都会调用一次RPOP去查看是否有新任务
     * 所以采用了BRPOP,BRPOP和RPOP的区别在于队列没有消息时,BRPOP会一直阻塞住连接,直到有新元素进来
     */
    @Test
    public void testQueue() {
        Long mQueue = jedis.lpush("mQueue", "123", "234");
        while (true) {
            List<String> messages = jedis.brpop(30, "mQueue");
            if (CollectionUtils.isNotEmpty(messages)) {
                messages.forEach(e -> System.out.println(e));
            }
        }
    }
```
厂里用redis实现队列还是比较简单,目前就商城用到redis实现的消息队列,如果包含多种类型的消息队列,那么久需要建立优先级队列,具体还是用到
LPUSH 和 BRPOP实现,其中是使用BRPOP可以传递多个参数的方式进行实现.实例代码如下:  

```
    /**
     * redis实现优先队列
     * BRPOP可以有多个参数,可以一次获取多个队列的数据,所以可以借此来实现优先队列
     *
     */
    @Test
    public void testFirstQueue() {
        Long mQueue1 = jedis.lpush("mQueue1", "111", "1111");
        Long mQueue2 = jedis.lpush("mQueue2", "222", "2222");
        while (true) {
            List<String> messages = jedis.brpop(30, "mQueue3", "mQueue2", "mQueue1");
            if (CollectionUtils.isNotEmpty(messages)) {
                messages.forEach(e -> System.out.println(e));
            }
        }
    }
```

## 定时任务  

我们厂是分布式系统,做定时任务发布时,采用了spring自带的定时任务注解@Scheduled去实现,然后在任务里面判断设定的服务器IP进行判断,
指定某一台服务器执行该定时任务,避免多台一起执行同一个定时任务.这种做法不好的地方就是一旦服务器地址改变后就又要改变配置文件.
后面跟隔壁组的技术经理聊天的时候说到,可以使用redis实现分布式锁的方式来替代配置IP的实现方案.   
具体的实现原理是:

