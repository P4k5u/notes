### 简介
AQS 全称 AbstractQueuedSynchronizer 抽象同步队列，它是实现同步器的基础组件，锁的底层就是用 AQS 实现的

AQS 的主要属性如下：
```java
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer {

    // Node status bits, also used as argument and return values
    static final int WAITING   = 1;          // must be 1
    static final int CANCELLED = 0x80000000; // must be negative
    static final int COND      = 2;          // in a condition wait

    /** CLH Nodes */
    abstract static class Node { }

    // Concrete classes tagged by type
    static final class ExclusiveNode extends Node { }
    static final class SharedNode extends Node { }

    static final class ConditionNode extends Node { }

    /**
     * Head of the wait queue, lazily initialized.
     */
    private transient volatile Node head;

    /**
     * Tail of the wait queue. After initialization, modified only via casTail.
     */
    private transient volatile Node tail;

    /**
     * The synchronization state.
     */
    private volatile int state;
}
```
主要是就是 state 值的操作和 node 的管理（等待队列）以及 条件队列
### state
AQS 中维护了一个单一的状态信息 state，可以通过 getState、setState、compareAndSetState 函数操作，不同实现的 state 含义也会不一样，比如：对于 ReentrantLock，state 可以表示获取锁的可重入次数；对于 ReentrantReadWriteLock，state 高16 位表示读权限次数，低16位表示写权限重入次数；对于 semaphore 来说，state 表示当前可用信号的个数
### state 的操作方式
state 的操作方式分为两种：
- 独占方式：acquire()、acquireInterruptibly()、release()
- 共享方式：acquireShared()、acquireSharedInterruptibly()、releaseShared()

独占方式获取的资源是与具体线程绑定的，如果一个线程获取到了资源，就会标记是这个线程获取到了，其他线程再尝试操作 state 获取资源时发现资源不是自己持有的，就会在获取失败后阻塞

共享方式的资源与具体线程是不相关的，当多个线程去请求资源时通过 CAS 方式竞争获取资源，当一个线程获取到资源后，另一个线程再去获取时，如果资源满足获取要求也能直接用 CAS 获取
#### acquire()
acquire(int arg) 方法是用来获取资源
```java
public final void acquire(int arg) {  
    if (!tryAcquire(arg))  
        acquire(null, arg, false, false, false, 0L);  
}
```
而下面多个参数的 acquire() 方法，则是管理 node 节点的关键方法
代码如下：
```java
/**  
 * Main acquire method, invoked by all exported acquire methods. * * @param node null unless a reacquiring Condition  
 * @param arg the acquire argument  
 * @param shared true if shared mode else exclusive  
 * @param interruptible if abort and return negative on interrupt  
 * @param timed if true use timed waits  
 * @param time if timed, the System.nanoTime value to timeout  
 * @return positive if acquired, 0 if timed out, negative if interrupted  
 */final int acquire(Node node, int arg, boolean shared,  
                  boolean interruptible, boolean timed, long time) {  
    Thread current = Thread.currentThread();  
    byte spins = 0, postSpins = 0;   // retries upon unpark of first thread  
    boolean interrupted = false, first = false;  
    Node pred = null;                // predecessor of node when enqueued  
  
    /*     * Repeatedly:     *  Check if node now first     *    if so, ensure head stable, else ensure valid predecessor     *  if node is first or not yet enqueued, try acquiring     *  else if node not yet created, create it     *  else if not yet enqueued, try once to enqueue     *  else if woken from park, retry (up to postSpins times)     *  else if WAITING status not set, set and retry     *  else park and clear WAITING status, and check cancellation     */  
    for (;;) {  
        if (!first && (pred = (node == null) ? null : node.prev) != null &&  
            !(first = (head == pred))) {  
            if (pred.status < 0) {  
                cleanQueue();           // predecessor cancelled  
                continue;  
            } else if (pred.prev == null) {  
                Thread.onSpinWait();    // ensure serialization  
                continue;  
            }  
        }  
        if (first || pred == null) {  
            boolean acquired;  
            try {  
                if (shared)  
                    acquired = (tryAcquireShared(arg) >= 0);  
                else  
                    acquired = tryAcquire(arg);  
            } catch (Throwable ex) {  
                cancelAcquire(node, interrupted, false);  
                throw ex;  
            }  
            if (acquired) {  
                if (first) {  
                    node.prev = null;  
                    head = node;  
                    pred.next = null;  
                    node.waiter = null;  
                    if (shared)  
                        signalNextIfShared(node);  
                    if (interrupted)  
                        current.interrupt();  
                }  
                return 1;  
            }  
        }  
        if (node == null) {                 // allocate; retry before enqueue  
            if (shared)  
                node = new SharedNode();  
            else  
                node = new ExclusiveNode();  
        } else if (pred == null) {          // try to enqueue  
            node.waiter = current;  
            Node t = tail;  
            node.setPrevRelaxed(t);         // avoid unnecessary fence  
            if (t == null)  
                tryInitializeHead();  
            else if (!casTail(t, node))  
                node.setPrevRelaxed(null);  // back out  
            else  
                t.next = node;  
        } else if (first && spins != 0) {  
            --spins;                        // reduce unfairness on rewaits  
            Thread.onSpinWait();  
        } else if (node.status == 0) {  
            node.status = WAITING;          // enable signal and recheck  
        } else {  
            long nanos;  
            spins = postSpins = (byte)((postSpins << 1) | 1);  
            if (!timed)  
                LockSupport.park(this);  
            else if ((nanos = time - System.nanoTime()) > 0L)  
                LockSupport.parkNanos(this, nanos);  
            else  
                break;  
            node.clearStatus();  
            if ((interrupted |= Thread.interrupted()) && interruptible)  
                break;  
        }  
    }  
    return cancelAcquire(node, interrupted, interruptible);  
}
```
当调用 tryAcquire() 获取锁失败后，就会调用 acquire() 方法
acuqire 会不断进行自旋，主要完成了三个功能：
- 将阻塞线程转换节点，并入队
- 调用 LockSupport.park() 方法阻塞线程
- 当阻塞线程被唤醒后继续尝试获取锁
#### release()
release() 方法主要用于唤醒线程
```java
public final boolean release(int arg) {  
    if (tryRelease(arg)) {  
        signalNext(head);  
        return true;  
    }  
    return false;  
}
```
其中的 signalNext(head) 通过调用 LockSupport.unpakr() 来唤醒队列中第一个节点的线程
代码如下：
```java
/**  
 * Wakes up the successor of given node, if one exists, and unsets its * WAITING status to avoid park race. This may fail to wake up an * eligible thread when one or more have been cancelled, but * cancelAcquire ensures liveness. */private static void signalNext(Node h) {  
    Node s;  
    if (h != null && (s = h.next) != null && s.status != 0) {  
        s.getAndUnsetStatus(WAITING);  
        LockSupport.unpark(s.waiter);  
    }  
}
```

#### acquireShared()
acquireShared() 方法用于操作获取共享资源，代码如下：
```java
public final void acquireShared(int arg) {  
    if (tryAcquireShared(arg) < 0)  
        acquire(null, arg, true, false, false, 0L);  
}
```
在调用 tryAcquireShared 失败后，也是调用 acquire() 方法来进行重试和入队操作，区别在于 shared 参数传入的为 true
#### releaseShared()
releaseShared() 方法用于释放共享资源，代码如下：
```java
public final boolean releaseShared(int arg) {  
    if (tryReleaseShared(arg)) {  
        signalNext(head);  
        return true;  
    }  
    return false;  
}
```


需要注意的是，tryAcquire() 、 tryRelease()、tryAcquiredShared() 和 tryReleaseShared() 方法需要具体的子类进行实现，子类在实现时根据具体场景来用 CAS 修改 state 状态值，成功返回 true，失败返回 flase
### Node
Node 时队列中的队列元素，结构如下：
```java
abstract static class Node {
        volatile Node prev;       // initially attached via casTail
        volatile Node next;       // visibly nonnull when signallable
        Thread waiter;            // visibly nonnull when enqueued
        volatile int status; 
}
```
可以看到，Node 是一个抽象类，所以有两个具体类型的类表示线程获取的是独占还是共享资源（就是线程请求获取的是啥资源，相当于是读请求或写请求）：
```java
// Concrete classes tagged by type  
static final class ExclusiveNode extends Node { }  
static final class SharedNode extends Node { }
```
Thread 类型的 waiter 就是存放被阻塞的线程，prev 、next 就是标记队列前后节点，status 就是表示当前节点的状态，状态属性如下：
```java
static final int WAITING   = 1;          // must be 1  
static final int CANCELLED = 0x80000000; // must be negative  
static final int COND      = 2;          // in a condition wait
```
还有个 ConditionNode 类型（用于条件队列），结构如下：
```java
    static final class ConditionNode extends Node
        implements ForkJoinPool.ManagedBlocker {
        ConditionNode nextWaiter;            // link to next waiting node

        /**
         * Allows Conditions to be used in ForkJoinPools without
         * risking fixed pool exhaustion. This is usable only for
         * untimed Condition waits, not timed versions.
         */
        public final boolean isReleasable() {
            return status <= 1 || Thread.currentThread().isInterrupted();
        }

        public final boolean block() {
            while (!isReleasable()) LockSupport.park();
            return true;
        }
    }
```
### ConditionObject
ConditionObject 是 AQS 的内部类，可以访问 AQS 内部的变量，每个 条件变量 内部都维护了一个条件队列，用来存放调用 await() 阻塞的线程
ConditionObject 结构如下（跟 AQS 类似的结构）：
```java
public class ConditionObject implements Condition, java.io.Serializable {
        private static final long serialVersionUID = 1173984872572414699L;
        /** First node of condition queue. */
        private transient ConditionNode firstWaiter;
        /** Last node of condition queue. */
        private transient ConditionNode lastWaiter;

        /**
         * Creates a new {@code ConditionObject} instance.
         */
        public ConditionObject() { }
```

#### signal()
调用 signal() 方法会把条件队列中的 condition node 移出到 同步队列 中
```java
        // Signalling methods

        /**
         * Removes and transfers one or all waiters to sync queue.
         */
        private void doSignal(ConditionNode first, boolean all) {
            while (first != null) {
                ConditionNode next = first.nextWaiter;
                if ((firstWaiter = next) == null)
                    lastWaiter = null;
                if ((first.getAndUnsetStatus(COND) & COND) != 0) {
                    enqueue(first);
                    if (!all)
                        break;
                }
                first = next;
            }
        }

        /**
         * Moves the longest-waiting thread, if one exists, from the
         * wait queue for this condition to the wait queue for the
         * owning lock.
         *
         * @throws IllegalMonitorStateException if {@link #isHeldExclusively}
         *         returns {@code false}
         */
        public final void signal() {
            ConditionNode first = firstWaiter;
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            if (first != null)
                doSignal(first, false);
        }
```
#### await()
调用 await() 方法会把阻塞的线程转换为 condition node 并放入到 条件队列 ，然后调用 LockSupport.setCurrentBlocker() 阻塞。
然后自旋查看 是否已经进入 同步队列 （有无调用 signal() 方法）并且可以重新进行锁的获取
代码如下：
```java
    /**  
 * Implements interruptible condition wait. * <ol>  
 * <li>If current thread is interrupted, throw InterruptedException.  
 * <li>Save lock state returned by {@link #getState}.  
 * <li>Invoke {@link #release} with saved state as argument,  
 *     throwing IllegalMonitorStateException if it fails. * <li>Block until signalled or interrupted.  
 * <li>Reacquire by invoking specialized version of  
 *     {@link #acquire} with saved state as argument.  
 * <li>If interrupted while blocked in step 4, throw InterruptedException.  
 * </ol>  
 */  
public final void await() throws InterruptedException {  
    if (Thread.interrupted())  
        throw new InterruptedException();  
    ConditionNode node = new ConditionNode();  
    int savedState = enableWait(node);  
    LockSupport.setCurrentBlocker(this); // for back-compatibility  
    boolean interrupted = false, cancelled = false, rejected = false;  
    while (!canReacquire(node)) {  
        if (interrupted |= Thread.interrupted()) {  
            if (cancelled = (node.getAndUnsetStatus(COND) & COND) != 0)  
                break;              // else interrupted after signal  
        } else if ((node.status & COND) != 0) {  
            try {  
                if (rejected)  
                    node.block();  
                else  
                    ForkJoinPool.managedBlock(node);  
            } catch (RejectedExecutionException ex) {  
                rejected = true;  
            } catch (InterruptedException ie) {  
                interrupted = true;  
            }  
        } else  
            Thread.onSpinWait();    // awoke while enqueuing  
    }  
    LockSupport.setCurrentBlocker(null);  
    node.clearStatus();  
    acquire(node, savedState, false, false, false, 0L);  
    if (interrupted) {  
        if (cancelled) {  
            unlinkCancelledWaiters(node);  
            throw new InterruptedException();  
        }  
        Thread.currentThread().interrupt();  
    }  
}
```
### AQS 队列
AQS 的队列存放的是被阻塞的线程，分为两种队列，一种是因为获取锁失败而阻塞的 同步队列 ，一种是因为条件变量获取失败而阻塞的 条件队列
#### 同步队列
当一个线程获取锁失败之后，该线程就会被转换为 Node 节点，然后进入同步队列
入队方式就跟正常队列的操作一样，是带有虚拟头节点的方式
#### 条件队列
Lock 可以创建多个条件变量，每个条件变量都有相应的条件队列
线程获取 条件变量 的前提是获取 条件变量对应的锁 

当调用了 ConditionObject 的 await() 方法时，线程会进入到 条件队列 里并阻塞
当别的线程调用了 signal() 方法时，会把条件队列里的线程唤醒并移动到 阻塞队列 里，继续等待获取资源