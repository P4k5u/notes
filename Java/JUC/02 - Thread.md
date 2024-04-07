### 简介

### 创建线程的方式
#### 继承 Thread 类
同样也是需要实现 run() 方法，因为 Thread 类也实现了 Runable 接口
```java
public class MyThread extends Thread {
    public void run() {
        // ...
    }
}
```

当调用 start() 方法启动一个线程时，虚拟机会将该线程放入就绪队列中等待被调度，当一个线程被调度时会执行该线程的 run() 方法
```java
public static void main(String[] args) {
    MyThread mt = new MyThread();
    mt.start();
}
```


#### 实现 Runnable 接口
需要实现 run() 方法
```java
public class MyRunnable implements Runnable {
    public void run() {
        // ...
    }
}
```
通过 Thread 调用 start() 方法来启动线程
```java
public static void main(String[] args) {
    MyRunnable instance = new MyRunnable();
    Thread thread = new Thread(instance);
    thread.start();
}
```

#### 实现 Callable 接口
与 Runnable 相比，Callable 可以有返回值，返回值通过 FutureTask 进行封装
```java
public class MyCallable implements Callable<Integer> {
    public Integer call() {
        return 123;
    }
}
```
同样也是 Thread 调用 start() 方法来启动
```java
public static void main(String[] args) throws ExecutionException, InterruptedException {
    MyCallable mc = new MyCallable();
    FutureTask<Integer> ft = new FutureTask<>(mc);
    Thread thread = new Thread(ft);
    thread.start();
    System.out.println(ft.get());
}
```

### 线程通知与等待

### 等待线程执行完成

### 线程睡眠

### 线程CPU让权

### 线程中断