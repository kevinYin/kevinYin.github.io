---
layout: post  
title:  "spring源码学习--IOC容器初始化代码"  
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

### 定位BeanDefinition的Resource  
从整体上看，这个环节包括设置Resource的文件路径，创建一个IOC容器，创建ResourceLoader对文件进行读取。  
从FileSystemXmlApplicationContext的构造器代码开始，查看到      

```
public FileSystemXmlApplicationContext(String[] configLocations, boolean refresh, ApplicationContext parent)
			throws BeansException {
		super(parent);
		setConfigLocations(configLocations);
		if (refresh) {
			refresh();
		}
	}
```
setConfigLocations明显是初始化Resource的文件路径，后面看到configLocations会被转化放进Resource[] 数组，然后逐个Resource进行解析。然后是关键**refresh()**进行一系列的复杂的功能的初始化工作，其中就包括bean的载入、注册等功能。   
 
```
	// Prepare this context for refreshing.
	prepareRefresh();
	// Tell the subclass to refresh the internal bean factory.
	
	ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

	// Prepare the bean factory for use in this context.
	prepareBeanFactory(beanFactory);

	try {
		// Allows post-processing of the bean factory in context subclasses.
		postProcessBeanFactory(beanFactory);

		// Invoke factory processors registered as beans in the context.
		invokeBeanFactoryPostProcessors(beanFactory);

		// Register bean processors that intercept bean creation.
		registerBeanPostProcessors(beanFactory);

		// Initialize message source for this context.
		initMessageSource();

		// Initialize event multicaster for this context.
		initApplicationEventMulticaster();

		// Initialize other special beans in specific context subclasses.
		onRefresh();

		// Check for listener beans and register them.
		registerListeners();

		// Instantiate all remaining (non-lazy-init) singletons.
		finishBeanFactoryInitialization(beanFactory);

		// Last step: publish corresponding event.
		finishRefresh();
	}
```    

而跟IOC容器初始化核心相关的则是 **ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();** 这个方法字面上是获取一个BeanFacotry，其实就是初始化IOC容器，包括Resource的读取解析，BeanDefinition的载入，BeanDefinition注册进BeanFactory。进入该方法，可以看到refreshBeanFactory(); 是核心的构建BeanFactory的方法，    
  
```
			DefaultListableBeanFactory beanFactory = createBeanFactory();
			beanFactory.setSerializationId(getId());
			customizeBeanFactory(beanFactory);
			loadBeanDefinitions(beanFactory);
```    

其中看到了第一个方法就是创建一个DefaultListableBeanFactory，这个就是spring默认的BeanFactory；然后最后一个**loadBeanDefinitions(beanFactory);**就是完成ReSource的解析 、bean的载入、以及注册进beanFactory. 
解析来进入loadBeanDefinitions方法，   

```
// Create a new XmlBeanDefinitionReader for the given BeanFactory.
		XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

		// Configure the bean definition reader with this context's
		// resource loading environment.
		beanDefinitionReader.setEnvironment(this.getEnvironment());
		beanDefinitionReader.setResourceLoader(this);
		beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

		// Allow a subclass to provide custom initialization of the reader,
		// then proceed with actually loading the bean definitions.
		initBeanDefinitionReader(beanDefinitionReader);
		loadBeanDefinitions(beanDefinitionReader);
```   

可以看到主要是做了创建一个XmlBeanDefinitionReader用于解析Resource 和 loadBeanDefinitions，进入loadBeanDefinitions可以看到   

```
Resource[] configResources = getConfigResources();
		if (configResources != null) {
			reader.loadBeanDefinitions(configResources);
		}
		String[] configLocations = getConfigLocations();
		if (configLocations != null) {
			reader.loadBeanDefinitions(configLocations);
		}
```   

可以看到这个环节的最终目的，定位配置的Resource文件地址。getConfigLocations()其实就是获取初始化FileSystemXmlApplicationContext的时候，放入 configLocations的文件地址“applicationContext.xml”。    

### 载入Resource描述的BeanDefinition  
上一步已经可以完成定位了bean的解析文件地址，这个环节就是文件描述的bean进行解析封装成BeanDefinition载入，这也是IOC容器初始化过程中最为复杂的一步。大体上来看解析Resource，并将bean封装成BeanDefinition的过程大致如下：   
 一步步跟踪reader.loadBeanDefinitions(configLocations);到 doLoadBeanDefinitions(InputSource inputSource, Resource resource)，发现是创建了一个Document对象进行协助处理的，
 ```
 Document doc = doLoadDocument(inputSource, resource);
 return registerBeanDefinitions(doc, resource);
 ```
 逐步进入doRegisterBeanDefinitions(Element root)，
 ```
preProcessXml(root);
parseBeanDefinitions(root, this.delegate);
postProcessXml(root);
 ```
parseBeanDefinitions(root, this.delegate);是最终的处理,该方法对bean的解析处理是使用了BeanDefinitionParserDelegate进行解析，
```
parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate);
```
到最后的调用 processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate)；   

```
BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
if (bdHolder != null) {
	bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
	try {
		// Register the final decorated instance.
		BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
	}
```    

delegate是将bean解析转化成一个BeanDefinitionHolder（里面包含有BeanDefinition 和 beanName）,然后解析来的就是将BeanDefinitionHolder注册进BeanFactory。   

### 将BeanDefinition注册到IOC容器   
IOC容器初始化的最后一步，将bean注册进IOC容器，简单的将就是将bean 以beanName为key，对应的BeanDefinition为value put进一个HashMap里面。代码如下：  

```
public static void registerBeanDefinition(
		BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
		throws BeanDefinitionStoreException {

	// Register bean definition under primary name.
	String beanName = definitionHolder.getBeanName();
	registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

	// Register aliases for bean name, if any.
	String[] aliases = definitionHolder.getAliases();
	if (aliases != null) {
		for (String alias : aliases) {
			registry.registerAlias(beanName, alias);
		}
	}
}
```
可以看到是通过BeanDefinitionHolder获取BeanDefinition，然后调用DefaultListableBeanFactory的registerBeanDefinition(String beanName, BeanDefinition beanDefinition)，将BeanDefinition放进DefaultListableBeanFactory 定义的beanDefinitionMap里面
```
this.beanDefinitionMap.put(beanName, beanDefinition);
```

### 总结    
IOC容器的初始化的代码流程大致如此，当然很多的细节没有一一列出来，读源码感觉收获更多的是spring整体大的实现思路以及设计是如何的，在读的过程中也感觉到源码的代码风格比较值得学习（比如核心的refresh()方法，风格跟厂里架构师一直推荐的风格类似，让人看起来思路非常的清晰）。
