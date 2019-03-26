---
layout: post  
title:  "解决presto long prepareStatement-一次开源项目的填坑之旅"  
date:   2018-03-08 01:20  
categories: SQL  
permalink: /Priest/presto-long-prepareStatement-

---

## 前言  
项目中用到presto，一个facebook开源的分布式查询引擎，号称查询速度是hive的10倍以上，是spark的3倍以上，因此在唯品会内部做实时统计的很多计算任务都是交给presto去做。但是presto有一个非常不好的地方，就是它的用户文档几乎是空的，除了基本的部署文档、基本使用文档，内部的实现细节几乎一点都没，出了问题完全只能自己debug代码去看。于是就有了这次填坑的旅程。

## 问题描述
所在组负责的系统承担着一部分唯品会日常海量数据的计算，系统除了给内部用户使用之外，还会开放给供应商使用，所以在使用presto进行SQL查询的时候，一开始就是使用了PrepareStatement的方式，为了避免SQL注入。但是在遇到用户执行比较长的SQL（大几kb，十几kb）的时候，presto-jdbc就会收到服务端返回500的异常。 具体信息如下：  
```
Caused by: java.lang.RuntimeException: Error fetching next at http://XXXXX/v1/statement/20190320_031845_00001_5jdem/2 returned an invalid response: JsonResponse{statusCode=500, statusMessage=Server Error, headers={connection=[keep-alive], date=[Wed, 20 Mar 2019 03:18:45 GMT], server=[nginx/1.13.5], transfer-encoding=[chunked]}, hasValue=false} [Error: ] 
        at com.facebook.presto.jdbc.internal.client.StatementClientV1.requestFailedException(StatementClientV1.java:436) ~[presto-jdbc-0.215.jar:0.215] 
        at com.facebook.presto.jdbc.internal.client.StatementClientV1.advance(StatementClientV1.java:383) ~[presto-jdbc-0.215.jar:0.215] 
```

## 排查
#### 第一步：查看服务端日志
看到500 ，首先就想到是服务端的问题，直接上presto-server（cordinator）的服务器查看日志，出乎意料的是，查看presto官方的三个日志文件server.log(运行日志),laucher.log(启动日志),http-request.log(请求日志)，一点异常栈信息都没，而且log的配置级别是info。  
### 第二步：google
在github找到一个用户遇到一样的问题 **https://github.com/prestodb/presto/issues/10049** 看到Facebook的开发者在最后说了还在开发过程中，可以使用商业版本的presto（日了狗），出于穷 + 风险大（商业版本很久之前已经从presto fork出去了，不敢随便用）的考虑，我们还是坚持使用官方的presto。
### 第三步：远程debug presto server
鉴于presto没有在log打印出异常栈，唯一觉得可以想得通的就是presto的代码把Exception给catch 掉，但是没有记录起来，所以关键是要找到抛出异常的那段代码，把Exception 打印出来，获取最原始的异常信息。  
于是搭了一个presto的集群，jvm参数加了 **-Xdebug -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5009**远程debug参数，通过无数次的请求，debug到最终的异常信息。  
### 第四步：理解presto query请求过程
debug presto的过程，发现presto 是通过一个内部的rest框架 **https://github.com/airlift/airlift** 来实现查询请求，请求过程大致如下：  
> 1.客户端通过http client将query SQL请求到presto-server 的rest接口，对应的类是StatementResource#createQuery 进行创建查询请求  
> 2.presto 查询有别于常规的SQL查询，它是创建请求后获取queryId，后续再不断请求服务端来获取数据，见StatementResource#getQueryResults
> 出乎意料的是getQueryResults不是同步返回结果的，它采用了guava工具类实现异步返回结果，这给debug带来困难  
> 3.然后根据社区的一些人的反馈和源码的阅读，找到了一个关键的类OutboundMessageContext#854L，用于正常的query完成后close connection时触发，正如源码的注释所写：**Happens when the client closed connection before receiving the full response**，看了源码显示如果有异常，会catch并log，但是远程debug的时候服务端并没有记录这个异常，所以这个时候使用intellij的功能（ctrl + U），临时执行一段代码**e.printStackTrace()** 可以在服务端清晰看到这个异常的信息是 **Response Header too large**，然后再debug到response的header 存放着preparestatement的SQL，这个时候一切都清晰了，是presto把prepareStatement的SQL放到http header里面，由于rest框架的默认限制repsonse header是8kb，导致在SQL长的时候报错，之所以没有在log打印异常，因为这个是属于rest框架的配置，presto默认没有配置这块，导致log没有记录起来。  

## 解决
确认是response header too large的原因之后，开始着手看Facebook内部的rest框架 airlift，之前在社区有看到一些问题的时候有用户提到配置http-server.max-request-header-size，那么没什么意外这个应该就是airlift的配置，clone源码之后，全局搜索下，果然找到在类HttpServerConfig里面有配置，然后发现它是封装了jetty，相应地我搜索下jetty 关于repsonse header size的配置，果然有。但是airlift 没有这个配置项，所以最终的解决方式是  
> 1.clone airlift代码，修改HttpServerConfig.java，添加一个http-server.max-repsonse-header-size的初始化操作   

```
public DataSize getMaxResponseHeaderSize()
{
    return maxRequestHeaderSize;
}

@Config("http-server.max-response-header-size")
public HttpServerConfig setMaxResponseHeaderSize(DataSize maxRequestHeaderSize)
{
    this.maxRequestHeaderSize = maxRequestHeaderSize;
    return this;
}
```

> 2.在airlift 的jetty配置初始化类HttpServer加上   

```
if (config.getMaxResponseHeaderSize() != null) {
    baseHttpConfiguration.setResponseHeaderSize(toIntExact(config.getMaxResponseHeaderSize().toBytes()));
}
```

重新maven打包，替换掉部署的presto对应的jar包，重启，即可解决