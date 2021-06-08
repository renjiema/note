# BeanDefinition理解

## 1、BeanDefinition接口

在 Spring 容器中，我们广泛使用的是一个一个的Bean，`BeanDefinition` 从名字上就可以看出是关于 Bean 的定义。

XML 文件中配置的 Bean 的各种属性，这些属性不仅仅是和对象相关，Spring 容器还要解决 Bean 的生命周期、销毁、初始化等等各种操作，我们定义的关于 Bean 的生命周期、销毁、初始化等操作总得有一个对象来承载，那么这个对象就是 `BeanDefinition`。

XML 中定义的各种属性都会先加载到`BeanDefinition`上，然后通过 `BeanDefinition` 来生成一个 Bean，从这个角度来说，`BeanDefinition`和 Bean 的关系有点类似于类和对象的关系。

要理解 `BeanDefinition`，我们从 `BeanDefinition`的继承关系开始看起。

![image-20210608154844163](https://raw.githubusercontent.com/renjiema/images/main/blogs/20210608154844.png)

`BeanDefinition`是一个接口，继承自 `BeanMetadataElement` 和 `AttributeAccessor` 接口。

- `BeanMetadataElement`：该接口只有一个方法 getSource，该方法返回 Bean 的来源。
- `AttributeAccessor`：该接口主要规范了问任意对象元数据的方法。

我们来看下 `AttributeAccessor`：

```java
public interface AttributeAccessor {
    void setAttribute(String name, @Nullable Object value);
    @Nullable
    Object getAttribute(String name);
    @Nullable
    Object removeAttribute(String name);
    boolean hasAttribute(String name);
    String[] attributeNames();
}
```

这里定义了元数据的访问接口，具体的实现则是 `AttributeAccessorSupport`，这些数据采用 `LinkedHashMap` 进行存储。

这是 `BeanDefinition` 所继承的两个接口。接下来我们来看下 `BeanDefinition` 接口：

```java
public interface BeanDefinition extends AttributeAccessor, BeanMetadataElement {
	String SCOPE_SINGLETON = ConfigurableBeanFactory.SCOPE_SINGLETON;
	String SCOPE_PROTOTYPE = ConfigurableBeanFactory.SCOPE_PROTOTYPE;
	int ROLE_APPLICATION = 0;
	int ROLE_SUPPORT = 1;
	int ROLE_INFRASTRUCTURE = 2;

	// Modifiable attributes
	void setParentName(@Nullable String parentName);
	@Nullable
	String getParentName();
	void setBeanClassName(@Nullable String beanClassName);
	@Nullable
	String getBeanClassName();
	void setScope(@Nullable String scope);
	@Nullable
	String getScope();
	void setLazyInit(boolean lazyInit);
	boolean isLazyInit();
	void setDependsOn(@Nullable String... dependsOn);
	@Nullable
	String[] getDependsOn();
	void setAutowireCandidate(boolean autowireCandidate);
	boolean isAutowireCandidate();
	void setPrimary(boolean primary);
	boolean isPrimary();
	void setFactoryBeanName(@Nullable String factoryBeanName);
	@Nullable
	String getFactoryBeanName();
	void setFactoryMethodName(@Nullable String factoryMethodName);
	@Nullable
	String getFactoryMethodName();
	ConstructorArgumentValues getConstructorArgumentValues();
	default boolean hasConstructorArgumentValues() {
		return !getConstructorArgumentValues().isEmpty();
	}
	MutablePropertyValues getPropertyValues();
	default boolean hasPropertyValues() {
		return !getPropertyValues().isEmpty();
	}
	void setInitMethodName(@Nullable String initMethodName);
	@Nullable
	String getInitMethodName();
	void setDestroyMethodName(@Nullable String destroyMethodName);
	@Nullable
	String getDestroyMethodName();
	void setRole(int role);
	int getRole();
	void setDescription(@Nullable String description);
	@Nullable
	String getDescription();
    
        // Read-only attributes
	ResolvableType getResolvableType();
	boolean isSingleton();
	boolean isPrototype();
	boolean isAbstract();
	@Nullable
	String getResourceDescription();
	@Nullable
	BeanDefinition getOriginatingBeanDefinition();

}
```

BeanDefinition 中的方法虽然多，但是结合我们平时在 XML 中的配置，这些方法其实都很好理解：

1. 首先是用来描述 Bean 是否为单例的`scope`属性的`get/set`方法。
2. ROLE_xxx 用来描述一个 Bean 的角色，ROLE_APPLICATION 表示这个 Bean 是用户自己定义的 Bean；ROLE_SUPPORT 表示这个 Bean 是某些复杂配置的支撑部分；ROLE_INFRASTRUCTURE 表示这是一个 Spring 内部的 Bean，通过 setRole/getRole 可以修改。
3. `set/getParentName` 用来配置 parent 的名称。
4. `set/getBeanClassName` 配置 Bean 的 Class 全路径，对应 XML 中的 `<bean class="">` 配置。
5. `set/isLazyInit` 配置/获取 Bean 是否懒加载，对应 XML 中的 `<bean lazy-init="">` 配置。
6. `set/getDependsOn` 配置/获取 Bean 的依赖对象，这个对应了 XML 中的 `<bean depends-on="">` 配置。
7. `set/isAutowireCandidate` 配置/获取 Bean 是否是自动装配，对应 XML 中的 `<bean autowire-candidate="">` 配置。
8. `set/isPrimary` 配置/获取当前 Bean 是否为首选的 Bean，对应了 XML 中的 `<bean primary="">` 配置。
9. `set/getFactoryBeanName` 配置/获取 FactoryBean 的名字，对应了 XML 中的 `<bean factory-bean="">` 配置。
10. `set/getFactoryMethodName` 和上一条成对出现的，对应了 XML 中的 `<bean factory-method="">` 配置。
11. `getConstructorArgumentValues` 返回该 Bean 构造方法的参数值。
12. `hasConstructorArgumentValues` 判断上一条是否是空对象。
13. `getPropertyValues` 这个是获取普通属性的集合。
14. `hasPropertyValues` 判断上一条是否为空对象。
15. `setInitMethodName/setDestroyMethodName` 配置 Bean 的初始化方法、销毁方法。
16. `set/getDescription` 配置/返回 Bean 的描述。
17. `isSingleton` Bean 是否为单例。
18. `isPrototype` Bean 是否为原型。
19. `isAbstract` Bean 是否抽象。
20. `getResourceDescription` 返回定义 Bean 的资源描述。
21. `getOriginatingBeanDefinition` 如果当前 `BeanDefinition` 是一个代理对象，那么该方法可以用来返回原始的 `BeanDefinition` 。

## 2、BeanDefinition 实现类

上面只是 BeanDefinition 接口的定义，BeanDefinition 还拥有诸多实现类，我们也来大致了解下。

先来看一张继承关系图：

![image-20210608163218874](https://raw.githubusercontent.com/renjiema/images/main/blogs/20210608163218.png)

### 2.1 AbstractBeanDefinition

`AbstractBeanDefinition` 是一个抽象类，它根据 `BeanDefinition` 中定义的接口提供了相应的属性，并实现了 `BeanDefinition` 中定义的一部分方法。`BeanDefinition` 中原本只是定义了一系列的 get/set 方法，并没有提供对应的属性，在 `AbstractBeanDefinition` 中将所有的属性定义出来了。

后面其他的实现类也基本上都是在 `AbstractBeanDefinition` 的基础上完成的。

### 2.2 RootBeanDefinition

这是一个比较常用的实现类，对应了一般的元素标签。

### 2.3 ChildBeanDefinition

可以让子 `BeanDefinition` 定义拥有从父 `BeanDefinition` 那里继承配置的能力。

### 2.4 GenericBeanDefinition

`GenericBeanDefinition` 是从 Spring 2.5 以后新加入的 `BeanDefinition` 实现类。`GenericBeanDefinition` 可以动态设置父 Bean，同时兼具 `RootBeanDefinition` 和 `ChildBeanDefinition` 的功能。

### 2.5 AnnotatedBeanDefinition

表示注解类型 `BeanDefinition`，拥有获取注解元数据和方法元数据的能力。

### 2.6 AnnotatedGenericBeanDefinition

使用了 @Configuration 注解标记配置类会解析为 `AnnotatedGenericBeanDefinition`。

## 3.实践

接下来通过几行代码来测试下上述`BeanDefinition`的实现类

创建一个实体类`Spring`：

```java
public class Spring {
	private String name;
	private String version;

	@Override
	public String toString() {
		return "Spring{" +
				"name='" + name + '\'' +
				", version='" + version + '\'' +
				'}';
	}
	public String getName() {
		return name;
	}
	public void setName(String name) {
		this.name = name;
	}
	public String getVersion() {
		return version;
	}
	public void setVersion(String version) {
		this.version = version;
	}
}
```

首先我们验证`RootBeanDefinition`，手动定义一个`RootBeanDefinition`，并且将之注册到 Spring 容器中

```java
@Test
public void testRootBeanDefinition() {
	final AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
	final MutablePropertyValues pvs = new MutablePropertyValues();
	pvs.add("name", "spring框架")
			.add("version", "v5.2.15.RELEASE");
	final RootBeanDefinition rootBeanDefinition = new RootBeanDefinition(Spring.class, null, pvs);
	ctx.registerBeanDefinition("spring", rootBeanDefinition);
	ctx.refresh();
	final Spring spring = ctx.getBean(Spring.class);
	System.out.println("spring = " + spring);
}
```

`MutablePropertyValues` 是定义对象中的一个一个属性，构造 `RootBeanDefinition` 的时候，我们传入了类名称和属性集合，最终把 `RootBeanDefinition` 注册到容器中去。

最终输出结果如下：

```
spring = Spring{name='spring框架', version='v5.2.15.RELEASE'}
```

`ChildBeanDefinition` 具有从父 Bean 继承数据的能力，首先新建一个`Context`类，`Context`类在`Spring`类的基础上增加一个moduleName属性，这样`Context`就可以继承到`Spring`的 name和 version两个属性的值。

```java
public class Context{
	private String name;
	private String version;
	private String moduleName;

	@Override
	public String toString() {
		return "Context{" +
				"name='" + name + '\'' +
				", version='" + version + '\'' +
				", moduleName='" + moduleName + '\'' +
				'}';
	}
	public String getName() {
		return name;
	}
	public void setName(String name) {
		this.name = name;
	}
	public String getVersion() {
		return version;
	}
	public void setVersion(String version) {
		this.version = version;
	}
	public String getModuleName() {
		return moduleName;
	}
	public void setModuleName(String moduleName) {
		this.moduleName = moduleName;
	}
}
```

接下来测试`ChildBeanDefinition`：

```java
@Test
public void testChildBeanDefinition() {
	final AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
	final MutablePropertyValues pvs = new MutablePropertyValues();
	pvs.add("name", "spring框架")
			.add("version", "v5.2.15.RELEASE");
	final RootBeanDefinition rootBeanDefinition = new RootBeanDefinition(Spring.class, null, pvs);
	final ChildBeanDefinition childBeanDefinition = new ChildBeanDefinition("spring");
	childBeanDefinition.setBeanClass(Context.class);
	childBeanDefinition.getPropertyValues().add("moduleName", "spring-context");
	ctx.registerBeanDefinition("spring", rootBeanDefinition);
	ctx.registerBeanDefinition("context", childBeanDefinition);
	ctx.refresh();
	final Spring spring = ctx.getBean(Spring.class);
	final Context context = ctx.getBean(Context.class);
	System.out.println("spring = " + spring);
	System.out.println("context = " + context);
}
```

首先定义 `RootBeanDefinition` 并注册到 Spring 容器中，然后再定义 `ChildBeanDefinition`，`ChildBeanDefinition` 继承了 `RootBeanDefinition` 中现有的属性值。

最终输出结果如下：

```
spring = Spring{name='spring框架', version='v5.2.15.RELEASE'}
context = Context{name='spring框架', version='v5.2.15.RELEASE', moduleName='spring-context'}
```

`RootBeanDefinition` 和 `ChildBeanDefinition` 都可以被 `GenericBeanDefinition` 代替，效果一样，如下：

```java
@Test
public void testGenericBeanDefinition() {
	final AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
	final MutablePropertyValues pvs = new MutablePropertyValues();
	pvs.add("name", "spring框架")
			.add("version", "v5.2.15.RELEASE");
	final GenericBeanDefinition rootBeanDefinition = new GenericBeanDefinition();
	rootBeanDefinition.setBeanClass(Spring.class);
	rootBeanDefinition.setPropertyValues(pvs);
	ctx.registerBeanDefinition("spring", rootBeanDefinition);
	final GenericBeanDefinition childBeanDefinition = new GenericBeanDefinition();
	childBeanDefinition.setParentName("spring");
	childBeanDefinition.setBeanClass(Context.class);
	childBeanDefinition.getPropertyValues().add("moduleName", "spring-context");
	ctx.registerBeanDefinition("context", childBeanDefinition);
	ctx.refresh();
	final Spring spring = ctx.getBean(Spring.class);
	final Context context = ctx.getBean(Context.class);
	System.out.println("spring = " + spring);
	System.out.println("context = " + context);
}
```

运行结果同上。

在 Spring Boot 广泛流行之后，Java 配置使用越来越多，以 @Configuration 注解标记配置类会被解析为 `AnnotatedGenericBeanDefinition`；以 @Bean 注解标记的 Bean 会被解析为 `ConfigurationClassBeanDefinition`。

我们新建一个 `MyConfig` 配置类，如下

```java
@Configuration
public class MyConfig {
    @Bean
    Spring spring() {
        return new Spring();
    }
}
```

查看获取到的 BeanDefinition 结果如下：

![image-20210608183025905](https://raw.githubusercontent.com/renjiema/images/main/blogs/20210608183025.png)

而其他 @Service、@Controller、@Repository 以及 @Component 等注解标记的 Bean 则会被识别为 `ScannedGenericBeanDefinition`。

