---
layout: post  
title:  "maven-shade-plugin踩坑"  
date:   2016-12-27 23:16  
categories: 工具  
permalink: /Priest/maven-shade-plugin

---




## 背景
在用dubbo接入cat的时候，集成后放到服务器上一直报错，异常如下：  
 
```java
java.lang.RuntimeException: Unable to get component: interface org.unidal.initialization.Module.  
at org.unidal.initialization.DefaultModuleContext.lookup(DefaultModuleContext.java:98)  
at com.dianping.cat.Cat.initialize(Cat.java:112)  
at com.dianping.cat.Cat.initialize(Cat.java:107)   
      ······  
Caused by: org.codehaus.plexus.component.repository.exception.ComponentLookupException: Component descriptor cannot be found in the component repository  
role: org.unidal.initialization.Module  
roleHint: cat-client  
classRealm: none specified
```




但是在本地启动跑单元测时候是没问题的，同时，web系统接入cat也是一切正常。  

## 分析  
从异常上看，cat初始化的时候是没有加载组件Module，再从源码上分析org.unidal.initialization.Module，发现源头是一个plexus的组件，plexus是类似于spring的东西，类似IOC容器、AOP等功能也有，而cat使用plexus进行开发的，进而推测是部署到服务器的时候没有完成初始化plexus IOC容器。但是在本地跑单元测试的时候没有问题的，web系统接入cat也是没问题的，所以推测：  
**1.本地运行的时候的classpath 跟 在服务器运行的classpath不一致导致dubbo服务初始化cat失败**  
**2.web系统的启动方式与dubbo服务的启动方式的区别导致dubbo服务启动失败**   

## 验证  

### classpath的推测  
将在intellij idea跑的单元测试的classpath的jar包打包传到服务器，然后用 服务器启动dubbo的方式进行启动，也就是调用com.alibaba.dubbo.container.Main进行初始化。  
试验结果：成功启动。  
结论：**确定是classpath的jar包差异导致。**   
查看dubbo服务打包的方式，是采用了 maven-shade-plugin 插件打成一个jar包，shade插件主要做这几件事情：  

> 把依赖的class重新放到指定的包下；  
> 改写相关class的字节码，对应于重定义的包路径；  
> 把相关依赖的class打进一个jar包；  

此外shade还有一个很重要的功能就是Spring Framework的多个jar包中包含相同的文件spring.handlers和spring.schemas，shade提供AppendingTransformer来对文件内容追加合并，避免运行时会出现读取XML schema文件出错。  
但是，比较少人了解到shade插件有个坑就是：**如果第三方包中有反射相关的代码，则shade后可能出现不能正常工作,因为它在打包的时候忽略了用字符串写的类名或者包名，比如servlet.addServletWithMapping("org.mortbay.jetty.servlet.DefaultServlet",
    URIUtil.SLASH)。** 详情请看：http://stackoverflow.com/questions/8992025/transformer-for-maven-shade-plugin-to-deal-with-java-reflection     

### web项目与 dubbo服务启动方式的区别  
有了classpath的分析，基本确定，再拿web项目与dubbo服务的pom.xml去对比，发现wbe项目没有用到shade插件（也没必要用），所以基本确定是maven-shade-plugin影响到cat依赖的包，猜测应该是 plexus关联的jar包，包含了反射的代码，shade打包的时候忽略了导致运行失败。  

## 解决  
换一种方式打包。我的解决方案是： 采用maven-jar-plugin 和 maven-dependency-plugin 替代shade进行打包。代码如下：  

``` xml
<plugin>  
      <groupId>org.apache.maven.plugins</groupId>  
      <artifactId>maven-jar-plugin</artifactId>  
       <version>2.6</version>  
      <configuration>  
          <archive>  
              <manifest>  
                  <addClasspath>true</addClasspath>
                  <classpathPrefix></classpathPrefix>
                  <mainClass>com.alibaba.dubbo.container.Main</mainClass>  
              </manifest>  
          </archive>  
      </configuration>  
  </plugin>  
  <plugin>  
      <groupId>org.apache.maven.plugins</groupId>  
      <artifactId>maven-dependency-plugin</artifactId>  
      <version>2.10</version>  
      <executions>  
          <execution>  
              <id>copy</id>  
              <phase>install</phase>  
              <goals>  
                  <goal>copy-dependencies</goal>  
              </goals>  
              <configuration>  
                  <outputDirectory>  
                      ${project.build.directory}  
                  </outputDirectory>  
              </configuration>  
          </execution>  
      </executions>  
  </plugin>
```

原理是将依赖的jar存放到跟跟当前dubbo服务打成的jar包放在同一个目录，shell脚本启动的时候将该目录引进classpath即可。总体上只需要修改pom.xml，算是比较低成本的解决方案。  
