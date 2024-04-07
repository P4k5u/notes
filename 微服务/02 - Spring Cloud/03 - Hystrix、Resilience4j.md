### 简介
熔断用来防治服务不可用的时候引发的雪崩效应，可以通过隔离不可用的服务来保证其他相关依赖服务的可用
服务降级用来应对系统繁忙的情况，当服务能力达到上限时，通过服务降级返回静态资源来适当缓解服务压力，防止请求挤压造成瘫痪
分布式中的限流组件 Hystrix、Resilience4j 就是用来以上的问题
### Hystrix

#### 功能简介
- 防止单个服务的故障导致其依赖和相关的容器线程资源耗尽
- 减少负载，并提供请求快速失败的功能，而不是排队
- 提供服务降级功能
- 使用隔离技术来限制某个服务出现问题所带来的影响
- 提供实时监控信息
Hystrix 的底层实现是 RxJava，是流形式的，采用观察者模式实现
#### 使用方法
使用 @EnableCircuitBreaker 就可以使用启动断路器了
```java
package com.spring.cloud.product.main; 
@SpringBootApplication(scanBasePackages="com.spring.cloud.product")
@EnableCircuitBreaker public class ProductApplication { ...... }
```
定一个 Facade 接口，表示的是外部微服务的调用
```java
package com.spring.cloud.product.facade; import com.spring.cloud.common.vo.ResultMessage; public interface UserFacade { public ResultMessage timeout(); public ResultMessage exp(String msg); }
```
实现如下：
```java
package com.spring.cloud.product.facade.impl;
/**** imports ****/
@Service
public class UserFacadeImpl implements UserFacade {
   // 注入RestTemplate，在Ribbon中我们标注了@LoadBalance，用以实现负载均衡
   @Autowired
   private RestTemplate restTemplate = null;

   @Override
   // @HystrixCommand将方法推给Hystrix进行监控
   // 配置项fallbackMethod指定了降级服务的方法
   @HystrixCommand(fallbackMethod = "fallback1")
   public ResultMessage timeout() {
      String url = "http://USER/hystrix/timeout";
      return restTemplate.getForObject(url, ResultMessage.class);
   }

   @Override
   @HystrixCommand(fallbackMethod = "fallback2")
   public ResultMessage exp(String msg) {
      String url = "http://USER/hystrix/exp/{msg}";
      return restTemplate.getForObject(url, ResultMessage.class, msg);
   }

   // 降级方法1
   public ResultMessage fallback1() {
      return new ResultMessage(false, "超时了");
   }

   /**
   * 降级方法2，带有参数
   * @Param msg -- 消息
   * @Return ResultMessage -- 结果消息 
   **/
   public ResultMessage fallback2(String msg) {
      return new ResultMessage(false, "调用产生异常了，参数:" + msg);
   }
}
```
通过加入 @HystrixCommand 注解来将整个方法封装成一个 Hystrix 命令，这是通过 AOP 实现的。`fallbackMethod` 配置项用来指定降级方法，当正常的服务无法按预想完成（如超时、异常、服务繁忙等）就会跳转到降级方法执行
#### 工作原理
Hystrix 的工作流程如下：![[Pasted image 20240116155908.png]]
##### Hystrix Command
工作流程中的，第一步就是要封装 Hystrix Command，一个是 HystrixCommand（同步请求命令） ，另一个是 HystrixObservableCommand （异步请求命令）
Hystrix 会把服务消费者的请求封装成一个 HystrixCommand 对象或HystrixObservableCommand 对象，从而可以对不同的请求进行客制化，这便是一种命令模式，可以达到“行为请求者”和“行为执行实现者”解耦的目的
HystrixCommand 是传递参数，然后单个数据单元响应；HystrixObservableCommand 是返回一个可观察者来发射请求
```java
HystrixCommand command = new HystrixCommand(arg1, arg2);

HystrixObservableCommand command = new HystrixObservableCommand(arg1, arg2);
```

执行命令的四种方式：
- execute()：该方法是阻塞的，从请求中接受单个响应，出错时抛出异常
- queue()：从请求中返回一个包含单个响应的 Future 对象
- observe()：从请求中返回一个代表响应的 Observable 对象
- toObservable()：返回一个 Observable 对象，只有当你订阅它时，它才会执行 Hystrix 命令并发射响应
```java
K value   = command.execute();
Future<K> fValue = command.queue();
Observable<K> ohValue = command.observe();         // hot observable
Observable<K> ocValue = command.toObservable();    // cold observable
```
在 HystrixObservableCommand 中有 hot observable 和 cold observable
- hot observable：观察者只要订阅了消息源，消息源就会立刻将消息传递给观察者
- cold observable：观察者订阅了消息源，也不会立刻将消息传递给观察者，观察者需要再次发送命令，才会传递消息

HystrixCommand 的 execute() 方法本质上也是调用了 queue() 方法
```java
public R execute() {
   try { // 调用queue方法
      return queue().get();
   } catch (Exception e) {
      throw Exceptions.sneakyThrow(decomposeException(e));
   }
}

public Future<R> queue() { 
   // 调用toObservable方法，进行cold observable，生成一个阻塞的观察者，
   //这是一个任务代理。
   final Future<R> delegate = toObservable().toBlocking().toFuture();
   // 创建Future对象，返回异步
   final Future<R> f = new Future<R>() {
      ......
   };
   /* 对于出现的异常和错误，立即进行处理 */
   if (f.isDone()) {
      try {
         f.get();
         return f;
      } catch (Exception e) { // 异常处理
         Throwable t = decomposeException(e);
         if (t instanceof HystrixBadRequestException) {
            return f;
         } else if (t instanceof HystrixRuntimeException) {
            HystrixRuntimeException hre = (HystrixRuntimeException) t;
            switch (hre.getFailureType()) {
            case COMMAND_EXCEPTION: // 命令错误
            case TIMEOUT: // 超时
               // we don't throw these types from queue() 
               // only from queue().get() as they are execution errors
               return f;
            default: // 其他情况
               // these are errors we throw from queue() 
               // as they as rejection type errors
               throw hre;
            }
         } else { // 其他异常，抛出
            throw Exceptions.sneakyThrow(t);
         }
      }
   }
   return f;
}
```
HystrixCommand的execute方法，其实是调用自己的queue方法来实现的，而queue方法是调用toObservable方法来实现Observable的。queue方法的最后目的只是获取一个Future对象，这个对象包含最后返回的结果或者异常的处理
##### 断路器
当命令查询没有缓存的时候，请求就会到达断路器，在 Hystrix 中，断路器有 3 种状态
- CLOSED
- OPEN
- HALF_OPEN
状态之间的转换如下图：
![[Pasted image 20240116163116.png]]
- CLOSED -> OPEN
	通过 subscribeToStream() 方法获取统计分析的数据，用来判断是否转变状态
- OPEN -> HALF_OPEN
	状态转变为 OPEN 之后，就会阻隔请求，打开超过一定时间之后，就会进入 HALF_OPEN 状态，此时可以进行尝试请求，调用 attemptExecution() 方法，成功 -> CLOSED；失败 -> OPEN（调用 markNonSuccess() 方法）
熔断器 HystrixCircuitBreaker 的接口定义如下：
```java
public interface HystrixCircuitBreaker {

   // 判断断路器是否允许发送请求
   boolean allowRequest();

   // 判断断路器是否已经打开
   boolean isOpen();

   // 当执行成功时，记录结果，可能重新关闭断路器（CLOSED）
   void markSuccess();

   // 当执行不成功时，记录结果
   void markNonSuccess();

   // 尝试执行Hystrix命令，这是一个非幂等性的方法，可以修改断路器内部的状态
   boolean attemptExecution();

   // 使用工厂生产断路器
   class Factory {......}

   // 提供默认实现
   class HystrixCircuitBreakerImpl implements HystrixCircuitBreaker {......}

   // 提供空实现
   static class NoOpCircuitBreaker implements HystrixCircuitBreaker{......}
}
```
HystrixCommandMetrics 命令度量，用来决定断路器状态的转变，核心概念是 时间窗 和 桶
判定是否打开断路器的算法是 滑动窗口统计法，在每一个时间段内，统计服务调用的结果，桶 是窗口数据数据的最小单位，通过把时间窗口再细分为桶，来保证统计的数据具有实时性，当调用溢出桶的大小时，会丢弃最早的记录
默认配置断路器的工作流程如下：
![[Pasted image 20240116165120.png]]
最后两排数据就是时间窗，当中的每个格子就是桶，每个桶存放4个数字的数据，分别记录成功（Success）、失败（Failure）、超时（Timeout）和拒绝（Rejection）的次数，这样就可以分析时间窗（包含10个桶）的数据，决定断路器的状态了。
当超过10秒，需要进行新的统计的时候，看第二排数据，可以看到它会废弃最旧的桶的数据，然后创建一个新桶进行统计，这样就可以始终保持10个桶存放度量数据了

##### 隔离
在 Hystrix 中，使用的隔离为 舱壁（Bulkhead）
在正常情况下，单个线程池是可以工作的，能够完成对各个服务的调用，但是都某个服务的请求增多，就会导致线程池占满，导致其他服务阻塞 ![[Pasted image 20240116165356.png]]
使用舱壁模式，将某个服务的调度单独隔离出来，这样就不会影响到别的服务 ![[Pasted image 20240116165437.png]]

但是使用了舱壁模式后，需要维护多个线程池，以及服务调用的分配，尤其是线程上下文的相互切换，这使得系统更复杂，开销更大。关于这些，Netflix在设计Hystrix的时候已经考虑到了，Netflix觉得舱壁模式隔离所带来的好处远远超过了使用单线程池的模式，并且认为这样的模式不会带来过大的开销与成本。Hystrix在子线程执行construct()方法和run()方法时会计算延迟，还会计算父线程从端到端执行的总时间。为了让大家打消对这种选择的疑虑，Netflix公司对其做了大量的测试，以下是Netflix发布的关于Hystrix的分析数据![[Pasted image 20240116165706.png]]

在极端场景下，性能可能差距会较大，Hystrix 提供了信号量 的处理方式
```java
public static void useSemaphore() {
   // 线程池
   ExecutorService pool = Executors.newCachedThreadPool();
   // 信号量，这里采用3个许可信号，线程间采用公平机制
   final Semaphore semaphore = new Semaphore(3, true);
   // 循环10次
   for (int i = 0; i < 10; i++) {
      Runnable runnable = new Runnable() {
         @Override
         public void run() {
            try {
               // 获取许可信号，获取不到的线程会被阻塞，等待唤起
               semaphore.acquire();
               System.out.println("线程：" + Thread.currentThread().getName()
                      + " 进入当前系统的并发数是：" 
                      + (3 - semaphore.availablePermits()));
               // 线程休眠一个随机时间戳（1秒内）
               Thread.sleep(new Random().nextInt(1000));
            } catch (InterruptedException e) {
               e.printStackTrace();
            }
            System.out.println("线程：" + Thread.currentThread().getName()
                   + " 即将离开");
            // 释放许可信号
            semaphore.release();
            System.out.println("线程：" + Thread.currentThread().getName()
                  + " 已经离开，当前系统的并发数是："
                  + (3 - semaphore.availablePermits()));
         }
      };
      pool.execute(runnable); // 执行线程池任务
   }
}
```
#### 封装 Hystrix Command
##### 使用 @HystrixCommand 同步执行
```java
@HystrixCommand(fallbackMethod = "fallback1")  
public ResultMessage timeout() {  
    String url = "http://USER/hystrix/timeout";  
    return restTemplate.getForObject(url, ResultMessage.class);  
}  
  
public ResultMessage fallback1() {  
    return new ResultMessage(false, "超时了");  
}
```
##### 使用 @HystrixCommand 异步执行
```java
@HystrixCommand(fallbackMethod = "fallback1")  
public Future<ResultMessage> asyncTimeout() {  
    return new AsyncResult<ResultMessage>() {  
        @Override  
        public ResultMessage invoke() {  
            String url = "http://USER/hystrix/timeout";  
            return restTemplate.getForObject(url, ResultMessage.class);  
        }  
    };  
}
```
这里使用抽象类 AsyncResult ，来得到一个 Future 对象，再通过 get 方法来拉取数据
##### 使用 @HystrixObservable 来执行 HystrixObservableCommand 
```java
@HystrixCommand(fallbackMethod = "fallback3",  
        // 执行模式  
        observableExecutionMode = ObservableExecutionMode.EAGER)  
public Observable<ResultMessage> asyncExp(String[] params) {  
    String url = "http://USER/hystrix/exp/{msg}";  
    // 行为描述  
    Observable.OnSubscribe<ResultMessage> onSubs = (resSubs) ->{  
        try {  
            int count = 0; // 计数器  
            if (!resSubs.isUnsubscribed()) {  
                for (String param : params) {  
                    count ++;  
                    System.out.println("第【" + count + "】次发送 ");  
                    // 观察者发射单次参数到微服务  
                    ResultMessage resMsg  
                            = restTemplate.getForObject(  
                            url, ResultMessage.class, param);  
                    resSubs.onNext(resMsg);  
                }  
                // 遍历所有参数后，发射完结  
                resSubs.onCompleted();  
            }  
        } catch (Exception ex) {  
            // 异常处理  
            resSubs.onError(ex);  
        }  
    };  
    return Observable.create(onSubs);  
}  
  
public ResultMessage fallback3(String[] params) {  
    return new ResultMessage(false, "调用产生异常了，参数:" + params);  
}
```
#### 请求合并
Hystrix 提供了请求合并的功能，在一个很短的时间戳内，按照一定的规则进行判断，如果觉得是同样的请求，就将其合并为同一个请求，只用一条线程进行请求，然后响应多个请求
请求合并的作用域可以是 全局有效的（GLOBAL），也可以是 单次请求有效的（REQUEST），默认为单次请求有效

Hystrix 中提供请求合并的类是 HystrixCollapser ，是一个抽象类，可以通过 @HystrixCollapser 来实现
注解的源码如下：
```java
@Target({ElementType.METHOD})  
@Retention(RetentionPolicy.RUNTIME)  
@Documented  
public @interface HystrixCollapser {  
    // 合并器键  
    String collapserKey() default "";  
  
    // 合并方法，要求是一个Hystrix命令  
    String batchMethod();  
  
    // 合并器作用域，默认请求范围（可以配置为全局性的）  
    Scope scope() default Scope.REQUEST;  
  
    // 合并器属性  
    HystrixProperty[] collapserProperties() default {};  
}
```
使用方法如下：
```java
@Override  @HystrixCollapser(collapserKey="userGroup",  
        // 指定合并方法，必需项  
        batchMethod = "findUsers2",  
        //合并器作用域  
        scope = com.netflix.hystrix.HystrixCollapser.Scope.GLOBAL,  
        collapserProperties = {  
                // 限定合并时间戳为50 ms  
                @HystrixProperty(name = "timerDelayInMilliseconds", value = "50"),  
                // 合并最大请求数设置为3  
                @HystrixProperty(name="maxRequestsInBatch", value="3")  
        })  
public Future<UserInfo> getUser2(Long id) {  
    // 不需要任何逻辑  
    return null;  
}  
  
// 定义合并Hystrix命令  
@HystrixCommand(commandKey="userGroup")  
@SuppressWarnings("unchecked")  
@Override  
public List<UserInfo> findUsers2(List<Long> ids) {  
    String url = "http://USER/user/infoes/{ids}";  
    String strIds = StringUtils.join(ids, ",");  
    System.out.println("准备批量发送请求=》" + strIds);  
    // 定义转换最终类型  
    ParameterizedTypeReference<List<UserInfo>> responseType  
            = new ParameterizedTypeReference<List<UserInfo>>(){};  
    // 发生GET请求  
    ResponseEntity<List<UserInfo>> userEntity = restTemplate  
            .exchange(url, HttpMethod.GET, null, responseType,  strIds);  
    return userEntity.getBody();  
}
```
#### 请求缓存
Hystrix 的缓存是基于单次请求的，所以非本次请求是没法读取请求缓存的。一个请求在通过服务调用访问数据后，会将其缓存，在该请求再次通过同样服务调用获取数据时，就可以直接读取缓存
但是两次读取缓存期间的时间间隔可能会引发数据的不一致，可以通过缩小缓存超时时间来缓解

在 Hystrix 中可以使用注解来启用缓存，主要逻辑是覆盖 AbstractCommand<> 中的 getCacheKey() 方法

提供了三个注解：
- @CacheResult：将请求结果缓存，通过配置项 cacheKeyMethod 指定缓存 key 的生成方法
- @CacheRemove：将请求结果删除
- @CacheKey：作用在参数上，标记缓存的 key ，如果没有标注，则使用所有的参数
代码如下：
```java
    @CacheResult  
// 在默认情况下，命令键（commandKey）指向方法名getUserInfo  
    @HystrixCommand  
    @Override  
// @CacheKey 将参数id设置为缓存key  
    public UserInfo getUserInfo(@CacheKey Long id) {  
        String url = "http://USER/user/info/{id}";  
        System.out.println("获取用户" + id);  
        return restTemplate.getForObject(url, UserInfo.class, id);  
    }  
  
    // commandKey指定命令键，指向getUserInfo方法  
    @CacheRemove(commandKey ="getUserInfo")  
    @HystrixCommand  
    @Override  
    public UserInfo updateUserInfo(@CacheKey("id") UserInfo user) {  
        String url = "http://USER/user/info";  
        // 请求头  
        HttpHeaders headers = new HttpHeaders();  
        headers.setContentType(MediaType.APPLICATION_JSON_UTF8);  
        // 封装请求实体对象，将用户信息对象设置为请求体  
        HttpEntity<UserInfo> request = new HttpEntity<>(user, headers);  
        System.out.println("执行更新用户" + user.getId());  
        // 更新用户信息  
        restTemplate.put(url, request);  
        return user;  
    }
```
也可以通过配置项 cacheKeyMethod 来指定 cacheKey 生成方法
```java
// 将结果缓存，cacheKeyMethod指定key的生成方法  
    @CacheResult(cacheKeyMethod = "getCacheKey")  
// commandKey声明Hystrix命令键为“user_get”   
@HystrixCommand(commandKey = "user_get")  
    @Override  
// 由于@CacheResult的配置项cacheKeyMethod高于@CacheKey，因此@CacheKey此处无效  
    public UserInfo getUserInfo(@CacheKey Long id) {  
        String url = "http://USER/user/info/{id}";  
        System.out.println("获取用户" + id);  
        return restTemplate.getForObject(url, UserInfo.class, id);  
    }  
  
    // commandKey指定命令键，从而指向getUserInfo方法； cacheKeyMethod指定key的生成方法  
    @CacheRemove(commandKey = "user_get", cacheKeyMethod = "getCacheKey")  
    @HystrixCommand  
    @Override  
    public UserInfo updateUserInfo(UserInfo user) {  
        String url = "http://USER/user/info";  
        // 请求头  
        HttpHeaders headers = new HttpHeaders();  
        headers.setContentType(MediaType.APPLICATION_JSON_UTF8);  
        // 封装请求实体对象，将用户信息对象设置为请求体  
        HttpEntity<UserInfo> request = new HttpEntity<>(user, headers);  
        System.out.println("执行更新用户" + user.getId());  
        // 更新用户信息  
        restTemplate.put(url, request);  
        return user;  
    }  
  
    private static final String CACHE_PREFIX = "user_"; // 前缀   
/** 两个方法参数和命令方法保持一致 **/  
    public String getCacheKey(Long id) {  
        return CACHE_PREFIX + id;  
    }  
  
    public String getCacheKey(UserInfo user) {  
        return CACHE_PREFIX + user.getId();  
    }
```
想要 Hystrix 缓存生效，要先启动 HystrixRequestContext ，这个可以放进过滤器中
代码如下：
```java
// @WebFilter表示Servlet过滤器，  
@WebFilter(filterName = "HystrixRequestContextFilter",  
        //urlPatterns定义拦截的地址  
        urlPatterns = "/user/info/cache/*")  
@Component // 表示被Spring扫描，装配为Servlet过滤器  
public class HystrixRequestContextFilter implements Filter {  
  
    @Override  
    public void doFilter(ServletRequest request, ServletResponse response,  
                         FilterChain chain) throws IOException, ServletException {  
        // 初始化上下文  
        HystrixRequestContext context  
                = HystrixRequestContext.initializeContext();  
        try {  
            chain.doFilter(request, response);  
        } finally {  
            // 关闭上下文  
            context.shutdown();  
        }  
    }  
}
```


#### 线程池划分
Hystrix 隔壁是采用舱壁模式进行隔离的，也就是会存在很多独立的线程池

Hystrix 中的三个概念：
- groupKey：必须配置，表示组别
- commandKey：不必需，默认为当前类名，表示命令
- threadPoolKey：表示不用的线程池，需要在同一组中分线程池时才配置

Hystrix 对线程池的分配是按照组别分配的。在需要统计的时候，它还会根据命令键的维度进行统计。通过制定对应的组别键，就可以让那些命令在同一个线程池内运行了
在同一个组别键下，安排不同的线程池来运行不同的 Hystrix command，就需要指定 threadPoolKey

用 @HystrixCommand 来配置
```java
@HystrixCommand(
    // 命令键，默认值为标注方法名称
    commandKey = "user_get",
    // 组别键，默认值为当前运行类名称
    groupKey = "userGroup",
    // 线程池键
    threadPoolKey = "pool-user-1" 
)
```

#### 监控
### Resilience4j

### Spring Cloud Circuit Breaker