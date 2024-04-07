### 简介
IoC 容器就是具有依赖注入功能的容器。IoC 容器负责实例化、定位、配置应用程序中的对象及建立这些对象间的依赖。应用程序无需直接在代码中 new 相关的对象，应用程序由 IoC 容器进行组装。在 Spring 中 BeanFactory 是 IoC 容器的实际代表者。

### BeanFactory 和 ApplicationContext
在 Spring 中，有两种 IoC 容器：
- *BeanFactory*：BeanFactory 是 Spring 基础 IoC 容器，提供了 Spring 容器的配置框架和基本功能
- *ApplicationContext*：ApplicationContext 是具备应用特性的 BeanFactory 的子接口。同时还扩展了其他接口来支持更丰富的功能，如：国际化、访问资源、事件机制、更方便的支持 AOP、在 web 应用中指定应用层上下文等
实际开发中，更推荐使用 `ApplicationContext` 作为 IoC 容器，因为它的功能远多于 `BeanFactory`

#### BeanFactory
BeanFactory 作为最顶层的接口，它定义了 IoC 容器的基本功能规范
```java
public interface BeanFactory {    
      
    //用于取消引用实例并将其与FactoryBean创建的bean区分开来。例如，如果命名的bean是FactoryBean，则获取将返回Factory，而不是Factory返回的实例。
    String FACTORY_BEAN_PREFIX = "&"; 
        
    //根据bean的名字和Class类型等来得到bean实例    
    Object getBean(String name) throws BeansException;    
    Object getBean(String name, Class requiredType) throws BeansException;    
    Object getBean(String name, Object... args) throws BeansException;
    <T> T getBean(Class<T> requiredType) throws BeansException;
    <T> T getBean(Class<T> requiredType, Object... args) throws BeansException;

    //返回指定bean的Provider
    <T> ObjectProvider<T> getBeanProvider(Class<T> requiredType);
    <T> ObjectProvider<T> getBeanProvider(ResolvableType requiredType);

    //检查工厂中是否包含给定name的bean，或者外部注册的bean
    boolean containsBean(String name);

    //检查所给定name的bean是否为单例/原型
    boolean isSingleton(String name) throws NoSuchBeanDefinitionException;
    boolean isPrototype(String name) throws NoSuchBeanDefinitionException;

    //判断所给name的类型与type是否匹配
    boolean isTypeMatch(String name, ResolvableType typeToMatch) throws NoSuchBeanDefinitionException;
    boolean isTypeMatch(String name, Class<?> typeToMatch) throws NoSuchBeanDefinitionException;

    //获取给定name的bean的类型
    @Nullable
    Class<?> getType(String name) throws NoSuchBeanDefinitionException;

    //返回给定name的bean的别名
    String[] getAliases(String name);
     
}
```
为了区分在 Spring 内部对象的传递和转化过程中的对象数据访问的限制
BeanFactory 定义了多个子类：
- ListableBeanFactory：该接口定义了访问容器中 Bean 基本信息的若干方法，如查看Bean 的个数、获取某一类型 Bean 的配置名、查看容器中是否包括某一 Bean 等方法
- HierarchicalBeanFactory：父子级联 IoC 容器的接口，子容器可以通过接口方法访问父容器； 通过 HierarchicalBeanFactory 接口， Spring 的 IoC 容器可以建立父子层级关联的容器体系，子容器可以访问父容器中的 Bean，但父容器不能访问子容器的 Bean。Spring 使用父子容器实现了很多功能，比如在 Spring MVC 中，展现层 Bean 位于一个子容器中，而业务层和持久层的 Bean 位于父容器中。这样，展现层 Bean 就可以引用业务层和持久层的 Bean，而业务层和持久层的 Bean 则看不到展现层的 Bean
- ConfigurableBeanFactory：是一个重要的接口，增强了 IoC 容器的可定制性，它定义了设置类装载器、属性编辑器、容器初始化后置处理器等方法
- ConfigurableListableBeanFactory：ListableBeanFactory 和 ConfigurableBeanFactory的融合
- AutowireCapableBeanFactory：定义了将容器中的 Bean 按某种规则（如按名字匹配、按类型匹配等）进行自动装配的方法

#### ApplicationContext
ApplicationContext 的整体结构如下：
![[Pasted image 20240123164409.png]]
- HierarchicalBeanFactory 和 ListableBeanFactory： ApplicationContext 继承了 HierarchicalBeanFactory 和 ListableBeanFactory 接口，在此基础上，还通过多个其他的接口扩展了 BeanFactory 的功能
- ApplicationEventPublisher：让容器拥有发布应用上下文事件的功能，包括容器启动事件、关闭事件等。实现了 ApplicationListener 事件监听接口的 Bean 可以接收到容器事件 ， 并对事件进行响应处理 。 在 ApplicationContext 抽象实现类AbstractApplicationContext 中，我们可以发现存在一个 ApplicationEventMulticaster，它负责保存所有监听器，以便在容器产生上下文事件时通知这些事件监听者
- MessageSource：为应用提供 i18n 国际化消息访问的功能
- ResourcePatternResolver ： 所有 ApplicationContext 实现类都实现了类似于PathMatchingResourcePatternResolver 的功能，可以通过带前缀的 Ant 风格的资源文件路径装载 Spring 的配置文件
- LifeCycle：该接口是 Spring 2.0 加入的，该接口提供了 start()和 stop()两个方法，主要用于控制异步处理过程。在具体使用时，该接口同时被 ApplicationContext 实现及具体 Bean 实现， ApplicationContext 会将 start/stop 的信息传递给容器中所有实现了该接口的 Bean，以达到管理和控制 JMX、任务调度等目的

ApplicationContext 接口的实现如下：![[Pasted image 20240123170003.png]]
根据是否需要 refresh 容器衍生出两个抽象类：
- GenericApplicationContext： 是初始化的时候就创建容器，往后的每次refresh都不会更改
- AbstractRefreshableApplicationContext： AbstractRefreshableApplicationContext及子类的每次refresh都是先清除已有(如果不存在就创建)的容器，然后再重新创建，AbstractRefreshableApplicationContext及子类无法做到GenericApplicationContext混合搭配从不同源头获取bean的定义信息

根据不同的 Bean 配置方式有着 不同的资源加载方式，这衍生了很多种实现类，ApplicationContext 的常用实现类：
- AnnotationConfigApplicationContext：从一个或多个基于java的配置类中加载上下文定义，适用于java注解的方式。
- ClassPathXmlApplicationContext：从类路径下的一个或多个xml配置文件中加载上下文定义，适用于xml配置的方式。
- FileSystemXmlApplicationContext：从文件系统下的一个或多个xml配置文件中加载上下文定义，也就是说系统盘符中加载xml配置文件
- AnnotationConfigWebApplicationContext：专门为web应用准备的，适用于注解方式
- XmlWebApplicationContext ：从web应用下的一个或多个xml配置文件加载上下文定义，适用于xml配置方式
### IoC 容器结构设计
IoC 容器的整体功能如下：
![[Pasted image 20240122164715.png]]
- 加载 Bean 的配置（xml配置）
- 根据 Bean 的定义加载生成 Bean 的实例，并放入Bean 容器（依赖注入、嵌套、缓存）
- 针对企业级业务的特殊 Bean（国际化、Event）
- 对容器中的 Bean 进行统一的管理和调用（工厂模式管理）
### IoC 容器初始化
容器初始化过程就是加载，解析，生成 BeanDefinition 并注册到 IoC 容器中
以 xml 配置为例，具体流程如下图：![[Pasted image 20240123155852.png]]
#### 初始化的入口
配置 IoC 容器的方式有多种，如：xml 配置、注解配置、Java 配置，不同的配置方式会选用不同的 ApplicationConext
通过 SpringApplication.run() 方法，可以得知
Spring 会根据 webApplicationType 字段的值来确定加载哪一个 ApplicationContext![[Pasted image 20240123170915.png]]
然后就对 容器 继续相关设置![[Pasted image 20240123171642.png]]

#### 加载 Bean Definition 
IoC 容器初始化的主体流程是 Bean Definition 的解析和加载，这个过程 在 AbstractApplicationContext.refresh() 方法里实现
```java
public void refresh() throws BeansException, IllegalStateException {  
    synchronized(this.startupShutdownMonitor) {  
        StartupStep contextRefresh = this.applicationStartup.start("spring.context.refresh");  
        this.prepareRefresh();  
        ConfigurableListableBeanFactory beanFactory = this.obtainFreshBeanFactory();  
        this.prepareBeanFactory(beanFactory);  
  
        try {  
            this.postProcessBeanFactory(beanFactory);  
            StartupStep beanPostProcess = this.applicationStartup.start("spring.context.beans.post-process");  
            this.invokeBeanFactoryPostProcessors(beanFactory);  
            this.registerBeanPostProcessors(beanFactory);  
            beanPostProcess.end();  
            this.initMessageSource();  
            this.initApplicationEventMulticaster();  
            this.onRefresh();  
            this.registerListeners();  
            this.finishBeanFactoryInitialization(beanFactory);  
            this.finishRefresh();  
        } catch (BeansException var10) {  
            if (this.logger.isWarnEnabled()) {  
                this.logger.warn("Exception encountered during context initialization - cancelling refresh attempt: " + var10);  
            }  
  
            this.destroyBeans();  
            this.cancelRefresh(var10);  
            throw var10;  
        } finally {  
            this.resetCommonCaches();  
            contextRefresh.end();  
        }  
  
    }  
}
```
refresh() 方法是一个模版方法，作用在于：在创建IoC容器前，如果已经有容器存在，则需要把已有的容器销毁和关闭，以保证在refresh之后使用的是新建立起来的IoC容器，并对 Bean Definition进行载入
refresh() 方法的大体流程如下：
![[Pasted image 20240123172607.png]]
##### 初始化 BeanFactory
通过 AbstractApplicationContext.refreshBeanFactory() 方法来进行初始化，不同子类有不同实现，这里以 AbstractRefreshableApplicationContext 为例，代码如下：
```java
protected final void refreshBeanFactory() throws BeansException {

    // 如果已经有容器存在，则需要把已有的容器销毁和关闭，以保证在refresh之后使用的是新建立起来的IoC容器
    if (this.hasBeanFactory()) {  
        this.destroyBeans();  
        this.closeBeanFactory();  
    }  
  
    try {
    // 创建DefaultListableBeanFactory，并调用loadBeanDefinitions(beanFactory)装载bean定义
        DefaultListableBeanFactory beanFactory = this.createBeanFactory();  
        beanFactory.setSerializationId(this.getId());  
// 对IoC容器进行定制化，如设置启动参数，开启注解的自动装配等
this.customizeBeanFactory(beanFactory);
// 调用载入Bean定义的方法，主要这里又使用了一个委派模式，在当前类中只定义了抽象的loadBeanDefinitions方法，具体的实现调用子类容器
this.loadBeanDefinitions(beanFactory);  
        this.beanFactory = beanFactory;  
    } catch (IOException var2) {  
        throw new ApplicationContextException("I/O error parsing bean definition source for " + this.getDisplayName(), var2);  
    }  
}
```

##### 读取 BeanDefinition
通过 loadBeanDefinitions() 来读取 BeanDefinition，这里以 AnnotationConfigWebApplicationContext 为例，代码如下：
```java
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) {  
    AnnotatedBeanDefinitionReader reader = this.getAnnotatedBeanDefinitionReader(beanFactory);  
    ClassPathBeanDefinitionScanner scanner = this.getClassPathBeanDefinitionScanner(beanFactory);  
    BeanNameGenerator beanNameGenerator = this.getBeanNameGenerator();  
    if (beanNameGenerator != null) {  
        reader.setBeanNameGenerator(beanNameGenerator);  
        scanner.setBeanNameGenerator(beanNameGenerator);  
        beanFactory.registerSingleton("org.springframework.context.annotation.internalConfigurationBeanNameGenerator", beanNameGenerator);  
    }  
  
    ScopeMetadataResolver scopeMetadataResolver = this.getScopeMetadataResolver();  
    if (scopeMetadataResolver != null) {  
        reader.setScopeMetadataResolver(scopeMetadataResolver);  
        scanner.setScopeMetadataResolver(scopeMetadataResolver);  
    }  
  
    if (!this.componentClasses.isEmpty()) {  
        if (this.logger.isDebugEnabled()) {  
            this.logger.debug("Registering component classes: [" + StringUtils.collectionToCommaDelimitedString(this.componentClasses) + "]");  
        }  
  
        reader.register(ClassUtils.toClassArray(this.componentClasses));  
    }  
  
    if (!this.basePackages.isEmpty()) {  
        if (this.logger.isDebugEnabled()) {  
            this.logger.debug("Scanning base packages: [" + StringUtils.collectionToCommaDelimitedString(this.basePackages) + "]");  
        }  
  
        scanner.scan(StringUtils.toStringArray(this.basePackages));  
    }  
  
    String[] configLocations = this.getConfigLocations();  
    if (configLocations != null) {  
        String[] var7 = configLocations;  
        int var8 = configLocations.length;  
  
        for(int var9 = 0; var9 < var8; ++var9) {  
            String configLocation = var7[var9];  
  
            try {  
                Class<?> clazz = ClassUtils.forName(configLocation, this.getClassLoader());  
                if (this.logger.isTraceEnabled()) {  
                    this.logger.trace("Registering [" + configLocation + "]");  
                }  
  
                reader.register(new Class[]{clazz});  
            } catch (ClassNotFoundException var13) {  
                if (this.logger.isTraceEnabled()) {  
                    this.logger.trace("Could not load class for config location [" + configLocation + "] - trying package scan. " + var13);  
                }  
  
                int count = scanner.scan(new String[]{configLocation});  
                if (count == 0 && this.logger.isDebugEnabled()) {  
                    this.logger.debug("No component classes found for specified class/package [" + configLocation + "]");  
                }  
            }  
        }  
    }  
  
}
```
通过获取相应 context 的 reader （注解方式 AnnotatedBeanDefinitionReader）来进行配置读取，在注解方式下，即将带有注解的类进行读取解析，并封装成 BeanDefinition

##### 注册 BeanDefinition
最后，通过 doRegisterBean() 方法来将 BeanDefinition 注册到容器中
```java
private <T> void doRegisterBean(Class<T> beanClass, @Nullable String name, @Nullable Class<? extends Annotation>[] qualifiers, @Nullable Supplier<T> supplier, @Nullable BeanDefinitionCustomizer[] customizers) {  
    AnnotatedGenericBeanDefinition abd = new AnnotatedGenericBeanDefinition(beanClass);  
    if (!this.conditionEvaluator.shouldSkip(abd.getMetadata())) {  
        abd.setInstanceSupplier(supplier);  
        ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(abd);  
        abd.setScope(scopeMetadata.getScopeName());  
        String beanName = name != null ? name : this.beanNameGenerator.generateBeanName(abd, this.registry);  
        AnnotationConfigUtils.processCommonDefinitionAnnotations(abd);  
        int var10;  
        int var11;  
        if (qualifiers != null) {  
            Class[] var9 = qualifiers;  
            var10 = qualifiers.length;  
  
            for(var11 = 0; var11 < var10; ++var11) {  
                Class<? extends Annotation> qualifier = var9[var11];  
                if (Primary.class == qualifier) {  
                    abd.setPrimary(true);  
                } else if (Lazy.class == qualifier) {  
                    abd.setLazyInit(true);  
                } else {  
                    abd.addQualifier(new AutowireCandidateQualifier(qualifier));  
                }  
            }  
        }  
  
        if (customizers != null) {  
            BeanDefinitionCustomizer[] var13 = customizers;  
            var10 = customizers.length;  
  
            for(var11 = 0; var11 < var10; ++var11) {  
                BeanDefinitionCustomizer customizer = var13[var11];  
                customizer.customize(abd);  
            }  
        }  
  
        BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(abd, beanName);  
        definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);  
        BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry);  
    }  
}
```
IoC 容器本质上就是一个 BeanDefinitionMap， 注册即将 BeanDefinition 放到 Map 中

### Bean 实例化
实例化就是从 BeanDefinition 中实例化 Bean 对象
#### BeanFactory.getBean()
实例化过程主要就是 getBean() 方法，BeanFactory 的 getBean() 方法定义如下：
```java
// 根据bean的名字和Class类型等来得到bean实例    
Object getBean(String name) throws BeansException;    
Object getBean(String name, Class requiredType) throws BeansException;    
Object getBean(String name, Object... args) throws BeansException;
<T> T getBean(Class<T> requiredType) throws BeansException;
<T> T getBean(Class<T> requiredType, Object... args) throws BeansException;
```
getBean() 方法的实现在 AbstractBeanFactory 中
```java
public Object getBean(String name) throws BeansException {
  return doGetBean(name, null, null, false);
}
public <T> T getBean(String name, Class<T> requiredType) throws BeansException {
  return doGetBean(name, requiredType, null, false);
}
public Object getBean(String name, Object... args) throws BeansException {
  return doGetBean(name, null, args, false);
}
public <T> T getBean(String name, @Nullable Class<T> requiredType, @Nullable Object... args)
    throws BeansException {
  return doGetBean(name, requiredType, args, false);
}
```
逻辑都在 doGetBean() 这个方法中
```java
// 参数typeCheckOnly：bean实例是否包含一个类型检查
protected <T> T doGetBean(
      String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly)
      throws BeansException {

  // 解析bean的真正name，如果bean是工厂类，name前缀会加&，需要去掉
  String beanName = transformedBeanName(name);
  Object beanInstance;

  // Eagerly check singleton cache for manually registered singletons.
  Object sharedInstance = getSingleton(beanName);
  if (sharedInstance != null && args == null) {
    // 无参单例从缓存中获取
    beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, null);
  }

  else {
    // 如果bean实例还在创建中，则直接抛出异常
    if (isPrototypeCurrentlyInCreation(beanName)) {
      throw new BeanCurrentlyInCreationException(beanName);
    }

    // 如果 bean definition 存在于父的bean工厂中，委派给父Bean工厂获取
    BeanFactory parentBeanFactory = getParentBeanFactory();
    if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
      // Not found -> check parent.
      String nameToLookup = originalBeanName(name);
      if (parentBeanFactory instanceof AbstractBeanFactory) {
        return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
            nameToLookup, requiredType, args, typeCheckOnly);
      }
      else if (args != null) {
        // Delegation to parent with explicit args.
        return (T) parentBeanFactory.getBean(nameToLookup, args);
      }
      else if (requiredType != null) {
        // No args -> delegate to standard getBean method.
        return parentBeanFactory.getBean(nameToLookup, requiredType);
      }
      else {
        return (T) parentBeanFactory.getBean(nameToLookup);
      }
    }

    if (!typeCheckOnly) {
      // 将当前bean实例放入alreadyCreated集合里，标识这个bean准备创建了
      markBeanAsCreated(beanName);
    }

    StartupStep beanCreation = this.applicationStartup.start("spring.beans.instantiate")
        .tag("beanName", name);
    try {
      if (requiredType != null) {
        beanCreation.tag("beanType", requiredType::toString);
      }
      RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
      checkMergedBeanDefinition(mbd, beanName, args);

      // 确保它的依赖也被初始化了.
      String[] dependsOn = mbd.getDependsOn();
      if (dependsOn != null) {
        for (String dep : dependsOn) {
          if (isDependent(beanName, dep)) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
          }
          registerDependentBean(dep, beanName);
          try {
            getBean(dep); // 初始化它依赖的Bean
          }
          catch (NoSuchBeanDefinitionException ex) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                "'" + beanName + "' depends on missing bean '" + dep + "'", ex);
          }
        }
      }

      // 创建Bean实例：单例
      if (mbd.isSingleton()) {
        sharedInstance = getSingleton(beanName, () -> {
          try {
            // 真正创建bean的方法
            return createBean(beanName, mbd, args);
          }
          catch (BeansException ex) {
            // Explicitly remove instance from singleton cache: It might have been put there
            // eagerly by the creation process, to allow for circular reference resolution.
            // Also remove any beans that received a temporary reference to the bean.
            destroySingleton(beanName);
            throw ex;
          }
        });
        beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
      }
      // 创建Bean实例：原型
      else if (mbd.isPrototype()) {
        // It's a prototype -> create a new instance.
        Object prototypeInstance = null;
        try {
          beforePrototypeCreation(beanName);
          prototypeInstance = createBean(beanName, mbd, args);
        }
        finally {
          afterPrototypeCreation(beanName);
        }
        beanInstance = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
      }
      // 创建Bean实例：根据bean的scope创建
      else {
        String scopeName = mbd.getScope();
        if (!StringUtils.hasLength(scopeName)) {
          throw new IllegalStateException("No scope name defined for bean ´" + beanName + "'");
        }
        Scope scope = this.scopes.get(scopeName);
        if (scope == null) {
          throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
        }
        try {
          Object scopedInstance = scope.get(beanName, () -> {
            beforePrototypeCreation(beanName);
            try {
              return createBean(beanName, mbd, args);
            }
            finally {
              afterPrototypeCreation(beanName);
            }
          });
          beanInstance = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
        }
        catch (IllegalStateException ex) {
          throw new ScopeNotActiveException(beanName, scopeName, ex);
        }
      }
    }
    catch (BeansException ex) {
      beanCreation.tag("exception", ex.getClass().toString());
      beanCreation.tag("message", String.valueOf(ex.getMessage()));
      cleanupAfterBeanCreationFailure(beanName);
      throw ex;
    }
    finally {
      beanCreation.end();
    }
  }

  return adaptBeanInstance(name, beanInstance, requiredType);
}
```
主要流程如下：
- 解析bean的真正name，如果bean是工厂类，name前缀会加&，需要去掉
- 无参单例先从缓存中尝试获取
- 如果bean实例还在创建中，则直接抛出异常
- 如果bean definition 存在于父的bean工厂中，委派给父Bean工厂获取
- 标记这个beanName的实例正在创建
- 确保它的依赖也被初始化
- 真正创建过程
	- 单例时
	- 原型时
	- 根据bean的scope创建

### 解决依赖循环
Spring 只是解决了单例模式下属性依赖的循环问题，为了解决单例的循环依赖问题，使用了三级缓存

三级缓存如下：
```java
/** Cache of singleton objects: bean name --> bean instance */
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<String, Object>(256);
 
/** Cache of early singleton objects: bean name --> bean instance */
private final Map<String, Object> earlySingletonObjects = new HashMap<String, Object>(16);

/** Cache of singleton factories: bean name --> ObjectFactory */
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<String, ObjectFactory<?>>(16);
```
- **第一层缓存（singletonObjects）**：单例对象缓存池，已经实例化并且属性赋值，这里的对象是**成熟对象**
- **第二层缓存（earlySingletonObjects）**：单例对象缓存池，已经实例化但尚未属性赋值，这里的对象是**半成品对象**
- **第三层缓存（singletonFactories）**: 单例工厂的缓存

在 BeanFactory#getBean() 方法中，会先调用
```java
// Eagerly check singleton cache for manually registered singletons.  
Object sharedInstance = getSingleton(beanName);
```
来获取 bean 实例，其中就是调用了AbstractBeanFactory#doGetBean() 方法，代码如下
```java
@Nullable  
protected Object getSingleton(String beanName, boolean allowEarlyReference) {  
    // Quick check for existing instance without full singleton lock
    // 从单例缓存中加载bean  
    Object singletonObject = this.singletonObjects.get(beanName); 
            // 单例缓存中没有获取到bean，同时bean在创建中 
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {  
                // 从 earlySingletonObjects 获取(二级缓存)
       singletonObject = this.earlySingletonObjects.get(beanName);  
       if (singletonObject == null && allowEarlyReference) {  
          synchronized (this.singletonObjects) {  
             // Consistent creation of early reference within full singleton lock                      //再次从三级缓存中加载bean
             singletonObject = this.singletonObjects.get(beanName);  
             if (singletonObject == null) {  
                singletonObject = this.earlySingletonObjects.get(beanName);  
                if (singletonObject == null) {  
                   ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);  
                   if (singletonFactory != null) {  
                      singletonObject = singletonFactory.getObject();  
                      this.earlySingletonObjects.put(beanName, singletonObject);  
                      this.singletonFactories.remove(beanName);  
                   }  
                }  
             }  
          }  
       }  
    }  
    return singletonObject;  
}
```
这个方法就是从三级缓存中获取Bean对象，可以看到这里先从一级缓存`singletonObjects`中查找，没有找到的话接着从二级缓存`earlySingletonObjects`，还是没找到的话最终会去三级缓存`singletonFactories`中查找，需要注意的是如果在三级缓存中找到，

就会从三级缓存**升级**到二级缓存了。所以，二级缓存存在的意义，就是缓存三级缓存中的 ObjectFactory 的 `#getObject()` 方法的执行结果，提早曝光的单例 Bean 对象

如果还是没有找到 bean，就 doGetBean() 方法，就会创建一个 Bean
```java
// Create bean instance.  
if (mbd.isSingleton()) {  
    sharedInstance = getSingleton(beanName, () -> {  
       try {  
          return createBean(beanName, mbd, args);  
       }  
       catch (BeansException ex) {  
          // Explicitly remove instance from singleton cache: It might have been put there  
          // eagerly by the creation process, to allow for circular reference resolution.          // Also remove any beans that received a temporary reference to the bean.          destroySingleton(beanName);  
          throw ex;  
       }  
    });  
    beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);  
}
```
createBean() 方法中的关键逻辑，提前曝光半成品 bean，代码如下
```java
// Eagerly cache singletons to be able to resolve circular references  
// even when triggered by lifecycle interfaces like BeanFactoryAware.  
boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&  
       isSingletonCurrentlyInCreation(beanName));  
if (earlySingletonExposure) {  
    if (logger.isTraceEnabled()) {  
       logger.trace("Eagerly caching bean '" + beanName +  
             "' to allow for resolving potential circular references");  
    }
    // 提前将创建的 bean 实例加入到 singletonFactories 中
     // 这里是为了后期避免循环依赖  
    addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));  
}
```
addSignleFactory() 的代码如下：
```java
protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
    Assert.notNull(singletonFactory, "Singleton factory must not be null");
    synchronized (this.singletonObjects) {
        if (!this.singletonObjects.containsKey(beanName)) {
            this.singletonFactories.put(beanName, singletonFactory);
            this.earlySingletonObjects.remove(beanName);
            this.registeredSingletons.add(beanName);
        }
    }
}
```
同时这段代码发生在 `#createBeanInstance(...)` 方法之后，也就是说这个 bean 其实已经被创建出来了，**但是它还不是很完美（没有进行属性填充和初始化），但是对于其他依赖它的对象而言已经足够了（可以根据对象引用定位到堆中对象），能够被认出来了**

#### 为什么要采用三级缓存
正常情况下（没有循环依赖），`Spring`都是在创建好`完成品Bean`之后才创建对应的`代理对象`。为了处理循环依赖，`Spring`有两种选择：
1. 不管有没有`循环依赖`，都`提前`创建好`代理对象`，并将`代理对象`放入缓存，出现`循环依赖`时，其他对象直接就可以取到代理对象并注入。
2. 不提前创建好代理对象，在出现`循环依赖`被其他对象注入时，才实时生成`代理对象`。这样在没有`循环依赖`的情况下，`Bean`就可以按着`Spring设计原则`的步骤来创建

显然`Spring`使用了三级缓存，选择第二种方案，原因如下：
**如果要使用`二级缓存`解决`循环依赖`，意味着Bean在`构造`完后就创建`代理对象`，这样违背了`Spring设计原则`。Spring结合AOP跟Bean的生命周期，是在`Bean创建完全`之后通过`AnnotationAwareAspectJAutoProxyCreator`这个后置处理器来完成的，在这个后置处理的`postProcessAfterInitialization`方法中对初始化后的Bean完成AOP代理。如果出现了`循环依赖`，那没有办法，只有给Bean先创建代理，但是没有出现循环依赖的情况下，设计之初就是让Bean在生命周期的最后一步完成代理而不是在实例化后就立马完成代理**