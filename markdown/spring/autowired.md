在看spring的doCreateBean源码时发现源码中有autowireByName的方法
## @Autowried
默认情况下必须要求依赖对象必须存在，如果要允许null值，可以设置它的required属性为false，如：@Autowired(required=false) 
- autowireByName
    ```java
        protected void autowireByName(
                String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {
    
            String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
            for (String propertyName : propertyNames) {
                if (containsBean(propertyName)) {
                    Object bean = getBean(propertyName);
                    pvs.add(propertyName, bean);
                    registerDependentBean(propertyName, beanName);
                    if (logger.isDebugEnabled()) {
                        logger.debug("Added autowiring by name from bean name '" + beanName +
                                "' via property '" + propertyName + "' to bean named '" + propertyName + "'");
                    }
                }
                else {
                    if (logger.isTraceEnabled()) {
                        logger.trace("Not autowiring property '" + propertyName + "' of bean '" + beanName +
                                "' by name: no matching bean found");
                    }
                }
            }
        }
    ```
- autowireByType
    ```java
    protected void autowireByType(
                String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {
    
            TypeConverter converter = getCustomTypeConverter();
            if (converter == null) {
                converter = bw;
            }
    
            Set<String> autowiredBeanNames = new LinkedHashSet<>(4);
            String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
            for (String propertyName : propertyNames) {
                try {
                    PropertyDescriptor pd = bw.getPropertyDescriptor(propertyName);
                    // Don't try autowiring by type for type Object: never makes sense,
                    // even if it technically is a unsatisfied, non-simple property.
                    if (Object.class != pd.getPropertyType()) {
                        MethodParameter methodParam = BeanUtils.getWriteMethodParameter(pd);
                        // Do not allow eager init for type matching in case of a prioritized post-processor.
                        boolean eager = !PriorityOrdered.class.isInstance(bw.getWrappedInstance());
                        DependencyDescriptor desc = new AutowireByTypeDependencyDescriptor(methodParam, eager);
                        Object autowiredArgument = resolveDependency(desc, beanName, autowiredBeanNames, converter);
                        if (autowiredArgument != null) {
                            pvs.add(propertyName, autowiredArgument);
                        }
                        for (String autowiredBeanName : autowiredBeanNames) {
                            registerDependentBean(autowiredBeanName, beanName);
                            if (logger.isDebugEnabled()) {
                                logger.debug("Autowiring by type from bean name '" + beanName + "' via property '" +
                                        propertyName + "' to bean named '" + autowiredBeanName + "'");
                            }
                        }
                        autowiredBeanNames.clear();
                    }
                }
                catch (BeansException ex) {
                    throw new UnsatisfiedDependencyException(mbd.getResourceDescription(), beanName, propertyName, ex);
                }
            }
        }
    ```
## @Qualifier
Spring通常使用@Autowire根据类型自动注入，但是容器中可能存在两个相同类型的不同Bean，此时会抛出
NoUniqueBeanDefinitionException
```java
/**
 * Create a new {@code NoUniqueBeanDefinitionException}.
 * @param type required type of the non-unique bean
 * @param beanNamesFound the names of all matching beans (as a Collection)
 */
public NoUniqueBeanDefinitionException(Class<?> type, Collection<String> beanNamesFound) {
    super(type, "expected single matching bean but found " + beanNamesFound.size() + ": " +
			StringUtils.collectionToCommaDelimitedString(beanNamesFound));
    this.numberOfBeansFound = beanNamesFound.size();
    this.beanNamesFound = beanNamesFound;
}
```
如果我们想使用名称装配可以结合@Qualifier注解进行使用，以下代码来自于spring官方文档

    ```java
    public class MovieRecommender {  
        @Autowired
        @Qualifier("main")
        private MovieCatalog movieCatalog;

        // ...
    }   
    
## @Resource
@Resource是JDK1.6支持的注解， 默认根据名字注入，其次根据类型注入，名称可以通过name属性进行指定，如果没有指定name属性，
当注解写在字段上时，默认取字段名，按照名称查找，如果注解写在setter方法上默认取属性名进行装配。
当找不到与名称匹配的bean时才按照类型进行装配。但是需要注意的是，如果name属性一旦指定，就只会按照名称进行装配


