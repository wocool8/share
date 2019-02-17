# BeanFactory
--
spring的核心是ICO，ICO的核心是容器，容器的基础是BeanFactory
### 一 DefaultListableBeanFactory
![DefaultListableBeanFactory](../../picture/spring/DefaultListableBeanFactory.JPG)

- AliasRegistry : 定义对Alias简单的增删改查操作
    ```java
    public interface AliasRegistry {
    
        /**
         * Given a name, register an alias for it.
         */
        void registerAlias(String name, String alias);
    
        /**
         * Remove the specified alias from this registry.
         */
        void removeAlias(String alias);
    
        /**
         * Determine whether this given name is defines as an alias
         * (as opposed to the name of an actually registered component).
         */
        boolean isAlias(String name);
    
        /**
         * Return the aliases for the given name, if defined.
         */
        String[] getAliases(String name);
    
    }
    ```

- SimpleAliasRegistry : 以map作为alias的缓存，并对AliasRegistry进行实现
    ```java
    public class SimpleAliasRegistry implements AliasRegistry {
    
        /** Map from alias to canonical name. */
        private final Map<String, String> aliasMap = new ConcurrentHashMap<>(16);
        
    }
    ```
- SingletonBeanRegistry : 定义对Singleton的注册及获取
    ```java
    public interface SingletonBeanRegistry {
    
        /**
         * Register the given existing object as singleton in the bean registry,
         * under the given bean name.
          */
        void registerSingleton(String beanName, Object singletonObject);
    
        /**
         * Return the (raw) singleton object registered under the given name.
         */
        @Nullable
        Object getSingleton(String beanName);
    
        /**
         * Check if this registry contains a singleton instance with the given name.
         */
        boolean containsSingleton(String beanName);
    
        /**
         * Return the names of singleton beans registered in this registry.
         */
        String[] getSingletonNames();
    
        /**
         * Return the number of singleton beans registered in this registry.
         */
        int getSingletonCount();
    
        /**
         * Return the singleton mutex used by this registry (for external collaborators).
         */
        Object getSingletonMutex();
    
    }
    ```    
    
### 二 XMLBeanFactory
```java
public class BeanFactoryTest {

    @Test
    public void testSimpleLoad() {
        BeanFactory beanFactory = new XmlBeanFactory(new ClassPathResource("beanFactory.xml"));
        CustomBean customBean = (CustomBean) beanFactory.getBean("customBean");
    }
}
```
XmlBeanFactory在Spring 3.1版本弃用
```java
 /* @see org.springframework.beans.factory.support.DefaultListableBeanFactory
  * @see XmlBeanDefinitionReader
  * @deprecated as of Spring 3.1 in favor of {@link DefaultListableBeanFactory} and
  * {@link XmlBeanDefinitionReader}
  */
 @Deprecated
 @SuppressWarnings({"serial", "all"})
 public class XmlBeanFactory extends DefaultListableBeanFactory {
     
 }
```

### 三 ApplicationContext

```java
public class BeanFactoryTest {

    @Test
    public void testSimpleLoad() {
        ApplicationContext context = new ClassPathXmlApplicationContext("beanFactory.xml");
        CustomBean customBean = (CustomBean) context.getBean("customBean");
    }
}
```
获取bean的过程分为两步，第一步构建BeanFactory，第二步骤从BeanFactory中获取bean