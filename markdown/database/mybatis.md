# Mybatis
---
![mybatis层次结构](../../picture/mybatis/mybatis.png)
![mybatis层次结构](../../picture/mybatis/level.png)

## 一 Mybatis插件
### 插件是基于拦截四大对象实现
- Executor
- StatementHandler
- ParameterHandler
- ResultSetHandler 


###
```java
public interface Interceptor {

  // mybatis插件重写的核心方法
  Object intercept(Invocation invocation) throws Throwable;
  
  // 生成动态代理对象方法，通常只是串连调用
  Object plugin(Object target);
  
  // mybatis插件在配置时通常需要配置一些参数，这个方法是对参数设置及转换
  void setProperties(Properties properties);

}

```
## 二 插件执行顺序


