### Atomic
JDK 中提供了 12 个原子操作类

#### 原子更新基本类型
- AtomicBoolean
- AtomicInteger
- AtomicLong

#### 原子更新数组
- AtomicIntegerArray
- AtomicLongArray
- AtomicReferenceArray

#### 原子更新引用类型
- AtomicReference
- AtomicStampedReference
- AtomicMarkableReference

#### 原子更新字段
- AtomicIntegerFieldUpdater
- AtomicLongFieldUpdater
- AtomicReferenceFieldUpdater

### CAS
AS的全称为 Compare-And-Swap，直译就是对比交换。这是一条CPU的原子指令，其作用是让CPU先进行比较两个值是否相等，然后原子地更新某个位置的值，其实现方式是基于硬件平台的汇编指令，就是说CAS是靠硬件实现的，JVM只是封装了汇编调用，比如 AtomicInteger 类便是使用了这些封装后的接口。  

简单来说，CAS操作需要输入两个数值，一个旧值(操作前原本的值)和一个新值，操作期间先比较下旧值有没有发生变化，如果没有发生变化，才交换成新值，发生了变化则不交换

CAS操作是原子性的，所以多线程并发使用CAS更新数据时，可以不使用锁。JDK中大量使用了CAS来更新数据而防止加锁（synchronized 重量级锁）来保持原子更新

其实就跟 sql 语句的条件更新一样，`update set id=3 from table where id=2`

#### 相关使用
Java 中为我们提供了AtomicInteger 原子类（底层基于CAS进行更新数据），不需要加锁就在多线程并发场景下实现数据的一致性
```java
public class Test {
    private  AtomicInteger i = new AtomicInteger(0);
    public int add(){
        return i.addAndGet(1);
    }
}
```

#### ABA 问题
因为CAS需要在操作值的时候，检查值有没有发生变化，比如没有发生变化则更新，但是如果一个值原来是A，变成了B，又变成了A，那么使用CAS进行检查时则会发现它的值没有发生变化，但是实际上却变化了

ABA问题的解决思路就是使用版本号。在变量前面追加上版本号，每次变量更新的时候把版本号加1，那么A->B->A 就会变成 1A->2B->3A。

从 Java 1.5 开始，JDK 的 Atomic 包里提供了一个类AtomicStampedReference 来解决ABA问题。这个类的 compareAndSet 方法的作用是首先检查当前引用是否等于预期引用，并且检查当前标志是否等于预期标志，如果全部相等，则以原子方式将该引用和该标志的值设置为给定的更新值

#### 只能保证一个共享变量的原子操作
当对一个共享变量执行操作时，我们可以使用循环CAS的方式来保证原子操作，但是对多个共享变量操作时，循环CAS就无法保证操作的原子性，这个时候就可以用锁。还有一个取巧的办法，就是把多个共享变量合并成一个共享变量来操作。比如，有两个共享变量i = 2，j = a，合并一下ij = 2a，然后用CAS来操作ij
从 Java 1.5 开始，JDK提供了 AtomicReference 类来保证引用对象之间的原子性，就可以把多个变量放在一个对象里来进行CAS操作

### Unsafe
