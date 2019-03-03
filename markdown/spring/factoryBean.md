# FactoryBean
---
## FactoryBean的使用
一般情况下spring利用反射实例化对应bean 但是某种情况下用户可以通过实现给定的接口定制实例化bean的逻辑

```java
public interface FactoryBean<T> {

	@Nullable
	T getObject() throws Exception;

	@Nullable
	Class<?> getObjectType();

	default boolean isSingleton() {
		return true;
	}

}
```
用户自定义的类可以通过实现接口 覆写getObject方法自定义bean的实例化方式 
注意：此时getBean方法返回的不是FactoryBean本身 而是FactoryBean.getObject()返回的对象



## Beanfactory的getBean()方法中调用getObjectForBeanInstance获取实例
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

