### 简介
AOP（Aspect Oriented Programming），即面向切面编程
`Spring AOP` 是利用 **CGLIB 和 JDK 动态代理**等方式来实现运行期动态方法增强，其目的是_将与业务无关的代码单独抽离出来，使其逻辑不再与业务代码耦合，从而降低系统的耦合性，提高程序的可重用性和开发效率_。因而 AOP 便成为了**日志记录、监控管理、性能统计、异常处理、权限管理、统一认证**等各个方面被广泛使用的技术

我们之所以能无感知地在 Spring 容器 bean 对象方法前后随意添加代码片段进行逻辑增强，是由于 Spring 在运行期帮我们把切面中的代码逻辑动态“织入”到了bean 对象方法内，所以说 AOP 本质上就是一个**代理模式**

AOP 的理念就是将分散在各个业务逻辑代码中相同的代码通过横向切割的方式抽取一个独立的模块![[Pasted image 20240202171309.png]]
这是针对业务处理过程中的切面进行提取，它所面对的是处理过程的某个步骤或阶段，以获得逻辑过程的中各部分之间低耦合的隔离效果

### 相关术语
AOP 的大致工作流程如下：
![[Pasted image 20240202175507.png]]

#### AOP 概念
- Joinpoint（连接点）：表示需要在程序中插入横切关注点的扩展点，**连接点可能是类初始化、方法执行、方法调用、字段调用或处理异常等等**，Spring只支持方法执行连接点，在AOP中表示为**在哪里执行**
- Pointcut（切入点）：选择一组相关连接点的模式，即可以认为连接点的集合，Spring支持perl5正则表达式和AspectJ切入点模式，Spring默认使用AspectJ语法，在AOP中表示为**在哪里执行的集合**
- Advice（通知）：在连接点上执行的行为，通知提供了在AOP中需要在切入点所选择的连接点处进行扩展现有行为的手段
- Aspect（切面）：横切关注点的模块化，可以认为是通知、引入和切入点的组合，在 Spring 中可以使用 Schema 和 @AspectJ 方式进行组织实现
- inter-type declaration （引入）：也称为内部类型声明，为已有的类添加额外新的字段或方法，Spring 允许引入新的接口（必须对应一个实现）到所有被代理对象（目标对象）
- Target Object（目标对象）：需要被织入横切关注点的对象，即该对象是切入点选择的对象，需要被通知的对象，从而也可称为被通知对象；由于Spring AOP 通过代理模式实现，从而这个对象永远是被代理对象
- Weaving（织入）：把切面连接到其它的应用程序类型或者对象上，并创建一个被通知的对象。这些可以在编译时（例如使用AspectJ编译器），类加载时和运行时完成。Spring和其他纯Java AOP框架一样，在运行时完成织入
- AOP Proxy（AOP 代理）：AOP框架使用代理模式创建的对象，从而实现在连接点处插入通知（即应用切面），就是通过代理来对目标对象应用切面。在Spring中，AOP代理可以用JDK动态代理或CGLIB代理实现，而通过拦截器模型应用切面

#### 通知类型
- **前置通知（Before advice）**：在某连接点之前执行的通知，但这个通知不能阻止连接点之前的执行流程（除非它抛出一个异常）
- **后置通知（After returning advice）**：在某连接点正常完成后执行的通知：例如，一个方法没有抛出任何异常，正常返回
- **异常通知（After throwing advice）**：在方法抛出异常退出时执行的通知
- **最终通知（After finally advice）**：当某连接点退出的时候执行的通知（不论是正常返回还是异常退出）
- **环绕通知（Around Advice）**：包围一个连接点的通知，如方法调用。这是最强大的一种通知类型。环绕通知可以在方法调用前后完成自定义的行为。它也会选择是否继续执行连接点或直接返回它自己的返回值或抛出异常来结束执行

### Spring AOP 的配置方式
Spring AOP 支持 XML 模式和 @AspectJ 注解的两种配置方式

#### XML Schema 配置方式

#### AspectJ 注解方式
AspectJ 是一个 java 实现的 AOP 框架，能够对 Java 代码进行 AOP 编译（使用特定的编译器），让 java 代码具有 AOP 功能

Spring AOP 使用纯 Java 实现，不需要专门的编译过程，一个重要的原则就是无侵入性，所以 Spring 使用了与 AspectJ 一样的注解，但是在运行的时候依旧是 Spring AOP，并不依赖于 AspectJ 的编译器

对于 织入 的方式，Spring AOP 与 AspectJ 采用的是不一样的方式：
- Spring AOP 采用的是基于动态代理的 动态织入 方式，在运行时就能完成，通过 Java JDK 或者 CGLIB 的动态代理来实现
- AspectJ 采用的是 静态织入 的方式，通过在编译期间使用 特殊编译器 acj 来进行编译织入

##### 相关注解
Spring 使用了 @AspectJ 框架为AOP的实现提供了一套注解
- @Aspect：声明这是一个切面类，需要与 @Component 一起使用，交给 Spring 管理
- @Pointcut：定义一个切点，有两种方式，execution() 匹配方法、annotation() 匹配注解
- @Around：增强处理，表示环绕增强，可以任意执行
- @Before：表示在切点方法前执行
- @After：表示在切点方法后执行
- @AfterReturing：与 @After 类似，但是可以捕获切点方法返回值
- @AfterThrowing：在方法抛出异常时执行

相关注解使用的代码例子如下：
```java
@EnableAspectJAutoProxy  
@Component  
@Aspect  
public class LogAspect {  
  
    /**  
     * define point cut.     */    @Pointcut("execution(* tech.pdai.springframework.service.*.*(..))")  
    private void pointCutMethod() {  
    }  
  
  
    /**  
     * 环绕通知.  
     *     * @param pjp pjp  
     * @return obj  
     * @throws Throwable exception  
     */    @Around("pointCutMethod()")  
    public Object doAround(ProceedingJoinPoint pjp) throws Throwable {  
        System.out.println("环绕通知: 进入方法");  
        Object o = pjp.proceed();  
        System.out.println("环绕通知: 退出方法");  
        return o;  
    }  
  
    /**  
     * 前置通知.  
     */    @Before("pointCutMethod()")  
    public void doBefore() {  
        System.out.println("前置通知");  
    }  
  
  
    /**  
     * 后置通知.  
     *     * @param result return val  
     */    @AfterReturning(pointcut = "pointCutMethod()", returning = "result")  
    public void doAfterReturning(String result) {  
        System.out.println("后置通知, 返回值: " + result);  
    }  
  
    /**  
     * 异常通知.  
     *     * @param e exception  
     */    @AfterThrowing(pointcut = "pointCutMethod()", throwing = "e")  
    public void doAfterThrowing(Exception e) {  
        System.out.println("异常通知, 异常: " + e.getMessage());  
    }  
  
    /**  
     * 最终通知.  
     */    @After("pointCutMethod()")  
    public void doAfter() {  
        System.out.println("最终通知");  
    }  
  
}
```




