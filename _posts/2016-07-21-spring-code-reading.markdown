---
layout: post  
title:  "spring源码学习--IOC容器初始化"  
date:   2016-07-20 00:16  
categories: spring  
permalink: /Priest/spring-ioc  

---

**前言：**spring源码是我觉得对于java开发来讲最值得去阅读的框架代码，不仅是因为它强大，更吸引我的是它整体的设计思路非常优秀，也是因此，厂里架构师非常推荐我们去读读源码。
 

## IOC容器的相关实现   

IOC容器最底层的是一个BeanFactory，它本身是一个接口，然后有各种实现方案，从大的方向看，它是分了2种层次实现。一种是实现BeanFacoty接口，这是IOC容器最基本功能的实现，具体实现比如XmlBeanFactory，DefaulListableFactory；另一种是ApplicationContext应用上下文的高级形态的实现，具体实现比如ClassPathApplicationContext，WebApplicationContext，而实际上，第二种实现是在第一种方式的基础上进行继承，并且拓展了一些高级的方法。

## IOC容器的初始化过程   

**IOC容器初始化过程，主要是包括3个步骤，一是定位BeanDefinition的Resource；二是载入Resource描述的BeanDefinition；三是将BeanDefinition注册到IOC容器。**

### 1.1定位BeanDefinition的Resource   

通俗地讲，BeanDefinition就是IOC容器里面的一个个bean，而Resource其实就是描述bean的配置文件，比如常见的就是applicationContext.xml文件。那么在spring里面是如何实现的呢。看个例子：   
``  
 ApplicationContext context = new FileSystemXmlApplicationContext("applicationContext.xml");
``
上面有提到FileSystemXmlApplicationContext是IOC的一种高级形态，看下它的父类结构：  
![UML图片](http://7xrmyq.com1.z0.glb.clouddn.com/FileSystemXmlApplicationContext.png)   
所以可以看到它是由 BeanFacotry和ResourceLoader一路继承下来的，很明显BeanFacotry其实用来装bean的，而ResourceLoader，就是读取bean的相关Resource，然后将读取的bean载入beanFacotry。   
了解到整体的结构后，开始进入到具体的代码实现上。从整体来讲，具体代码实现有这么几个步骤：   

> 1.初始化bean的Resource的具体地址（文件）  
> 2.创建一个BeanFactory  
> 3.获取ResourceLoader进行读取ReSource文件

### 1.2载入BeanDefinition   
这个环节主要是将配置文件进行解析，将里面描述的bean封装成BeanDefinition. 配置文件的解析是通过一个文档解析器BeanDefinitionDocumentReader进行解析。在这个环节也可简单分为   

> 1. 创建一个BeanDefinitionDocumentReader
> 2. BeanDefinitionDocumentReader对Resource进行解析，将bean封装成BeanDefinition

### 1.3 BeanDefinition注册进BeanFactory   
这个环节主要是将BeanDefinition注册进BeanFactory，它的实现原理是将解析出来的bean封装成BeanDefinition放入一个HashMap里面。IOC容器主要默认的BeanFacotry是 DefaulListableFactory，所以这个hashMap就是在它里面。   

## 总结   
spring大致的架构 以及 核心的IOC容器的初始化过程，有了大致的了解后，接下来就逐个环节的代码进行细致的了解。
