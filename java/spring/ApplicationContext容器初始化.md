# ApplicationContext容器初始化

IOC容器的初始化包括**`BeanDefinition`**的**`Resource 定位、加载和注册`**过程。`ApplicationContext`系列容器也许是我们最熟悉的，因为Web项目中使用的XmlWebApplicationContext就属于这个继承体系，还有ClasspathXmlApplicationContext等，其继承体系如下图所示：

![ClasspathXmlApplicationContext继承体系](https://raw.githubusercontent.com/renjiema/images/main/blogs/20210607195558.png)

`ApplicationContext`允许上下文嵌套，通过保持父上下文可以维持一个上下文体系。

对于Bean的查找可以在这个上下文体系中发生，首先检查当前上下文，其次是父上下文，逐级向上，这样为不同的Spring应用提供了一个共享的Bean定义环境。

## 1、程序入口

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

## 2、获得配置路径

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

## 3、开始启动IOC容器

Spring IOC 容器对 Bean 配置资源的载入是从`refresh()`函数开始的，`refresh()`是一个模板方法规定了IOC容器的启动流程 ，有些逻辑要交给其子类去实现。

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

## 4、创建IOC容器

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

首先判断`BeanFactory`是否存在，如果存在则销毁beans并关闭beanFactory，接着创建 `DefaultListableBeanFactory`(默认的IOC容器，一个完整的、功能成熟的 IOC 容器)，并调用`loadBeanDefinitions(beanFactory)`装载BeanDefinition。

## 5、载入配置路径

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
    @Nullable
	protected Resource[] getConfigResources() {
		return null;
	}
}
```

`XmlBeanDefinitionReader`调用其父类`AbstractBeanDefinitionReader`的`reader.loadBeanDefinitions()`方法读取Bean配置资源。 

由于我们使用`ClassPathXmlApplicationContext`作为例子分析，`getConfigResources()`方法的返回值为null，因此程序执行 `reader.loadBeanDefinitions(configLocations)`分支。

## 6、分配路径处理策略

在 `XmlBeanDefinitionReader`的抽象父类`AbstractBeanDefinitionReader`中定义了载入过程。

`AbstractBeanDefinitionReader` 的 `loadBeanDefinitions()`方法源码如下： 

```java
@Override
public int loadBeanDefinitions(String... locations) throws BeanDefinitionStoreException {
	Assert.notNull(locations, "Location array must not be null");
	int count = 0;
	for (String location : locations) {
		count += loadBeanDefinitions(location);
	}
	return count;
}
@Override
public int loadBeanDefinitions(String location) throws BeanDefinitionStoreException {
	return loadBeanDefinitions(location, null);
}
public int loadBeanDefinitions(String location, @Nullable Set<Resource> actualResources) throws BeanDefinitionStoreException {
	//获取在 IOC 容器初始化过程中设置的资源加载器
	ResourceLoader resourceLoader = getResourceLoader();
	if (resourceLoader == null) {
		throw new BeanDefinitionStoreException(
				"Cannot load bean definitions from location [" + location + "]: no ResourceLoader available");
	}
	if (resourceLoader instanceof ResourcePatternResolver) {
		// Resource pattern matching available.
		try {
			//将指定位置的 Bean 配置信息解析为 Spring IOC 容器封装的资源
			Resource[] resources = ((ResourcePatternResolver) resourceLoader).getResources(location);
			//委派调用其子类 XmlBeanDefinitionReader 的方法，实现加载功能
			int count = loadBeanDefinitions(resources);
			if (actualResources != null) {
				Collections.addAll(actualResources, resources);
			}
			if (logger.isTraceEnabled()) {
				logger.trace("Loaded " + count + " bean definitions from location pattern [" + location + "]");
			}
			return count;
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException(
					"Could not resolve bean definition resource pattern [" + location + "]", ex);
		}
	}
	else {
		// Can only load single resources by absolute URL.
		//将指定位置的 Bean 配置信息解析为 Spring IOC 容器封装的资源
		Resource resource = resourceLoader.getResource(location);
		//委派调用其子类 XmlBeanDefinitionReader 的方法，实现加载功能
		int count = loadBeanDefinitions(resource);
		if (actualResources != null) {
			actualResources.add(resource);
		}
		if (logger.isTraceEnabled()) {
			logger.trace("Loaded " + count + " bean definitions from location [" + location + "]");
		}
		return count;
	}
}
```

`AbstractXmlApplicationContext`的`loadBeanDefinitions(XmlBeanDefinitionReader reader)`方法实际上是调用`AbstractBeanDefinitionReader`的`loadBeanDefinitions()`方法。 

从对 `AbstractBeanDefinitionReader` 的 `loadBeanDefinitions()`方法源码分析可以看出该方法就做了两件事： 

* 调用资源加载器的获取资源方法 `resourceLoader.getResource(location)`，获取到要加载的资源。 
* 其次，真正执行加载功能是其子类 `XmlBeanDefinitionReader` 的 `loadBeanDefinitions()`方法。

在`loadBeanDefinitions()`方法中调用了`AbstractApplicationContext`的`getResources()`方法，跟进去之后发现 getResources()方法其实定义在 `ResourcePatternResolver`中，此时，我们有必要来看一下`ResourcePatternResolver`的类图： 

![image-20210608095414366](https://raw.githubusercontent.com/renjiema/images/main/blogs/20210608095414.png)

从上面可以看到 ResourceLoader 与 ApplicationContext 的继承关系，可以看出其实际调用的是DefaultResourceLoader中的getSource() 方法定位Resource。

因为`ClassPathXmlApplicationContext`本身就是`DefaultResourceLoader`的实现类，所以此时又回到了`ClassPathXmlApplicationContext`中来。

## 7、解析配置文件路径

`XmlBeanDefinitionReader`通 过 调 用`ClassPathXmlApplicationContext`的父类 `DefaultResourceLoader` 的 `getResource()`方法获取要指定的资源，其源码如下 :

```java
@Override
public Resource getResource(String location) {
	Assert.notNull(location, "Location must not be null");
	for (ProtocolResolver protocolResolver : getProtocolResolvers()) {
		Resource resource = protocolResolver.resolve(location, this);
		if (resource != null) {
			return resource;
		}
	}
	if (location.startsWith("/")) {
		return getResourceByPath(location);
	}
	else if (location.startsWith(CLASSPATH_URL_PREFIX)) {
		//如果是类路径的方式，那需要使用 ClassPathResource 来得到 bean 文件的资源对象
		return new ClassPathResource(location.substring(CLASSPATH_URL_PREFIX.length()), getClassLoader());
	}
	else {
		try {
			// Try to parse the location as a URL...
			// 如果是 URL 方式，使用 UrlResource 作为 bean 文件的资源对象
			URL url = new URL(location);
			return (ResourceUtils.isFileURL(url) ? new FileUrlResource(url) : new UrlResource(url));
		}
		catch (MalformedURLException ex) {
			// No URL -> resolve as resource path.
			//如果既不是 classpath 标识，又不是 URL 标识的 Resource 定位，则调用
			//容器本身的 getResourceByPath 方法获取 Resource
			return getResourceByPath(location);
		}
	}
```

`DefaultResourceLoader`提供了`getResourceByPath()`方法的实现，用于处理Resource既不是classpath标识，又不是URL标识的这种情况。

```java
protected Resource getResourceByPath(String path) {
	return new ClassPathContextResource(path, getClassLoader());
}
```

在`ClassPathResource`中完成了对整个路径的解析。这样，就可以从类路径上对 IOC 配置文件进行加载，当然我们可以按照这个逻辑从任何地方加载，在 Spring 中我们看到它提供的各种资源抽象，比如 ClassPathResource、URLResource、FileSystemResource 等来供我们使用。

上面我们看到的是定位Resource 的一个过程，而这只是加载过程的一部分。例如 FileSystemXmlApplication 容器就重写了getResourceByPath()方法： 

```java
@Override
protected Resource getResourceByPath(String path) {
	if (path.startsWith("/")) {
		path = path.substring(1);
	}
	return new FileSystemResource(path);
}
```

通过子类的覆盖，巧妙地完成了将类路径变为文件路径的转换。

## 8、开始读取配置内容

继续回到 `XmlBeanDefinitionReader` 的 `loadBeanDefinitions(Resource …)`方法看到代表 bean 文件的资源定义以后的载入过程。

```java
public class XmlBeanDefinitionReader extends AbstractBeanDefinitionReader {
    @Override
    public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
    	//将读入的 XML 资源进行特殊编码处理
    	return loadBeanDefinitions(new EncodedResource(resource));
    }
    public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
    	Assert.notNull(encodedResource, "EncodedResource must not be null");
    	...
    	Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
    	if (!currentResources.add(encodedResource)) {
    		throw new BeanDefinitionStoreException(
    				"Detected cyclic loading of " + encodedResource + " - check your import definitions!");
    	}

    	//将资源文件转为 InputStream 的 IO 流
    	try (InputStream inputStream = encodedResource.getResource().getInputStream()) {
    		//从 InputStream 中得到 XML 的解析源
    		InputSource inputSource = new InputSource(inputStream);
    		if (encodedResource.getEncoding() != null) {
    			inputSource.setEncoding(encodedResource.getEncoding());
    		}
    		//具体的读取过程
    		return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
    	}
    	catch (IOException ex) {
    		throw new BeanDefinitionStoreException(
    				"IOException parsing XML document from " + encodedResource.getResource(), ex);
    	}
    	finally {
    		currentResources.remove(encodedResource);
    		if (currentResources.isEmpty()) {
    			this.resourcesCurrentlyBeingLoaded.remove();
    		}
    	}
    }
    protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
    		throws BeanDefinitionStoreException {
    	try {
    		Document doc = doLoadDocument(inputSource, resource);
    		//注册bean定义
    		int count = registerBeanDefinitions(doc, resource);
    		if (logger.isDebugEnabled()) {
    			logger.debug("Loaded " + count + " bean definitions from " + resource);
    		}
    		return count;
    	}
    	...
    }
}
```

载入配置信息的最后一步是将Bean配置信息转换为Document对象，该过程由`documentLoader()`方法实现。

## 9、准备Document对象

DocumentLoader 将 Bean 配置资源转换成 Document 对象的源码如下：

```java
public class DefaultDocumentLoader implements DocumentLoader {
	@Override
	public Document loadDocument(InputSource inputSource, EntityResolver entityResolver,
			ErrorHandler errorHandler, int validationMode, boolean namespaceAware) throws Exception {
		//创建文件解析器工厂
		DocumentBuilderFactory factory = createDocumentBuilderFactory(validationMode, namespaceAware);
		if (logger.isTraceEnabled()) {
			logger.trace("Using JAXP provider [" + factory.getClass().getName() + "]");
		}
		//创建文档解析器
		DocumentBuilder builder = createDocumentBuilder(factory, entityResolver, errorHandler);
		//解析配置文档
		return builder.parse(inputSource);
	}
	protected DocumentBuilderFactory createDocumentBuilderFactory(int validationMode, boolean namespaceAware)
			throws ParserConfigurationException {
		DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
		factory.setNamespaceAware(namespaceAware);
		//设置解析 XML 的校验
		if (validationMode != XmlValidationModeDetector.VALIDATION_NONE) {
			factory.setValidating(true);
			if (validationMode == XmlValidationModeDetector.VALIDATION_XSD) {
				// Enforce namespace aware for XSD...
				factory.setNamespaceAware(true);
				try {
					factory.setAttribute(SCHEMA_LANGUAGE_ATTRIBUTE, XSD_SCHEMA_LANGUAGE);
				}
				catch (IllegalArgumentException ex) {
					ParserConfigurationException pcex = new ParserConfigurationException(
							"Unable to validate using XSD: Your JAXP provider [" + factory +
							"] does not support XML Schema. Are you running on Java 1.4 with Apache Crimson? " +
							"Upgrade to Apache Xerces (or Java 1.5) for full XSD support.");
					pcex.initCause(ex);
					throw pcex;
				}
			}
		}
		return factory;
	}
}
```

上面的解析过程是调用 JavaEE 标准的 JAXP 标准进行处理。

至此 Spring IOC容器根据定位的配置信息，将其加载读入并转换成为Document对象过程完成。

接下来分析 Spring IOC 容器将载入的 Bean 配置信息转换为 Document 对象之后，是如何将其解析为Spring IOC管理的 Bean 对象并将其注册到容器中的。

## 10、分配解析策略

`XmlBeanDefinitionReader`类中的`doLoadBeanDefinition()`方法是从特定 XML 文件中实际载入BeanDefinition的方法，该方法在载入 BeanDefinition之后将其转换为 Document 对象。

接下来调用registerBeanDefinitions()启动Spring IOC容器对BeanDefinition的解析过程 ， registerBeanDefinitions()方法源码如下：  

```java
public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
	//得到 BeanDefinitionDocumentReader 来对 xml 格式的 BeanDefinition 解析
	BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
	//获得容器中注册的 Bean 数量
	int countBefore = getRegistry().getBeanDefinitionCount();
	//解析过程入口，这里使用了委派模式，BeanDefinitionDocumentReader 只是个接口,
	//具体的解析实现过程有实现类 DefaultBeanDefinitionDocumentReader 完成
	documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
	//统计解析的 Bean 数量
	return getRegistry().getBeanDefinitionCount() - countBefore;
}
```

Bean 配置资源的载入解析分为以下两个过程： 

首先，通过调用 XML 解析器将 Bean 配置信息转换得到 Document 对象，但是这些 Document 对象并没有按照 Spring 的 Bean 规则进行解析。

其次，在完成通用的 XML 解析之后，按照 Spring Bean 的定义规则对 Document 对象进行解析，其解析过程是在接口`BeanDefinitionDocumentReader`的实现类`DefaultBeanDefinitionDocumentReader`中实现。 

## 11、将配置载入内存

BeanDefinitionDocumentReader 接口通过 registerBeanDefinitions()方法调用其实现类DefaultBeanDefinitionDocumentReader 对 Document 对象进行解析，解析的代码如下：

```java
public class DefaultBeanDefinitionDocumentReader implements BeanDefinitionDocumentReader {
	@Override
	public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
		this.readerContext = readerContext;
		doRegisterBeanDefinitions(doc.getDocumentElement());
	}
	protected void doRegisterBeanDefinitions(Element root) {
		BeanDefinitionParserDelegate parent = this.delegate;
		this.delegate = createDelegate(getReaderContext(), root, parent);
		if (this.delegate.isDefaultNamespace(root)) {
			String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
			if (StringUtils.hasText(profileSpec)) {
				String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
						profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
				if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
					if (logger.isDebugEnabled()) {
						logger.debug("Skipped XML bean definition file due to specified profiles [" + profileSpec +
								"] not matching: " + getReaderContext().getResource());
					}
					return;
				}
			}
		}
		//在解析前进行，自定义处理
		preProcessXml(root);
		//解析bean定义
		parseBeanDefinitions(root, this.delegate);
		//在解析后进行，自定义处理
		postProcessXml(root);
		this.delegate = parent;
	}
        //创建BeanDefinitionParserDelegate，用于完成doc解析
	protected BeanDefinitionParserDelegate createDelegate(
			XmlReaderContext readerContext, Element root, @Nullable BeanDefinitionParserDelegate parentDelegate) {

		BeanDefinitionParserDelegate delegate = new BeanDefinitionParserDelegate(readerContext);
		delegate.initDefaults(root, parentDelegate);
		return delegate;
	}
}
```

`BeanDefinitionParserDelegate parent = this.delegate;`的用于处理beans标签的嵌套。

`doRegisterBeanDefinitions()`方法流程比较简单，首先检查了一下有没有 profile 需要处理，处理完 profile 就是解析，解析有一个前置处理方法 preProcessXml 和后置处理方法 postProcessXml，不过这两个方法默认都是空方法，真正的解析方法是 `parseBeanDefinitions`：

```java
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
	//Bean定义的doc对象使用了spring默认的xml命名空间
	if (delegate.isDefaultNamespace(root)) {
		//获取根元素的子节点
		NodeList nl = root.getChildNodes();
		for (int i = 0; i < nl.getLength(); i++) {
			Node node = nl.item(i);
			if (node instanceof Element) {
				Element ele = (Element) node;
				if (delegate.isDefaultNamespace(ele)) {
					//处理spring规则节点
					parseDefaultElement(ele, delegate);
				}
				else {
					//处理自定义节点
					delegate.parseCustomElement(ele);
				}
			}
		}
	}
	else {
		delegate.parseCustomElement(root);
	}
}
```

在该方法中进行节点的解析，最终会来到 parseDefaultElement 方法中:

```java
private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
	if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
		//处理import
		importBeanDefinitionResource(ele);
	}
	else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
		//处理alias
		processAliasRegistration(ele);
	}
	else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
		//处理bean
		processBeanDefinition(ele, delegate);
	}
	else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
		// recurse
		//处理beans
		doRegisterBeanDefinitions(ele);
	}
}
```

在该方法中，我们可以看到，节点一共被分为了四大类：

- import
- alias
- bean
- beans

每一个节点都好理解，需要注意的是， beans 节点会再次调用`doRegisterBeanDefinitions`方法进行递归解析，源码上面还给了一个注释 recurse，意思就是递归。

在Spring 配置文件中可以使用<import>元素来导入 IOC 容器所需要的其他资源，Spring IOC 容器在解析时会首先将指定导入的资源加载进容器中。

```java
protected void importBeanDefinitionResource(Element ele) {
	//获取指定的resource
	String location = ele.getAttribute(RESOURCE_ATTRIBUTE);
	if (!StringUtils.hasText(location)) {
		getReaderContext().error("Resource location must not be empty", ele);
		return;
	}
	// Resolve system properties: e.g. "${user.dir}"
	//解析系统属性
	location = getReaderContext().getEnvironment().resolveRequiredPlaceholders(location);
	Set<Resource> actualResources = new LinkedHashSet<>(4);
	// Discover whether the location is an absolute or relative URI
	boolean absoluteLocation = false;
	try {
		absoluteLocation = ResourcePatternUtils.isUrl(location) || ResourceUtils.toURI(location).isAbsolute();
	}
	catch (URISyntaxException ex) {
		// cannot convert to an URI, considering the location relative
		// unless it is the well-known Spring prefix "classpath*:"
	}
	// Absolute or relative?
	if (absoluteLocation) {
		try {
			//加载绝对路径
			int importCount = getReaderContext().getReader().loadBeanDefinitions(location, actualResources);
			if (logger.isTraceEnabled()) {
				logger.trace("Imported " + importCount + " bean definitions from URL location [" + location + "]");
			}
		}
		catch (BeanDefinitionStoreException ex) {
			getReaderContext().error(
					"Failed to import bean definitions from URL location [" + location + "]", ele, ex);
		}
	}
	else {
		// No URL -> considering resource location as relative to the current file.
		try {
			int importCount;
			//创建相对路径资源
			Resource relativeResource = getReaderContext().getResource().createRelative(location);
			if (relativeResource.exists()) {
				//存在,加载import的资源
				importCount = getReaderContext().getReader().loadBeanDefinitions(relativeResource);
				actualResources.add(relativeResource);
			}
			else {
				//不存在,获取spring IOC容器资源读取的基本路径
				String baseLocation = getReaderContext().getResource().getURL().toString();
				importCount = getReaderContext().getReader().loadBeanDefinitions(
						StringUtils.applyRelativePath(baseLocation, location), actualResources);
			}
			if (logger.isTraceEnabled()) {
				logger.trace("Imported " + importCount + " bean definitions from relative location [" + location + "]");
			}
		}
		catch (IOException ex) {
			getReaderContext().error("Failed to resolve current resource location", ele, ex);
		}
		catch (BeanDefinitionStoreException ex) {
			getReaderContext().error(
					"Failed to import bean definitions from relative location [" + location + "]", ele, ex);
		}
	}
	Resource[] actResArray = actualResources.toArray(new Resource[0]);
	//发送import处理完成事件
	getReaderContext().fireImportProcessed(location, actResArray, extractSource(ele));
}
```

## 12、载入bean元素

对 Bean 配置信息中使用最多的<bean>元素交由 BeanDefinitionParserDelegate来解析， 来首先看`processBeanDefinition`方法： 

```java
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
	//BeanDefinitionHolder是对BeanDefinition的封装
	BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
	if (bdHolder != null) {
		bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
		try {
			// Register the final decorated instance.
			//向IOC容器注册Bean定义
			BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
		}
		catch (BeanDefinitionStoreException ex) {
			getReaderContext().error("Failed to register bean definition with name '" +
					bdHolder.getBeanName() + "'", ele, ex);
		}
		// Send registration event.
		getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
	}
}
```

先调用代理类`BeanDefinitionParserDelegate`对元素进行解析，解析的结果会保存在`BeanDefinitionHolder`中，也就是 bean 节点中配置的元素 class、id、name 等属性，在经过这一步的解析之后，都会保存到 bdHolder 中。

如果 bdHolder 不为空，那么接下来对子节点的属性继续解析，同时对 bdHolder 进行注册，最终发出事件，通知这个 bean 节点已经加载完了。

如此看来，整个解析的核心过程应该在 delegate.parseBeanDefinitionElement(ele) 方法中：

```java
public class BeanDefinitionParserDelegate {
	@Nullable
	public BeanDefinitionHolder parseBeanDefinitionElement(Element ele) {
		return parseBeanDefinitionElement(ele, null);
	}
	@Nullable
	//解析Bean元素,主要处理id、name、alias
	public BeanDefinitionHolder parseBeanDefinitionElement(Element ele, @Nullable BeanDefinition containingBean) {
		String id = ele.getAttribute(ID_ATTRIBUTE);
		String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);
		List<String> aliases = new ArrayList<>();
		if (StringUtils.hasLength(nameAttr)) {
			//将name存放到alias中,使用',; '分割
			String[] nameArr = StringUtils.tokenizeToStringArray(nameAttr, MULTI_VALUE_ATTRIBUTE_DELIMITERS);
			aliases.addAll(Arrays.asList(nameArr));
		}
		String beanName = id;
		if (!StringUtils.hasText(beanName) && !aliases.isEmpty()) {
			//id为空,将第一个alias设置为beanName
			beanName = aliases.remove(0);
			if (logger.isTraceEnabled()) {
				logger.trace("No XML 'id' specified - using '" + beanName +
						"' as bean name and " + aliases + " as aliases");
			}
		}
		if (containingBean == null) {
			//检查beanName和aliases唯一性
			checkNameUniqueness(beanName, aliases, ele);
		}
		//解析BeanDefinition
		AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean);
		if (beanDefinition != null) {
			if (!StringUtils.hasText(beanName)) {
				//beanName为空,生成beanName
				try {
					if (containingBean != null) {
						beanName = BeanDefinitionReaderUtils.generateBeanName(
								beanDefinition, this.readerContext.getRegistry(), true);
					}
					else {
						beanName = this.readerContext.generateBeanName(beanDefinition);
						String beanClassName = beanDefinition.getBeanClassName();
						if (beanClassName != null &&
								beanName.startsWith(beanClassName) && beanName.length() > beanClassName.length() &&
								!this.readerContext.getRegistry().isBeanNameInUse(beanClassName)) {
							aliases.add(beanClassName);
						}
					}
					if (logger.isTraceEnabled()) {
						logger.trace("Neither XML 'id' nor 'name' specified - " +
								"using generated bean name [" + beanName + "]");
					}
				}
				catch (Exception ex) {
					error(ex.getMessage(), ele);
					return null;
				}
			}
			String[] aliasesArray = StringUtils.toStringArray(aliases);
			//返回BeanDefinitionHolder
			return new BeanDefinitionHolder(beanDefinition, beanName, aliasesArray);
		}
		return null;
	}
}
```

方法中所作的事情我们可以大致分为 5 个步骤：

1. 提取出 id 和 name 属性值。
2. 检查 beanName 是否唯一。
3. 对节点做进一步的解析，解析出 beanDefinition 对象，真是的类型是 `GenericBeanDefinition`。
4. 如果 beanName 属性没有值，则使用默认的规则生成 beanName（默认规则是类名全路径）。
5. 最终将获取到的信息封装成一个 `BeanDefinitionHolder` 返回。

Bean元素的其他属性解析，主要是在 `parseBeanDefinitionElement()` 方法中完成：

```java
@Nullable
public AbstractBeanDefinition parseBeanDefinitionElement(
		Element ele, String beanName, @Nullable BeanDefinition containingBean) {
	//记录解析的<Bean>
	this.parseState.push(new BeanEntry(beanName));
	String className = null;
	if (ele.hasAttribute(CLASS_ATTRIBUTE)) {
		//获取class属性
		className = ele.getAttribute(CLASS_ATTRIBUTE).trim();
	}
	String parent = null;
	if (ele.hasAttribute(PARENT_ATTRIBUTE)) {
		//获取parent属性
		parent = ele.getAttribute(PARENT_ATTRIBUTE);
	}
	try {
		AbstractBeanDefinition bd = createBeanDefinition(className, parent);
		//对<Bean>中的一些属性进行解析和配置，比如singleton
		parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
		//设置description
		bd.setDescription(DomUtils.getChildElementValueByTagName(ele, DESCRIPTION_ELEMENT));
		//解析meta信息
		parseMetaElements(ele, bd);
		//解析lookup-method属性
		parseLookupOverrideSubElements(ele, bd.getMethodOverrides());
		//解析replaced-method属性
		parseReplacedMethodSubElements(ele, bd.getMethodOverrides());
		//解析构造参数
		parseConstructorArgElements(ele, bd);
		//解析<property>元素
		parsePropertyElements(ele, bd);
		//解析qualifier属性
		parseQualifierElements(ele, bd);
		//设置资源和依赖对象
		bd.setResource(this.readerContext.getResource());
		bd.setSource(extractSource(ele));
		return bd;
	}
	...
	finally {
		this.parseState.pop();
	}
	return null;
}
```

上面方法中一些对一些配置如元信息(meta)、qualifier 等的解析，我们在 Spring 中配置时使用的也不多，还有一些比较常规，例如 parseBeanDefinitionAttributes 方法用来解析各种各样的节点属性，这些节点属性可能大家都比较熟悉:

```java
public AbstractBeanDefinition parseBeanDefinitionAttributes(Element ele, Stri
		@Nullable BeanDefinition containingBean, AbstractBeanDefinition bd) {
	if (ele.hasAttribute(SINGLETON_ATTRIBUTE)) {
		error("Old 1.x 'singleton' attribute in use - upgrade to 'scope' decl
	}
	else if (ele.hasAttribute(SCOPE_ATTRIBUTE)) {
		bd.setScope(ele.getAttribute(SCOPE_ATTRIBUTE));
	}
	else if (containingBean != null) {
		// Take default from containing bean in case of an inner bean definit
		bd.setScope(containingBean.getScope());
	}
	if (ele.hasAttribute(ABSTRACT_ATTRIBUTE)) {
		bd.setAbstract(TRUE_VALUE.equals(ele.getAttribute(ABSTRACT_ATTRIBUTE)
	}
	String lazyInit = ele.getAttribute(LAZY_INIT_ATTRIBUTE);
	if (isDefaultValue(lazyInit)) {
		lazyInit = this.defaults.getLazyInit();
	}
	bd.setLazyInit(TRUE_VALUE.equals(lazyInit));
	String autowire = ele.getAttribute(AUTOWIRE_ATTRIBUTE);
	bd.setAutowireMode(getAutowireMode(autowire));
	if (ele.hasAttribute(DEPENDS_ON_ATTRIBUTE)) {
		String dependsOn = ele.getAttribute(DEPENDS_ON_ATTRIBUTE);
		bd.setDependsOn(StringUtils.tokenizeToStringArray(dependsOn, MULTI_VA
	}
	String autowireCandidate = ele.getAttribute(AUTOWIRE_CANDIDATE_ATTRIBUTE)
	if (isDefaultValue(autowireCandidate)) {
		String candidatePattern = this.defaults.getAutowireCandidates();
		if (candidatePattern != null) {
			String[] patterns = StringUtils.commaDelimitedListToStringArray(c
			bd.setAutowireCandidate(PatternMatchUtils.simpleMatch(patterns, b
		}
	}
	else {
		bd.setAutowireCandidate(TRUE_VALUE.equals(autowireCandidate));
	}
	if (ele.hasAttribute(PRIMARY_ATTRIBUTE)) {
		bd.setPrimary(TRUE_VALUE.equals(ele.getAttribute(PRIMARY_ATTRIBUTE)))
	}
	if (ele.hasAttribute(INIT_METHOD_ATTRIBUTE)) {
		String initMethodName = ele.getAttribute(INIT_METHOD_ATTRIBUTE);
		bd.setInitMethodName(initMethodName);
	}
	else if (this.defaults.getInitMethod() != null) {
		bd.setInitMethodName(this.defaults.getInitMethod());
		bd.setEnforceInitMethod(false);
	}
	if (ele.hasAttribute(DESTROY_METHOD_ATTRIBUTE)) {
		String destroyMethodName = ele.getAttribute(DESTROY_METHOD_ATTRIBUTE)
		bd.setDestroyMethodName(destroyMethodName);
	}
	else if (this.defaults.getDestroyMethod() != null) {
		bd.setDestroyMethodName(this.defaults.getDestroyMethod());
		bd.setEnforceDestroyMethod(false);
	}
	if (ele.hasAttribute(FACTORY_METHOD_ATTRIBUTE)) {
		bd.setFactoryMethodName(ele.getAttribute(FACTORY_METHOD_ATTRIBUTE));
	}
	if (ele.hasAttribute(FACTORY_BEAN_ATTRIBUTE)) {
		bd.setFactoryBeanName(ele.getAttribute(FACTORY_BEAN_ATTRIBUTE));
	}
	return bd;
}
```

可以看到，这里解析的节点属性，从上往下，依次是：

1. 解析 singleton 属性（该属性已废弃，使用 scope 替代）。
2. 解析 scope 属性，如果未指定 scope 属性，但是存在 containingBean，则使用 containingBean 的 scope 属性值。
3. 解析 abstract 属性。
4. 解析 lazy-init 属性。
5. 解析 autowire 属性。
6. 解析 depends-on 属性。
7. 解析 autowire-candidate 属性。
8. 解析 primary 属性。
9. 解析 init-method 属性。
10. 解析 destroy-method 属性。
11. 解析 factory-method 属性。
12. 解析 factory-bean 属性。

除此之外配置最多的是<property>属性，下面继续分析源码，了解 Bean 的<property>如何解析。

### 13、载入property元素

BeanDefinitionParserDelegate 在解析<Bean>调用 parsePropertyElements()方法解析<Bean>元素中的<property>属性子元素，解析源码如下： 

```java
public void parsePropertyElements(Element beanEle, BeanDefinition bd) {
	NodeList nl = beanEle.getChildNodes();
	for (int i = 0; i < nl.getLength(); i++) {
		Node node = nl.item(i);
		if (isCandidateElement(node) && nodeNameEquals(node, PROPERTY_ELEMENT)) {
			parsePropertyElement((Element) node, bd);
		}
	}
}
public void parsePropertyElement(Element ele, BeanDefinition bd) {
	String propertyName = ele.getAttribute(NAME_ATTRIBUTE);
	if (!StringUtils.hasLength(propertyName)) {
		error("Tag 'property' must have a 'name' attribute", ele);
		return;
	}
	this.parseState.push(new PropertyEntry(propertyName));
	try {
		//如果已存在同名的property,取第一个
		if (bd.getPropertyValues().contains(propertyName)) {
			error("Multiple 'property' definitions for property '" + propertyName + "'", ele);
			return;
		}
		Object val = parsePropertyValue(ele, bd, propertyName);
		//创建pv对象
		PropertyValue pv = new PropertyValue(propertyName, val);
		//解析property元素属性
		parseMetaElements(ele, pv);
		pv.setSource(extractSource(ele));
		bd.getPropertyValues().addPropertyValue(pv);
	}
	finally {
		this.parseState.pop();
	}
}
@Nullable
public Object parsePropertyValue(Element ele, BeanDefinition bd, @Nullable String propertyName) {
	String elementName = (propertyName != null ?
			"<property> element for property '" + propertyName + "'" :
			"<constructor-arg> element");
	// Should only have one child element: ref, value, list, etc.
	//子元素只能是: ref, value, list, et中的一种,且最多只有一个
	NodeList nl = ele.getChildNodes();
	Element subElement = null;
	for (int i = 0; i < nl.getLength(); i++) {
		Node node = nl.item(i);
		if (node instanceof Element && !nodeNameEquals(node, DESCRIPTION_ELEMENT) &&
				!nodeNameEquals(node, META_ELEMENT)) {
			// Child element is what we're looking for.
			if (subElement != null) {
				error(elementName + " must not contain more than one sub-element", ele);
			}
			else {
				subElement = (Element) node;
			}
		}
	}
	boolean hasRefAttribute = ele.hasAttribute(REF_ATTRIBUTE);
	boolean hasValueAttribute = ele.hasAttribute(VALUE_ATTRIBUTE);
	if ((hasRefAttribute && hasValueAttribute) ||
			((hasRefAttribute || hasValueAttribute) && subElement != null)) {
		error(elementName +
				" is only allowed to contain either 'ref' attribute OR 'value' attribute OR sub-element", ele);
	}
	if (hasRefAttribute) {
		String refName = ele.getAttribute(REF_ATTRIBUTE);
		if (!StringUtils.hasText(refName)) {
			error(elementName + " contains empty 'ref' attribute", ele);
		}
		RuntimeBeanReference ref = new RuntimeBeanReference(refName);
		ref.setSource(extractSource(ele));
		return ref;
	}
	else if (hasValueAttribute) {
		TypedStringValue valueHolder = new TypedStringValue(ele.getAttribute(VALUE_ATTRIBUTE));
		valueHolder.setSource(extractSource(ele));
		return valueHolder;
	}
	else if (subElement != null) {
		//解析property子元素
		return parsePropertySubElement(subElement, bd);
	}
	else {
		// Neither child element nor "ref" or "value" attribute found.
		error(elementName + " must specify a ref or value", ele);
		return null;
	}
}
```

通过对上述源码的分析，我们可以了解在 Spring 配置文件中，<Bean>元素中<property>元素的相关配置是如何处理的： 

1. ref 被封装为指向依赖对象一个引用。 

2. value 配置都会封装成一个字符串类型的对象。 

3. ref 和 value 都通过“解析的数据类型属性值.setSource(extractSource(ele));”方法将属性值/引用与所引用的属性关联起来。 

在方法的最后对于<property>元素的子元素通过 `parsePropertySubElement()`方法解析，我们继续分析该方法的源码，了解其解析过程。 

### 14、载入property的子元素

在 `BeanDefinitionParserDelegate` 类中的 parsePropertySubElement()方法对<property>中的子元素解析，源码如下： 

```java
@Nullable
public Object parsePropertySubElement(Element ele, @Nullable BeanDefinition bd) {
	return parsePropertySubElement(ele, bd, null);
}
@Nullable
public Object parsePropertySubElement(Element ele, @Nullable BeanDefinition bd, @Nullable String defaultValueType) {
	if (!isDefaultNamespace(ele)) {
		return parseNestedCustomElement(ele, bd);
	}
	else if (nodeNameEquals(ele, BEAN_ELEMENT)) {
		BeanDefinitionHolder nestedBd = parseBeanDefinitionElement(ele, bd);
		if (nestedBd != null) {
			nestedBd = decorateBeanDefinitionIfRequired(ele, nestedBd, bd);
		}
		return nestedBd;
	}
	else if (nodeNameEquals(ele, REF_ELEMENT)) {
		// A generic reference to any name of any bean.
		String refName = ele.getAttribute(BEAN_REF_ATTRIBUTE);
		boolean toParent = false;
		if (!StringUtils.hasLength(refName)) {
			// A reference to the id of another bean in a parent context.
			refName = ele.getAttribute(PARENT_REF_ATTRIBUTE);
			toParent = true;
			if (!StringUtils.hasLength(refName)) {
				error("'bean' or 'parent' is required for <ref> element", ele);
				return null;
			}
		}
		if (!StringUtils.hasText(refName)) {
			error("<ref> element contains empty target attribute", ele);
			return null;
		}
		RuntimeBeanReference ref = new RuntimeBeanReference(refName, toParent);
		ref.setSource(extractSource(ele));
		return ref;
	}
	else if (nodeNameEquals(ele, IDREF_ELEMENT)) {
		return parseIdRefElement(ele);
	}
	else if (nodeNameEquals(ele, VALUE_ELEMENT)) {
		return parseValueElement(ele, defaultValueType);
	}
	else if (nodeNameEquals(ele, NULL_ELEMENT)) {
		// It's a distinguished null value. Let's wrap it in a TypedStringValue
		// object in order to preserve the source location.
		TypedStringValue nullHolder = new TypedStringValue(null);
		nullHolder.setSource(extractSource(ele));
		return nullHolder;
	}
	else if (nodeNameEquals(ele, ARRAY_ELEMENT)) {
		return parseArrayElement(ele, bd);
	}
	else if (nodeNameEquals(ele, LIST_ELEMENT)) {
		return parseListElement(ele, bd);
	}
	else if (nodeNameEquals(ele, SET_ELEMENT)) {
		return parseSetElement(ele, bd);
	}
	else if (nodeNameEquals(ele, MAP_ELEMENT)) {
		return parseMapElement(ele, bd);
	}
	else if (nodeNameEquals(ele, PROPS_ELEMENT)) {
		return parsePropsElement(ele);
	}
	else {
		error("Unknown property sub-element: [" + ele.getNodeName() + "]", ele);
		return null;
	}
}
```

上述源码展示了 Spring 配置文件中，对<property>元素中配置的 array、list、set、map、prop 等各种集合子元素的都通过上述方法解析，生成对应的数据对象，比如 ManagedList、 ManagedArray、ManagedSet 等，这些 Managed 类是 Spring 对象 BeanDefiniton 的数据封装，对集合数据类型的具体解析有各自的解析方法实现，我们对<list>集合元素的解析方法进行源码分析，了解其实现过程。 

#### 15、载入list子元素

在`BeanDefinitionParserDelegate`类中的 `parseListElement()`方法就是具体实现解析<property>元素中的<list>集合子元素，源码如下：

```java
public List<Object> parseListElement(Element collectionEle, @Nullable BeanDefinition bd) {
	String defaultElementType = collectionEle.getAttribute(VALUE_TYPE_ATTRIBUTE);
	NodeList nl = collectionEle.getChildNodes();
	ManagedList<Object> target = new ManagedList<>(nl.getLength());
	target.setSource(extractSource(collectionEle));
	target.setElementTypeName(defaultElementType);
	target.setMergeEnabled(parseMergeAttribute(collectionEle));
	parseCollectionElements(nl, target, bd, defaultElementType);
	return target;
}
protected void parseCollectionElements(
		NodeList elementNodes, Collection<Object> target, @Nullable BeanDefinition bd, String defaultElementType) {
	for (int i = 0; i < elementNodes.getLength(); i++) {
		Node node = elementNodes.item(i);
		if (node instanceof Element && !nodeNameEquals(node, DESCRIPTION_ELEMENT)) {
			target.add(parsePropertySubElement((Element) node, bd, defaultElementType));
		}
	}
}
```

经过对 Spring Bean 配置信息转换的 Document 对象中的元素层层解析，Spring IOC 现在已经将 XML形式定义的 Bean 配置信息转换为 Spring IOC 所识别的数据结构——BeanDefinition，它是 Bean 配置信息中配置的 POJO 对象在 Spring IOC 容器中的映射。

通过 Spring IOC 容器对 Bean 配置资源的解析后，IOC 容器大致完成了管理 Bean 对象的准备工作，即初始化过程，但是最为重要的依赖注入还没有发生，现在在 IOC 容器中 BeanDefinition 存储的只是一些静态信息，接下来需要向容器注册 Bean 定义信息才能全部完成 IOC 容器的初始化过程

## 16、分配注册策略

让我们继续跟踪程序的执行顺序，接下来我们来分析`DefaultBeanDefinitionDocumentReader`对Bean定义转换的 Document 对象解析的流程中， 在其 `parseDefaultElement()` 方法中完成对Document 对 象 的 解 析 后 得 到 封 装 BeanDefinition的BeanDefinitionHold 对象 ， 然后调用`BeanDefinitionReaderUtils.registerBeanDefinition()`方法向IOC容器注册解析的Bean ， `BeanDefinitionReaderUtils` 的注册的源码如下：

```java
public abstract class BeanDefinitionReaderUtils {
	public static void registerBeanDefinition(
			BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
			throws BeanDefinitionStoreException {

		// Register bean definition under primary name.
		String beanName = definitionHolder.getBeanName();
		//向IOC容器注册BeanDefinition
		registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

		// Register aliases for bean name, if any.
		String[] aliases = definitionHolder.getAliases();
		if (aliases != null) {
			for (String alias : aliases) {
				//注册别名
				registry.registerAlias(beanName, alias);
			}
		}
	}
}
```

当调用 BeanDefinitionReaderUtils 向 IOC 容器注册解析的 BeanDefinition 时，真正完成注册功能的是`DefaultListableBeanFactory`。 

## 17、向容器注册

`DefaultListableBeanFactory`中使用一个HashMap的集合对象存放IOC容器中注册解析的BeanDefinition，`DefaultListableBeanFactory`的类图如下：

![image-20210608142300497](https://raw.githubusercontent.com/renjiema/images/main/blogs/20210608142300.png)

 IOC 容器注册的主要源码如下：

```java
public class DefaultListableBeanFactory extends AbstractAutowireCapableBeanFactory
		implements ConfigurableListableBeanFactory, BeanDefinitionRegistry, Serializable {
	private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>(256);
	@Override
	public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
			throws BeanDefinitionStoreException {

		Assert.hasText(beanName, "Bean name must not be empty");
		Assert.notNull(beanDefinition, "BeanDefinition must not be null");

		if (beanDefinition instanceof AbstractBeanDefinition) {
			try {
				//验证BeanDefinition
				((AbstractBeanDefinition) beanDefinition).validate();
			}
			catch (BeanDefinitionValidationException ex) {
				throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
						"Validation of bean definition failed", ex);
			}
		}

		BeanDefinition existingDefinition = this.beanDefinitionMap.get(beanName);
		if (existingDefinition != null) {
			if (!isAllowBeanDefinitionOverriding()) {
				throw new BeanDefinitionOverrideException(beanName, beanDefinition, existingDefinition);
			}
			else if (existingDefinition.getRole() < beanDefinition.getRole()) {
				// e.g. was ROLE_APPLICATION, now overriding with ROLE_SUPPORT or ROLE_INFRASTRUCTURE
				...
			}
			else if (!beanDefinition.equals(existingDefinition)) {
				...
			}
			else {
				...
			}
			this.beanDefinitionMap.put(beanName, beanDefinition);
		}
		else {
			if (hasBeanCreationStarted()) {
				// Cannot modify startup-time collection elements anymore (for stable iteration)
				//同步处理
				synchronized (this.beanDefinitionMap) {
					this.beanDefinitionMap.put(beanName, beanDefinition);
					List<String> updatedDefinitions = new ArrayList<>(this.beanDefinitionNames.size() + 1);
					updatedDefinitions.addAll(this.beanDefinitionNames);
					updatedDefinitions.add(beanName);
					this.beanDefinitionNames = updatedDefinitions;
					removeManualSingletonName(beanName);
				}
			}
			else {
				// Still in startup registration phase
				this.beanDefinitionMap.put(beanName, beanDefinition);
				this.beanDefinitionNames.add(beanName);
				removeManualSingletonName(beanName);
			}
			this.frozenBeanDefinitionNames = null;
		}
		if (existingDefinition != null || containsSingleton(beanName)) {
			//重置BeanDefinition缓存
			resetBeanDefinition(beanName);
		}
		else if (isConfigurationFrozen()) {
			clearByTypeCache();
		}
	}
}
```

registerBeanDefinition 方法是在 BeanDefinitionRegistry 接口中声明的，DefaultListableBeanFactory 类实现了 BeanDefinitionRegistry 接口，并实现了该方法，我们来看分析下该方法：

1. 首先对传入的 beanDefinition 对象进行校验，这也是注册前的最后一次校验，不过这个时候 BeanDefinition 对象已经到手了，所以这个校验并非 XML 文件校验，这里主要是对 methodOverrides 的校验。
2. 接下来会根据 beanName 从 beanDefinitionMap 中获取 BeanDefinition，看看当前 Bean 是否已经定义过了。beanDefinitionMap 是一个 Map 集合，这个集合中 key 是 beanName，value 是 BeanDefinition 对象。
3. 如果 BeanDefinition 已经存在了，那么接下来会判断是否允许 BeanDefinition 覆盖，如果不允许，就直接抛出异常，如果允许 BeanDefinition 的覆盖，那就向 beanDefinitionMap 中再次存一次值，覆盖之前的值。
4. 如果 BeanDefinition 不存在，那就直接注册。直接注册分两种情况：项目已经运行了和项目还没运行。
5. 如果项目已经运行，由于 beanDefinitionMap 是一个全局变量，可能存在并发问题，所以要加锁处理。否则就直接注册，所谓的注册就是把对象存入 beanDefinitionMap 中，同时将 beanName 都存入 beanDefinitionNames 集合中。

Bean 配置信息中配置的 Bean 被解析过后，已经注册到 IOC 容器中，真正完成了 IOC 容器初始化所做的全部工作。现在 IOC 容器中已经建立了整个 Bean 的配置信息，这些BeanDefinition 信息已经可以使用，并且可以被检索，IOC 容器的作用就是对这些注册的 Bean 定义信息进行处理和维护。这些的注册的 Bean 定义信息是 IOC 容器控制反转的基础，容器才可以进行依赖注入。

参考：

[Spring 源码解读第七弹！bean 标签的解析](https://juejin.cn/post/6859153061563072526)

[《Spring源码解析（四）》从源码深处体验Spring核心技术--基于Xml的IOC容器的初始化](https://mp.weixin.qq.com/s?__biz=MzI5MzE4MzYxMw==&mid=2247487449&idx=1&sn=c99a7dcfd3f48d980a233fbd27290247&source=41#wechat_redirect)

