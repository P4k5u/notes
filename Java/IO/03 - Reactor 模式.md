### 简介
因为 I/O 多路复用是面向过程的方式，所以基于面向对象的方式，对 I/O 多路复用作了一层封装，这就是 Reactor 模式

顾名思义，这里的 react 指的是对 事件 反应，也就是来了一个事件，Reactor 就有相应的反应。

Reactor 的工作模式，即 I/O 多路复用监听事件，收到事件后，根据事件类型分配给某个进程/线程

Reactor 模式主要由两个核心部分组成：
- Reactor：负责监听和分发事件，事件类型包含连接事件、读写事件
- 处理资源池：负责处理事件，如: read -> 业务逻辑 -> send

Reactor 模式有 3 个比较常用的方案：
- 单 Reactor 单进程 / 线程
- 单 Reactor 多线程 / 进程
- 多 Reactor 多进程 / 线程

### 单 Reactor 单线程
这个方案的示意图如下
![[Pasted image 20231206162946.png]]
一共有三个组件：
- Reactor：监听和分发事件，一般是accept、read、write
- Acceptor：用于获取连接
- Handler：处理业务
对象里的 select、accept、read、send 是系统调用函数，dispatch 和 业务处理 是需要完成的操作，其中 dispatch 是分发事件操作

大致流程如下：
- Reactor 对象通过 select() 监听事件，收到事件后通过 dispatch 进行分发，具体分发给 Acceptor 对象还是 Handler 对象，还要看收到的事件类型
- 如果是连接建立的事件，则交由 Acceptor 对象进行处理，Acceptor 对象会通过 accept 方法 获取连接，并创建一个 Handler 对象来处理后续的响应事件
- 如果不是连接建立事件， 则交由当前连接对应的 Handler 对象来进行响应
- Handler 对象通过 read -> 业务处理 -> send 的流程来完成完整的业务流程

代码例子：

Handler
```java
public class Handler implements Runnable{  
    private final SocketChannel channel;  
  
    public Handler(SocketChannel channel) {  
        this.channel = channel;  
    }  
  
    @Override  
    public void run() {  
        try {  
            ByteBuffer buffer = ByteBuffer.allocate(128);  
            channel.read(buffer);  
            buffer.flip();  
            System.out.println("接收到客户端数据："+new String(buffer.array(), 0, buffer.remaining()));  
            channel.write(ByteBuffer.wrap("已收到！".getBytes()));  
        }catch (IOException e){  
            e.printStackTrace();  
        }  
    }  
}
```
Acceptor
```java
/**  
 * Acceptor主要用于处理连接操作  
 */  
public class Acceptor implements Runnable{  
  
    private final ServerSocketChannel serverChannel;  
    private final Selector selector;  
  
    public Acceptor(ServerSocketChannel serverChannel, Selector selector) {  
        this.serverChannel = serverChannel;  
        this.selector = selector;  
    }  
  
    @Override  
    public void run() {  
        try{  
            SocketChannel channel = serverChannel.accept();  
            System.out.println("客户端已连接，IP地址为："+channel.getRemoteAddress());  
            channel.configureBlocking(false);  
            //这里在注册时，创建好对应的Handler，这样在Reactor中分发的时候就可以直接调用Handler了  
            channel.register(selector, SelectionKey.OP_READ, new Handler(channel));  
        }catch (IOException e){  
            e.printStackTrace();  
        }  
    }  
}
```
Reactor
```java
public class Reactor implements Closeable, Runnable{  
  
    private final ServerSocketChannel serverChannel;  
    private final Selector selector;  
    public Reactor() throws IOException{  
        serverChannel = ServerSocketChannel.open();  
        selector = Selector.open();  
    }  
  
    @Override  
    public void run() {  
        try {  
            serverChannel.bind(new InetSocketAddress(8080));  
            serverChannel.configureBlocking(false);  
            //注册时，将Acceptor作为附加对象存放，当选择器选择后也可以获取到  
            serverChannel.register(selector, SelectionKey.OP_ACCEPT, new Acceptor(serverChannel, selector));  
            while (true) {  
                int count = selector.select();  
                System.out.println("监听到 "+count+" 个事件");  
                Set<SelectionKey> selectionKeys = selector.selectedKeys();  
                Iterator<SelectionKey> iterator = selectionKeys.iterator();  
                while (iterator.hasNext()) {  
                    this.dispatch(iterator.next());   //通过dispatch方法进行分发  
                    iterator.remove();  
                }  
            }  
        }catch (IOException e) {  
            e.printStackTrace();  
        }  
    }  
  
    //通过此方法进行分发  
    private void dispatch(SelectionKey key){  
        Object att = key.attachment();   //获取attachment，ServerSocketChannel和对应的客户端Channel都添加了的  
        if(att instanceof Runnable) {  
            ((Runnable) att).run();   //由于Handler和Acceptor都实现自Runnable接口，这里就统一调用一下  
        }   //这样就实现了对应的时候调用对应的Handler或是Acceptor了  
    }  
  
    //用了记得关，保持好习惯，就像看完视频要三连一样  
    @Override  
    public void close() throws IOException {  
        serverChannel.close();  
        selector.close();  
    }  
}
```

### 单 Reactor 多线程 
这个方案的示意图如下：
![[Pasted image 20231207151251.png]]

其实就是把 Handler 中处理业务的部分放在 线程池 中处理

修改一下 Handler
```java
public class Handler implements Runnable{  
    //把线程池给安排了，10个线程  
    private static final ExecutorService POOL = Executors.newFixedThreadPool(10);  
    private final SocketChannel channel;  
    public Handler(SocketChannel channel) {  
        this.channel = channel;  
    }  
  
    @Override  
    public void run() {  
        try {  
            ByteBuffer buffer = ByteBuffer.allocate(1024);  
            channel.read(buffer);  
            buffer.flip();  
            POOL.submit(() -> {  
                try {  
                    System.out.println("接收到客户端数据："+new String(buffer.array(), 0, buffer.remaining()));  
                    channel.write(ByteBuffer.wrap("已收到！".getBytes()));  
                }catch (IOException e){  
                    e.printStackTrace();  
                }  
            });  
        } catch (IOException e) {  
            throw new RuntimeException(e);  
        }  
    }  
}
```

### 多 Reactor 多线程
单个 Reactor 的压力可能会有点大，所以把 Reactor 做成一主多从的形式，主 Reactor 负责 Accept 操作，从 Reactor 负责 dispatch 操作![[Pasted image 20231207152308.png]]

新增 SubReactor
```java
//SubReactor作为从Reactor  
public class SubReactor implements Runnable, Closeable {  
    //每个从Reactor也有一个Selector  
    private final Selector selector;  
  
    //创建一个4线程的线程池，也就是四个从Reactor工作  
    private static final ExecutorService POOL = Executors.newFixedThreadPool(4);  
    private static final SubReactor[] reactors = new SubReactor[4];  
    private static int selectedIndex = 0;  //采用轮询机制，每接受一个新的连接，就轮询分配给四个从Reactor  
    static {   //在一开始的时候就让4个从Reactor跑起来  
        for (int i = 0; i < 4; i++) {  
            try {  
                reactors[i] = new SubReactor();  
                POOL.submit(reactors[i]);  
            } catch (IOException e) {  
                e.printStackTrace();  
            }  
        }  
    }  
    //轮询获取下一个Selector（Acceptor用）  
    public static Selector nextSelector(){  
        Selector selector = reactors[selectedIndex].selector;  
        selectedIndex = (selectedIndex + 1) % 4;  
        return selector;  
    }  
  
    private SubReactor() throws IOException {  
        selector = Selector.open();  
    }  
  
    @Override  
    public void run() {  
        try {   //启动后直接等待selector监听到对应的事件即可，其他的操作逻辑和Reactor一致  
            while (true) {  
                int count = selector.select();  
                System.out.println(Thread.currentThread().getName()+" >> 监听到 "+count+" 个事件");  
                Set<SelectionKey> selectionKeys = selector.selectedKeys();  
                Iterator<SelectionKey> iterator = selectionKeys.iterator();  
                while (iterator.hasNext()) {  
                    this.dispatch(iterator.next());  
                    iterator.remove();  
                }  
            }  
        }catch (IOException e) {  
            e.printStackTrace();  
        }  
    }  
  
    private void dispatch(SelectionKey key){  
        Object att = key.attachment();  
        if(att instanceof Runnable) {  
            ((Runnable) att).run();  
        }  
    }  
  
    @Override  
    public void close() throws IOException {  
        selector.close();  
    }  
}
```

修改一下 Acceptor
```java
public class Acceptor implements Runnable{  
  
    private final ServerSocketChannel serverChannel;   //只需要一个ServerSocketChannel就行了  
  
    public Acceptor(ServerSocketChannel serverChannel) {  
        this.serverChannel = serverChannel;  
    }  
  
    @Override  
    public void run() {  
        try{  
            SocketChannel channel = serverChannel.accept();   //还是正常进行Accept操作，得到SocketChannel  
            System.out.println(Thread.currentThread().getName()+" >> 客户端已连接，IP地址为："+channel.getRemoteAddress());  
            channel.configureBlocking(false);  
            Selector selector = SubReactor.nextSelector();   //选取下一个从Reactor的Selector  
            selector.wakeup();    //在注册之前唤醒一下防止卡死  
            channel.register(selector, SelectionKey.OP_READ, new Handler(channel));  //注意现在注册的是从Reactor的Selector  
        }catch (IOException e){  
            e.printStackTrace();  
        }  
    }  
}
```