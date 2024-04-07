### 简介
旧网关 Netflix Zuul 性能一般并且版本更新迟迟未上，所以新版的 Spring Cloud 不打算捆绑 Zuul 了，将使用 Spring Cloud Gateway 代替，Gateway 并非使用传统的 Jakarta EE 的 Servlet 容器，采用响应式编程的方式进行开发，所以运行环境需要 Spring Boot 和 Spring WebFlux 提供的基于 Netty 的支持

由于 Gateway 是基于响应式方式的，内部执行方式的不同导致性能完全不一样
Zuul 的执行方式如下：![[Pasted image 20240118154053.png]]
从执行的原理来说，Zuul 会为一个请求分配一条线程，然后通过执行不同类型的过滤器来完成路由的功能，但是这条线程会等 route 类型的过滤器去调用源服务器，这就导致了性能一般

Gateway 的组件执行方式如下：![[Pasted image 20240118154538.png]]
- 创建一条线程，通过过滤器拦截请求
- 对源服务器转发请求，但注意，Gateway 并不会等待请求调用源服务器的过程，而是将处理线程挂起，这样就不会占用资源
- 等源服务器返回消息后，再通过寻址的方式来响应之前客户端发送的请求
所以，Gateway 线程在处理请求的时候，仅仅是负责转发请求到源服务器，并不会等待源服务器执行完成，所以性能会更好

### 使用方法
引入依赖之后，Gateway 网关就会自动开启，但是有要注意的地方：
- Gateway 依赖 WebFlux，而 WebFlux 和 Spring Web MVC 的包冲突，再引入 spring-boot-starter-web 就会发生异常
- 当前 Gateway 只支持 Netty 容器，不支持其他容器，所以引入 tomcat 或 jetty 会引发问题

通过代码配置：
```java
package com.spring.cloud.gateway.main;
/**** imports ****/
@SpringBootApplication(scanBasePackages = "com.spring.cloud.gateway.*")
public class GatewayApplication {

   public static void main(String[] args) {
      SpringApplication.run(GatewayApplication.class, args);
   } 

   /**
    * 创建路由规则
    * @param builder -- 路由构造器
    * @return 路由规则
    */
   @Bean
   public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
      return builder.routes() //开启路由配置
         // 匹配路径
         .route(f -> f.path("/user/**") // route方法逻辑是一个断言，后续会论述
         // 转发到具体的URI
         .uri("http://localhost:6001"))
         // 创建
         .build();
   }
}
```
核心代码是 customRouteLocator 方法，由构造器 RouteLocatorBuilder 构造的，采用了响应式编程方法

通过yaml文件配置：
```yaml
spring:
  cloud:
    gateway:
      # 开始配置路径
      routes:
        # 路径匹配
        - id: fund
          # 转发URI
          uri: http://localhost:7001
          # 断言配置
          predicates:
          - Path=/r4j/**
```

### 工作原理
Gateway 执行请求的过滤和转发的过程如下：![[Pasted image 20240118161135.png]]
有三个关键组件：
- Route（路由）：路由网关是一个最基本的组件，它由ID、目标URI、断言集合和过滤器集合共同组成，当断言判定为 true 时，才会匹配到路由
- Predicate（断言）：它主要是判定当前请求如何匹配路由，采用的是 Java 8 的断言，可以存在多个断言，每个断言的入参都是 Spring 的 ServerWebExchange 对象类型。它允许开发者匹配来自 HTTP 请求的任何内容，例如：URL、请求头或请求参数，当这些断言都返回 true 时才执行路由
- Filter（过滤器）：使用特定工厂构造的 SpringFrameworkGatewayFilter 实例，作用是在发送下游请求之前或之后，修改请求和响应
执行过程如下：![[Pasted image 20240118161652.png]]
一个请求先通过 HandlerMapping 机制找到对应的 WebHandler ，然后通过各类（代理）过滤器进行处理
Gateway 是通过 Spring WebFlux 来实现的，而在 WebFlux 中，最核心的接口是 WebHandler ，它是一个请求处理器，存在多个实现类
![[Pasted image 20240118162537.png]]
主要关注 FilterWebHandler 和 DispatcherHandler，其中 DispatcherHandler 是一个转发处理器，而 FilteringWebHandler 是 GateWay 实际执行逻辑的处理器
#### DispatcherHandler
既然 DispatcherHandler 是一个转发处理器，那么它一定需要一个处理器映射（HandlerMapping）来找到对应的处理器
在配置类 GatewayAutoConfiguration 中，有一个 routePredicateHandlerMapping() 方法
```java
/**
* 构建带有断言（Predicate）的路由
* @param webHandler -- 处理器（具体为FilteringWebHandler对象）
* @param routeLocator -- 路由信息
* @param globalCorsProperties -- 全局跨域配置属性
* @param environment -- 上下文环境
*/
@Bean
public RoutePredicateHandlerMapping routePredicateHandlerMapping(
       FilteringWebHandler webHandler, RouteLocator routeLocator,
       GlobalCorsProperties globalCorsProperties, Environment environment) {
   return new RoutePredicateHandlerMapping(webHandler, routeLocator,
          globalCorsProperties, environment);
}
```
Gateway 会自动构建一个 RoutePredicateHandlerMapping 对象，它便是 HandlerMapping，RoutePredicateHandlerMapping 中会通过 Predicate 去判断当前路由是否和请求匹配，这就是断言的作用
DispatcherHandler 通过 HandlerMapping 和请求的 URL，可以路由到具体的处理器，而具体的处理类是 FilteringWebHandler 对象

#### FilteringWebHandler
FilteringWebHandler 的核心处理逻辑：
```java
package org.springframework.cloud.gateway.handler;
/**** imports ****/
public class FilteringWebHandler implements WebHandler {
   ......
   // 全局过滤器
   private final List<GatewayFilter> globalFilters; 
   // 构造方法，注意参数类型为List<GlobalFilter>，全局过滤器 ①
   public FilteringWebHandler(List<GlobalFilter> globalFilters) { 
      this.globalFilters = loadFilters(globalFilters);
   }
   // 初始化全局过滤器
   private static List<GatewayFilter> loadFilters(
         List<GlobalFilter> filters) {
      return filters.stream()
            .map(filter -> {
               GatewayFilterAdapter gatewayFilter = new GatewayFilterAdapter(filter);
               if (filter instanceof Ordered) {
                  int order = ((Ordered) filter).getOrder();
                  return new OrderedGatewayFilter(gatewayFilter, order);
               }
               return gatewayFilter;
            }).collect(Collectors.toList());
   }
   // 实际逻辑方法
   @Override
   public Mono<Void> handle(ServerWebExchange exchange) {
      Route route = exchange.getRequiredAttribute(GATEWAY_ROUTE_ATTR);
      // 当前路由过滤器
      List<GatewayFilter> gatewayFilters = route.getFilters();
      List<GatewayFilter> combined = new ArrayList<>(this.globalFilters);//②
      // 装配当前路由过滤器 ③
      combined.addAll(gatewayFilters);
      //TODO: needed or cached?
      AnnotationAwareOrderComparator.sort(combined); // 排序
      if (logger.isDebugEnabled()) {
         logger.debug("Sorted gatewayFilterFactories: "+ combined);
      }
      // 构建过滤器责任链，然后执行过滤器，返回转发请求的结果 ④
      return new DefaultGatewayFilterChain(combined).filter(exchange);
   }
   ......
}
```
过滤器分为 全局过滤器 和 局部过滤器，其中全局过滤器需要实现接口 GlobaFilter，而局部过滤器则需要实现 GatewayFilter 接口。全局过滤器对所有路由有效，局部过滤器只是对某个具体的路由有效
### Route 路由
Route 的源码如下：
```java
package org.springframework.cloud.gateway.route;
/**** imports ****/
public class Route implements Ordered {
   // 编号
   private final String id;
   // 匹配URI
   private final URI uri;
   // 排序
   private final int order;
   // 断言
   private final AsyncPredicate<ServerWebExchange> predicate;
   // 过滤器列表
   private final List<GatewayFilter> gatewayFilters;
   /**** 其他代码****/
}
```

### Predicate 断言
Gateway 是通过断言来筛选路由，断言是通过工厂来实现的，定义工厂的接口是 RoutePredicateFactory，子类如下![[Pasted image 20240118170216.png]]
可以看出，断言工厂都是以 "xxxRoutePredicateFactory" 格式命名，Gateway 提供了配置它们的方式，可以通过代码或配置文件来实现
#### 断言工厂
##### 时间类断言
Before 路由断言是一个关于时间的断言，这可以判断路由在什么时间之前有效，过了这个时间点则无效。类似这样的时间断言还有 After 路由断言 和 Between 路由断言
断言工厂都采用 UTC 时间制度
###### 使用 Before 路由断言工厂
采用代码形式：
```java
/**
 * 创建路由规则
 * @param builder -- 路由构造器
 * @return 路由规则
 */
@Bean
public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
   ZonedDateTime datetime = LocalDateTime.now()//获取当前时间
          // 两分钟后路由失效
          .plusMinutes(2)
          // 定义国际化区域
          .atZone(ZoneId.systemDefault()); // 定义UTC时间 ①
   return builder.routes()
          // 匹配
          .route("/user/**", r -> r.before(datetime) // 使用断言 ②
                 // 转发到具体的URI
                 .uri("http://localhost:6001"))
          // 创建
          .build();
}
```
采用配置形式：
```yaml
spring:
  cloud:
    gateway:
      # 开始配置路径
      routes:
        # 路径匹配
        - id: fund
          # 转发URI
          uri: http://localhost:7001
          # 断言配置
          predicates:
            - Path=/r4j/** # 路径匹配
            - Before=2019-09-14T10:58:00.9896784+08:00[Asia/Shanghai]
```
配置项 Before 就是 ”xxxRoutePredicateFactory“，这样就映射到了 BeforeRoutePredicateFactory

##### Method 路由断言

##### Path 路由断言

##### Query 路由断言

##### RemoteAddr 路由断言

##### Weight 路由断言

### Filter 过滤器
断言是为了路由的匹配，过滤器则是在请求源服务器之前或之后对 HTTP 请求和响应的拦截，以便对请求和响应做出相应的修改
和断言一样，过滤器也有不同的过滤工厂，并且已经内置，可以直接进行使用。如：请求头、响应头、跳转、参数处理、响应状态、Hystrix 熔断和限速器

过滤器工厂是通过接口 GatewayFilterFactory 进行定义的，该接口还声明了一个 apply 方法，返回类型为 GatewayFilter，过滤器工厂有以下类型：![[Pasted image 20240118175346.png]]
也是通过 “xxxGatewayFilterFactory” 的形式来命名，因此可以通过名称来区分过滤器类型
#### 过滤器工厂
##### AddRequestHeader 过滤器
AddRequestHeader是一个添加请求头参数的过滤器工厂，通过它可以增加请求参数

通过代码方式：
```java
@Bean
public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
   return builder.routes()
      // 设置请求路径满足ANT风格“/user/**”的路由
      .route("user-service", r-> r.path("/user/**")
         // 通过增加请求头过滤器添加请求头参数
         .filters(f->f.addRequestHeader("id", "1")) // ①
         // 匹配路径
         .uri("http://localhost:6001"))
      .build();
}
```

通过配置文件方式：
```yaml
spring:
  cloud:
    gateway:
      routes:
        # 路由编号
        - id: user-service
          # 转发URI
          uri: http://localhost:6001
          # 断言配置
          predicates:
            # 路径匹配
          - Path=/user/**
          # 过滤器工厂
          filters:
            # 请求头参数
          - AddRequestHeader=id, 1
```
配置项 AddRequestHeader 就可以匹配到 “xxxGatewayFilterFactory"，指定 AddRequestHeaderGatewayFilterFactory 这个过滤器工厂类

##### AddRequestParameter 过滤器

##### AddResponseHeader 过滤器

##### Retry 过滤器

##### Hystrix 过滤器

##### RequestRateLimiter 过滤器
RequestRateLimiter工厂用于限制请求流量，避免过大的流量进入系统，从而保证系统在一个安全的流量下可用。在Gateway中提供了RateLimiter<>接口来定义限流器，该接口中唯一的非抽象实现类是RedisRateLimiter，也就是当前只能通过Redis来实现限流。使用Redis的好处是，可以通过Redis随时监控


##### StripPrefix 过滤器
可以通过这个过滤器来删除请求 uri 的前缀
代码实现方式：
```java
@Bean
public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
   return builder.routes()
      // 设置请求路径满足ANT风格“/u/**”的路由
      .route("user-service", r-> r.path("/u/**")
         // 通过StripPrefix过滤器添加响应参数
         .filters(f->f.stripPrefix(1)) // 
         // 匹配路径
         .uri("http://localhost:6001"))
      .build();
}
```
stripPrefix(1) 表示路由到源地址时删除 1 个前缀，如：请求 Gateway 服务的路径为 /u/user/info/1，就会路由到源服务器 /user/info/1

通过配置文件
```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: user-service
        # 转发URI
        uri: http://localhost:6001
        # 断言配置
        predicates:
        - Path=/u/**
        filters:
        - StripPrefix=1
```

##### RewritePath 过滤器
RewritePath过滤器工厂和StripPrefix过滤器工厂类似，只是功能比StripPrefix过滤器工厂更强大，可以直接重写请求路径

#### 自定义过滤器

##### 使用 Resilience4j 限流

##### 转发 token
全局过滤器的接口定义是 GlobalFilter，代码如下：
```java
package org.springframework.cloud.gateway.filter;
/**** imports ****/
public interface GlobalFilter {
   Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain);
}
```
和 GatewayFilter 接口几乎一样，都是同一的 filter 方法和参数。在 Gateway 机制中，只要一个类实现了 GlobalFilter 接口，并且装配到 IOC 容器中，Gateway 就会将它识别为全局过滤器。对于全局过滤器，执行路由命令的处理（FilteringWebHandler）会把它转变为 GatewayFilter ，放到过滤器链中

在分布式开发中，有时候需要一个 token 进行跨服务器鉴权，在通过网关时，往往需要截取这个 token，放到请求头中作为登录凭证，再路由到源服务器。

自定义 token 转发过滤器，代码如下：
```java
@Bean // 装配为Spring Bean
public GlobalFilter tokenFilter() {
   // Lambda表达式
   return (exchange, chain) -> {
      // 判定请求头token参数是否存在
      boolean flag = !StringUtils.isBlank(
             exchange.getRequest().getHeaders().getFirst("token"));
      if (flag) { // 存在，直接放行路由
         return chain.filter(exchange);
      }
      // 获取token参数
      String token = exchange.getRequest()
            .getQueryParams().getFirst("token");
      ServerHttpRequest request = null;
      // token参数不为空，路由时将它放入请求头
      if (!StringUtils.isBlank(token)) {
         request  = exchange.getRequest().mutate() // 构建请求头
                .header("token", token)
                .build();
         // 构造请求头后执行责任链
         return chain.filter(exchange.mutate().request(request).build());
      }
      // 放行
      return chain.filter(exchange);
   };
}
```
可以通过 GlobalFilter 的 getOrder 方法来进行排序：
```java
@Override
public int getOrder(){
   return HIGHEST_PRECEDENCE + 100001;
}
```