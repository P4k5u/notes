### 简介
IoC（Inversion of Control），即 控制反转，不是什么技术，而是一种设计思想，在 Java 开发中，IoC 意味着将你设计好的对象交给容器控制，而不是传统的在你的对象内部直接控制

### IoC 的作用
传统应用程序都是由我们在类内部主动创建依赖对象，从而导致类与类之间高耦合，难于测试
有了 IoC 容器之后，**把创建和查找依赖对象的控制权交给了容器，由容器进行注入组合对象，所以对象与对象之间是 松散耦合，这样也方便测试，利于功能复用，更重要的是使得程序的整个体系结构变得非常灵活**

在 IoC/DI 思想中，应用程序变成了被动，被动等待 IoC 容器来创建并注入它所需要的资源

IoC 很好的体现了面向对象的法则之一 —— 好莱坞法则：“别找我们，我们找你”，即由 IoC 容器帮对象找相应的依赖对象并注入，而不是由对象主动去找

### DI 依赖注入
DI（Dependency Injection），即依赖注入，组件之间的依赖关系由容器在运行期决定。依赖注入的目的是为了提升组件重用的频率
而 IoC 就是通过 DI 来实现的，被注入对象依赖 IoC 容器配置依赖对象

### IoC 的配置方式

#### xml 配置
将 bean 的信息配置在 .xml 文件里，通过 Spring 加载文件为我们创建 bean。这种方式出现在 SSM 项目中会比较多
```xml
<?xml version="1.0" encoding="UTF-8"?> <beans xmlns="http://www.springframework.org/schema/beans" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd"> <!-- services --> <bean id="userService" class="tech.pdai.springframework.service.UserServiceImpl"> <property name="userDao" ref="userDao"/> <!-- additional collaborators and configuration for this bean go here --> </bean> <!-- more bean definitions for services go here --> </beans>
```
#### Java 配置
将类的创建交给配置好的 JavaConfig 类来完成，Spring 只负责维护和管理，采用纯 Java 创建方式，其本质上就是把 XML 上的配置声明转移到 Java 配置类中，常用于配置第三方资源
```java
@Configuration  
public class BeansConfig {  
    /**  
     * @return user dao  
     */
    @Bean("userDao")  
    public UserDaoImpl userDao() {  
        return new UserDaoImpl();  
    }  
  
    /**  
     * @return user service  
     */
    @Bean("userService")  
    public UserServiceImpl userService() {  
        UserServiceImpl userService = new UserServiceImpl();  
        userService.setUserDao(userDao());  
        return userService;  
    }  
}
```
#### 注解配置
通过在类上加注解的方式，来声明一个类交给Spring管理，Spring会自动扫描带有 @Component，@Controller，@Service，@Repository 这四个注解的类，然后帮我们创建并管理，前提是需要先配置Spring的注解扫描器
```java
@Service
public class UserServiceImpl {

    /**
     * user dao impl.
     */
    private UserDaoImpl userDao;

    /**
     * find user list.
     *
     * @return user list
     */
    public List<User> findUserList() {
        return userDao.findUserList();
    }

}
```

### 依赖注入的方式

#### setter 方式
在 xml 配置方式中，property 都是 setter 方式注入的
使用注解和 Java 配置时，对应的 service 类如下：
```java
public class UserServiceImpl {

    /**
     * user dao impl.
     */
    private UserDaoImpl userDao;

    /**
     * init.
     */
    public UserServiceImpl() {
    }

    /**
     * find user list.
     *
     * @return user list
     */
    public List<User> findUserList() {
        return this.userDao.findUserList();
    }

    /**
     * set dao.
     *
     * @param userDao user dao
     */
     @Autowired
    public void setUserDao(UserDaoImpl userDao) {
        this.userDao = userDao;
    }
}
```
#### 构造函数
**在XML配置方式中**，`<constructor-arg>` 是通过构造函数参数注入的
本质上就是通过 new UserServiceImpl(userDao) 方法来创建对象，service 类如下：
```java
 @Service
public class UserServiceImpl {

    /**
     * user dao impl.
     */
    private final UserDaoImpl userDao;

    /**
     * init.
     * @param userDaoImpl user dao impl
     */
    @Autowired // 这里@Autowired也可以省略
    public UserServiceImpl(final UserDaoImpl userDaoImpl) {
        this.userDao = userDaoImpl;
    }

    /**
     * find user list.
     *
     * @return user list
     */
    public List<User> findUserList() {
        return this.userDao.findUserList();
    }

}
```
#### 注解方式
以 `@Autowired` 注解注入为例，修饰符有三个属性：Constructor、byType、byName 默认按照 byType 注入
- constructor：通过构造方法进行自动注入，Spring 会匹配与构造方法参数类型一致的 bean 进行注入，如果有一个多参数的构造方法，一个只有一个参数的构造方法，Spring 会优先将 bean 注入到多参数的构造方法中
- byName： id 的首字母需要小写，配合 @Qualifier 指定 id 来使用
- byType：将符合参数类型的 bean 注入

在字段属性上使用，代码如下：
```java
@Service
public class UserServiceImpl {

    /**
     * user dao impl.
     */
    @Autowired
    private UserDaoImpl userDao;

    /**
     * find user list.
     *
     * @return user list
     */
    public List<User> findUserList() {
        return userDao.findUserList();
    }

}
```
在set方法上使用：
```java
@Service
public class UserServiceImpl {

    /**
     * user dao impl.
     */
    private UserDaoImpl userDao;

    /**
     * find user list.
     *
     * @return user list
     */
    public List<User> findUserList() {
        return userDao.findUserList();
    }

    @Autowired
    public void setUserDao(UserDao userDao{
        this.userDao = userDao;
    }

}
```
在构造方法上使用：
```java
@Service
public class UserServiceImpl {

    /**
     * user dao impl.
     */
    private UserDaoImpl userDao;

    @Autowired //只有一个构造方法时，可以省略
    public UserServiceImpl(UserDaoImpl userDao){
       this.userDao = userDao;
    }

}
```

#### Spring 推荐使用构造器注入方式的原因
Spring 在文档的解释：
构造器注入的方式**能够保证注入的组件不可变，并且确保需要的依赖不为空**。此外，构造器注入的依赖总是能够在返回客户端（组件）代码的时候保证完全初始化的状态
- 依赖不可变：通过 final 关键字修饰参数
- 依赖不为空：由于自己实现了有参构造函数，就不会调用默认的构造函数，所有在容器传入参数时，没有该类型的参数就会自动报错
- 完全初始化的状态：这个可以跟上面的依赖不为空结合起来，向构造器传参之前，要确保注入的内容不为空，那么肯定要调用依赖组件的构造方法完成实例化。而在Java类加载实例化的过程中，构造方法是最后一步（之前如果有父类先初始化父类，然后自己的成员变量，最后才是构造方法），所以返回来的都是初始化之后的状态



#### `@Autowired` 、`@Resource` 和 `@Inject` 的区别

`@Autowired`
Spring自带的注解，通过AutowiredAnnotationBeanPostProcessor 类实现的依赖注入
可以作用在CONSTRUCTOR、METHOD、PARAMETER、FIELD、ANNOTATION_TYPE
默认是根据类型（byType ）进行自动装配的

`@Resource`
是JSR250规范的实现，在javax.annotation包下
可以作用TYPE、FIELD、METHOD上
是默认根据属性名称进行自动装配的，如果有多个类型一样的Bean候选者，则可以通过name进行指定进行注入

`@Inject`
是JSR330 (Dependency Injection for Java)中的规范，需要导入javax.inject.Inject jar包 ，才能实现注入
可以作用CONSTRUCTOR、METHOD、FIELD上
根据类型进行自动装配的，如果需要按名称进行装配，则需要配合@Named
