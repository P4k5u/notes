String 被声明为 final，因此它不可被继承。

内部使用 char 数据存储数据，该数据被声明为 final，这意味着 value 数组初始化之后就不能再引用其它数组。并且 String 内部没有改变 value 数组的方法，因此可以保证 String 不可变
```java
public final class String implements java.io.Serializable,Comparable<String>, CharSequence { 
/** The value is used for character storage. */ 
private final char value[];
...
}
```

### 不可变的好处
1. ** 可以缓存 hash 值 **
		因为 String 的 hash 值经常被使用，例如：String 用做 HashMap 的 key。不可变的特性可以使得 hash 值也不可变，因此只需要进行一次计算。
2. ** String Pool 的需要 **
		如果一个 String 对象已经被创建过了，那么就会从 String Pool 中取得引用。只有 String 是不可变的，才可能使用 String Pool
		![[Pasted image 20231024182433.png]]
3. ** 安全性 **
		String 经常作为参数，String 不可变性可以保证参数不可变。例如：在作为网络连接参数的情况下，如果 String 时可变的，那么在网络连接过程中，String 被改变，会误认为在与其它主机连接
4. ** 线程安全 **
		String 不可变性天生具备线程安全，可以在多个线程中安全地使用

### String, StringBuffer and StringBuilder
1. 可变性
   - String 不可变
   - StringBuffer 和 StringBuilder 可变
2. 线程安全
   - String 不可变，因此是线程安全的
   - StringBuilder 不是线程安全的
   - StringBuffer 是线程安全的，内部使用 synchronized 进行同步

### String.intern()
使用 String.intern() 可以保证相同内容的字符串变量引用同一个内存对象

下面示例中，s1 和 s2 采用 new String() 的方式新建了两个不同的对象，而 s3 是通过 s1.intern() 方法取得一个对象引用
intern() 首先把 s1 引用的对象放到 String Pool （字符串常量池）中，然后返回这个对象引用。因此 s3 和 s1 引用的是同一个字符串常量池的对象
```java
String s1 = new String("aaa");
String s2 = new String("aaa");
sout(s1 == s2); // false 
String s3 = s1.intern();
sout(s1.intern() == s3); // true
```
如果采用的是 "bbb" 这种使用双引号的形式来创建字符串实例，会自动地将新建的对象放入 String Pool 中
```java
String s4 = "bbb";
String s5 = "bbb";
sout(s4 == s5);  // true
```

### HotSpot 中字符常量池保存在哪里？
1. 运行时常量池（Runtime Constant Pool）是虚拟机规范中是方法区的一部分，在加载类和结构到虚拟机后，就会创建对应的运行时常量池，而字符串常量池是这个过程中常量字符串的存放位置。所以从这个角度，字符串常量池属于虚拟机规范中的方法区，它是一个逻辑上的概念。而堆区，永久代以及元空间是实际的存放位置
2. 不同的虚拟机对虚拟机的规范（比如方法区）是不一样的，只有 HotSpot 才有永久代的概念
3. HotSpot 也是在持续改进，对于不同版本的JDK，实际的存储位置是不一样的，具体看如下表格：

   | JDK版本      | 是否有永久代，字符串常量池存放位置                                                             | 方法区定义，实际位置                                                                                                         |
   | ------------ | ---------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
   | JDK1.6及之前 | 有永久代，运行时常量池（包括字符串常量池），静态变量存放在永久代上                             | 这个时期方法区在HotSpot中是由永久代来实现的，以至于**这个时期说方法区就是指永久代**                                          |
   | JDK1.7       | 有永久代，但已经逐步“去永久代”，字符串常量池、静态变量移除，保存在堆中                         | 这个时期方法区在HotSpot中由**永久代**（类型信息、字段、方法、常量）和**堆**（字符串常量池、静态变量）共同实现                |
   | JDK1.8及之后 | 取消永久代，类型信息、字段、方法、常量保存在本地内存的元空间，但字符串常量池、静态变量仍在堆中 | 这个时期方法区在HotSpot中由本地内存的** 元空间**（类型信息、字段、方法、常量）和 ** 堆 ** （字符串常量池、静态变量）共同实现 | 
