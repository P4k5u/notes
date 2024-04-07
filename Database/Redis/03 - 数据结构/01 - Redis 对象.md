Redis 数据结构的底层设计如下：
![[Pasted image 20240331112006.png]]
它反映了 Redis 的每种对象都由对象结构（redis object）与对应的底层数据结构组合而成

不用的编码方式对应的底层数据结构是不同的
对象结构里包含的成员变量：
- type：标识该对象是什么类型的对象（String 对象、 List 对象、Hash 对象、Set 对象和 Zset 对象）；
- encoding：标识该对象使用了哪种底层的数据结构；
- **ptr：指向底层数据结构的指针**。

所以要从两个角度开始研究：
- 对象设计机制：对象结构（redis object）
- 编码类型和底层数据结构：对应编码的数据结构

### Redis Object 的作用
为了区别不同命令对不同键的处理，并且在不同场景下进行优化。
Redis 必须让每个键都带有类型信息，使得程序可以检查键的类型，并为它选择合适的处理方式，同时还要根据数据类型的不同编码进行多态处理

为了实现这个需求，Redis 构建了自己的类型系统，主要功能包括：
- Redis Object 对象
- 基于 Redis Object 对象的类型检查
- 基于 Redis Object 对象的显式多态函数
- 对 Redis Object 进行分配、共享和销毁的机制

### Redis Object 数据结构
Redis Object 是 Redis 类型系统的核心，数据库中的每个键、值，以及 Redis 本身处理的参数，都为这种数据类型，在创建一个键值对是，会至少创建两个 redis object（键、值）
```lua
/*
 * Redis 对象
 */
typedef struct redisObject {

    // 类型
    unsigned type:4;

    // 编码方式
    unsigned encoding:4;

    // LRU - 24位, 记录最末一次访问时间（相对于lru_clock）; 或者 LFU（最少使用的数据：8位频率，16位访问时间）
    unsigned lru:LRU_BITS; // LRU_BITS: 24

    // 引用计数
    int refcount;

    // 指向底层数据结构实例
    void *ptr;

} robj;
```
其中 type、encoding 和 ptr 最重要的三个属性
- type：记录了对象所保存的值的类型。值为以下常量：
```lua
/*
* 对象类型
*/
#define OBJ_STRING 0 // 字符串
#define OBJ_LIST 1 // 列表
#define OBJ_SET 2 // 集合
#define OBJ_ZSET 3 // 有序集
#define OBJ_HASH 4 // 哈希表
```
- encoding：记录了对象所保存的值的编码
```lua
/*
* 对象编码
*/
#define OBJ_ENCODING_RAW 0     /* Raw representation */
#define OBJ_ENCODING_INT 1     /* Encoded as integer */
#define OBJ_ENCODING_HT 2      /* Encoded as hash table */
#define OBJ_ENCODING_ZIPMAP 3  /* 注意：版本2.6后不再使用. */
#define OBJ_ENCODING_LINKEDLIST 4 /* 注意：不再使用了，旧版本2.x中String的底层之一. */
#define OBJ_ENCODING_ZIPLIST 5 /* Encoded as ziplist */
#define OBJ_ENCODING_INTSET 6  /* Encoded as intset */
#define OBJ_ENCODING_SKIPLIST 7  /* Encoded as skiplist */
#define OBJ_ENCODING_EMBSTR 8  /* Embedded sds string encoding */
#define OBJ_ENCODING_QUICKLIST 9 /* Encoded as linked list of ziplists */
#define OBJ_ENCODING_STREAM 10 /* Encoded as a radix tree of listpacks */
```
- ptr：这是一个指针，指向实际保存值的数据结构
### 类型检查与命令多态
Redis 用于操作键的命令基本上可以分为两种类型：
- 对任何类型的键执行：如 DEL、EXPIRE、RENAME、TYPE 命令等
- 对特定类型的键执行：如 SET、GET 针对 String 类型，LPOP 针对列表键执行
#### 类型检查的实现
在执行一个类型特定的命令之前，会先查输入键的类型是否正确
类型检查时通过redis object 结构的 type 属性来实现的：
- 执行命令之前，会检查键的值对象是否为执行命令所需的类型，如果是，则执行命令
- 如果不是，就拒绝执行命令，并返回一个类型错误
如执行 LLEN 命令这个例子：![[Pasted image 20231115165712.png]]
#### 多态命令的实现
Redis 会根据值对象的编码方式，选择正确的命令实现代码来执行命令
LLEN 命令执行时，会判断列表对象编码为 ziplist 还是 linkedlist，从而用不同的函数来返回列表的长度
DEL、EXPIRE、TYPE等命令也是多态命令，不过是基于类型的多态

执行命令的整个过程如下：![[Pasted image 20231115170042.png]]

### 共享对象
Redis 一般会把一些常见的值放到一个共享对象中：
- 各种命令的返回值：如成功时返回的OK，错误时的ERROR
- 0 ～ 9999 的整数
共享对象只能被字典和双向链表这类带有指针的数据结构使用
### 对象空转时长
redis object 中的 lru 属性，记录对象最后一次被命令程序访问的时间
空转时长：当前时间减去键的值对象的lru时间，就是该键的空转时长
在释放内存时，优先释放空转时长较高的
### 内存回收
redis object 中有 refcount 属性，是对象的引用计数，显然计数 0 就是可以回收