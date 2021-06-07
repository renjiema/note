# ApplicationContext容器初始化

IOC容器的初始化包括**`BeanDefinition`**的**`Resource 定位、加载和注册`**过程。`ApplicationContext`系列容器也许是我们最熟悉的，因为Web项目中使用的XmlWebApplicationContext就属于这个继承体系，还有ClasspathXmlApplicationContext等，其继承体系如下图所示：

![ClasspathXmlApplicationContext继承体系](https://raw.githubusercontent.com/renjiema/images/main/blogs/20210607195558.png)

`ApplicationContext`允许上下文嵌套，通过保持父上下文可以维持一个上下文体系。

对于Bean的查找可以在这个上下文体系中发生，首先检查当前上下文，其次是父上下文，逐级向上，这样为不同的Spring应用提供了一个共享的Bean定义环境。

**1、寻找入口**

这里使用`ClassPathXmlApplicationContext`，通过main()方法启动:

```java
ApplicationContext app = new ClassPathXmlApplicationContext("application.xml"); 
```

先看其构造函数的调用：

```java
public ClassPathXmlApplicationContext(String configLocation) throws BeansException { 
    this(new String[]{configLocation}, true, (ApplicationContext)null); }
```

其实际调用的构造函数为：

```java
public ClassPathXmlApplicationContext(
		String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
		throws BeansException {
	super(parent);
	setConfigLocations(configLocations);
	if (refresh) {
		refresh();
	}
}
```

其他实现类如：`AnnotationConfigApplicationContext`、`FileSystemXmlApplicationContext`、`XmlWebApplicationContext`等都继承自父容器`AbstractApplicationContext`主要用到了装饰器模式和策略模式，最终都是调用`refresh()`方法。

**2、获得配置路径**

通过分析`ClassPathXmlApplicationContext`的源代码可以知道，在创建`ClassPathXmlApplicationContext`容器时，构造方法做以下两项重要工作：

首先，调用父类容器的构造方法(`super(parent)`)为容器设置Bean的ResourceLoader。

然后，再调用父类`AbstractRefreshableConfigApplicationContext`的`setConfigLocations(configLocations)`方法设置Bean配置信息的定位路径。

通过追踪ClassPathXmlApplicationContext的继承体系，发现其父类的父类AbstractApplicationContext中初始化IOC容器所做的主要源码如下：

```java
public abstract class AbstractApplicationContext extends DefaultResourceLoader
		implements ConfigurableApplicationContext {
    static {
		// Eagerly load the ContextClosedEvent class to avoid weird classloader issues
		// on application shutdown in WebLogic 8.1. (Reported by Dustin Woods.)
		ContextClosedEvent.class.getName();
	}
    public AbstractApplicationContext() {
		this.resourcePatternResolver = getResourcePatternResolver();
	}
    public AbstractApplicationContext(@Nullable ApplicationContext parent) {
		this();
		setParent(parent);
	}
    //获取一个 Spring Source 的加载器用于读入 Spring Bean 配置信息
	protected ResourcePatternResolver getResourcePatternResolver() {
		//AbstractApplicationContext 继承 DefaultResourceLoader，因此也是一个资源加载器
		//Spring 资源加载器，其 getResource(String location)方法用于载入资源
		return new PathMatchingResourcePatternResolver(this);
	}
}
```

`AbstractApplicationContext`的默认构造方法中有调用`PathMatchingResourcePatternResolver`的构造方法创建Spring资源加载器：

```java
public PathMatchingResourcePatternResolver(ResourceLoader resourceLoader) { 
    Assert.notNull(resourceLoader, "ResourceLoader must not be null"); 
    //设置 Spring 的资源加载器 
    this.resourceLoader = resourceLoader; 
}
```

在设置容器的资源加载器之后，接下来`ClassPathXmlApplicationContext`执行`setConfigLocations()`方法通过调用其父类`AbstractRefreshableConfigApplicationContext`的方法进行对Bean配置信息的定位，该方法的源码如下：

```java
public void setConfigLocation(String location) {
	//多个资源文件路径之间用” ,; \t\n”分隔，解析成数组形式
	setConfigLocations(StringUtils.tokenizeToStringArray(location, CONFIG_LOCATION_DELIMITERS));
}

public void setConfigLocations(@Nullable String... locations) {
	if (locations != null) {
		Assert.noNullElements(locations, "Config locations must not be null");
		this.configLocations = new String[locations.length];
		for (int i = 0; i < locations.length; i++) {
			// resolvePath 将字符串解析为路径
			this.configLocations[i] = resolvePath(locations[i]).trim();
		}
	}
	else {
		this.configLocations = null;
	}
}
```

通过上面的源码看出，使用一个字符串来配置多个 Spring Bean 配置信息，也可以使用字符串数组： 

```java
ClassPathResource res = new ClassPathResource("a.xml,b.xml"); 
```

多个资源文件路径之间可以是用” , ; \t\n”等分隔。 

```java
ClassPathResource res =new ClassPathResource(new String[]{"a.xml","b.xml"}); 
```

至此，Spring IOC 容器在初始化时将配置的 Bean 配置信息定位为 Spring 封装的Resource。

**3、开始启动** 

Spring IOC 容器对 Bean 配置资源的载入是从`refresh()`函数开始的，`refresh()`是一个模板方法，规定了IOC容器的启动流程 ，有些逻辑要交给其子类去实现。

它对Bean配置资源进行载入`ClassPathXmlApplicationContext`通过调用其父类`AbstractApplicationContext`的`refresh()`函数启动整个IOC容器对Bean定义的载入过程，现在我们来详细看看`refresh()`中的逻辑处理：

```java
@Override
public void refresh() throws BeansException, IllegalStateException {
	synchronized (this.startupShutdownMonitor) {
		// Prepare this context for refreshing.
		//1、调用容器prepareRefresh()方法，获取容器的当时时间，同时给容器设置同步标识
		prepareRefresh();
		// Tell the subclass to refresh the internal bean factory.
		//告诉子类启动refreshBeanFactory()方法，Bean定义资源文件的载入从子类的refreshBeanFactory()方法启动
		ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
		// Prepare the bean factory for use in this context.
		//为 BeanFactory 配置容器特性，例如类加载器、事件处理器等
		prepareBeanFactory(beanFactory);
		try {
			// Allows post-processing of the bean factory in context subclasses.
			//4、为容器的某些子类指定特殊的 BeanPost 事件处理器
			postProcessBeanFactory(beanFactory);
			// Invoke factory processors registered as beans in the context.
			//5、调用所有注册的 BeanFactoryPostProcessor 的 Bean
			invokeBeanFactoryPostProcessors(beanFactory);
			// Register bean processors that intercept bean creation.
			//6、为 BeanFactory 注册 BeanPost 事件处理器.
			registerBeanPostProcessors(beanFactory);
			// Initialize message source for this context.
			//7、初始化信息源，和国际化相关.
			initMessageSource();
			// Initialize event multicaster for this context.
			//8、初始化容器事件传播器.
			initApplicationEventMulticaster();
			// Initialize other special beans in specific context subclasses.
			//9、调用子类的某些特殊 Bean 初始化方法
			onRefresh();
			// Check for listener beans and register them.
			//10、为事件传播器注册事件监听器.
			registerListeners();
			// Instantiate all remaining (non-lazy-init) singletons.
			//11、初始化所有剩余的单例 Bean
			finishBeanFactoryInitialization(beanFactory);
			// Last step: publish corresponding event.
			//12、初始化容器的生命周期事件处理器，并发布容器的生命周期事件
			finishRefresh();
		}
		catch (BeansException ex) {
			if (logger.isWarnEnabled()) {
				logger.warn("Exception encountered during context initialization - " +
						"cancelling refresh attempt: " + ex);
			}
			// Destroy already created singletons to avoid dangling resources.
			destroyBeans();
			// Reset 'active' flag.
			//取消 refresh 操作，重置容器的同步标识.
			cancelRefresh(ex);
			// Propagate exception to caller.
			throw ex;
		}
		finally {
			// Reset common introspection caches in Spring's core, since we
			// might not ever need metadata for singleton beans anymore...
			//重置公共缓存
			resetCommonCaches();
		}
	}
}
```

`refresh()`方法主要为IOC容器Bean的生命周期管理提供条件，Spring IOC容器载入Bean配置信息从其子类容器的refreshBeanFactory()方法启动。

所以整个refresh()中`ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();`

后的代码都是注册容器的信息源和生命周期事件，我们前面说的载入就是从这句代码开始启动。

`refresh()`方法的主要作用：在创建IOC容器前，如果已经有容器存在，则需要把已有的容器销毁和关闭，保证在`refresh()`之后使用的是新建立起来的IOC容器。

类似重启IOC容器，在新建立好的容器中对容器进行初始化，对Bean配置资源进行载入。

**4、创建容器** 

`obtainFreshBeanFactory()`方法调用子类容器的 `refreshBeanFactory()`方法，启动容器载入 Bean 配置信息的过程，代码如下： 

```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
	//使用了委派设计模式，父类定义了抽象的refreshBeanFactory()方法，调用子类容器的refreshBeanFactory()实现
	refreshBeanFactory();
	return getBeanFactory();
}
```

`AbstractApplicationContext`类中只抽象定义了`refreshBeanFactory()`方法，容器真正调用的是其子类 `AbstractRefreshableApplicationContext` 实现的 `refreshBeanFactory()`方法，源码如下:

```java
@Override
protected final void refreshBeanFactory() throws BeansException {
	//如果已经有容器，销毁容器中的 bean，关闭容器
	if (hasBeanFactory()) {
		destroyBeans();
		closeBeanFactory();
	}
	try {
		//创建beanFactory容器
		DefaultListableBeanFactory beanFactory = createBeanFactory();
		beanFactory.setSerializationId(getId());
		//对 IOC 容器进行定制化，如设置启动参数，开启注解的自动装配等
		customizeBeanFactory(beanFactory);
		//调用加载BeanDefinition方法，又使用一个委派模式，在当前类中只定义了抽象的loadBeanDefinitions方法
		loadBeanDefinitions(beanFactory);
		//赋值IOC 容器
		this.beanFactory = beanFactory;
	}
	catch (IOException ex) {
		throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
	}
}
```

首先判断`BeanFactory`是否存在，如果存在则销毁beans并关闭beanFactory，接着创建 `DefaultListableBeanFactory`，并调用`loadBeanDefinitions(beanFactory)`装载BeanDefinition。

**5、载入配置路径** 

`AbstractRefreshableApplicationContext`中只定义了抽象的loadBeanDefinitions 方法，容器真正调用的是其子类 `AbstractXmlApplicationContext`对该方法的实现，`AbstractXmlApplicationContext`的主要源码如下： 

```java
public abstract class AbstractXmlApplicationContext extends AbstractRefreshableConfigApplicationContext { 
    @Override
    protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException,         IOException {
	    // Create a new XmlBeanDefinitionReader for the given BeanFactory.
	    //创建 XmlBeanDefinitionReader，并通过回调设置到BeanFactory中，容器使用该Reader读取Bean配置资源
	    XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);
	    // Configure the bean definition reader with this context's
	    // resource loading environment.
        //为Bean Reader设置Spring资源加载器，AbstractXmlApplicationContext的父类继承了DefaultResourceLoader
	    beanDefinitionReader.setEnvironment(this.getEnvironment());
	    beanDefinitionReader.setResourceLoader(this);
	    //为 Bean Reader设置 SAX xml 解析器
	    beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));
	    // Allow a subclass to provide custom initialization of the reader,
	    // then proceed with actually loading the bean definitions.
	    //当 Bean Reader读取 Bean 定义的 Xml 资源文件时，启用 Xml 的校验机制
	    initBeanDefinitionReader(beanDefinitionReader);
	    //Bean Reader真正实现加载的方法
	    loadBeanDefinitions(beanDefinitionReader);
    }
    protected void initBeanDefinitionReader(XmlBeanDefinitionReader reader) {
		reader.setValidating(this.validating);
	}
    //加载Bean配置资源
    protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
		//获取 Bean 配置资源的定位，使用委托模式
		Resource[] configResources = getConfigResources();
		if (configResources != null) {
			//Xml BeanReader调用其父类 AbstractBeanDefinitionReader 读取定位的 Bean 配置资源
			reader.loadBeanDefinitions(configResources);
		}
		// 获取 ClassPathXmlApplicationContext构造方法中 setConfigLocations 方法设置的资源
		String[] configLocations = getConfigLocations();
		if (configLocations != null) {
			reader.loadBeanDefinitions(configLocations);
		}
	}
}
```

