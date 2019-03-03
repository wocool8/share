# bean的加载

## spring的入门实例

``` java
public class BeanFactoryTest {
  @Test
  public void testSimpleLoad() {
    BeanFactory bf = new XmlBeanFactory(new ClassPathResource("beanFactory.xml"));
    MyBean b = (MyBean) bf.getBean("myBean");
    // bean的调用
  }
  // 注意现在多用applicationContext 取代BeanFactory的用法 功能更全
}
```  
## BeanFactory的getBean的实现
```java
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
    @Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {

  // 提取bean名
  final String beanName = transformedBeanName(name);
  Object bean;

  // 检查cache中或者singletonBeanFactory中是否有相应的实例
  //
  // Eagerly check singleton cache for manually registered singletons.
  Object sharedInstance = getSingleton(beanName);
  if (sharedInstance != null && args == null) {
    if (logger.isDebugEnabled()) {
      if (isSingletonCurrentlyInCreation(beanName)) {
        logger.debug("Returning eagerly cached instance of singleton bean '" + beanName +
            "' that is not fully initialized yet - a consequence of a circular reference");
      }
      else {
        logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
      }
    }
    // 返回对应的实例！
    bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
  }

  else {
    // 对于bean如果是原型模式 不会尝试去解决循环依赖 直接抛异常
    // 循环依赖： A中有B的属性 B中有A的属性 当依赖注入时 就会产生当A还未创建完的时候因为对于b的创建再次创建A 造成循环依赖
    // Fail if we're already creating this bean instance:
    // We're assumably within a circular reference.
    if (isPrototypeCurrentlyInCreation(beanName)) {
      throw new BeanCurrentlyInCreationException(beanName);
    }

    // 首先尝试从parentBeanFactory 中获取bean的实例 也是先从缓存中取
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
      else {
        // No args -> delegate to standard getBean method.
        return parentBeanFactory.getBean(nameToLookup, requiredType);
      }
    }

    //如果不是仅仅做类型检查 需要记录
    if (!typeCheckOnly) {
      markBeanAsCreated(beanName);
    }

    // 尝试通过bean名得到RootBeanDefinition 如果要获取的bean是子bean则合并父类的相应属性
    try {
      final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
      checkMergedBeanDefinition(mbd, beanName, args);

      // Guarantee initialization of beans that the current bean depends on.
      String[] dependsOn = mbd.getDependsOn();
      if (dependsOn != null) {
        // 递归实例化依赖的bean
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

      // Create bean instance.开始实例化
      if (mbd.isSingleton()) { // 单例模式的创建
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

      else if (mbd.isPrototype()) { // 原型模式的创建
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

      // 通过制定scope的方式实例化
      else {
```         
[通过自定义scope的方式实例化](/markdown/spring/beanScope.md)
```java 
      } catch (IllegalStateException ex) {
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

  // Check if required type matches the type of the actual bean instance. 类型检查
  if (requiredType != null && !requiredType.isInstance(bean)) {
    try {
      T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
      if (convertedBean == null) {
        throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
      }
      return convertedBean;
    }
    catch (TypeMismatchException ex) {
      if (logger.isDebugEnabled()) {
        logger.debug("Failed to convert bean '" + name + "' to required type '" +
            ClassUtils.getQualifiedName(requiredType) + "'", ex);
      }
      throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
    }
  }
  return (T) bean;
}
```
代码中涉及到的加载过程：

- 转换bean名   
  1. 去除[FactoryBean](/markdown/spring/factoryBean.md)的修饰符
  2. 如果是通过alias指定的bean名 取到最终bean名
- 尝试从缓存中加载bean实例 如果加载不成功则从singletonFactories中加载 原因：避免循环依赖
spring中创建bean的原则是不等bean创建完成就将创建bean的ObjectFactory提前放到缓存中，一旦下一个bean创建时需要依赖上一个bean则直接使用ObjectFactory
- bean的实例化 getObjectForBeanInstance 最重要的方法
- 对于原因模式需要进行依赖检查
- 如果缓存中没有数据只能到父类工厂中去加载
- 将GenericBeanDefinition转换成RootBeanDefinition
- 先加载其依赖
- 针对不同的[scope](/markdown/spring/beanScope.md)进行bean的创建
- 类型转换  将返回的bean转换成用户需要的类型 requiredType决定

### 缓存中获取单例bean
```java
@Nullable
	protected Object getSingleton(String beanName, boolean allowEarlyReference) {
		Object singletonObject = this.singletonObjects.get(beanName);
		if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
			synchronized (this.singletonObjects) {
				singletonObject = this.earlySingletonObjects.get(beanName);
				if (singletonObject == null && allowEarlyReference) {
					ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
					if (singletonFactory != null) {
						singletonObject = singletonFactory.getObject();
						this.earlySingletonObjects.put(beanName, singletonObject);
						this.singletonFactories.remove(beanName);
					}
				}
			}
		}
		return singletonObject;
	}
```

实现的逻辑大致先从缓存（map）中拿，再从earlySingletonObjects中拿 如果还没有就通过singletonFactory.getObejct()拿  基本就是反射, 然后放缓存

### getObjectForBeanInstance
返回的工厂bean中定义的factory-method方法中返回的bean
首先是一些准备工作：

- 对FactoryBean的正确性进行验证
- 对非FactoryBean不做任何处理
- 对bean进行转换
- 将Factory中解析bean的操作委托个getObjectFromFactoryBean 然后又委托给doGetObjectFromFactoryBean

所以真正的实现在doGetObjectFromFactoryBean中
```java
private Object doGetObjectFromFactoryBean(final FactoryBean<?> factory, final String beanName)
    throws BeanCreationException {

  Object object;
  try {
    if (System.getSecurityManager() != null) {
      AccessControlContext acc = getAccessControlContext();
      try {
        object = AccessController.doPrivileged((PrivilegedExceptionAction<Object>) factory::getObject, acc);
      }
      catch (PrivilegedActionException pae) {
        throw pae.getException();
      }
    }
    else {
      object = factory.getObject();
    }
  }
  catch (FactoryBeanNotInitializedException ex) {
    throw new BeanCurrentlyInCreationException(beanName, ex.toString());
  }
  catch (Throwable ex) {
    throw new BeanCreationException(beanName, "FactoryBean threw exception on object creation", ex);
  }

  // Do not accept a null value for a FactoryBean that's not fully
  // initialized yet: Many FactoryBeans just return null then.
  if (object == null) {
    if (isSingletonCurrentlyInCreation(beanName)) {
      throw new BeanCurrentlyInCreationException(
          beanName, "FactoryBean which is currently in creation returned null from getObject");
    }
    object = new NullBean();
  }
  return object;
}
```
其实就是调用FactoryBean的getObject方法 没错 就是上面讲的那个 可以自定义的内个
但是 getObjectFromFactoryBean这个方法在调用后并没有直接返回 又做了一些后处理

### 获取单例getSingleton
具体代码不贴了 主要的逻辑

- 锁住 检查缓存是否加载过
- 如果没加载 记录beanName为正在加载 据说便可以对循环依赖进行检测 ？ 不懂
- 调用ObjectFactory的getObject方法实例化bean(老套路)
- 进行后处理 移除缓存中正在加载的状态
- 将加载的结果加缓存

### singletonFactory中定义的getObject方法的实现
真正的实现 return createBean(beanName, mbd, args); 在这里

```java
@Override
	protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {

		if (logger.isDebugEnabled()) {
			logger.debug("Creating instance of bean '" + beanName + "'");
		}
		RootBeanDefinition mbdToUse = mbd;

		// Make sure bean class is actually resolved at this point, and
		// clone the bean definition in case of a dynamically resolved Class
		// which cannot be stored in the shared merged bean definition.
		Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
		if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
			mbdToUse = new RootBeanDefinition(mbd);
			mbdToUse.setBeanClass(resolvedClass);
		}

		// Prepare method overrides.
		try {
			mbdToUse.prepareMethodOverrides();
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
					beanName, "Validation of method overrides failed", ex);
		}

		try {
			// Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
			Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
			if (bean != null) {
				return bean;
			}
		}
		catch (Throwable ex) {
			throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
					"BeanPostProcessor before instantiation of bean failed", ex);
		}

		try {
			Object beanInstance = doCreateBean(beanName, mbdToUse, args);
			if (logger.isDebugEnabled()) {
				logger.debug("Finished creating instance of bean '" + beanName + "'");
			}
			return beanInstance;
		}
		catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
			// A previously detected exception with proper bean creation context already,
			// or illegal singleton state to be communicated up to DefaultSingletonBeanRegistry.
			throw ex;
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
		}
	}
```
这个方法主要做的事情：

- 根据设置的class属性或者classname解析class 如果getClass有东西就直接返回 没有就找到classLoader然后反射的到class
- 对override属性进行标记 验证
prepareMethodOverride bean在实例化的时候如果检测存在methodOverrides属性 会动态的位当前bean生成代理并使用相应的拦截去为bean做增强处理 这个地方只是检车并标记
- 实例化bean的前置处理 resolveBeforeInstantiation，[基于BeanPostProcessor实现的AOP](/markdown/spring/beanLifecycle.md)是基础此处判断
返回代理类



