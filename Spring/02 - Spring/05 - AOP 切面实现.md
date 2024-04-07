### 简介
AOP 是基于 IoC 的 Bean 加载来实现的，通过对应的 AspectJAutoProxyBeanDefinitionParser 来解析
### AnnotationAwareAspectJAutoProxyCreator
注解切面代理创建类（AnnotationAwareAspectJAutoProxyCreator）的类结构关系如下：
![[Pasted image 20240219165117.png]]
实现了两类接口
- BeanFactoryAware
- InstantiationAwareBeanPostProcessor 和 BeanPostProcessor

而 BeanPostProcessor 作用在 bean 的初始化过程中，所以 AOP 的初始化方法也会在 postProcessBeforeInstantiation() 和postProcessAfterInitialization() 中实现

#### postProcessBeforeInstantiation()
主要是处理使用了@Aspect 注解的切面类，然后将切面类的所有切面方法根据使用的注解生成对应Advice，并将Advice连同切入点匹配器和切面类等信息一并封装到Advisor

代码如下：
```java
Object cacheKey = getCacheKey(beanClass, beanName);  
  
if (!StringUtils.hasLength(beanName) || !this.targetSourcedBeans.contains(beanName)) {  
    if (this.advisedBeans.containsKey(cacheKey)) {  
       return null;  
    }  
    if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {  
       this.advisedBeans.put(cacheKey, Boolean.FALSE);  
       return null;  
    }  
}  
  
// Create proxy here if we have a custom TargetSource.  
// Suppresses unnecessary default instantiation of the target bean:  
// The TargetSource will handle target instances in a custom fashion.  
TargetSource targetSource = getCustomTargetSource(beanClass, beanName);  
if (targetSource != null) {  
    if (StringUtils.hasLength(beanName)) {  
       this.targetSourcedBeans.add(beanName);  
    }  
    Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(beanClass, beanName, targetSource);  
    Object proxy = createProxy(beanClass, beanName, specificInterceptors, targetSource);  
    this.proxyTypes.put(cacheKey, proxy.getClass());  
    return proxy;  
}  
  
return null;
```

#### postProcessAfterInitialization()
主要负责将Advisor注入到合适的位置，创建代理（cglib、jdk)，为后面给代理进行增强实现做准备

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