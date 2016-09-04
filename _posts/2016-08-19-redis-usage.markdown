---
layout: post  
title:  "redis"  
date:   2016-07-25 01:16  
categories: spring  
permalink: /Priest/spring-ioc-code  

---

**前言：**了解了IOC容器的初始化过程后，接下来就逐个环节了解和学习他的具体实现的代码。我采用一个单元测试例子来查看spring IOC容器的初始化过程。   
```
@Test
    public void testIOC() {
        FileSystemXmlApplicationContext context = new FileSystemXmlApplicationContext("applicationContext.xml");
        ExpenseDetailService service = (ExpenseDetailService) context.getBean("expenseDetailService");
        assert service != null;
    }
```
