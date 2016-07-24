---
layout: post  
title:  "spring源码学习--IOC容器初始化代码"  
date:   2016-07-23 12:16  
categories: spring  
permalink: /Priest/spring-ioc-code  

---

**前言：**了解了IOC容器的初始化过程后，接下来就逐个环节了解和学习他的具体实现的代码。我采用一个单元测试例子来查看spring IOC容器的初始化过程。   
``
@Test
    public void testIOC() {
        FileSystemXmlApplicationContext context = new FileSystemXmlApplicationContext("applicationContext.xml");
        ExpenseDetailService service = (ExpenseDetailService) context.getBean("expenseDetailService");
        assert service != null;
    }
``

### 定位BeanDefinition的Resource  
从整体上看，这个环节包括设置Resource的文件路径，创建一个IOC容器，创建ResourceLoader对文件进行读取。  
从FileSystemXmlApplicationContext的构造器代码开始，查看到      

``
public FileSystemXmlApplicationContext(String[] configLocations, boolean refresh, ApplicationContext parent)
			throws BeansException {
		super(parent);
		setConfigLocations(configLocations);
		if (refresh) {
			refresh();
		}
	}
``
setConfigLocations明显是初始化Resource的文件路径，后面看到configLocations会被转化放进Resource[] 数组，然后逐个Resource进行解析。然后是关键**refresh()**进行一系列的复杂的功能的初始化工作，其中就包括bean的载入、注册等功能。    

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

而跟IOC容器初始化核心相关的则是 **ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();** 这个方法字面上是获取一个BeanFacotry，其实就是初始化IOC容器，包括Resource的读取解析，BeanDefinition的载入，BeanDefinition注册进BeanFactory。进入该方法，可以看到refreshBeanFactory(); 是核心的构建BeanFactory的方法，   
``
			DefaultListableBeanFactory beanFactory = createBeanFactory();
			beanFactory.setSerializationId(getId());
			customizeBeanFactory(beanFactory);
			loadBeanDefinitions(beanFactory);
``	
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
上一步已经可以完成定位了bean的解析文件地址，这个环节就是文件描述的bean进行解析封装成BeanDefinition载入，这也是IOC容器初始化过程中最为复杂的一步。
### 将BeanDefinition注册到IOC容器  
IOC容器初始化的最后一步，将bean注册进IOC容器，简单的将就是将bean 以beanName为key，对应的BeanDefinition为value put进一个HashMap里面。
