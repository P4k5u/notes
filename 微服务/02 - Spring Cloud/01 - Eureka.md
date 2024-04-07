### 简介
Eureka 作为一个微服务的治理中心，它是一个服务应用，可以接收其他服务的注册，也可以发现和治理服务实例
在微服务架构中，每一个微服务都可以通过集群或其他方式进行动态扩展，每一个微服务实例的网络地址都可能动态变化，这使得原本通过硬编码地址的调用失去了作用
所以，在微服务架构中，服务地址的变化和数量变化，迫切需要系统建立一个`中心化的组件`，对各个微服务实例信息进行登记和管理，同时让各个微服务之间能够互相发现，从而互相调用
- 服务注册：各个微服务连接到注册中心登记自己的相关信息（服务名、ip、端口等），通过注册表，各个微服务就能明确知道在网络环境中有哪些服务存在，以及如何联系上他们
- 服务发现：各个微服务会从注册中心下载注册表（所有登记到注册中心的微服务信息），通过注册表，各个微服务就能明确知道在网络环境有些服务存在，以及如何联系上他们![[Pasted image 20240110171554.png]]

Eureka 由两部分组成：
- Eureka Server，提供服务注册和发现功能
- Eureka Client，定时将自己的注信息登记到 Eureka Server 中，并从 Eureka Server 中下载包括其他 Eureka Client 信息的注册表
### 服务注册中心

基于 SpringBoot 来搭建 Eureka Server
启动类如下：
```java
@EnableEurekaServer
@SpringBootApplication 
public class EurekaServerApplication { public static void main(String[] args) { SpringApplication.run(EurekaServerApplication.class, args); } }
```
通过 @EnableEurekaServer 这个注解来启动 Eureka 服务器

相关的 application.yml 配置如下：
```yaml
server:
  port: 8888
eureka:
	# 开启之前需要修改一下客户端设置（虽然是服务端
  client:
  	# 由于我们是作为服务端角色，所以不需要获取服务端，改为false，默认为true
		fetch-registry: false
		# 暂时不需要将自己也注册到Eureka
    register-with-eureka: false
```
启动完成之后，直接输入地址+端口即可访问 Eureka 的管理后台
![[Pasted image 20240110175103.png]]
### 服务发现
搭建了服务注册中心之后，要往里面注册微服务及其实例
依赖于 spring-cloud-starter-netflix-eureka-client 包，依靠它把当前的 SpringBoot 模块注册到 Eureka 服务治理中心

启动类如下：
```java
@SpringBootApplication
//@EnableDiscoveryClient 这个注解在新版中不再使用
public class UserApplication{
  public static void main(String[] args){
  SpringApplication.run(UserApplication.class,args);
  }
}
```
代码中的 @EnableDiscoveryClient 注解是一个用于服务发现的注解，但是在新版本中就不再使用，当这个服务启动后，就会根据配置项 eureka.client.serviceUrl.defaultZone 发送相关的请求，注册实例。
服务注册功能是在服务启动成功后，间隔一个时间戳才会执行

运行之后，就把服务注册到 Eureka Server
在进行服务调用的时候，就可用服务名来替换地址
```java
User user = template.getForObject("http://localhost:8082/user/"+uid, User.class);
```
```java
User user = template.getForObject("http://userservice/user/"+uid, User.class);
```
### 高可用服务治理中心
注册多个 Eureka Server 来提供服务，配置如下
EurekaServer01：
```yaml
server:
  port: 8801
spring:
  application:
    name: eurekaserver
eureka:
  instance:
  	# 由于不支持多个localhost的Eureka服务器，但是又只有本地测试环境，所以就只能自定义主机名称了
  	# 主机名称改为eureka01
    hostname: eureka01
  client:
    fetch-registry: false
    #去掉register-with-eureka选项，让Eureka服务器自己注册到其他Eureka服务器，这样才能相互启用
    service-url:
    	# 注意这里填写其他Eureka服务器的地址，不用写自己的
      defaultZone: http://eureka01:8801/eureka
```
EurekaServer02：
```yaml
server:
  port: 8802
spring:
  application:
    name: eurekaserver
eureka:
  instance:
    hostname: eureka02
  client:
    fetch-registry: false
    service-url:
      defaultZone: http://eureka01:8801/eureka
```
EurekaClient 的配置也要修改下：
```yaml
eureka:
  client:
    service-url:
    	#将两个Eureka的地址都加入，这样就算有一个Eureka挂掉，也能完成注册
      defaultZone: http://localhost:8801/eureka, http://localhost:8802/eureka
```

### 服务注册中心工作原理

#### 微服务实例和服务注册中心的关系
任何的微服务都可以对 Eureka 服务注册中心发送 REST 风格的请求，在 Eureka 的机制中，一般是由具体的微服务来主动维持它们之间的关系。Eureka 客户端的请求类型包括注册、续约和下线
- 注册
	将具体的微服务实例注册到 Eureka 服务端时，是通过 REST 风格请求其配置的属性 `eureka.client.serviceUrl.defaultZone` 生成的 URL 来完成，微服务会把自身的信息传递给 Eureka 服务器。同时通过 `spring.application.name` 作为微服务名称来定义。配置项 `eureka.client.register-with-eureka` 代表是否将服务注册到注册中心
- 续约
	微服务实例会按照一个频率对 Eureka 服务器维持心跳，告诉 Eureka 服务端服务可用，不然就会被剔除出去，这样的行为称为 续约（Renew）
- 下线
	当服务不可用时，实例会对 Eureka 发送下线请求，告知服务注册中心，这样这个实例就不能被请求了

#### 服务注册中心
通过注册、续约和下限这三种服务，Eureka 可以有效地管理具体的微服务实例。但是服务注册中心和本身也会提供一定的服务，甚至可以说服务注册中心也是 Eureka 客户端，因为它也可以注册到其他的 Eureka 服务器中，被其他的 Eureka 服务器治理
- 相互复制
	Eureka 本身也会相互注册，以保证高可用和高性能。各个 Eureka 服务器之间也会相互复制，也就是当微服务发生注册、下线和续约这些操作时，Eureka 会将这些消息转发到其他服务注册中心上，这里的 Eureka 服务器之间采用的是对等模式（Peer-to-Peer），也就是每个 Eureka 都是等价的，这跟主从模式不太一样
- 服务剔除
	在实际工作中，有时候有些服务会因为网络故障、内存溢出或宕机而导致服务不能正常工作，这个时候就要剔除服务。Eureka Server 在启动时，会创建一个 定时任务，在默认情况下，每 60s 就会更新一次微服务实例的list，超过 90s 没续约的实例就会被剔除出去
- 自我保护
	在本机测试时，会触发 Eureka 的自我保护机制，实例启动之后都会查找 Eureka 进行注册，注册之后也会通过心跳来告诉自己还活着，在 15 分钟内低于 85% 的情况下心跳测试失败，就会出现警告

#### 微服务之间的调用
在调用过程中，可以用 Ribbon 来进行负载均衡，整体调用过程如下：
- 服务获取
	服务获取指的是向 Eureka 服务端获取其他服务实例清单的功能，同时会把清单缓存在本地，以一定的时间间隔去刷新清单
- 服务调用
	通过服务名在获取的服务清单中以某种负载均衡算法来进行服务实例的调用
#### Region 和 Zone
通过 Region 和 Zone 概念的设计，可以将机房设置在不同的地区，从而解决距离问题和各个地区业务的差异，进一步提高微服务系统的响应能力和灵活性

Region 和 Zone 是来自 AWS 平台的概念，之所以提出这样的概念，是因为亚马逊是全球服务的公司，它的站点是全球范围的。
不同地区的调用服务有出现两个问题：
- 距离问题：长距离的网络通信会造成一定的延迟，多达几十毫秒
- 地区差异问题：每个国家或地区的习俗和法制基本都不一样，存在着很大的地区差异，所以一个服务在不同的国家或地区需要采用不同的业务模式
为了解决这两个问题，提出了 Region 和 Zone 的概念，例如：在中国这个 Region 的基础上再进行划分，将南方地区划分为一个 Zone ，将服务站点设在广州，这样南方地区的请求就优先路由到广州

在 Eureka 中也是一样的，在需要大型分布式站点的时候，微服务之间的 REST 风格请求交互，也应该采用就近原则。配置如下：
```yaml
eureka:
  client:
    region: China
    availability-zones: beijing
```
这样，北方的站点就会主要路由到北京站点，微服务实例在协调的时候也会采用就近原则![[Pasted image 20240111162933.png]]

### Eureka 使用注意点
Eureka 是一个强调 AP 的组件，
可用性：Eureka 的机制是通过各种 REST 风格的请求来监控各个微服务以及其他 Eureka 服务器是否可用，在一定情况下会进行剔除