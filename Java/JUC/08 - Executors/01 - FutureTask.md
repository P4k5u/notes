### 简介
FutureTask 为了 Future 提供了基础实现，如：获取任务执行结果和取消任务等。如果任务尚未完成，获取任务执行结果时将会阻塞。一旦执行结束，任务就不能被重启或取消。
FutureTask 常用来封装 Callable 和 Runnable ，也可以作为一个任务提交到线程池中执行
FutureTask 的线程安全由 CAS 来保证

### 工作原理
FutureTask 类关系图如下：![[Pasted image 20240120215227.png]]
FutureTask 实现了 RunnableFuture 接口，而 RunnableFuture 接口继承了 Runnable 接口和 Future 接口，所以 FutureTask 既能当作一个 Runnable 被 Thread 执行，也能作为 Future 来得到 Callable 结果

#### 核心属性
FutureTask 的源码如下：
```java
/**  
 * The run state of this task, initially NEW.  The run state * transitions to a terminal state only in methods set, * setException, and cancel.  During completion, state may take on * transient values of COMPLETING (while outcome is being set) or * INTERRUPTING (only while interrupting the runner to satisfy a * cancel(true)). Transitions from these intermediate to final * states use cheaper ordered/lazy writes because values are unique * and cannot be further modified. * * Possible state transitions: * NEW -> COMPLETING -> NORMAL * NEW -> COMPLETING -> EXCEPTIONAL * NEW -> CANCELLED * NEW -> INTERRUPTING -> INTERRUPTED */private volatile int state;  
private static final int NEW          = 0;  
private static final int COMPLETING   = 1;  
private static final int NORMAL       = 2;  
private static final int EXCEPTIONAL  = 3;  
private static final int CANCELLED    = 4;  
private static final int INTERRUPTING = 5;  
private static final int INTERRUPTED  = 6;  
  
/** The underlying callable; nulled out after running */  
private Callable<V> callable;  
/** The result to return or exception to throw from get() */  
private Object outcome; // non-volatile, protected by state reads/writes  
/** The thread running the callable; CASed during run() */  
private volatile Thread runner;  
/** Treiber stack of waiting threads */  
private volatile WaitNode waiters;
```
其中，state 字段是 volatile 类型的，也就是说一个线程改变了这个变量，其他所有线程都能看见
各个状态的转换关系如下：![[Pasted image 20240120215717.png]]
outcome 字段用来保存执行结果
callable 字段用来保存传入的任务
runner 字段表示用来执行任务的线程

#### 构造函数
##### FutureTask(Callable<> callable)
```java
public FutureTask(Callable<V> callable) {  
    if (callable == null)  
        throw new NullPointerException();  
    this.callable = callable;  
    this.state = NEW;       // ensure visibility of callable  
}
```
这个构造函数会把传入的 Callable 变量保存在 this.callable 字段中，该字段定义为 `Callable<V> callable`，用来保存底层的调用，在被执行完成以后会指向 null

##### FutureTask(Runnable runnable, V result)
```java
public FutureTask(Runnable runnable, V result) {  
    this.callable = Executors.callable(runnable, result);  
    this.state = NEW;       // ensure visibility of callable  
}
```
这个构造函数会把传入的Runnable封装成一个Callable对象保存在callable字段中，同时如果任务执行成功的话就会返回传入的result。这种情况下如果不需要返回值的话可以传入一个null

Executors.callable() 用来把 Runnable 转换为 callable，代码如下：
```java
public static <T> Callable<T> callable(Runnable task, T result) {  
    if (task == null)  
        throw new NullPointerException();  
    return new RunnableAdapter<T>(task, result);  
}
```
采用的是适配器模式，调用 `RunnableAdapter<T>(task,result)` 方法来适配，代码如下：
```java
private static final class RunnableAdapter<T> implements Callable<T> {  
    private final Runnable task;  
    private final T result;  
    RunnableAdapter(Runnable task, T result) {  
        this.task = task;  
        this.result = result;  
    }  
    public T call() {  
        task.run();  
        return result;  
    }  
    public String toString() {  
        return super.toString() + "[Wrapped task = " + task + "]";  
    }  
}
```
这个适配器很简单，就是简单的实现了Callable接口，在call()实现中调用Runnable.run()方法，然后把传入的result作为任务的结果返回

在创建一个 FutureTask 对象之后，接下来就是在另一个线程中执行这个 Task，无论是通过直接创建一个Thread 还是通过线程池，执行的都是 run() 方法

#### run() 方法
run() 的代码如下：
```java
public void run() {
    //新建任务，CAS替换runner为当前线程
    if (state != NEW ||  
        !RUNNER.compareAndSet(this, null, Thread.currentThread()))  
        return;  
    try {  
        Callable<V> c = callable;  
        if (c != null && state == NEW) {  
            V result;  
            boolean ran;  
            try {  
                result = c.call();  
                ran = true;  
            } catch (Throwable ex) {  
                result = null;  
                ran = false;  
                setException(ex);  
            }  
            if (ran)  
                set(result);  
        }  
    } finally {  
        // runner must be non-null until state is settled to  
        // prevent concurrent calls to run()        runner = null;  
        // state must be re-read after nulling runner to prevent  
        // leaked interrupts        int s = state;  
        if (s >= INTERRUPTING)  
            handlePossibleCancellationInterrupt(s);  
    }  
}
```
逻辑步骤：
 运行任务，如果任务状态为NEW状态，则利用 CAS 修改为当前线程。执行完毕调用 set(result) 方法设置执行结果。代码如下：
```java
protected void set(V v) {  
    if (STATE.compareAndSet(this, NEW, COMPLETING)) {  
        outcome = v;  
        STATE.setRelease(this, NORMAL); // final state  
        finishCompletion();  
    }  
}
```
用 CAS 修改 state 状态为 COMPLETING，设置返回结果，然后使用 VarHandle.setRelease()的方式设置state状态为 NORMAL，最后调用 finishCompletion() 方法唤醒等待线程，代码如下：
```java
private void finishCompletion() {  
    // assert state > COMPLETING;  
    for (WaitNode q; (q = waiters) != null;) {  
        if (WAITERS.weakCompareAndSet(this, q, null)) {  
            for (;;) {  
                Thread t = q.thread;  
                if (t != null) {  
                    q.thread = null;  
                    LockSupport.unpark(t);  
                }  
                WaitNode next = q.next;  
                if (next == null)  
                    break;  
                q.next = null; // unlink to help gc  
                q = next;  
            }  
            break;  
        }  
    }  
  
    done();  
  
    callable = null;        // to reduce footprint  
}
```
#### get() 方法
get() 的代码如下：
```java
public V get() throws InterruptedException, ExecutionException {  
    int s = state;  
    if (s <= COMPLETING)  
        s = awaitDone(false, 0L);  
    return report(s);  
}
```
FutureTask 通过 get() 方法获取任务执行结果，如果任务处于未完成的状态，就调用 awaitDone() 方法等待任务完成。任务完成后，通过 report() 方法获取执行结果或抛出执行期间的异常
report() 方法的代码如下：
```java
private V report(int s) throws ExecutionException {  
    Object x = outcome;  
    if (s == NORMAL)  
        return (V)x;  
    if (s >= CANCELLED)  
        throw new CancellationException();  
    throw new ExecutionException((Throwable)x);  
}
```

### 使用方法
- Future + ExecutorService
- FutureTask + ExecutorService
- FutureTask + Thread
#### Future + ExecutorService
```java
          ExecutorService executorService = Executors.newCachedThreadPool();
          Future future = executorService.submit(new Callable<Object>() {
              @Override
              public Object call() throws Exception {
                  Long start = System.currentTimeMillis();
                  while (true) {
                      Long current = System.currentTimeMillis();
                     if ((current - start) > 1000) {
                         return 1;
                     }
                 }
             }
         });
  
         try {
             Integer result = (Integer)future.get();
             System.out.println(result);
         }catch (Exception e){
             e.printStackTrace();
         }
```

#### Future + ExecutorService
```java
        // 创建 ExecutorService
        ExecutorService executorService = Executors.newSingleThreadExecutor();

        // 创建 FutureTask
        FutureTask<String> futureTask = new FutureTask<>(() -> {
            // 模拟耗时任务
            Thread.sleep(2000);
            return "Task completed";
        });

        // 提交 FutureTask 到 ExecutorService
        executorService.submit(futureTask);

        System.out.println("Main thread is doing something else while waiting for the task to complete.");

        try {
            // 阻塞等待任务完成，并获取结果
            String result = futureTask.get();

            System.out.println("Task result: " + result);
        } catch (InterruptedException | ExecutionException e) {
            e.printStackTrace();
        } finally {
            // 关闭 ExecutorService
            executorService.shutdown();
        }
```

#### FutureTask + Thread
```java
        // 创建 FutureTask
        FutureTask<String> futureTask = new FutureTask<>(() -> {
            // 模拟耗时任务
            Thread.sleep(2000);
            return "Task completed";
        });

        // 创建 Thread 并启动
        Thread thread = new Thread(futureTask);
        thread.start();

        System.out.println("Main thread is doing something else while waiting for the task to complete.");

        try {
            // 阻塞等待任务完成，并获取结果
            String result = futureTask.get();

            System.out.println("Task result: " + result);
        } catch (InterruptedException | ExecutionException e) {
            e.printStackTrace();
        }
```