# AOP Proxy
---
## 一 JDK Proxy
JDK动态代理是利用反射机制生成一个实现代理接口的匿名类，在调用具体方法前调用InvokeHandler来处理
```java
    public class JDKProxy implements InvocationHandler {
    	//需要代理的目标对象
    	private Object targetObject;
    	// 构建代理类
    	private Object newProxy(Object targetObject) {
    		this.targetObject = targetObject;
    		return Proxy.newProxyInstance(targetObject.getClass().getClassLoader(), targetObject.getClass().getInterfaces(), this);
    	}
    	public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    		//模拟检查权限
    		checkPopedom();
    		//使用反射 调用代理类
    		Object value = method.invoke(proxy, args);
    		return value;
    	}
    	private void checkPopedom() {
    		System.out.println(".:检查权限 checkPopedom()!");
    	}
    }
```        
## 二 CGLIB Proxy
动态生成一个要代理类的子类，子类重写要代理的类的所有不是final的方法。在子类中采用方法拦截的技术拦截所有父类方法的调用，顺势织入横切逻辑。它比使用java反射的JDK动态代理要快，cglib底层使用字节码处理框架ASM，来转换字节码并生成新的类。不鼓励直接使用ASM，因为它要求你必须对JVM内部结构包括class文件的格式和指令集都很熟悉，由于cglib要生成子类，所以对final类是无法进行代理的
```java
    public class CGLIBProxy implements MethodInterceptor {
        private Object targetObject;
        private Object createProxyObject(Object targetObject) {
            this.targetObject = targetObject;
            Enhancer enhancer = new Enhancer();
            enhancer.setSuperclass(targetObject.getClass());
            enhancer.setCallback(this);
            Object proxyObject = enhancer.create();
            return proxyObject;
        }
    
        public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
            Object object = null;
            if ("addUser".equals(method.getName())) {// 过滤方法
                checkPopedom();// 检查权限
            }
            object = method.invoke(targetObject, objects);
            return object;
        }
        private void checkPopedom() {
            System.out.println(".:检查权限 checkPopedom()!");
        }
    }
```    
## 三 Spring proxy
### 3.1 代理方式选择规则(默认JDK Proxy)

```java
public class DefaultAopProxyFactory implements AopProxyFactory, Serializable {

    public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
        if (!config.isOptimize() && !config.isProxyTargetClass() && !this.hasNoUserSuppliedProxyInterfaces(config)) {
            return new JdkDynamicAopProxy(config);
        } else {
            Class targetClass = config.getTargetClass();
            if (targetClass == null) {
                throw new AopConfigException("TargetSource cannot determine target class: Either an interface or a target is required for proxy creation.");
            } else {
                return (AopProxy)(targetClass.isInterface() ? new JdkDynamicAopProxy(config) : DefaultAopProxyFactory.CglibProxyFactory.createCglibProxy(config));
            }
        }
    }
}    
```
1. 如果目标对象实现了接口，默认情况下会采用JDK的动态代理实现AOP
2. 如果目标对象实现了接口，可以强制使用CGLIB实现AOP
3. 如果目标对象没有实现了接口，必须采用CGLIB库，spring会自动在JDK动态代理和CGLIB之间转换
### 3.2 使用ProxyFactoryBean实现
```java
public class ProxyFactoryBean extends ProxyCreatorSupport implements FactoryBean<Object>, BeanClassLoaderAware, BeanFactoryAware {
    public Object getObject() throws BeansException {
        this.initializeAdvisorChain();
        if (this.isSingleton()) {
            return this.getSingletonInstance();
        } else {
            if (this.targetName == null) {
                this.logger.warn("Using non-singleton proxies with singleton targets is often undesirable. Enable prototype proxies by setting the 'targetName' property.");
            }

            return this.newPrototypeInstance();
        }
    }    
    private synchronized Object getSingletonInstance() {
        if (this.singletonInstance == null) {
            this.targetSource = this.freshTargetSource();
            if (this.autodetectInterfaces && this.getProxiedInterfaces().length == 0 && !this.isProxyTargetClass()) {
                Class targetClass = this.getTargetClass();
                if (targetClass == null) {
                    throw new FactoryBeanNotInitializedException("Cannot determine target class for proxy");
                }

                this.setInterfaces(ClassUtils.getAllInterfacesForClass(targetClass, this.proxyClassLoader));
            }

            super.setFrozen(this.freezeProxy);
            this.singletonInstance = this.getProxy(this.createAopProxy());
        }

        return this.singletonInstance;
    }    
    
    private synchronized Object newPrototypeInstance() {
        if (this.logger.isTraceEnabled()) {
            this.logger.trace("Creating copy of prototype ProxyFactoryBean config: " + this);
        }

        ProxyCreatorSupport copy = new ProxyCreatorSupport(this.getAopProxyFactory());
        TargetSource targetSource = this.freshTargetSource();
        copy.copyConfigurationFrom(this, targetSource, this.freshAdvisorChain());
        if (this.autodetectInterfaces && this.getProxiedInterfaces().length == 0 && !this.isProxyTargetClass()) {
            copy.setInterfaces(ClassUtils.getAllInterfacesForClass(targetSource.getTargetClass(), this.proxyClassLoader));
        }

        copy.setFrozen(this.freezeProxy);
        if (this.logger.isTraceEnabled()) {
            this.logger.trace("Using ProxyCreatorSupport copy: " + copy);
        }

        return this.getProxy(copy.createAopProxy());
    }    
}
```

### 3.2 CGLIB代理方式配置
#### 3.2.1 xml
在xml中配置如下标签
```xml
<aop:aspectj-autoproxy proxy-target-class="true"> 
```    
#### 3.2.1 springboot
在application.properties或者application.yml去设置如下属性
```yml
// application.yml
spring:
     aop:
         proxy-target-class: true
```           
```properties
# application.properties
spring.aop.proxy-target-class=true
``` 
## 四 FAQ
### 4.1 final修饰的类为什么不能使用CGLIB代理
由于CGLIB动态代理的底层实现是ASM，理论上是可以代理final类的，但是CGLIB实现[字节码增强](/markdown/jvm/bytecode.md)是使用Enhancer生成派生子类字节码
所以才不能代理final修饰类
           

