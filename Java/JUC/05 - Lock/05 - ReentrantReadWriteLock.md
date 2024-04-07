### 简介
读/写锁的实现更为复杂一点，当一个应用程序对于资源的读取远远大于写入，那么在并发情况下，多个线程读取资源会不会造成问题，但是如果一个线程尝试写入资源，那么同一时间内不能有其他读取或写入操作正在进行
为了解决这个允许多个 reader 但只允许一个  writer 的问题，可以使用 读/写锁
### 读写锁实现
先总结一下获取资源读写权限的条件：
- 读权限：没有线程正在写入并且没有线程有写请求
- 写权限：没有线程正在读取或写入

如果线程想要读取资源，只要没有线程写入该资源并且没有线程写请求。通过提高写访问请求的优先级，我们假设写请求比读请求更重要。
如果读取是最常发生的事情，而我们没有提高写入的优先级，则可能会发生饥饿，因为当新线程不断地获取读权限时，需要写权限的线程会无限阻塞，导致饥饿

当没有线程正在读或写资源时，可以授予想要对资源进行写访问的线程。有多少线程有写请求或以什么顺序请求写并不重要，除非想保证请求写访问的线程之间的公平性

代码如下：
```java
public class ReadWriteLock{  
  
    private int readers       = 0;  
    private int writers       = 0;  
    private int writeRequests = 0;  
  
    public synchronized void lockRead() throws InterruptedException{  
        while(writers > 0 || writeRequests > 0){  
            wait();  
        }  
        readers++;  
    }  
  
    public synchronized void unlockRead(){  
        readers--;  
        notifyAll();  
    }  
  
    public synchronized void lockWrite() throws InterruptedException{  
        writeRequests++;  
  
        while(readers > 0 || writers > 0){ 
            wait();  
        }  
        writeRequests--;  
        writers++;  
    }  
  
    public synchronized void unlockWrite() throws InterruptedException{  
        writers--;  
        notifyAll();  
    }  
}
```
读写锁有两个 lock 方法和两个 unlock 方法，分别为 读权限 和 写权限

读访问的规则在 lockRead() 方法中实现。所以线程都可以获得读权限，除非有一个线程具有写权限，或者一个或多个线程有写请求

写访问的规则在 lockWrite() 方法中实现，想要获得写权限，首先得进行写请求，然后检查是否可以获得写权限，如果没有线程正在读或写，则该线程可以获得写权限，有多少写请求并不重要

可以看到，unlockRead() 和 unlockWrite() 方法都是调用了 notifyAll() 而不是 notify()，原因如下：
- ReadWriteLock 内部有等待写权限的线程和等待读权限的线程，如果用 notify() 唤醒的是等待读权限的，那么它会被重新置于等待状态，因为还有其他写请求的线程。所以使用 notifyAll() 方法来唤醒所有等待的线程并检查是否可以获得所需的权限
- 当等待的线程全都是等待读读权限时，那么使用 notifyAll() 就可以让等待的线程一次性获取读权限，而不是一个个获取
### 可重入读写锁
上面实现的 ReadWriteLock 是不可重入的，如果一个获得写权限的线程再次发起写请求时，这会把自己给阻塞
此外，还有以下情况：
1. 线程 1 获得了 读权限
2. 线程 2 请求写，但是因为有线程在读所以被阻塞
3. 线程 1 再次请求 读权限，但是也被阻塞，因为此时有了 线程 2 的写请求
这种情况就导致了 ReadWriteLock 被锁住了，无法释放，类似于死锁

所以，ReadWriteLock 需要进行一些修改来处理可重入的情况，读 和 写 的可重入情况会分别处理
### 可重入读
可重入读的规则：
- 如果线程可以获得读权限（没有线程在写入或请求写），或者它已经具有读权限（无论有无写请求），则线程可以进行可重入读

为了检查线程是否具有读权限，会将线程的引用以及获取读权限的次数保存在 Map 中，在线程获取读权限前会先检查这个 Map 中的关系
lockRead() 和 unlockRead() 修改后如下：
```java
public class ReadWriteLock{

  private Map<Thread, Integer> readingThreads =
      new HashMap<Thread, Integer>();

  private int writers        = 0;
  private int writeRequests  = 0;

  public synchronized void lockRead() throws InterruptedException{
    Thread callingThread = Thread.currentThread();
    while(! canGrantReadAccess(callingThread)){
      wait();                                                                   
    }

    readingThreads.put(callingThread,
       (getAccessCount(callingThread) + 1));
  }


  public synchronized void unlockRead(){
    Thread callingThread = Thread.currentThread();
    int accessCount = getAccessCount(callingThread);
    if(accessCount == 1){ readingThreads.remove(callingThread); }
    else { readingThreads.put(callingThread, (accessCount -1)); }
    notifyAll();
  }


  private boolean canGrantReadAccess(Thread callingThread){
    if(writers > 0)            return false;
    if(isReader(callingThread) return true;
    if(writeRequests > 0)      return false;
    return true;
  }

  private int getReadAccessCount(Thread callingThread){
    Integer accessCount = readingThreads.get(callingThread);
    if(accessCount == null) return 0;
    return accessCount.intValue();
  }

  private boolean isReader(Thread callingThread){
    return readingThreads.get(callingThread) != null;
  }

}
```
可以看到在 canGrantReadAccess() 方法里，现在只关注有无线程正在写入，如果线程已经获得过读权限的话则可以无视全部的写请求

### 可重入写
可重写只允许已经拥有写权限的线程获取
lockWrite() 和 unlockWrite() 的修改如下：
```java
public class ReadWriteLock{

    private Map<Thread, Integer> readingThreads =
        new HashMap<Thread, Integer>();

    private int writeAccesses    = 0;
    private int writeRequests    = 0;
    private Thread writingThread = null;

  public synchronized void lockWrite() throws InterruptedException{
    writeRequests++;
    Thread callingThread = Thread.currentThread();
    while(! canGrantWriteAccess(callingThread)){
      wait();
    }
    writeRequests--;
    writeAccesses++;
    writingThread = callingThread;
  }

  public synchronized void unlockWrite() throws InterruptedException{
    writeAccesses--;
    if(writeAccesses == 0){
      writingThread = null;
    }
    notifyAll();
  }

  private boolean canGrantWriteAccess(Thread callingThread){
    if(hasReaders())             return false;
    if(writingThread == null)    return true;
    if(!isWriter(callingThread)) return false;
    return true;
  }

  private boolean hasReaders(){
    return readingThreads.size() > 0;
  }

  private boolean isWriter(Thread callingThread){
    return writingThread == callingThread;
  }
}
```

### 可重入转换

#### 读 到 写
有些时候一些拥有读权限的线程需要获取写权限。为此需要这个线程必须为 唯一的 reader
writeLock() 方法需要修改一下：
```java
public class ReadWriteLock{

    private Map<Thread, Integer> readingThreads =
        new HashMap<Thread, Integer>();

    private int writeAccesses    = 0;
    private int writeRequests    = 0;
    private Thread writingThread = null;

  public synchronized void lockWrite() throws InterruptedException{
    writeRequests++;
    Thread callingThread = Thread.currentThread();
    while(! canGrantWriteAccess(callingThread)){
      wait();
    }
    writeRequests--;
    writeAccesses++;
    writingThread = callingThread;
  }

  public synchronized void unlockWrite() throws InterruptedException{
    writeAccesses--;
    if(writeAccesses == 0){
      writingThread = null;
    }
    notifyAll();
  }

  private boolean canGrantWriteAccess(Thread callingThread){
    if(isOnlyReader(callingThread))    return true;
    if(hasReaders())                   return false;
    if(writingThread == null)          return true;
    if(!isWriter(callingThread))       return false;
    return true;
  }

  private boolean hasReaders(){
    return readingThreads.size() > 0;
  }

  private boolean isWriter(Thread callingThread){
    return writingThread == callingThread;
  }

  private boolean isOnlyReader(Thread thread){
      return readingThreads.size() == 1 &&
             readingThreads.get(callingThread) != null;
      }
  
}
```
新增了 inOnlyReader() 的判断
#### 写 到 读
当拥有写权限的线程想要获得读权限时，应该是可以立即获取的，因为拥有写权限的线程，说明没有其他线程正在写或读，所以直接获取读权限也没什么问题
canGrantReadAccess() 方法的修改如下：
```java
public class ReadWriteLock{

    private boolean canGrantReadAccess(Thread callingThread){
      if(isWriter(callingThread)) return true;
      if(writingThread != null)   return false;
      if(isReader(callingThread)  return true;
      if(writeRequests > 0)       return false;
      return true;
    }

}
```
增加一个判断 isWriter() ，判读当前线程是否拥有 写权限 即可

### 完整的可重入读写锁实现
至此，一个完整的可重入读写锁的实现如下：
```java
public class ReadWriteLock{

  private Map<Thread, Integer> readingThreads =
       new HashMap<Thread, Integer>();

   private int writeAccesses    = 0;
   private int writeRequests    = 0;
   private Thread writingThread = null;


  public synchronized void lockRead() throws InterruptedException{
    Thread callingThread = Thread.currentThread();
    while(! canGrantReadAccess(callingThread)){
      wait();
    }

    readingThreads.put(callingThread,
     (getReadAccessCount(callingThread) + 1));
  }

  private boolean canGrantReadAccess(Thread callingThread){
    if( isWriter(callingThread) ) return true;
    if( hasWriter()             ) return false;
    if( isReader(callingThread) ) return true;
    if( hasWriteRequests()      ) return false;
    return true;
  }


  public synchronized void unlockRead(){
    Thread callingThread = Thread.currentThread();
    if(!isReader(callingThread)){
      throw new IllegalMonitorStateException("Calling Thread does not" +
        " hold a read lock on this ReadWriteLock");
    }
    int accessCount = getReadAccessCount(callingThread);
    if(accessCount == 1){ readingThreads.remove(callingThread); }
    else { readingThreads.put(callingThread, (accessCount -1)); }
    notifyAll();
  }

  public synchronized void lockWrite() throws InterruptedException{
    writeRequests++;
    Thread callingThread = Thread.currentThread();
    while(! canGrantWriteAccess(callingThread)){
      wait();
    }
    writeRequests--;
    writeAccesses++;
    writingThread = callingThread;
  }

  public synchronized void unlockWrite() throws InterruptedException{
    if(!isWriter(Thread.currentThread()){
      throw new IllegalMonitorStateException("Calling Thread does not" +
        " hold the write lock on this ReadWriteLock");
    }
    writeAccesses--;
    if(writeAccesses == 0){
      writingThread = null;
    }
    notifyAll();
  }

  private boolean canGrantWriteAccess(Thread callingThread){
    if(isOnlyReader(callingThread))    return true;
    if(hasReaders())                   return false;
    if(writingThread == null)          return true;
    if(!isWriter(callingThread))       return false;
    return true;
  }


  private int getReadAccessCount(Thread callingThread){
    Integer accessCount = readingThreads.get(callingThread);
    if(accessCount == null) return 0;
    return accessCount.intValue();
  }


  private boolean hasReaders(){
    return readingThreads.size() > 0;
  }

  private boolean isReader(Thread callingThread){
    return readingThreads.get(callingThread) != null;
  }

  private boolean isOnlyReader(Thread callingThread){
    return readingThreads.size() == 1 &&
           readingThreads.get(callingThread) != null;
  }

  private boolean hasWriter(){
    return writingThread != null;
  }

  private boolean isWriter(Thread callingThread){
    return writingThread == callingThread;
  }

  private boolean hasWriteRequests(){
      return this.writeRequests > 0;
  }

}
```



### Java 中的 ReentrantReadWriteLock
ReentrantReadWriteLock 内部维护了一个 ReadLock 和一个 WriteLock，它们依赖 Sync 实现具体的功能，而 Sync 继承自 AQS，并且也提供了公平和非公平的实现
#### state 的定义
AQS 中只维护了一个 state 状态，而 ReentrantReadWriteLock 则需要维护读状态和写状态
所以，在读写锁中，state 的定义如下：
- state 的高16位表示读状态，表示获取读锁的次数![[Pasted image 20231226160649.png]]
- state 的低16位表示写状态，表示获取写锁线程的可重入次数![[Pasted image 20231226160657.png]]
#### WriteLock 的实现
所谓的实现就是通过 Sync 类来实现的，Sync 继承自 AQS ，通过重写相关方法来对 state 进行自定义的判断来实现功能，说白了就是根据 state 的含义来进行相关加锁的操作，就跟上面讲到的规则一样
##### tryAcquire()
写锁的规则跟上面的类似（好像没有可重入转换的逻辑），如下所示：
- 没有其他线程正在读或写，可以获取读权限
- 如果有线程正在写，并且写的线程是自己，可以获得读权限
逻辑思路也很简单，先检查可重入及规则，再检查公平机制，最后获取
![[Pasted image 20231227184726.png]]
代码如下：
```java
protected final boolean tryAcquire(int acquires) {  
    /*  
     * Walkthrough:     * 1. If read count nonzero or write count nonzero     *    and owner is a different thread, fail.     * 2. If count would saturate, fail. (This can only     *    happen if count is already nonzero.)     * 3. Otherwise, this thread is eligible for lock if     *    it is either a reentrant acquire or     *    queue policy allows it. If so, update state     *    and set owner.     */    Thread current = Thread.currentThread();  
    int c = getState();  
    int w = exclusiveCount(c);  
    if (c != 0) {  
        // (Note: if c != 0 and w == 0 then shared count != 0)  
        if (w == 0 || current != getExclusiveOwnerThread())  
            return false;  
        if (w + exclusiveCount(acquires) > MAX_COUNT)  
            throw new Error("Maximum lock count exceeded");  
        // Reentrant acquire  
        setState(c + acquires);  
        return true;  
    }  
    if (writerShouldBlock() ||  
        !compareAndSetState(c, c + acquires))  
        return false;  
    setExclusiveOwnerThread(current);  
    return true;  
}
```
当 c != 0 时，说明有其他线程获正在读或写，那么就进行可重入的判断，跟上面一样；如果还是原来的线程就直接获取读权限

当 c = 0 时，即没有线程正在读或写，那么就可以进行读权限的获取
这里增加了 writerShouldBlock() 方法的判断，用于公平锁和非公平锁的实现
公平锁会判断 AQS 同步队列前面还没有线程等待，如果有返回 true 直接阻塞，不获取锁；如果没有就返回 flase 直接获取，实现如下：
```java
final boolean writerShouldBlock() {  
    return hasQueuedPredecessors();  
}
```
非公平锁就直接返回 false ，直接抢占式获取锁，实现如下：
```java
final boolean writerShouldBlock() {  
    return false; // writers can always barge  
}
```
最后调用 setExclusiveOwnerThread(current) 方法，设置一下该读权限的当前线程
##### lock()
lock() 方法内部就是直接调用了 Sync 的 acquire (1)，代码如下：
```java
public void lock() {  
    sync.acquire(1);  
}
```
Sync 的 acquire() 方法其实就是 AQS 的 acquire() 方法，分别调用子类实现的 tryAcquire() 和 AQS 的 acquire() 代码如下：
```java
public final void acquire(int arg) {  
    if (!tryAcquire(arg))  
        acquire(null, arg, false, false, false, 0L);  
}
```
tryAcquire() 失败之后就进入到 acquire() 里不断重试获取
##### tryLock()
这个方法就是调用 Sync 的 tryWriteLock() 方法，实现逻辑跟 tryAcquire() 方法类似，区别就在于没有 writerShouldBlock() 方法，获取锁失败不会阻塞
代码如下：
```java
final boolean tryWriteLock() {  
    Thread current = Thread.currentThread();  
    int c = getState();  
    if (c != 0) {  
        int w = exclusiveCount(c);  
        if (w == 0 || current != getExclusiveOwnerThread())  
            return false;  
        if (w == MAX_COUNT)  
            throw new Error("Maximum lock count exceeded");  
    }  
    if (!compareAndSetState(c, c + 1))  
        return false;  
    setExclusiveOwnerThread(current);  
    return true;  
}
```

##### tryRelease()
tryRelease() 的实现逻辑其实就是两步：
- 判断释放锁的是不是原来读权限的线程
- 修改 state 值
代码如下：
```java
protected final boolean tryRelease(int releases) {  
    if (!isHeldExclusively())  
        throw new IllegalMonitorStateException();  
    int nextc = getState() - releases;  
    boolean free = exclusiveCount(nextc) == 0;  
    if (free)  
        setExclusiveOwnerThread(null);  
    setState(nextc);  
    return free;  
}
```
##### unlock()
调用了Sync 的 release() 方法
```java
public void unlock() {  
    sync.release(1);  
}
```

里面的逻辑就是 调用 tryRelease() 和 signalNext 方法来唤醒等待队列的第一个 Node
```java
public final boolean release(int arg) {  
    if (tryRelease(arg)) {  
        signalNext(head);  
        return true;  
    }  
    return false;  
}
```
#### ReadLock 的实现

##### tryAcquireShared()
获取读权限的规则也跟上面的类似：
- 没有正在写或写请求的线程时，可以获取
- 如果正在写的线程是当前要获取读权限的线程，可以获取（写 -> 读 可重入转换）

代码如下：
```java
protected final int tryAcquireShared(int unused) {  
    /*  
     * Walkthrough:     * 1. If write lock held by another thread, fail.     * 2. Otherwise, this thread is eligible for     *    lock wrt state, so ask if it should block     *    because of queue policy. If not, try     *    to grant by CASing state and updating count.     *    Note that step does not check for reentrant     *    acquires, which is postponed to full version     *    to avoid having to check hold count in     *    the more typical non-reentrant case.     * 3. If step 2 fails either because thread     *    apparently not eligible or CAS fails or count     *    saturated, chain to version with full retry loop.     */    Thread current = Thread.currentThread();  
    int c = getState();  
    if (exclusiveCount(c) != 0 &&  
        getExclusiveOwnerThread() != current)  
        return -1;  
    int r = sharedCount(c);  
    if (!readerShouldBlock() &&  
        r < MAX_COUNT &&  
        compareAndSetState(c, c + SHARED_UNIT)) {  
        if (r == 0) {  
            firstReader = current;  
            firstReaderHoldCount = 1;  
        } else if (firstReader == current) {  
            firstReaderHoldCount++;  
        } else {  
            HoldCounter rh = cachedHoldCounter;  
            if (rh == null ||  
                rh.tid != LockSupport.getThreadId(current))  
                cachedHoldCounter = rh = readHolds.get();  
            else if (rh.count == 0)  
                readHolds.set(rh);  
            rh.count++;  
        }  
        return 1;  
    }  
    return fullTryAcquireShared(current);  
}
```
这里有个 readerShouldBlock() 方法，也是分为 公平锁 和 非公平锁 的情况

公平锁的实现跟 WriteLock 的一致，先检查同步队列前面有无其他节点

非公平锁的情况，就是判断是否有写请求的线程（即 AQS 中等待队列的第一个节点是否为 exclusive ），如果有的话，那就获取失败，只能自旋重试了
代码如下：
```java
final boolean readerShouldBlock() {  
    /* As a heuristic to avoid indefinite writer starvation,  
     * block if the thread that momentarily appears to be head     * of queue, if one exists, is a waiting writer.  This is     * only a probabilistic effect since a new reader will not     * block if there is a waiting writer behind other enabled     * readers that have not yet drained from the queue.     */    return apparentlyFirstQueuedIsExclusive();  
}

final boolean apparentlyFirstQueuedIsExclusive() {  
    Node h, s;  
    return (h = head) != null && (s = h.next)  != null &&  
```
因为多个读线程同时获取时只能成功一个，所以剩下的通过调用 fullTryAcquireShared(current) 方法自旋继续获取，逻辑跟 tryAcquireShared 一样
代码如下：
```java
/**  
 * Full version of acquire for reads, that handles CAS misses * and reentrant reads not dealt with in tryAcquireShared. */final int fullTryAcquireShared(Thread current) {  
    /*  
     * This code is in part redundant with that in     * tryAcquireShared but is simpler overall by not     * complicating tryAcquireShared with interactions between     * retries and lazily reading hold counts.     */    HoldCounter rh = null;  
    for (;;) {  
        int c = getState();  
        if (exclusiveCount(c) != 0) {  
            if (getExclusiveOwnerThread() != current)  
                return -1;  
            // else we hold the exclusive lock; blocking here  
            // would cause deadlock.        } else if (readerShouldBlock()) {  
            // Make sure we're not acquiring read lock reentrantly  
            if (firstReader == current) {  
                // assert firstReaderHoldCount > 0;  
            } else {  
                if (rh == null) {  
                    rh = cachedHoldCounter;  
                    if (rh == null ||  
                        rh.tid != LockSupport.getThreadId(current)) {  
                        rh = readHolds.get();  
                        if (rh.count == 0)  
                            readHolds.remove();  
                    }  
                }  
                if (rh.count == 0)  
                    return -1;  
            }  
        }  
        if (sharedCount(c) == MAX_COUNT)  
            throw new Error("Maximum lock count exceeded");  
        if (compareAndSetState(c, c + SHARED_UNIT)) {  
            if (sharedCount(c) == 0) {  
                firstReader = current;  
                firstReaderHoldCount = 1;  
            } else if (firstReader == current) {  
                firstReaderHoldCount++;  
            } else {  
                if (rh == null)  
                    rh = cachedHoldCounter;  
                if (rh == null ||  
                    rh.tid != LockSupport.getThreadId(current))  
                    rh = readHolds.get();  
                else if (rh.count == 0)  
                    readHolds.set(rh);  
                rh.count++;  
                cachedHoldCounter = rh; // cache for release  
            }  
            return 1;  
        }  
    }  
}
```
