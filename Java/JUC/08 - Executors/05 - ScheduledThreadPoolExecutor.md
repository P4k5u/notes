### 简介
ScheduledThreadPool 继承自 ThreadPoolExecutor  可以指定一定延迟时间后或者定时进行任务调度执行的线程池
跟 ThreadPoolExecutor 的工作机制类似，只是会返回一个 FutureTask ，并且 task queue 用的是 DelayWorkQueue
重要的两个部分：
- ScheduleFutureTask
- DelayedWorkQueue

### 相关方法

#### 创建 ScheduledThreadPoolExecutor
代码如下：
```java
/**  
 * Creates a new {@code ScheduledThreadPoolExecutor} with the  
 * given core pool size. * * @param corePoolSize the number of threads to keep in the pool, even  
 *        if they are idle, unless {@code allowCoreThreadTimeOut} is set  
 * @throws IllegalArgumentException if {@code corePoolSize < 0}  
 */public ScheduledThreadPoolExecutor(int corePoolSize) {  
    super(corePoolSize, Integer.MAX_VALUE,  
          DEFAULT_KEEPALIVE_MILLIS, MILLISECONDS,  
          new DelayedWorkQueue());  
}
```
因为 ScheduleThreadPoolExecutor 继承自 ThreadPoolExecutor，所以调用的还是 ThreadPoolExecutor 的构造方法

#### schedule() 方法
该方法的作用是提交一个延迟执行的任务，任务从提交时间算起延迟单位为 unit 的 delay 时间后开始执行。提交的任务不是周期性任务，任务只会执行一次
代码如下：
```java
public ScheduledFuture<?> schedule(Runnable command,  
                                   long delay,  
                                   TimeUnit unit) {  
    if (command == null || unit == null)  
        throw new NullPointerException();  
    RunnableScheduledFuture<Void> t = decorateTask(command,  
        new ScheduledFutureTask<Void>(command, null,  
                                      triggerTime(delay, unit),  
                                      sequencer.getAndIncrement()));  
    delayedExecute(t);  
    return t;  
}
```
这里会把提交的 Runnable 任务转变为 ScheduledFutureTask，其实就是转变为 Callable ，然后放进延迟队列里
#### scheduleWithFixedDelay() 方法
该方法的作用是，当任务执行完毕后，让其延迟固定时间后再次运行（fixed-delay 任务），任务会一直重复运行直到任务运行中抛出了异常，被取消了，或者关闭了线程
如果一个任务执行时间较长，它不会影响下一个任务的开始时间。任务的执行时间不会叠加到间隔中，而是等到上一个任务执行完成后，再等待 `delay` 时间后开始下一个任务

代码如下：
```java
public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,  
                                                 long initialDelay,  
                                                 long delay,  
                                                 TimeUnit unit) {  
    if (command == null || unit == null)  
        throw new NullPointerException();  
    if (delay <= 0L)  
        throw new IllegalArgumentException();  
    ScheduledFutureTask<Void> sft =  
        new ScheduledFutureTask<Void>(command,  
                                      null,  
                                      triggerTime(initialDelay, unit),  
                                      -unit.toNanos(delay),  
                                      sequencer.getAndIncrement());  
    RunnableScheduledFuture<Void> t = decorateTask(command, sft);  
    sft.outerTask = t;  
    delayedExecute(t);  
    return t;  
}
```
实现原理就是，任务添加到延迟队列之后，当任务过期时，就从队列中移除，并执行。
当一个任务在执行中抛出了异常，那么这个任务就结束了，并不会影响其他任务的执行
#### scheduleAtFixedRate() 方法
这个方法不用等待任务执行完毕，而是直接开始计时，时间到了就开始执行

如果一个任务执行时间较长，它会影响下一个任务的开始时间。任务的执行时间会叠加到间隔中，如果任务执行时间超过间隔时间，可能会导致任务之间没有间隔

代码如下：
```java
public ScheduledFuture<?> scheduleAtFixedRate(Runnable command,  
                                              long initialDelay,  
                                              long period,  
                                              TimeUnit unit) {  
    if (command == null || unit == null)  
        throw new NullPointerException();  
    if (period <= 0L)  
        throw new IllegalArgumentException();  
    ScheduledFutureTask<Void> sft =  
        new ScheduledFutureTask<Void>(command,  
                                      null,  
                                      triggerTime(initialDelay, unit),  
                                      unit.toNanos(period),  
                                      sequencer.getAndIncrement());  
    RunnableScheduledFuture<Void> t = decorateTask(command, sft);  
    sft.outerTask = t;  
    delayedExecute(t);  
    return t;  
}
```