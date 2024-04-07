### 简介
线程池是一个池化的一些线程，可以重复利用来执行一些任务，所以每个线程可以执行更多的任务。
线程池解决了每个任务执行都需要创建一个线程的问题
此外，使用线程池可以更好的管理正在运行的线程数，可以更加精确的控制内存的消耗

### 线程池的工作方式
任务可以传递到线程池中，而不是为每个任务都开启一个线程。只要池中有空闲的线程就会分配一个任务来执行，任务被放到一个 阻塞队列 中，空闲的线程通过 阻塞队列 来获取任务。但是在获取任务的时候，其他的空闲线程也会被阻塞![[Pasted image 20231228211115.png]]

### Java 线程池的实现
下面是一个简单的线程池实现：
```java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;

public class ThreadPool {

    private BlockingQueue taskQueue = null;
    private List<PoolThreadRunnable> runnables = new ArrayList<>();
    private boolean isStopped = false;

    public ThreadPool(int noOfThreads, int maxNoOfTasks){
        taskQueue = new ArrayBlockingQueue(maxNoOfTasks);

        for(int i=0; i<noOfThreads; i++){
            PoolThreadRunnable poolThreadRunnable =
                    new PoolThreadRunnable(taskQueue);

            runnables.add(poolThreadRunnable);
        }
        for(PoolThreadRunnable runnable : runnables){
            new Thread(runnable).start();
        }
    }

    public synchronized void  execute(Runnable task) throws Exception{
        if(this.isStopped) throw
                new IllegalStateException("ThreadPool is stopped");

        this.taskQueue.offer(task);
    }

    public synchronized void stop(){
        this.isStopped = true;
        for(PoolThreadRunnable runnable : runnables){
            runnable.doStop();
        }
    }

    public synchronized void waitUntilAllTasksFinished() {
        while(this.taskQueue.size() > 0) {
            try {
                Thread.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

}
```
PoolThreadRunnable 是 Runnable 接口的实现：
```java
import java.util.concurrent.BlockingQueue;

public class PoolThreadRunnable implements Runnable {

    private Thread        thread    = null;
    private BlockingQueue taskQueue = null;
    private boolean       isStopped = false;

    public PoolThreadRunnable(BlockingQueue queue){
        taskQueue = queue;
    }

    public void run(){
        this.thread = Thread.currentThread();
        while(!isStopped()){
            try{
                Runnable runnable = (Runnable) taskQueue.take();
                runnable.run();
            } catch(Exception e){
                //log or otherwise report exception,
                //but keep pool thread alive.
            }
        }
    }

    public synchronized void doStop(){
        isStopped = true;
        //break pool thread out of dequeue() call.
        this.thread.interrupt();
    }

    public synchronized boolean isStopped(){
        return isStopped;
    }
}
```
最后是线程池的使用：
```java
public class ThreadPoolMain {

    public static void main(String[] args) throws Exception {

        ThreadPool threadPool = new ThreadPool(3, 10);

        for(int i=0; i<10; i++) {

            int taskNo = i;
            threadPool.execute(() -> {
                String message =
                        Thread.currentThread().getName()
                                + ": Task " + taskNo ;
                System.out.println(message);
            });
        }

        threadPool.waitUntilAllTasksFinished();
        threadPool.stop();

    }
}
```
线程池的实现由两部分组成，ThreadPool 是线程池的公共入口，PoolThread 类是线程执行任务的实现

当调用 ThreadPool.execute(Runnable r) 的时候，Runnable 会进入到 阻塞队列 中，等待出队

空闲的 PoolThread 会把 Runnable 出队并执行 PoolThread.run() 方法，执行完之后，会继续尝试从 阻塞队列 中获取任务