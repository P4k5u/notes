  ### 简介
ThreadPoolExecutor 实现自 ExecutorService 接口，有相关的 execute() 方法
实现原理跟 ThreadPool 类似，有一个 ThreadPool 的入口，一个阻塞队列 workQueue（task queue）用来存放任务，一个 workers 集合用来存放工作线程，新增加一个 RejectedExecutionHandler 用来处理被拒绝的任务![[Pasted image 20240102162019.png]]

控制池内线程数量的就两个参数：
- corePoolSize
- maximumPoolSize
线程池的工作机制也是围绕这两个参数来实现：
- 当少于 corePoolSize 时，就会创建的新的线程，即使在有空闲线程的情况下也会新建
- 当 Task Queue 满了，并且线程数量也达到或超过了 corePoolSize，就会继续创建新的线程，直到达到 maximumPoolSize
- 当达到 maximumPoolSize 的时候，就开始根据相关策略来拒绝新到来的任务

### 相关方法

#### 创建 ThreadPoolExecutor
代码如下：
```java
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), defaultHandler);
    }
```
参数就不用多说了，主要就是 corePoolSize 、maximumPoolSize 和 BlockingQueue

传入 ThreadFactory 的话可以对新建的线程进行自定义，比如线程名称之类的

Executors 工厂方法创建的线程池实际上就是用 ThreadPoolExecutor 创建的，一般有三种类型：
- SingleThreadExecutor
- FixedThreadPool
- CachedThreadPool
##### SingleThreadExecutor
```java
public static ExecutorService newSingleThreadExecutor() {  
    return new FinalizableDelegatedExecutorService  
        (new ThreadPoolExecutor(1, 1,  
                                0L, TimeUnit.MILLISECONDS,  
                                new LinkedBlockingQueue<Runnable>()));  
}
```
这个线程池永远只有一个线程在工作，所以能保证任务按顺序执行，相当于是一个同步的线程池
##### FixedThreadPool
```java
public static ExecutorService newFixedThreadPool(int nThreads) {  
    return new ThreadPoolExecutor(nThreads, nThreads,  
                                  0L, TimeUnit.MILLISECONDS,  
                                  new LinkedBlockingQueue<Runnable>());  
}
```
因为采用了无界队列 LinkedBlockingQueue 作为 task queue，所以在线程数量达到 corePoolSize 之后就不会再继续增加，但是由于是无界队列，所以任务会永远在队列里，不会拒绝
##### CachedThreadPool
```java
public static ExecutorService newCachedThreadPool() {  
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,  
                                  60L, TimeUnit.SECONDS,  
                                  new SynchronousQueue<Runnable>());  
}
```
这个会在线程的空闲时间超过 keepAliveTime 之后释放线程。当提交新任务时，又继续创建线程
这样会造成一定的系统开销
#### execute() 方法
代码如下：
```java
/**  
 * Executes the given task sometime in the future.  The task * may execute in a new thread or in an existing pooled thread. * * If the task cannot be submitted for execution, either because this * executor has been shutdown or because its capacity has been reached, * the task is handled by the current {@link RejectedExecutionHandler}.  
 * * @param command the task to execute  
 * @throws RejectedExecutionException at discretion of  
 *         {@code RejectedExecutionHandler}, if the task  
 *         cannot be accepted for execution * @throws NullPointerException if {@code command} is null  
 */public void execute(Runnable command) {  
    if (command == null)  
        throw new NullPointerException();  
    /*  
     * Proceed in 3 steps:     *     * 1. If fewer than corePoolSize threads are running, try to     * start a new thread with the given command as its first     * task.  The call to addWorker atomically checks runState and     * workerCount, and so prevents false alarms that would add     * threads when it shouldn't, by returning false.     *     * 2. If a task can be successfully queued, then we still need     * to double-check whether we should have added a thread     * (because existing ones died since last checking) or that     * the pool shut down since entry into this method. So we     * recheck state and if necessary roll back the enqueuing if     * stopped, or start a new thread if there are none.     *     * 3. If we cannot queue task, then we try to add a new     * thread.  If it fails, we know we are shut down or saturated     * and so reject the task.     */    int c = ctl.get();  
    if (workerCountOf(c) < corePoolSize) {  
        if (addWorker(command, true))  
            return;  
        c = ctl.get();  
    }  
    if (isRunning(c) && workQueue.offer(command)) {  
        int recheck = ctl.get();  
        if (! isRunning(recheck) && remove(command))  
            reject(command);  
        else if (workerCountOf(recheck) == 0)  
            addWorker(null, false);  
    }  
    else if (!addWorker(command, false))  
        reject(command);  
}
```
逻辑跟 ThreadPool 一样，没超过 corePoolSize  的就直接创建新 worker
超过 corePoolSize 之后就把任务放入 task queue，然后继续创建新的 worker
#### addWorker() 方法
代码如下：
```java
private boolean addWorker(Runnable firstTask, boolean core) {  
    retry:  
    for (int c = ctl.get();;) {  
        // Check if queue empty only if necessary.  
        if (runStateAtLeast(c, SHUTDOWN)  
            && (runStateAtLeast(c, STOP)  
                || firstTask != null  
                || workQueue.isEmpty()))  
            return false;  
  
        for (;;) {  
            if (workerCountOf(c)  
                >= ((core ? corePoolSize : maximumPoolSize) & COUNT_MASK))  
                return false;  
            if (compareAndIncrementWorkerCount(c))  
                break retry;  
            c = ctl.get();  // Re-read ctl  
            if (runStateAtLeast(c, SHUTDOWN))  
                continue retry;  
            // else CAS failed due to workerCount change; retry inner loop  
        }  
    }  
  
    boolean workerStarted = false;  
    boolean workerAdded = false;  
    Worker w = null;  
    try {  
        w = new Worker(firstTask);  
        final Thread t = w.thread;  
        if (t != null) {  
            final ReentrantLock mainLock = this.mainLock;  
            mainLock.lock();  
            try {  
                // Recheck while holding lock.  
                // Back out on ThreadFactory failure or if                // shut down before lock acquired.                int c = ctl.get();  
  
                if (isRunning(c) ||  
                    (runStateLessThan(c, STOP) && firstTask == null)) {  
                    if (t.getState() != Thread.State.NEW)  
                        throw new IllegalThreadStateException();  
                    workers.add(w);  
                    workerAdded = true;  
                    int s = workers.size();  
                    if (s > largestPoolSize)  
                        largestPoolSize = s;  
                }  
            } finally {  
                mainLock.unlock();  
            }  
            if (workerAdded) {  
                t.start();  
                workerStarted = true;  
            }  
        }  
    } finally {  
        if (! workerStarted)  
            addWorkerFailed(w);  
    }  
    return workerStarted;  
}
```
把 worker 放入 wokerSet 的时候要进行加锁，放入 workerSet 成功之后就开始运行 worker
#### runWorker() 方法
代码如下：
```java
final void runWorker(Worker w) {  
    Thread wt = Thread.currentThread();  
    Runnable task = w.firstTask;  
    w.firstTask = null;  
    w.unlock(); // allow interrupts  
    boolean completedAbruptly = true;  
    try {  
        while (task != null || (task = getTask()) != null) {  
            w.lock();  
            // If pool is stopping, ensure thread is interrupted;  
            // if not, ensure thread is not interrupted.  This            // requires a recheck in second case to deal with            // shutdownNow race while clearing interrupt            if ((runStateAtLeast(ctl.get(), STOP) ||  
                 (Thread.interrupted() &&  
                  runStateAtLeast(ctl.get(), STOP))) &&  
                !wt.isInterrupted())  
                wt.interrupt();  
            try {  
                beforeExecute(wt, task);  
                try {  
                    task.run();  
                    afterExecute(task, null);  
                } catch (Throwable ex) {  
                    afterExecute(task, ex);  
                    throw ex;  
                }  
            } finally {  
                task = null;  
                w.completedTasks++;  
                w.unlock();  
            }  
        }  
        completedAbruptly = false;  
    } finally {  
        processWorkerExit(w, completedAbruptly);  
    }  
}
```
在执行任务的时候，也要进行加锁，实现原理跟 ThreadPool 的一样，把 task 放入到 worker 中，然后 start worker 的线程，run 方法里就会去执行 task 的 run 的方法，相当于给 task 套了一层 worker 线程