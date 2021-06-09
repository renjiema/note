# 深入分析 FactoryBean

## 1.FactoryBean示例

FactoryBean 和 BeanFactory虽然只是因为单词前后颠倒了一下，但功能完全不一样，下面通过一个demo来了解FactoryBean的作用

创建一个UserService如下：

```java
public class UserService {
	public String name() {
		return "user service";
	}
}
```

然后再创建一个UserService的工厂类：

```java
public class UserServiceFactoryBean implements FactoryBean<UserService> {
	@Override
	public UserService getObject() throws Exception {
		return new UserService();
	}

	@Override
	public Class<?> getObjectType() {
		return UserService.class;
	}

	@Override
	public boolean isSingleton() {
		return FactoryBean.super.isSingleton();
	}
}
```

然后通过代码注入UserServiceFactoryBean到IoC容器中：

```java
@Test
public void testFactoryBean() {
	final DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
	final GenericBeanDefinition beanDefinition = new GenericBeanDefinition();
	beanDefinition.setBeanClass(UserServiceFactoryBean.class);
	factory.registerBeanDefinition("userService", beanDefinition);
	final Object userService = factory.getBean("userService");
	System.out.println(userService.getClass());
}
```

按照正常Bean注册，这里获取到的 Bean 应该是UserServiceFactoryBean的实例，但是实际打印结果如下：

```
class factory.bean.test.UserService
```

具体原因需要从FactoryBean接口开始分析。

## 2.FactoryBean 接口

FactoryBean 在 Spring 框架中具有重要地位，Spring 框架本身也为此提供了多种不同的实现类(未完全展示)：

![image-20210609181142693](https://raw.githubusercontent.com/renjiema/images/main/blogs/20210609181149.png)

从 Spring3.0 开始，FactoryBean 开始支持泛型，就是大家所看到的 FactoryBean形式，FactoryBean接口中一共有三个方法：

```java
public interface FactoryBean<T> {

	String OBJECT_TYPE_ATTRIBUTE = "factoryBeanObjectType";

	@Nullable
	T getObject() throws Exception;
	@Nullable
	Class<?> getObjectType();
	default boolean isSingleton() {
		return true;
	}
}
```

- getObject：该方法返回 FactoryBean 所创建的实例，如果在 XML 配置文件中，我们提供的 class 是一个 FactoryBean 的话，那么当我们调用 getBean 方法去获取实例时，最终获取到的是 getObject 方法的返回值。
- getObjectType：返回对象的类型。
- isSingleton：getObject 方法所返回的对象是否单例。

所以当我们调用 getBean 方法去获取 Bean 实例的时候，实际上获取到的是 getObject 方法的返回值，那如何返回UserServiceFactoryBean的实例呢？客通古如下代码：

```java
@Test
public void testFactoryBean() {
	final DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
	final GenericBeanDefinition beanDefinition = new GenericBeanDefinition();
	beanDefinition.setBeanClass(UserServiceFactoryBean.class);
	factory.registerBeanDefinition("userService", beanDefinition);
	final Object userService = factory.getBean("&userService");
	System.out.println(userService.getClass());
}
```

只需要在 Bean 的名字前面加上一个 & 符号，获取到的就是对应的FactoryBean 实例了。

## 3.源码分析

在 DefaultListableBeanFactory 中有一个很重要的方法：`preInstantiateSingletons()`，改可以提前注册 Bean，而且ApplicationContext接口创建后也是通过该方法实现提前注册，该方法是在 ConfigurableListableBeanFactory 接口中声明的，DefaultListableBeanFactory 类实现了 ConfigurableListableBeanFactory 接口并实现了接口中的方法：

```java
@Override
public void preInstantiateSingletons() throws BeansException {
   if (logger.isTraceEnabled()) {
      logger.trace("Pre-instantiating singletons in " + this);
   }
   // Iterate over a copy to allow for init methods which in turn register new bean definitions.
   // While this may not be part of the regular factory bootstrap, it does otherwise work fine.
   List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);
   // Trigger initialization of all non-lazy singleton beans...
   for (String beanName : beanNames) {
      RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
      if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
         if (isFactoryBean(beanName)) {
            Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
            if (bean instanceof FactoryBean) {
               FactoryBean<?> factory = (FactoryBean<?>) bean;
               boolean isEagerInit;
               if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
                  isEagerInit = AccessController.doPrivileged(
                        (PrivilegedAction<Boolean>) ((SmartFactoryBean<?>) factory)::isEagerInit,
                        getAccessControlContext());
               }
               else {
                  isEagerInit = (factory instanceof SmartFactoryBean &&
                        ((SmartFactoryBean<?>) factory).isEagerInit());
               }
               if (isEagerInit) {
                  getBean(beanName);
               }
            }
         }
         else {
            getBean(beanName);
         }
      }
   }
   // Trigger post-initialization callback for all applicable beans...
   for (String beanName : beanNames) {
      Object singletonInstance = getSingleton(beanName);
      if (singletonInstance instanceof SmartInitializingSingleton) {
         SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
         if (System.getSecurityManager() != null) {
            AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
               smartSingleton.afterSingletonsInstantiated();
               return null;
            }, getAccessControlContext());
         }
         else {
            smartSingleton.afterSingletonsInstantiated();
         }
      }
   }
}
```

该方法在 [ApplicationContext容器初始化](https://github.com/renjiema/note/blob/0f5b8f52685ac07cb0c60fdc93afff94b684ee11/java/spring/ApplicationContext%E5%AE%B9%E5%99%A8%E5%88%9D%E5%A7%8B%E5%8C%96.md#L1322)中有分析，`preInstantiateSingletons`方法的整体逻辑比较简单，就是遍历 beanNames，对符合条件的 Bean 进行实例化，这里所谓的提前初始化其实就是在我们调用 getBean 方法之前，它自己先调用了一下 getBean。

这里有几个比较关键的点。

第一个就是 isFactoryBean 方法的调用是根据 beanName 去获取 Bean 实例，进而判断是不是一个 FactoryBean，如果没有 Bean 实例（还没创建出来），则根据 BeanDefinition 去判断是不是一个 FactoryBean。

如果是 FactoryBean，则在 getBean 时自动加上了 FACTORY_BEAN_PREFIX 前缀，这个常量其实就是 &，这样获取到的实例实际上就是 FactoryBean 的实例。获取到 FactoryBean 实例之后，接下来判断是否需要在容器启动阶段，调用 getObject 方法初始化 Bean，如果 isEagerInit 为 true，则去初始化。

按照我们前面的定义，这里获取到的 isEagerInit 属性为 false，即不提前加载 Bean，而是在开发者手动调用 getBean 的方法的时候才去加载。如果希望这里能够提前加载，需要重新定义 UserServiceFactoryBean，并使之实现 SmartFactoryBean 接口，如下：

```java
public class UserServiceFactoryBean implements SmartFactoryBean<UserService> {
	@Override
	public UserService getObject() throws Exception {
		return new UserService();
	}
	@Override
	public Class<?> getObjectType() {
		return UserService.class;
	}
	@Override
	public boolean isSingleton() {
		return true;
	}
	@Override
	public boolean isEagerInit() {
		return true;
	}
}
```

实现了 SmartFactoryBean 接口之后，重写 isEagerInit 方法并返回 true,在容器启动时，就会提前调用 getBean 方法完成 Bean 的加载。

接下来我们来看 getBean 方法。

getBean 方法首先调用 AbstractBeanFactory#doGetBean，在该方法中，又会调用到 AbstractBeanFactory#getObjectForBeanInstance 方法：

```java
public abstract class AbstractBeanFactory extends FactoryBeanRegistrySupport implements ConfigurableBeanFactory {
	@Override
	public Object getBean(String name) throws BeansException {
		return doGetBean(name, null, null, false);
	}
	protected <T> T doGetBean(
			String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly)
			throws BeansException {
		String beanName = transformedBeanName(name);
		Object bean;
		// Eagerly check singleton cache for manually registered singletons.
		Object sharedInstance = getSingleton(beanName);
		if (sharedInstance != null && args == null) {
			...
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}
		else {
			// Fail if we're already creating this bean instance:
			// We're assumably within a circular reference.
			if (isPrototypeCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(beanName);
			}
			// Check if bean definition exists in this factory.
			BeanFactory parentBeanFactory = getParentBeanFactory();
			if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
				// Not found -> check parent.
				String nameToLookup = originalBeanName(name);
				if (parentBeanFactory instanceof AbstractBeanFactory) {
					return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
							nameToLookup, requiredType, args, typeCheckOnly);
				}
				else if (args != null) {
					// Delegation to parent with explicit args.
					return (T) parentBeanFactory.getBean(nameToLookup, args);
				}
				else if (requiredType != null) {
					// No args -> delegate to standard getBean method.
					return parentBeanFactory.getBean(nameToLookup, requiredType);
				}
				else {
					return (T) parentBeanFactory.getBean(nameToLookup);
				}
			}
			if (!typeCheckOnly) {
				markBeanAsCreated(beanName);
			}
			try {
				RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				checkMergedBeanDefinition(mbd, beanName, args);
				// Guarantee initialization of beans that the current bean depends on.
				String[] dependsOn = mbd.getDependsOn();
				if (dependsOn != null) {
					for (String dep : dependsOn) {
						if (isDependent(beanName, dep)) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
						}
						registerDependentBean(dep, beanName);
						try {
							getBean(dep);
						}
						catch (NoSuchBeanDefinitionException ex) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"'" + beanName + "' depends on missing bean '" + dep + "'", ex);
						}
					}
				}
				// Create bean instance.
				if (mbd.isSingleton()) {
					sharedInstance = getSingleton(beanName, () -> {
						try {
							return createBean(beanName, mbd, args);
						}
						catch (BeansException ex) {
							// Explicitly remove instance from singleton cache: It might have been put there
							// eagerly by the creation process, to allow for circular reference resolution.
							// Also remove any beans that received a temporary reference to the bean.
							destroySingleton(beanName);
							throw ex;
						}
					});
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}
				else if (mbd.isPrototype()) {
					// It's a prototype -> create a new instance.
					Object prototypeInstance = null;
					try {
						beforePrototypeCreation(beanName);
						prototypeInstance = createBean(beanName, mbd, args);
					}
					finally {
						afterPrototypeCreation(beanName);
					}
					bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
				}
				else {
					String scopeName = mbd.getScope();
					if (!StringUtils.hasLength(scopeName)) {
						throw new IllegalStateException("No scope name defined for bean ´" + beanName + "'");
					}
					Scope scope = this.scopes.get(scopeName);
					if (scope == null) {
						throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
					}
					try {
						Object scopedInstance = scope.get(beanName, () -> {
							beforePrototypeCreation(beanName);
							try {
								return createBean(beanName, mbd, args);
							}
							finally {
								afterPrototypeCreation(beanName);
							}
						});
						bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
					}
					catch (IllegalStateException ex) {
						throw new BeanCreationException(beanName,
								"Scope '" + scopeName + "' is not active for the current thread; consider " +
								"defining a scoped proxy for this bean if you intend to refer to it from a singleton",
								ex);
					}
				}
			}
			catch (BeansException ex) {
				cleanupAfterBeanCreationFailure(beanName);
				throw ex;
			}
		}
		// Check if required type matches the type of the actual bean instance.
		if (requiredType != null && !requiredType.isInstance(bean)) {
			try {
				T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
				if (convertedBean == null) {
					throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
				}
				return convertedBean;
			}
			catch (TypeMismatchException ex) {
				if (logger.isTraceEnabled()) {
					logger.trace("Failed to convert bean '" + name + "' to required type '" +
							ClassUtils.getQualifiedName(requiredType) + "'", ex);
				}
				throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
			}
		}
		return (T) bean;
	}
	protected Object getObjectForBeanInstance(
			Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {

		// Don't let calling code try to dereference the factory if the bean isn't a factory.
		if (BeanFactoryUtils.isFactoryDereference(name)) {
			if (beanInstance instanceof NullBean) {
				return beanInstance;
			}
			if (!(beanInstance instanceof FactoryBean)) {
				throw new BeanIsNotAFactoryException(beanName, beanInstance.getClass());
			}
			if (mbd != null) {
				mbd.isFactoryBean = true;
			}
			return beanInstance;
		}

		// Now we have the bean instance, which may be a normal bean or a FactoryBean.
		// If it's a FactoryBean, we use it to create a bean instance, unless the
		// caller actually wants a reference to the factory.
		if (!(beanInstance instanceof FactoryBean)) {
			return beanInstance;
		}

		Object object = null;
		if (mbd != null) {
			mbd.isFactoryBean = true;
		}
		else {
			object = getCachedObjectForFactoryBean(beanName);
		}
		if (object == null) {
			// Return bean instance from factory.
			FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
			// Caches object obtained from FactoryBean if it is a singleton.
			if (mbd == null && containsBeanDefinition(beanName)) {
				mbd = getMergedLocalBeanDefinition(beanName);
			}
			boolean synthetic = (mbd != null && mbd.isSynthetic());
			object = getObjectFromFactoryBean(factory, beanName, !synthetic);
		}
		return object;
	}
}
```

