# BeanFactory
spring的核心是ICO，ICO的核心是容器，容器的基础是BeanFactory
--
## 常见的BeanFactory

### DefaultListableBeanFactory
![DefaultListableBeanFactory](../../picture/spring/DefaultListableBeanFactory.PNG)
### XMLBeanFactory
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

### ApplicationContext

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