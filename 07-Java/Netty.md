# NIO 基础

## 1. 三大组件

### Channel

channel 是读写数据的 **双向通道**，类似于 stream， 但 stream 是 **单向通道**，要么是输入，要么是输出。

常见的 Channel 有：FileChannel、DatagramChannel、SocketChannel、ServerSocketChannel

```mermaid
graph LR
channel --> buffer
buffer --> channel
```

### Selector

selector 的作用就是配合一个线程来管理多个 channel，获取这些 channel 上发生的事件。

这些 channel 工作在 **非阻塞模式** 下，不会让线程吊死在 单个 channel 上。

调用 selector 的 select() 会阻塞直到某个 channel 发生了读写就绪事件，select 方法返回这些事件交给 thread 来处理

Selector 对 SelectionKey : 1 对 n

SelectionKey 对 Channel : 1 对 1

<img src="pictures/image-20210519205153117.png" alt="image-20210519205153117" style="zoom:67%;" />

**selector.select()** 何时不阻塞：

1. 事件发生时
2. 调用 selector.wakeup()
3. 调用 selector.close()
4. selector.select() 所在线程 interrupt 时





### Buffer

buffer 是 io 缓冲区，用来缓冲读写数据（**优化组件**）。常见的 Buffer 有：Byte Short Int Long Float Double Char ... 

buffer 是 **非线程安全的**

**DirectByteBuf  零拷贝**

buffer 属性：**capacity、position、limit**

编码姿势：

1. 向 buffer 写入数据，例如调用 channel.read(buffer)

2. 🔥读 buffer 前调用 **flip()** 切换至 **读模式**

3. 从 buffer 读取数据，buffer.get() 改变 position，buffer.get(i) 不变

4. 调用 **clear()** 或 **compact()** 切换至 **写模式**

写模式下，position 是写入位置，limit 等于容量，下图表示写入了 4 个字节后的状态

<img src="pictures/image-20210413145240982.png" alt="image-20210413145240982" style="zoom: 67%;" />

flip 动作发生后，position 切换为读取位置，limit 切换为读取限制

<img src="pictures/image-20210413145309286.png" alt="image-20210413145309286" style="zoom: 67%;" />

compact 动作，是把未读完的部分向前压缩，然后切换至写模式

<img src="pictures/image-20210413145525145.png" alt="image-20210413145525145" style="zoom: 60%;" />

clear 动作，切换成初始化写状态

<img src="pictures/image-20210413145356678.png" alt="image-20210413145356678" style="zoom: 67%;" />



## 2. 文件编程

FileChannel 只能工作在 **阻塞模式** 下，无法直接打开 FileChannel，须通过 FileInputStream、FileOutputStream 或者 RandomAccessFile 的 getChannel 方法来获取 FileChannel

* 通过 FileInputStream 获取的 channel 只能读
* 通过 FileOutputStream 获取的 channel 只能写
* 通过 RandomAccessFile 是否能读写根据构造 RandomAccessFile 时的读写模式决定



## 3. 网络编程

### 处理 accept 事件

```java
public class Server {
    public static void main(String[] args) {
        try (ServerSocketChannel channel = ServerSocketChannel.open()) {
            channel.bind(new InetSocketAddress(8080));
            System.out.println(channel);
            Selector selector = Selector.open();
            channel.configureBlocking(false);
            channel.register(selector, SelectionKey.OP_ACCEPT);
            while (true) {
                int count = selector.select();
                // 获取所有事件
                Set<SelectionKey> keys = selector.selectedKeys();
                // 遍历所有事件，逐一处理
                Iterator<SelectionKey> iter = keys.iterator();
                while (iter.hasNext()) {
                    SelectionKey key = iter.next();
                    // 判断事件类型
                    if (key.isAcceptable()) {
                        ServerSocketChannel c = (ServerSocketChannel) key.channel();
                        // 必须处理
                        SocketChannel sc = c.accept();
                    }
                    // 处理完毕，必须将事件移除
                    iter.remove();
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

``` java
public class Client {
    public static void main(String[] args) {
        try (Socket socket = new Socket("localhost", 8080)) {
            System.out.println(socket);
            socket.getOutputStream().write("world".getBytes());
            System.in.read();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

**注意：**事件发生后，**要么处理，要么取消（cancel）**，不能什么都不做，否则下次该事件仍会触发，因为 select() 是水平触发模式

**select()、poll()** 模型都是水平触发模式；**epoll()** 模型即支持水平触发，也支持边缘触发，默认是水平触发

+ **Level_triggered(水平触发)：** 当被监控的文件描述符上有可读写事件发生时，epoll_wait()会通知处理程序去读写。如果这次没有把数据一次性全部读写完(如读写缓冲区太小)，那么下次调用 epoll_wait()时，它还会通知你，在上没读写完的文件描述符上继续读写，

+ **Edge_triggered(边缘触发)：** 当被监控的文件描述符上有可读写事件发生时，epoll_wait()会通知处理程序去读写。如果这次没有把数据一次性全部读写完(如读写缓冲区太小)，那么下次调用epoll_wait()时，它不会通知你，也就是它只会通知你一次，直到该文件描述符上出现第二次可读写事件才会通知你。



### 处理 read 事件

``` java
public class Server {
    public static void main(String[] args) {
        try (ServerSocketChannel channel = ServerSocketChannel.open()) {
            channel.bind(new InetSocketAddress(8080));
            System.out.println(channel);
            Selector selector = Selector.open();
            channel.configureBlocking(false);
            channel.register(selector, SelectionKey.OP_ACCEPT);
            while (true) {
                int count = selector.select();
                // 获取所有事件
                Set<SelectionKey> keys = selector.selectedKeys();
                // 遍历所有事件，逐一处理
                Iterator<SelectionKey> iter = keys.iterator();
                while (iter.hasNext()) {
                    SelectionKey key = iter.next();
                    // 判断事件类型
                    if (key.isAcceptable()) {
                        ServerSocketChannel c = (ServerSocketChannel) key.channel();
                        // 必须处理
                        SocketChannel sc = c.accept();
                        sc.configureBlocking(false);
                        sc.register(selector, SelectionKey.OP_READ);
                    } else if (key.isReadable()) {
                        SocketChannel sc = (SocketChannel) key.channel();
                        ByteBuffer buffer = ByteBuffer.allocate(128);
                        int read = sc.read(buffer);
                        if(read == -1) {
                            key.cancel();
                            sc.close();
                        } else {
                            buffer.flip();
                            debug(buffer);
                        }
                    }
                    // 处理完毕，必须将事件移除
                    iter.remove();
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

**注意：** keys != **selected**Keys  两个不同的集合

1.  iter.remove 作用：select 在事件发生后，就会将相关的 key 放入 **selected**Keys 集合，但不会在处理完后从 selectedKeys 集合中移除，需要我们自己编码删除

2. cancel 作用：cancel 会取消注册在 selector 上的 channel，并从 **keys** 集合中删除 key 后续不会再监听事件

3. 如何处理消息边界（**TCP 粘包问题，底层是数据流，没有固定数据包大小**）：

   <img src="pictures/image-20210519205527920.png" alt="image-20210519205527920" style="zoom:80%;" />

   + 思路一：**固定足够大的消息长度**，服务器按预定长度读取，缺点是浪费带宽
   + 思路二：**按分隔符拆分**，缺点是效率低
   + 思路三：**TLV 格式**，即 Type 类型、Length 长度、Value 数据。在类型和长度已知的情况下，即可分配大小合适的 Buffer



### 处理 write 事件

``` java
public class Server {
    public static void main(String[] args) throws IOException {
        ServerSocketChannel ssc = ServerSocketChannel.open();
        ssc.configureBlocking(false);
        ssc.bind(new InetSocketAddress(8080));
        Selector selector = Selector.open();
        ssc.register(selector, SelectionKey.OP_ACCEPT);
        while(true) {
            selector.select();
            Iterator<SelectionKey> iter = selector.selectedKeys().iterator();
            while (iter.hasNext()) {
                SelectionKey key = iter.next();
                iter.remove();
                if (key.isAcceptable()) {
                    SocketChannel sc = ssc.accept();
                    sc.configureBlocking(false);
                    SelectionKey sckey = sc.register(selector, SelectionKey.OP_READ);
                    // 1. 向客户端发送内容
                    StringBuilder sb = new StringBuilder();
                    for (int i = 0; i < 3000000; i++) {
                        sb.append("a");
                    }
                    ByteBuffer buffer = Charset.defaultCharset().encode(sb.toString());
                    int write = sc.write(buffer);
                    // 3. write 表示实际写了多少字节
                    System.out.println("实际写入字节:" + write);
                    // 4. 如果有剩余未读字节，才需要关注写事件
                    if (buffer.hasRemaining()) {
                        // read 1  write 4
                        // 在原有关注事件的基础上，多关注 写事件
                        sckey.interestOps(sckey.interestOps() + SelectionKey.OP_WRITE);
                        // 把 buffer 作为附件加入 sckey
                        sckey.attach(buffer);
                    }
                } else if (key.isWritable()) {
                    ByteBuffer buffer = (ByteBuffer) key.attachment();
                    SocketChannel sc = (SocketChannel) key.channel();
                    int write = sc.write(buffer);
                    System.out.println("实际写入字节:" + write);
                    if (!buffer.hasRemaining()) { // 写完了
                        key.interestOps(key.interestOps() - SelectionKey.OP_WRITE);
                        key.attach(null);
                    }
                }
            }
        }
    }
}
```

**注意：由于可能无法一次性全部写入buffer**

+ 非阻塞模式下，无法保证把 buffer 中所有数据都写入 channel，**因此需要追踪 write 方法的返回值（代表实际写入字节数）**
+ 用 selector 监听所有 channel 的可写事件，每个 channel 都需要一个 key 来跟踪 buffer**【sckey.attach(buffer)】**，但这样又会导致占用内存过多，因此使用 **二阶段策略**：当消息处理器**第一次写入消息时**，才将 channel 注册到 selector 上；selector 检查 channel 上的可写事件，**如果所有的数据写完了**，就取消 channel 的注册（如果不取消，每次缓冲区可写时，均会触发 write 事件，故应当只在缓冲区 **写不下时** 再关注可写事件）



### 单 Reactor 多线程模型

<img src="pictures/image-20210602190526990.png" alt="image-20210602190526990" style="zoom: 80%;" />



``` java
class BossEventLoop implements Runnable {

    private Selector selector;
    private WorkerEventLoop[] workers;
    private volatile boolean start = false;
    private AtomicInteger index = new AtomicInteger();

    public void register() throws IOException {
        if (!start) {
            ServerSocketChannel ssc = ServerSocketChannel.open();
            ssc.bind(new InetSocketAddress(8888));
            ssc.configureBlocking(false);
            selector = Selector.open();
            log.info("boss selector is {}", selector.toString());
            SelectionKey sscKey = ssc.register(selector, SelectionKey.OP_ACCEPT, null);
            this.workers = initWorkers();
            new Thread(this, "boss").start();
            log.info("boss start ....");
            start = true;
        }
    }

    private WorkerEventLoop[] initWorkers() {
        WorkerEventLoop[] eventLoops = new WorkerEventLoop[Runtime.getRuntime().availableProcessors()];
        log.debug("available processors cnt: {}", Runtime.getRuntime().availableProcessors());
        for (int i = 0; i < eventLoops.length; i++) {
            eventLoops[i] = new WorkerEventLoop(i);
        }
        return eventLoops;
    }

    @Override
    public void run() {
        while (true) {
            try {
                int cnt = selector.select();
                log.info("boss selectedkey cnt : {}", cnt);
                Iterator<SelectionKey> iter = selector.selectedKeys().iterator();
                while (iter.hasNext()) {
                    SelectionKey key = iter.next();
                    iter.remove();
                    if (key.isAcceptable()) {
                        ServerSocketChannel ssc = (ServerSocketChannel) key.channel();
                        SocketChannel sc = ssc.accept();
                        sc.configureBlocking(false);
                        log.debug("{} connected", sc.getRemoteAddress());
                        workers[index.getAndIncrement() % workers.length].register(sc);
                    }
                }
            } catch (Exception e) {
                e.printStackTrace();
            }
        }

    }
}

class WorkerEventLoop implements Runnable {

    private Selector selector;
    private volatile boolean start = false;
    private int index;

    public WorkerEventLoop(int index) {
        this.index = index;
    }

    public void register(SocketChannel sc) throws IOException {
        if (!start) {
            selector = Selector.open();
            log.info("worker-{} selector is {}", index, selector.toString());
            new Thread(this, "worker-" + index).start();
            log.info("worker-{} start ....", index);
            start = true;
        }
        try {
            // 防止 selector.select 阻塞时，sc.register(selector, SelectionKey.OP_READ, null) 失败
            selector.wakeup();
            sc.register(selector, SelectionKey.OP_READ, null);
            int cnt = selector.selectNow();
            log.info("worker-{} selectNow selectedkey cnt : {}", index, cnt);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    @Override
    public void run() {
        while (true) {
            try {
                int cnt = selector.select();
                log.info("worker-{} select selectedkey cnt : {}", index, cnt);
                Iterator<SelectionKey> iter = selector.selectedKeys().iterator();
                while (iter.hasNext()) {
                    SelectionKey key = iter.next();
                    iter.remove();
                    if (key.isReadable()) {
                        SocketChannel sc = (SocketChannel) key.channel();
                        ByteBuffer buffer = ByteBuffer.allocate(1024);
                        int read = sc.read(buffer);
                        try {
                            if (read == -1) {
                                key.cancel();
                                sc.close();
                            } else {
                                log.info("{} message: ", sc.getRemoteAddress());
                                System.out.println(byteBufferToString(buffer));
                            }
                        } catch (Exception e) {
                            e.printStackTrace();
                            key.cancel();
                            sc.close();
                        }
                    }
                }
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

}


class Client {
    public static void main(String[] args) {
        try {
            SocketChannel sc = SocketChannel.open();
            ByteBuffer source = ByteBuffer.allocate(32);
            sc.connect(new InetSocketAddress("localhost", 8888));
            source.put("flip before read\n".getBytes());
            source.flip();
            sc.write(source);
            source.clear();
            source.put("hello world\n".getBytes());
            source.flip();
            sc.write(source);
            sc.write(Charset.defaultCharset().encode("hello world\n"));
            sc.write(Charset.defaultCharset().encode("so slow\n"));
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

**注意：**

+ ServerSocketChannel.open()、Selector.open() 并 **不是单例模式**
+ selector 阻塞时对其 register 会报错，需要 **select.wakeup()** 使其此次 select 不阻塞



# Netty

## 1. 基本架构

### 多 Reactor 多线程模型

<img src="pictures/image-20210526203224405.png" alt="image-20210526203224405" style="zoom: 60%;" />



## 2.基本组件

### EventLoop

EventLoop 本质是 **一个线程 + 执行器（submit）**，维护了一个 Selector，里面有 run 方法处理 Channel 上源源不断的 io 事件，以及其它提交的普通任务。

EventLoopGroup 是**一组 EventLoop**，Channel 一般会调用 EventLoopGroup 的 register 方法来**绑定** 其中一个 EventLoop（负载均衡），**后续这个 Channel 上的 io 事件都由此 EventLoop 来处理**（保证了 io 事件处理时的线程安全）

```java
// 内部创建了两个 EventLoop, 每个 EventLoop 维护一个线程
DefaultEventLoopGroup group = new DefaultEventLoopGroup(2);
System.out.println(group.next());
System.out.println(group.next());
// 实现了 Iterable 接口提供遍历 EventLoop 的能力
for (EventExecutor eventLoop : group) {
    System.out.println(eventLoop);
}
```

**nio 工作线程** 和 **非nio工作线程** 共同处理 io 事件

```java
// 2个 非nio工作线程
DefaultEventLoopGroup normalWorkers = new DefaultEventLoopGroup(2);
new ServerBootstrap()
    // 2个 nio工作线程
    .group(new NioEventLoopGroup(1), new NioEventLoopGroup(2))
    .channel(NioServerSocketChannel.class)
    .childHandler(new ChannelInitializer<NioSocketChannel>() {
        @Override
        protected void initChannel(NioSocketChannel ch)  {
            ch.pipeline().addLast(new LoggingHandler(LogLevel.DEBUG));
            ch.pipeline().addLast(normalWorkers,"myhandler",
              new ChannelInboundHandlerAdapter() {
                @Override
                public void channelRead(ChannelHandlerContext ctx, Object msg) {
                    ByteBuf byteBuf = msg instanceof ByteBuf ? ((ByteBuf) msg) : null;
                    if (byteBuf != null) {
                        byte[] buf = new byte[16];
                        ByteBuf len = byteBuf.readBytes(buf, 0, byteBuf.readableBytes());
                        log.debug(new String(buf));
                    }
                }
            });
        }
    }).bind(8080).sync();
```

可以看到，nio group 和 非nio group 中的线程， **负载均衡地注册到 channel**（LoggingHandler 由 nio 工人执行，而我们自己的 handler 由非 nio 工人执行）



<img src="pictures/image-20210602201451579.png" alt="image-20210602201451579" style="zoom:80%;" />



NioEventLoop 处理 **普通任务** 和 **定时任务**

``` java
NioEventLoopGroup nioWorkers = new NioEventLoopGroup(2);
nioWorkers.execute(()->{
    log.debug("normal task...");
});
nioWorkers.scheduleAtFixedRate(() -> {
    log.debug("timed task...");
}, 0, 1, TimeUnit.SECONDS);
```



### Channel & **Future & Promise**

**channel** 主要的方法：

+ 关闭 channel : close() 同步、closeFuture() [ sync 同步、addListener 异步 ]
+ 响应数据：write() 写入、 writeAndFlush() 写入并刷出
+ 添加处理器：pipeline()



**Future** 主要的方法：

+ JDK Future 只能同步等待任务结束（或成功、或失败）才能得到结果

+ Netty Future 可以同步等待任务结束得到结果，也可以异步回调方式得到结果

+ Netty Promise  是可写的 Future，脱离了任务独立存在，作为两个线程间传递结果的**容器**

  | 功能/名称    | jdk Future                     | netty Future                                                 | Promise      |
  | ------------ | ------------------------------ | ------------------------------------------------------------ | ------------ |
  | cancel       | 取消任务                       | -                                                            | -            |
  | isCanceled   | 任务是否取消                   | -                                                            | -            |
  | isDone       | 任务是否完成，不能区分成功失败 | -                                                            | -            |
  | get          | 获取任务结果，阻塞等待         | -                                                            | -            |
  | getNow       | -                              | 获取任务结果，非阻塞，还未产生结果时返回 null                | -            |
  | await        | -                              | 等待任务结束，如果任务失败，不会抛异常，而是通过 isSuccess 判断 | -            |
  | sync         | -                              | 等待任务结束，如果任务失败，抛出异常                         | -            |
  | isSuccess    | -                              | 判断任务是否成功                                             | -            |
  | cause        | -                              | 获取失败信息，非阻塞，如果没有失败，返回null                 | -            |
  | addLinstener | -                              | 添加回调，异步接收结果                                       | -            |
  | setSuccess   | -                              | -                                                            | 设置成功结果 |
  | setFailure   | -                              | -                                                            | 设置失败结果 |



**ChannelFuture**

```java
ChannelFuture channelFuture = new Bootstrap().group().channel().handler().connect();
channelFuture.sync().channel().writeAndFlush("send res .");
```

注意：Bootstrap().connect() 是异步的，意味着不等连接建立，方法执行就返回了 ChannelFuture。ChannelFuture 作用是通过 channel() 方法来获取 Channel 对象



**CloseFuture**

``` java
ChannelFuture closeFuture = channel.closeFuture();
closeFuture.sync();  // 同步阻塞关闭
closeFuture.addListener(new ChannelFutureListener() {  // 异步关闭
    @Override
    public void operationComplete(ChannelFuture future) throws Exception {
        log.debug("处理关闭之后的操作");
        group.shutdownGracefully();
    }
});
```



**Promise 处理 成功/失败 任务**

``` java
DefaultEventLoop eventExecutors = new DefaultEventLoop();
DefaultPromise<Integer> promise = new DefaultPromise<>(eventExecutors);

eventExecutors.execute(()->{
    try {
        Thread.sleep(1000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    promise.setSuccess(10);
    promise.setFailure(e);
});

log.debug("{}",promise.getNow()); // 还没有结果
log.debug("{}",promise.get());  // 同步阻塞

// 异步回调
promise.addListener(future -> {
    log.debug("{}",future.getNow());
});

// 对于 promise.setFailure(e) 时，有 get / sync / await 三种方式处理任务失败：
// sync 直接抛出异常
// get 用 ExecutionException 包一层异常，再抛出 （set failure, java.lang.RuntimeException: error）
// await 不会抛出异常，只输出信息
```













### Handler & Pipeline

ChannelHandler 用来处理 Channel 上的各种事件，分为入站、出站两种。所有 ChannelHandler 被连成一串，就是 Pipeline

* 入站处理器： ChannelInboundHandlerAdapter 的子类，主要用来读取客户端数据，写回结果
* 出站处理器： ChannelOutboundHandlerAdapter 的子类，主要对写回结果进行加工

每个 Channel 是一个产品的加工车间，Pipeline 是车间中的流水线，ChannelHandler 就是流水线上的各道工序， ByteBuf 是原材料，经过很多工序的加工：先经过一道道入站工序，再经过一道道出站工序最终变成产品

<img src="pictures/image-20210526203301316.png" alt="image-20210526203301316" style="zoom: 60%;" />



**Handler 执行顺序**

``` java
new ServerBootstrap()
    .group(new NioEventLoopGroup())
    .channel(NioServerSocketChannel.class)
    .childHandler(new ChannelInitializer<NioSocketChannel>() {
        protected void initChannel(NioSocketChannel ch) {
            ch.pipeline().addLast(new ChannelInboundHandlerAdapter(){
                @Override
                public void channelRead(ChannelHandlerContext ctx, Object msg) {
                    System.out.println(1);
                    ctx.fireChannelRead(msg); // 1
                }
            });
            ch.pipeline().addLast(new ChannelInboundHandlerAdapter(){
                @Override
                public void channelRead(ChannelHandlerContext ctx, Object msg) {
                    System.out.println(2);
                    ctx.fireChannelRead(msg); // 2
                }
            });
            ch.pipeline().addLast(new ChannelInboundHandlerAdapter(){
                @Override
                public void channelRead(ChannelHandlerContext ctx, Object msg) {
                    System.out.println(3);
                    ctx.channel().write(msg); // 3
                }
            });
            ch.pipeline().addLast(new ChannelOutboundHandlerAdapter(){
                @Override
                public void write(ChannelHandlerContext ctx, Object msg, 
                                  ChannelPromise promise) {
                    System.out.println(4);
                    ctx.write(msg, promise); // 4
                }
            });
            ch.pipeline().addLast(new ChannelOutboundHandlerAdapter(){
                @Override
                public void write(ChannelHandlerContext ctx, Object msg, 
                                  ChannelPromise promise) {
                    System.out.println(5);
                    ctx.write(msg, promise); // 5
                }
            });
            ch.pipeline().addLast(new ChannelOutboundHandlerAdapter(){
                @Override
                public void write(ChannelHandlerContext ctx, Object msg, 
                                  ChannelPromise promise) {
                    System.out.println(6);
                    ctx.write(msg, promise); // 6
                }
            });
        }
    })
    .bind(8080);

```

服务端打印顺序： 1 2 3 6 5 4

ChannelInboundHandlerAdapter 是按照 addLast 的顺序执行的，而 ChannelOutboundHandlerAdapter 是按照 addLast 的逆序执行的

+ 入站处理器中，ctx.fireChannelRead(msg) 是 **调用下一个入站处理器**
+ 出站处理器中，ctx.channel().write(msg) 会 **固定从尾部开始触发** 出站处理器的执行
+ 出站处理器中，ctx.write(msg, promise) 的调用会 **从当前节点触发上一个出站处理器**

![image-20210606083203312](pictures/image-20210606083203312.png)





### ByteBuf

对比 NIO ByteBuffer 优势：

* 可以重用池中 ByteBuf 实例，节约内存，提高性能（池化）
* 读写指针分离，不需要像 ByteBuffer 一样切换读写模式
* 支持容器的自动扩容、支持链式调用
* 很多方法体现零拷贝，例如 slice、duplicate、CompositeByteBuf



**创建**

``` java
// 堆内存
ByteBuf buffer = ByteBufAllocator.DEFAULT.heapBuffer(10);

// 直接内存 （读写性能高，少一次内存复制）
// 直接内存不受JVM垃圾回收管理，注意主动释放
ByteBuf buffer = ByteBufAllocator.DEFAULT.directBuffer(10);

// handler 中用 ctx 创建 ByteBuf
ByteBuf response = ctx.alloc().buffer();

// 开启池化
// -Dio.netty.allocator.type={unpooled|pooled}
```



**组成：** 读指针、写指针、容量、最大可扩容量

<img src="pictures/image-20210606091758560.png" alt="image-20210606091758560" style="zoom:90%;" />



**写入：** 自动扩容、链式调用

| 方法名                                                       | 备注                                             |
| ------------------------------------------------------------ | ------------------------------------------------ |
| writeInt(int value)                                          | Big Endian，即 0x250，写入后 00 00 02 50（习惯） |
| writeIntLE(int value)                                        | Little Endian，即 0x250，写入后 50 02 00 00      |
| writeBoolean、writeByte、writeShort、writeLong、writeChar、<br />writeFloat、writeDouble、writeBytes、writeCharSequence |                                                  |
另外还有 set 开头的方法，也可以写入数据，但不会改变写指针位置。



**读取：** read、get（不会改变读指针）



**垃圾回收：** retain、release

由于 Netty 中有堆外内存的 ByteBuf 实现，**堆外内存**最好是手动来释放，而不是等 GC 垃圾回收

1. Netty 采用了 引用计数法 来控制回收内存，每个 ByteBuf 都实现了 ReferenceCounted 接口
   + 每个 ByteBuf 对象的初始计数为 1，调用 retain 方法计数加 1
   + 调用 release 方法计数减 1，如果计数为 0，ByteBuf 内存被回收，这时即使 ByteBuf 对象还在，其各个方法均无法正常使用

2. 手动管理原则：**谁是最后使用者，谁负责 release**
   + 入站 ByteBuf 处理原则
     + 对原始 ByteBuf 不做处理，调用 ctx.fireChannelRead(msg) 向后传递，这时无须 release
     + 如果不调用 ctx.fireChannelRead(msg) 向后传递，则需要 release
     + 将原始 ByteBuf 转换为其它类型的 Java 对象，此时 ByteBuf 就没用了，必须 release
     + 如果出现各种异常，ByteBuf 没有成功传递到下一个 ChannelHandler，必须 release
     + **默认情况下**，假设消息一直向后传，则由 tailCtx 节点会负责释放未处理消息（原始的 ByteBuf）
   + 出站 ByteBuf 处理原则，场景同上。不同的是，**默认情况下**，假设消息一直向前传，则由 headCtx 节点负责释放未处理消息（原始的 ByteBuf）
   + 异常处理原则：循环调用 release 直至返回 true



**零拷贝：** slice、duplicate、CompositeByteBuf

