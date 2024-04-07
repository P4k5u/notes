### 任务委托
当线程把任务委托给 ExecutorService 之后，该线程继续执行自己的任务，独立该任务
ExecutorService 并发执行指令，独立于委托任务的线程
![[Pasted image 20231228214347.png]]

### ExecutorService 使用例子
来个简单的使用例子，代码如下：
```java
ExecutorService executorService = Executors.newFixedThreadPool(10);

executorService.execute(new Runnable() {
    public void run() {
        System.out.println("Asynchronous task");
    }
});

executorService.shutdown();
```

### ExecutorService 相关实现
Java ExecutorService 跟线程池很像。事实上，现在 ExecutorService 在 Java 中的实现就是一个线程池的实现 

ExecutorService 是一个接口，有以下两种实现：
- ThreadPoolExecutor
- ScheduledThreadPoolExecutor
### ExecutorService 使用

#### 创建 ExecutorService
如何创建 ExecutorService 取决于你使用哪个实现，但你可以用 Executors 工厂类来创建 ExecutorService 实例，下面是使用方法：
```java
ExecutorService executorService1 = Executors.newSingleThreadExecutor();

ExecutorService executorService2 = Executors.newFixedThreadPool(10);

ExecutorService executorService3 = Executors.newScheduledThreadPool(10);
```

#### 相关方法
有几种方法可以将任务委托给 ExecutorService：
- execute(Runnable)
- submit(Runnable)
- submit(Callable)
- invokeAny(...)
- invokeAll(...)

##### Execute Runnable
Executor execute(Runnable) 方法传入一个 Runnable 对象，并且异步执行
代码如下：
```java
ExecutorService executorService = Executors.newSingleThreadExecutor();

executorService.execute(new Runnable() {
    public void run() {
        System.out.println("Asynchronous task");
    }
});

executorService.shutdown();
```
这会没有办法获得执行结果，如果要获取结果的话，可以使用 Callable
##### Submit Runnable
ExecutorService submit(Runnable) 同样是传入一个 Runnable，但是他返回了一个 Future 对象，这个 Future 对象可以用来查看 Runnable 是否执行完毕

以下是一个例子：
```java
Future future = executorService.submit(new Runnable() {
    public void run() {
        System.out.println("Asynchronous task");
    }
});

future.get();  //returns null if the task has finished correctly.
```

##### Submit Callable
ExecutorService submit(Callable) 方法跟 submit(Runnable) 类似，但是会返回一个 Future 对象来包装结果

代码如下：
```java
Future future = executorService.submit(new Callable(){
    public Object call() throws Exception {
        System.out.println("Asynchronous Callable");
        return "Callable Result";
    }
});

System.out.println("future.get() = " + future.get());
```
输出结果如下：
```java
Asynchronous Callable
future.get() = Callable Result
```

##### invokeAny()
invokeAny() 方法需要传入一个 Callable 集合，这会随机返回一个 Callable 执行的结果（不是返回 Future）
当其中一个 Callable 执行完成或抛出异常，剩下的 Callable 会被取消

代码如下：
```java
ExecutorService executorService = Executors.newSingleThreadExecutor();

Set<Callable<String>> callables = new HashSet<Callable<String>>();

callables.add(new Callable<String>() {
    public String call() throws Exception {
        return "Task 1";
    }
});
callables.add(new Callable<String>() {
    public String call() throws Exception {
        return "Task 2";
    }
});
callables.add(new Callable<String>() {
    public String call() throws Exception {
        return "Task 3";
    }
});

String result = executorService.invokeAny(callables);

System.out.println("result = " + result);

executorService.shutdown();
```

##### invokeAll()
同样也是传入一个 Callable 集合，但是返回的是 Future 对象的集合来获取每个 Callable 的执行结果

需要注意的是，这只是返回结果，Future 对象不知道是否执行成功
代码如下：
```java
ExecutorService executorService = Executors.newSingleThreadExecutor();

Set<Callable<String>> callables = new HashSet<Callable<String>>();

callables.add(new Callable<String>() {
    public String call() throws Exception {
        return "Task 1";
    }
});
callables.add(new Callable<String>() {
    public String call() throws Exception {
        return "Task 2";
    }
});
callables.add(new Callable<String>() {
    public String call() throws Exception {
        return "Task 3";
    }
});

List<Future<String>> futures = executorService.invokeAll(callables);

for(Future<String> future : futures){
    System.out.println("future.get = " + future.get());
}

executorService.shutdown();
```
##### Runnable 和 Callable 的区别
Runnable 可以被 线程 或 ExecutorService 执行，Callable 只可以被 ExecutorService 执行

同样，在两者的接口中可以更加清楚的看到差别
Runnable 的接口声明如下：
```java
public interface Runnable {
    public void run();
}
```
Callable 的接口声明如下：
```java
public interface Callable{
    public Object call() throws Exception;
}
```
主要区别就是 Callable 的 call() 方法可以返回一个 对象 和 抛出异常

所以当你需要结果时最好用 callable 
##### Cancel Task
可以通过调用 Future.cancel() 方法来取消 task，前提是 task 还没开始执行

代码如下：
```java
future.cancel();
```

##### ExecutorService Shutdown
当使用 ExecutorService 之后，应该把它关掉，这样线程就不会继续运行，如果应用是在 main() 方法启动，主线程退出的时候如果还有活动的线程在 ExecutorService 里，那么应用程序会继续进行，因为 ExecutorService 里的线程可以防止被 JVM 关闭

###### shutdown()
调用 shutdown() 后，ExecutorService 不会立即停止，而是会等待所有的 task 完成之后，才会停止，
在 shutdown 之前 submit 的 task 也会被执行

代码如下：
```java
executorService.shutdown();
```

###### shutdownNow()
这会立刻停止所有正在执行的任务，被提交的还没开始执行的任务会被忽略

代码如下：
```java
executorService.shutdownNow();
```