### 简介
OpenFeign 是一种声明式调用，我们只需要按照一定的规则描述接口，它就能帮我们完成 REST 风格的调用，大大减少代码的编写量。
Github 的 OpenFeign 是一个第三方组件，Spring Cloud 将这个开源组件进行了封装，给出的规则完成采用了 Spring MVC 的风格
通过引入`spring-cloud-starter-openfeign`这个依赖来使用 OpenFeign，这个依赖包含了对 Ribbon 和 Hystrix 的支持

### OpenFeign 的使用

#### 基本使用
在 OpenFeign 中，只需要声明接口，而且风格是 Spring MVC 的，就可以使用
代码如下：
```java
@FeignClient("user") //声明为OpenFeign的客户端
public interface UserFacade{

/**
* 获取用户信息
*/
@GetMapping("/user/info/{id}")
  UserInfo getUser(@PathVariable("id") Long id);

/**
* 修改用户信息
*/
@PutMapping("/user/info")
  UserInfo putUser(@RequestBody UserInfo userInfo);
}
```
- @FeignClient("user")：表示这个几个接口是一个 OpenFeign 的客户端，底层将使用 Ribbon 执行 REST 风格调用，配置的 “user” 是一个微服务的名称
- @GetMapping("/user/info/{id}")：表示用 GET 请求调用用户微服务，其中的 `{id}`占位符与方法中的参数 @PathVariable("id") 对应
- @PutMapping("/user/info")：表示用 PUT 请求服务，这个请求需要一个 JSON 的请求体，因此参数使用 @RequestBody 来修饰 UserInfo，转换为 JSON 请求体
还需要在启动类中加上 `@EnableFeignClints` 注解来扫描带有 `@FeignClient` 的接口
```java
@SpringBootApplication
@EnableFeignClints
public class BorrowApplication{
public static void main(String[] args){
SpringApplication.run(BorrowApplication.class, args)
}
}
```
#### 常用的传参场景
还有三种传参方式：
- URL 传参，在地址后面加入参数
- 请求头传参，参数放在了请求头中
- 文件传参，调用的时候需要传递文件
代码如下：
```java
/**
* URL 传参
*/
@GetMapping("/user/infoes")
public ResponseEntity<List<UserInof>> findUser(@RequestParam("ids") Long[] ids);

/**
* 请求头传参
*/
@DeleteMapping("/user/info")
public ResultMessage deleteUser(@RequestHeader("id") Long id);

/**
* 传递文件流
*/
@RequestMapping(value="/user/upload",consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
public ResultMessage uploadFile(@RequestPart("file") MultipartFile file);
```

OpenFeign 客户端接口还可以提供继承的功能
通过在 facade 中指定相关的方法，发布到公共依赖中，然后其他服务再进行继承实现，由微服务的开发者来维护 facade 接口
```java
@FeignClient  
public interface UserFacade {  
    /**  
     * 获取用户信息 * @param id -- 用户编号 * @return 用户信息  
     */  
    @GetMapping("/user/info/{id}") // 注意方法和注解的对应选择  
    UserInfo getUser(@PathVariable("id") Long id);  
}
```
客户端代码如下：
```java
package com.spring.cloud.fund.client;
/**** imports ****/
@FeignClient(value="user")
// 此处继承UserFacade，那么OpenFeign定义的方法也能继承下来
public interface UserClient extends UserFacade {
}
```


### 配置 Hystrix
只用将 feign.hystrix.enabled 配置为 true 即可，OpenFeign 就会将接口方法的调用包装成一个 Hystrix 命令，然后采用断路器机制运行
#### 服务降级
先创建一个 UserFacade 接口的实现类，作为降级类（Fund 模块）
```java
package com.spring.cloud.fund.facade;  
  
/**** imports ****/ // 此处删除注解  
package com.spring.cloud.fund.fallback;  
/**** imports ****/  
/**  
 * 要使类提供降级方法，需要满足3个条件：  
 * 1. 实现OpenFeign接口定义的方法  
 * 2. 将Bean注册为Spring Bean  
 * 3. 使用@FeignClient的fallback配置项指向当前类  
 * @author ykzhen  
 * */@Component // 注册为Spring Bean ①  
public class UserFallback implements UserFacade { // ②  
  
    @Override  
    public UserInfo getUser(Long id) {  
        return new UserInfo(null, null, null);  
    }  
  
    @Override  
    public UserInfo putUser(UserInfo userInfo) {  
        return new UserInfo(null, null, null);  
    }  
  
    @Override  
    public ResponseEntity<List<UserInfo>> findUsers2(  
            // @RequestParam代表请求参数  
            @RequestParam("ids") Long []ids) {  
        return null;  
    }  
  
    @Override  
    public ResultMessage deleteUser(Long id) {  
        return new ResultMessage(false, "降级服务");  
    }  
  
  
    @Override  
    public ResultMessage uploadFile(MultipartFile file) {  
        return new ResultMessage(false, "降级服务");  
    }  
}
```
在 UserFacade 中配置降级类
```java
package com.spring.cloud.fund.facade;  
/**** imports ****/  
@FeignClient(value="user", fallback = UserFallback.class // ①  
/*, configuration=UserFacade.UserFeignConfig.class*/ // ②  
)  
public interface UserFacade  {   
}
```
但是这样的降级服务不能获取异信息，使用 fallbackFactory 配置项可以用 降级工厂 来获取异常信息
```java
package com.spring.cloud.fund.fallback;  
  
/**** imports ****/  
  
/**  
 * 定义OpenFeign降级工厂（FallbackFactory）分3步：  
 * 1. 实现接口FallbackFactory<T>定义的create方法  
 * 2. 将降级工厂定义为一个Spring Bean  
 * 3. 使用@FeignClient的fallbackFactory配置项定义  
 * @author ykzhen  
 * */@Component // ①  
public class UserFallbackFactory implements FallbackFactory<UserFacade> { // ②  
    /**  
     * 通过FallbackFactory接口定义的create方法参数获取异常信息  
     */  
    @Override  
    public UserFacade create(Throwable err) {  
        // 返回一个OpenFeign接口的实现类  
        return new UserFacade() {  
            // 错误编号  
            private Long ERROR_ID = Long.MAX_VALUE;  
  
            @Override  
            public UserInfo getUser(Long id) {  
                return new UserInfo(ERROR_ID, null, err.getMessage());  
            }  
  
            @Override  
            public UserInfo putUser(UserInfo userInfo) {  
                return new UserInfo(ERROR_ID, null, err.getMessage());  
            }  
  
            @Override  
            public ResponseEntity<List<UserInfo>> findUsers2(  
                    // @RequestParam代表请求参数  
                    @RequestParam("ids") Long []ids) {  
                return null;  
            }  
  
            @Override  
            public ResultMessage deleteUser(Long id) {  
                return new ResultMessage(false, err.getMessage());  
            }  
  
            @Override  
            public ResultMessage uploadFile(MultipartFile file) {  
                return new ResultMessage(false, err.getMessage());  
            }  
  
        };  
    }  
}
```
配置降级工厂：
```java
package com.spring.cloud.fund.facade;  
/**** imports ****/  
@FeignClient(value="user",  
/* fallback = UserFallback.class,*/  
// 配置降级工厂  
        fallbackFactory= UserFallbackFactory.class  
/* , configuration=UserFacade.UserFeignConfig.class*/)  
public interface UserFacade  {  
   ......  
}
```

#### 超时时间
在启用Hystrix的情况下，OpenFeign的调用会分为两个层次，一个是Ribbon，另一个是Hystrix
Ribbon或者Hystrix的配置中都有超时时间，且它们会相互影响。
在默认的情况下，Ribbon和Hystrix默认的超时都是1秒，在首次请求的时候，由于系统需要初始化很多对象且需要连通服务提供者，因此这有可能会导致OpenFeign客户端调用出现超时的情况。为了解决这个问题，我们可以通过在YAML文件中配置对应的超时时间来应对
```yaml
# Ribbon配置
ribbon:
   # 连接服务器超时时间（单位毫秒）
   ConnectTimeout: 3000
   # 调用超时时间（单位毫秒）
   ReadTimeout: 6000

# Hystrix配置
hystrix:
  # 自动配置一个Hystrix并发策略插件的hook，
  # 这个hook会将SecurityContext从主线程传输到Hystrix的命令。
  shareSecurityContext: true
  command:
    default: 
      execution:
        timeout:
          #是否启用Hystrix超时时间
          enable: true
        isolation:
          thread:
            # 配置Hystrix断路器超时时间（单位毫秒）
            timeoutInMilliseconds: 5000
```
我们需要将 Ribbon 的超时时间配置得大于 Hystrix 的默认超时时间，否则 Hystrix 的超时时间就没有意义了，因为 Hystrix 还未处理熔断时，Ribbon 就超时了

PS. hystrix 的超时指定的是 command 执行时的超时