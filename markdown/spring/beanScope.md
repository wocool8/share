## Bean Scope 
---
### 作用域
|scope|描述|
|:-|:-|
|singleton|IOC容器仅创建一个Bean实例，IOC容器每次返回的是同一个Bean实例|
|prototype|IOC容器可以创建多个Bean实例，每次返回的都是一个新的实例|
|request|仅对HTTP请求产生作用，每次HTTP请求都会创建一个新的Bean，仅适用于WebApplicationContext|
|session|仅用于HTTP Session，同一个Session共享一个Bean实例。仅适用于WebApplicationContext|
|application|一个ServletContext共享一个Bean实例仅适用于WebApplicationContext|
|websocket|一个websocket共享一个Bean实例|仅适用于WebApplicationContext|

- WebApplicationContext中定义的scope类型如下
    ```java
    public interface WebApplicationContext extends ApplicationContext {
        String ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE = WebApplicationContext.class.getName() + ".ROOT";
        String SCOPE_REQUEST = "request";
        String SCOPE_SESSION = "session";
        String SCOPE_GLOBAL_SESSION = "globalSession";
        String SCOPE_APPLICATION = "application";
    }
    ```

- spring-web包中定义了@SessionScope、@RequestScope、@ApplicationScope
    ```java
    @Target({ElementType.TYPE, ElementType.METHOD})
    @Retention(RetentionPolicy.RUNTIME)
    @Documented
    @Scope("session")
    public @interface SessionScope {
        @AliasFor(
            annotation = Scope.class
        )
        ScopedProxyMode proxyMode() default ScopedProxyMode.TARGET_CLASS;
    }
    ```
### 自定义Scope
- 实现 org.springframework.beans.factory.config.Scope
    ```java
    public class CustomScope implements org.springframework.beans.factory.config.Scope {
        @Override
        public Object get(String s, ObjectFactory<?> objectFactory) {
            return null;
        }
    
        @Override
        public Object remove(String s) {
            return null;
        }
    
        @Override
        public void registerDestructionCallback(String s, Runnable runnable) {
    
        }
    
        @Override
        public Object resolveContextualObject(String s) {
            return null;
        }
    
        @Override
        public String getConversationId() {
            return null;
        }
    }
    ```
- 注册自定义Scope是通过ConfigurableBeanFactory的registerScope()方法进行的

    ```java
    ConfigurableApplicationContext ctx = new ClassPathXmlApplicationContext("beans.xml");
    ctx.getBeanFactory().registerScope("custom", new CustomScope());
    ```

由于 CustomScopeConfigurer 实现了 BeanFactoryPostProcessor，SpringIoC容器允许BeanFactoryPostProcessor
在容器实例化任何bean之前读取bean的定义(配置元数据)，并可以修改它

```java
public class CustomScopeConfigurer implements BeanFactoryPostProcessor, BeanClassLoaderAware, Ordered {

    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        if (this.scopes != null) {
            Iterator var2 = this.scopes.entrySet().iterator();

            while(var2.hasNext()) {
                Entry<String, Object> entry = (Entry)var2.next();
                String scopeKey = (String)entry.getKey();
                Object value = entry.getValue();
                if (value instanceof Scope) {
                    beanFactory.registerScope(scopeKey, (Scope)value);
                } else {
                    Class scopeClass;
                    if (value instanceof Class) {
                        scopeClass = (Class)value;
                        Assert.isAssignable(Scope.class, scopeClass);
                        beanFactory.registerScope(scopeKey, (Scope)BeanUtils.instantiateClass(scopeClass));
                    } else {
                        if (!(value instanceof String)) {
                            throw new IllegalArgumentException("Mapped value [" + value + "] for scope key [" + scopeKey + "] is not an instance of required type [" + Scope.class.getName() + "] or a corresponding Class or String value indicating a Scope implementation");
                        }

                        scopeClass = ClassUtils.resolveClassName((String)value, this.beanClassLoader);
                        Assert.isAssignable(Scope.class, scopeClass);
                        beanFactory.registerScope(scopeKey, (Scope)BeanUtils.instantiateClass(scopeClass));
                    }
                }
            }
        }

    }
}
```