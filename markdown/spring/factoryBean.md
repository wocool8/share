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
用户自定义的类可以通过实现接口 覆写getObject方法自定义bean的实例化方式 注意：此时getBean方法返回的不是FactoryBean本身 而是FactoryBean.getObject()返回的对象




