### 简介
Java 的自动内存管理主要是针对对象内存的回收和对象内存的分配，其中最核心的功能就是 堆内存中对象的分配与回收

Java 堆事垃圾收集器管理的主要区域，因此也被称为 GC 堆（Garbage Collected Heap）

### 垃圾回收算法

### 垃圾收集器
垃圾收集器是内存回收的具体实现
根据具体应用场景选择适合自己的垃圾收集器

JDK 默认垃圾收集器（使用 java -XX:+PrintCommandLineFlags -version 命令查看）
- JDK 8：Parallel Scavenge（新生代）+ Parallel Old（老年代）
- JDK 9 ～ JDK 20：G1
#### Serial
Serial 收集器是最基本、历史最悠久的垃圾收集器。这是一个单线程收集器，在进行垃圾收集工作的时候必须暂停其他所有的工作线程（Stop the world），直到收集结束

针对新生代，采用 标记-复制 算法

优势：简单高效，采用单线程避免了上下文之间的切换

劣势：不可控的 Stop the world

使用场景
- Client 模式（桌面应用）
- 单核服务器

#### Serial Old
Serial Old 是 Serial 收集器的老年代版本，同样是一个单线程收集器

针对老年代，使用 标记-整理 算法

优劣势与 Serial 一样

#### ParNew
ParNew 收集器是 Serial 收集器的多线程版本，除了使用多线程进行垃圾回收之外，其余和 Serial 收集器一致

使用场景
- Server 模式下使用，除 Serial 外，只有它能与 CMS 收集器配合工作

#### Parallel Scavenge
Parallel Scavenge 是一款用于新生代的多线程收集器，采用 标记-复制 算法，与 ParNew 不同的是 Parallel Scavenge 收集器的目的是达到一个可控制的吞吐量，而 ParNew 收集关注点在于尽可能的缩短垃圾收集时用户线程的停顿时间

优势：追求高吞吐量，高效利用 CPU ，吞吐量优先且能进行精确控制

劣势：或者说特点，在吞吐量和停顿时间选择了吞吐量，所以单个 GC 周期的停顿时间会变长

使用场景
- 具有多个 CPU ，对暂停时间没有特别高的要求，程序主要在后台进行计算，不需要与用户进行太多交互的场景，如：批量处理，订单 等

参数设置
- -XX:MaxGCPauseMillis：控制最大垃圾收集停顿时间，大于 0 的毫秒数
- -XX:GCTimeRatio：设置垃圾收集时间占总时间的比率，0 < n < 100 的整数
#### Parallel Old
Parallel Old 是 Parallel Scavenge 收集器的老年代版本，使用多线程，采用 标记-整理 算法，可以充分利用多核 CPU 的计算能力

使用场景
- 在注重吞吐量以及 CPU 资源敏感的场景，就有了 Parallel Scavenge（新生代）+ Parallel Old（老年代）收集器的黄金组合

参数设置
- -XX:UseParallelOldGC：指定使用 Parralle Old 收集器

#### CMS
CMS（Concurrent Mark Sweep）收集器是一种以获取最短回收停顿时间为目标的收集器，采用的算法是 标记-清除，针对老年代

运行过程：
- 初始标记：标记 GC Roots 能够直接关联到达对象
- 并发标记：进行 GC Roots Tracing 的过程
- 重新标记：修正并发标记期间因用户程序继续运行而导致标记产生变动的那一部分标记记录
- 并发清除：用 标记-清除 算法清除对象

优势：停顿时间短，吞吐量大，并发收集

劣势：对 CPU 资源敏感，无法收集浮动垃圾，容易产生大量内存碎片

使用场景
- 与用户交互较多的场景
- 希望系统停顿时间最短，注重服务的响应速度

参数设置
- -XX:+UserConcMarkSweepGC：指定使用 CMS 收集器
#### G1
G1（Garbage-First）是 JDK7 推出的商用收集器
能充分利用多 CPU，多核环境下的硬件优势，能进行分代收集，不用与其他收集器合作

从整体上来看基于 标记-整理 算法，从局部上看是基于 标记-复制 算法，因此 G1 运行期间不回产生空间碎片

在后台维护一个优先列表，每次根据允许的收集时间，优先选择回收价值最大的 Region，用 Region 划分内存空间并采用优先级的回收方式使得收集效率很高

G1 能够建立可预测的时间停顿模型，能让使用者明确指定一个长度为 M 毫秒的时间片段内，消耗在垃圾收集上的时间不得超过 N 毫秒

优势：充分利用多 CPU 硬件优势，能独立管理整个 GC 堆（新生代和老年代），不会产生内存碎片，除了追求低停顿，还能建立可预测的停顿时间模型

劣势：内存占用可能会大一点，CMS 在小内存上表现好一点，内存界限在 6 GB ～ 8 GB，可以无视这点差距，无脑使用

使用场景
- 无脑使用

参数设置
- -XX:+UserG1GC：指定使用 G1 收集器
- -XX:InitiatingHeapoccupancyPercent：当整个 java 堆占用率达到参数值时，开始并发标记阶段
- -XX:MaxGCPauseMillis：为 G1 设置暂停时间目标
- -XX:G1HeapRegionSize：设置每个 Region 大小