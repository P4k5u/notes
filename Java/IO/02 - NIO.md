### Overview
Java NIO（New IO）是 java 的替代 IO API，意味着标准 Java IO 和 Java 网络 API 的替代品。NIO 提供了与传统 IO API 不同的 IO 模型。
有时 NIO 被称为 非阻塞 IO，然后这不是他的本意，因为 NIO 有一些 IO 操作也是阻塞的，例如：文件相关的API。

NIO 可以让你进行 非阻塞的IO操作 。例如：一个线程可以请求 channel 将数据读到 buffer，在读取的时候，线程可以去干别的事情，当数据读取到 buffer 后，线程再回来接着操作![[Pasted image 20231206160131.png]]

Java NIO 有三个核心组件：
- Channels
- Buffers
- Selectors
### Channel
在 NIO 里，所有的 IO 操作都始于 channel ，channel 有点像 stream 。buffer 从 channel 里读数据，也可以往 channel 里写数据
![[Pasted image 20231130143759.png]]

各类型的 Channel ：
- FileChannel : 文件
- DatagramChannel : UDP 网络
- SocketChannel : TCP 网络
- ServerSocketChannel : 监听到达的 TCP 连接
这些类型几乎涵盖 UDP + TCP 网络IO 以及 文件IO

简单例子：
使用 FileChannel 读取数据到 buffer 里
```java
RandomAccessFile aFile = new RandomAccessFile("data/nio-data.txt", "rw");
    FileChannel inChannel = aFile.getChannel();

    ByteBuffer buf = ByteBuffer.allocate(48);

    int bytesRead = inChannel.read(buf);
    while (bytesRead != -1) {

      System.out.println("Read " + bytesRead);
      buf.flip();

      while(buf.hasRemaining()){
          System.out.print((char) buf.get());
      }

      buf.clear();
      bytesRead = inChannel.read(buf);
    }
    aFile.close();
```

#### 分散 / 聚集
其实也是 Java NIO 具有 分散 / 聚集 支持的原因，就是在 channel 的读取和写入时所使用的 分散 / 聚集 概念

##### 分散读取
一个 channel 可以从被多个 buffer 同时读取![[Pasted image 20231201160522.png]]
代码例子：
```java
ByteBuffer header = ByteBuffer.allocate(128);
ByteBuffer body   = ByteBuffer.allocate(1024);

ByteBuffer[] bufferArray = { header, body };

channel.read(bufferArray);
```
bufferArray 里的 buffer 会依次进行读取，读取满了之后再到下一个 buffer，这也反映出 分散读取的方式不适合用于读取动态大小的数据
##### 聚集写入
一个 channel 可以被多个 buffer 同时写入![[Pasted image 20231201160946.png]]
代码例子：
```java
ByteBuffer header = ByteBuffer.allocate(128);
ByteBuffer body   = ByteBuffer.allocate(1024);

//write data into channel

ByteBuffer[] bufferArray = { header, body };

channel.write(bufferArray);
```

### Buffer
Java NIO 缓冲区在与 NIO 通道交互时使用，数据从通道读取到缓冲区，并从缓冲区写入通道

缓冲区本质上是一内存块，可以在其中写入数据，然后可以再次读取数据。该内存块包装在 NIO Buffer 对象中，该对象提供了一组方法，可以更轻松地使用该内存块
#### Buffer 的基本使用
用 buffer 操作数据可以分为 4 步：
- 数据写入 buffer
- 调用 buffer.flip()
- 从 buffer 读取数据
- 调用 buffer.clear() 或者 buffer.compact()

当你往 buffer 写入数据的时候，buffer 会追踪你写了多少数据，当需要读取数据时，调用 flip() 方法，变成 read mode 之后就可以进行读取你写入的数据

读取完所有数据之后，需要清空 buffer ，准备下一次的写入。清空的方法有两种：调用 clear() 或 compact() 方法，clear() 方法会清空 buffer 里的所有数据，compact() 方法会清空你已经读取过的数据，未读取的数据就移动到 buffer 的开始位置，新写入的数据紧随其后

简单的 buffer 使用例子：
```java
RandomAccessFile aFile = new RandomAccessFile("data/nio-data.txt", "rw");
FileChannel inChannel = aFile.getChannel();

//create buffer with capacity of 48 bytes
ByteBuffer buf = ByteBuffer.allocate(48);

int bytesRead = inChannel.read(buf); //read into buffer.
while (bytesRead != -1) {

  buf.flip();  //make buffer ready for read

  while(buf.hasRemaining()){
      System.out.print((char) buf.get()); // read 1 byte at a time
  }

  buf.clear(); //make buffer ready for writing
  bytesRead = inChannel.read(buf);
}
aFile.close();
```

#### Buffer Capacity, Position and Limit
Buffer 有三个比较重要的属性：
- capacity
- position
- limit

position 和 limit 的值随着 buffer 是处于 write mode 还是 read mode 而变动，capacity 是固定的。

下面是 write mode 和 read mode 下，各个参数的说明![[Pasted image 20231130153856.png]]
##### Capacity
作为一个内存块，buffer 有一定的固定大小，只能将一定容量的bytes，longs，chars等写入buffer，一旦buffer满了就的清空它才能继续写入数据
##### Position
当你往 buffer 中写入数据时，会往某个 position 写入。最初的 position 为 0 ，当数据写入后，position 就会往后移一个单元（+1），position 最多为 capacity -1。

当你从 buffer 中读取时，一样会从某个 position 读取，当调用 flip() 方法后， position 已经被重置为 0
##### Limit
限制数据能操作到哪个位置
在写入时，limit = capacity
在读取时，limit = position，就是你写入多少数据，就读取多少数据
#### Buffer 类型
Java NIO 的 buffer 有以下几种类型：
- ByteBuffer
- MappedByteBuffer
- CharBuffer
- DoubleBuffer
- FloatBuffer
- IntBuffer
- LongBuffer
- ShortBuffer
这些 buffer 类型代表着不同的数据类型，可以让你将 buffer 中的字节作为 char, short, long, int 等类型来处理

#### 分配 Buffer
要获取 Buffer 对象，要先进行分配，每个 Buffer 类都有一个 allocate() 方法来执行此操作
```java
ByteBuffer buf = ByteBuffer.allocate(48);
```

#### 写入数据到 Buffer
有两种方式可以将数据写入 buffer：
- 通过 channel 写入
```java
int bytesRead = inChannel.read(buf); //read into buffer.
```
- 通过调用 buffer 的 put() 方法
```java
buf.put(127);
```

#### flip()
flip() 方法可以将 buffer 从 写入模式 转换到 读取模式，调用 flip() 会把 position 设置为 0 ，并把 limit 设置到 position 之前的位置

#### 从 Buffer 中读取数据
一般有两种方法可以从 buffer 中读取数据：
- 通过 channel
```java
//read from buffer into channel.
int bytesWritten = inChannel.write(buf);
```
- 通过 buffer 的 get() 方法
```java
byte aByte = buf.get();
```

#### rewind()
当使用 rewind() 时也是转换为 读取模式，也会把 position 设置为 0 ，但跟 flip() 的区别是 limit 的值不会改变，还是保持在写入时的在 capacity 位置，所以这能读取所有数据
#### clear() 和 compact()
当 buffer 中的数据被读取完，准备继续写入的时候，可以调用 clear() 或 compact() 方法

调用 clear() 时，position 变为 0 ，limit 变为 capacity，换言之整个buffer都被 '清空' 了，但是没完全清空，因为数据还在，只是 position、limit 这些标记被重置了，找不到这些数据了

调用 compact() 就不一样了，当你读了一半，但是又想继续往里面写点数据，那就调用 compact() ，这个方法会把未读取的数据复制到 buffer 的起始位置，然后设置 position 设置在未读取数据的末尾，limit 值还是在 capacity 位置，这样你就可以接着写，且不会覆盖到旧数据
#### mark() 和 reset()
你可以调用 mark() 方法来标记当前 buffer 的 position 位置，然后调用 reset() 方法来回到标记的位置，例如：
```java
buffer.mark();
//call buffer.get() a couple of times, e.g. during parsing.
buffer.reset();  //set position back to mark.
```
#### equals() 和 compareTo()
可以调用 equals() 和 compareTo() 方法来比较两个 buffers
##### equals()
如果两个 buffer 相等：
1. buffer 类型一样
2. 剩余的字节数一样
3. 剩余的字节全都相等
equals() 方法只比较 buffer 里的剩余部分（limit - position 的部分），不是全部
##### compareTo()
compareTo() 方法也是比较两个 buffer 里的剩余元素，当满足以下条件时，就会认为其中一个 buffer 小于 另一个 buffer ：
1. 当两部的元素出现差异时，差异的元素比另一个小
2. 剩余的元素比另一个少
### Selector
Selector 允许一个线程来处理多个 channel ，这能以少量的资源来方便地管理多个连接channel，并确定哪些 channel 已经准备好读取或写入等操作

#### 为什么使用 Selector ？
使用 Selector 的好处就是一个线程就能处理多个 channel ，这样的话就能减少线程切换的开销和线程资源

但是，现在的 CPU 都是多核，多线程工作能力得到了提高，如果只用一个线程的话，可能会浪费 CPU 的性能
![[Pasted image 20231201162326.png]]

#### 创建 Selector
调用 Selector.open() 方法来创建 Selector
```java
Selector selector = Selector.open();
```

#### 在 Selector 中注册 Channel
为了在 Selector 中使用 channel ，需要把 channel 注册到 selector，调用 channel 的 register() 方法来进行注册：
```java
channel.configureBlocking(false);

SelectionKey key = channel.register(selector, SelectionKey.OP_READ);
```
在 selector 里使用的 channel 一定要是 非阻塞 模式，这意味着你不能在 selector 里使用 FileChannel ，因为 FileChannel 不能使用 非阻塞模式

register() 方法里的第二个参数，是 兴趣集 ，表示你有兴趣通过 selector 在这个 channel 中监听哪种事情，一般有四种不同的事件可以监听：
1. Connect
2. Accept
3. Read
4. Write
当 channel 触发事件，就说明这个 channel 已经准备好了，如：准备好写入数据的 channel ，就是触发 write 事件

四个事件分别对应着四个 SelectionKey 常量：
1. SelectionKey.OP_CONNECT
2. SelectionKey.OP_ACCEPT
3. SelectionKey.OP_READ
4. SelectionKey.OP_WRITE

当对多个 event 感兴趣的时候，可以使用 OR 来表示：
```java
int interestSet = SelectionKey.OP_READ | SelectionKey.OP_WRITE;
```

#### SelectionKey
当在 Selector 中注册 channel 时，register() 方法回返回一个 SelectionKey 对象，这个对象包含一些重要属性：
- interest set
- ready set
- channel
- selector
- attached object（optional）

##### Interest Set
interest set 就是注册的时候所设置的感兴趣的事件，它是一个 int 类型，每一位表示一个是事件，可以用 selectionkey 来操作 insterest set ：
```java
int interestSet = selectionKey.interestOps();

boolean isInterestedInAccept  = SelectionKey.OP_ACCEPT  == (interests & SelectionKey.OP_ACCEPT);
boolean isInterestedInConnect = SelectionKey.OP_CONNECT == (interests & SelectionKey.OP_CONNECT);
boolean isInterestedInRead    = SelectionKey.OP_READ    == (interests & SelectionKey.OP_READ);
boolean isInterestedInWrite   = SelectionKey.OP_WRITE   == (interests & SelectionKey.OP_WRITE
```
当对 interestSet 和 SelectionKey 事件常量进行 AND 操作，就可以判断是否在监视该事件

##### Ready Set
ready set 存放的就是已经准备好的 channel ，可以通过调用下面的方法来访问：
```java
int readySet = selectionKey.readyOps();
```
使用下面四个方法，可以知道 channel 是否准备好：
```java
selectionKey.isAcceptable();
selectionKey.isConnectable();
selectionKey.isReadable();
selectionKey.isWritable();
```

##### Channel、Selector
从 selectionKey 访问 channel、selector 十分简单，下面就搞定了：
```java
Channel  channel  = selectionKey.channel();

Selector selector = selectionKey.selector();
```

##### Attaching Objects
可以把对象附加到 SelectionKey，这是识别 channel 或把更多信息附加到 channel 的方法，比如，你可以增加一个使用这个 channel 的 buffer ，或者一个有更多信息的对象，下面代码：
```java
selectionKey.attach(theObject);

Object attachedObj = selectionKey.attachment();
```
当给 selector 注册 channel 的时候，也可以在 register() 方法里附加对象：
```java
SelectionKey key = channel.register(selector, SelectionKey.OP_READ, theObject);
```

#### 通过 Selector 选择 Channel
当把一个或多个 channel 注册到 selector 时，如果你感兴趣的 event 的 channel 已经准备好了，那么调用 select() 方法就能获取准备好接收到达 event 的 channel

select() 方法有下面几种：
- int select()
- int select(long timeout)
- int selectNow()

**select()** 会阻塞至少一个 channel 直到准备好接收感兴趣的 event 为止
**select(long timeout)** 跟 select() 一样，就是多了个 milliseconds 的时间参数
**selectNow()** 不会阻塞，如果任意 channel 准备好之后就立即返回

select() 返回的 int 表示有多少个 channel 已经准备好了。但这个数量指的是距离上一次调用 select() 的期间产生的 channel，比如：你调用了 select() 返回了 1（表示有一个 channel 准备好了），然后你再调用一次，会再次返回 1 （又一个 channel 准备好了）

这时你会发现，已经有两个 channel 准备好了，但是你每次调用 select() 方法只返回了一个

##### selectedKeys()
通过 "selected key set" 来访问那些已经准备好的 channel ，调用 selectedKeys() 来获得 selected key set：
```java
Set<SelectionKey> selectedKeys = selector.selectedKeys();
```
通过遍历 selected key set 来访问已经准备好的 channel：
```java
Set<SelectionKey> selectedKeys = selector.selectedKeys();

Iterator<SelectionKey> keyIterator = selectedKeys.iterator();

while(keyIterator.hasNext()) {
    
    SelectionKey key = keyIterator.next();

    if(key.isAcceptable()) {
        // a connection was accepted by a ServerSocketChannel.

    } else if (key.isConnectable()) {
        // a connection was established with a remote server.

    } else if (key.isReadable()) {
        // a channel is ready for reading

    } else if (key.isWritable()) {
        // a channel is ready for writing
    }

    keyIterator.remove();
}
```
在这循环中，可以对你相应类型的 channel 测试一下是否准备好，准备好的话就调用 SelectionKey.channel() 获得该 channel ，但是要强转一下来获得具体类型的 channel 比如，ServerSocketChannel 或 SocketChannel

keyIterator.remove() 方法需要在每次遍历调用一下，因为 Selector 不会删除 selectionKey 实例，如果不删除的话，当你操作这个channel，下一次准备好的时候又会加入到 selected key set 里。

##### wakeUp()
当一个线程因为调用 select() 阻塞时，可以强制退出方法，由另一个线程调用 Selectr.wakeUp() 方法来唤醒因为调用 select() 方法而阻塞的线程。

当没有线程调用 select() 阻塞时，调用 wakeUp() 方法的话，那么下一个调用 select() 的线程将会立刻被唤醒

##### close()
调用 close() 方法会关闭 Selector 并且失效所有已经注册的 SelectionKey，但是相关的 channel 不会被关闭

#### 完整使用例子
打开 selector，注册一个 channel 并监控 event 是否已经到达
```java
Selector selector = Selector.open();

channel.configureBlocking(false);

SelectionKey key = channel.register(selector, SelectionKey.OP_READ);


while(true) {

  int readyChannels = selector.selectNow();

  if(readyChannels == 0) continue;


  Set<SelectionKey> selectedKeys = selector.selectedKeys();

  Iterator<SelectionKey> keyIterator = selectedKeys.iterator();

  while(keyIterator.hasNext()) {

    SelectionKey key = keyIterator.next();

    if(key.isAcceptable()) {
        // a connection was accepted by a ServerSocketChannel.

    } else if (key.isConnectable()) {
        // a connection was established with a remote server.

    } else if (key.isReadable()) {
        // a channel is ready for reading

    } else if (key.isWritable()) {
        // a channel is ready for writing
    }

    keyIterator.remove();
  }
}
```