### 简介
创建代理的方法是postProcessAfterInitialization：如果bean被子类标识为代理，则使用配置的拦截器创建一个代理
代码如下：
```java
@Override  
public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {  
    if (bean != null) {  
       Object cacheKey = getCacheKey(bean.getClass(), beanName);  
       if (this.earlyProxyReferences.remove(cacheKey) != bean) {  
          return wrapIfNecessary(bean, beanName, cacheKey);  
       }  
    }  
    return bean;  
}
```
wrapIfNecessary() 方法主要用于判断是否需要创建代理，如果 Bean 能够获取到 advisor 才需要创建代理
代码如下：
```java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {  
    if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {  
       return bean;  
    }  
    if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {  
       return bean;  
    }  
    if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {  
       this.advisedBeans.put(cacheKey, Boolean.FALSE);  
       return bean;  
    }  
  
    // Create proxy if we have advice.  
    Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);  
    if (specificInterceptors != DO_NOT_PROXY) {  
       this.advisedBeans.put(cacheKey, Boolean.TRUE);  
       Object proxy = createProxy(  
             bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));  
       this.proxyTypes.put(cacheKey, proxy.getClass());  
       return proxy;  
    }  
  
    this.advisedBeans.put(cacheKey, Boolean.FALSE);  
    return bean;  
}
```

### 获取所有的代理

### 创建代理的入口方法

### 依据条件创建代理( jdk 或 cglib)