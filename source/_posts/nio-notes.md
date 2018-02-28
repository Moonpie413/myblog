---
title: Java NIO笔记
date: 2018-02-21 20:03:59
tags:
---

基本的Java NIO笔记, 为学习Netty做准备

<!-- more -->

四种IO模型
----------

主流的IO模型主要有如下四种

1. 同步阻塞 (Blocking IO)

  > 即标准的JavaIO实现, 进行IO操作时会阻塞当前线程

2. 同步非阻塞 (Non-Blocking IO)

  > 同步非阻塞IO即在同步阻塞的基础之上将socket设置为NONBLOCK,这样用户线程在发起IO操作之后可以立即返回, 
  > 但是用户线程需要不断轮询来请求数据

3. IO多路复用 (IO Multiplexing)

  **目前不是很明白这个解释, 将来再做整理**

  > 多路复用模型从流程上和同步阻塞的区别不大, 主要区别在于操作系统为用户提供了同时轮询多个IO句柄来查看是否有IO事件的接口,
  > 从根本上允许用户可以使用单个线程来管理多个IO句柄的问题

4. 异步IO (Asynchronous IO)

  > 不再需要用户去轮询IO事件, 当事件就绪时内核会开启一个独立的线程去执行IO操作
  > 此时用户线程可以直接读取内核线程准备好的数据

Java中的NIO
-----------

Java中的NIO实现主要通过 `Channel`, `Selector`, `Buffer` 这三个核心组件

### Selector

多路复用器, 可以将多个 `Channel` 注册到 `Selector` 上, 每次注册时将返回一个
`SelectionKey`, 代表注册的一个token, `SelectionKey` 由 `Selector` 维护, `Selector`
包含三个 `SelectionKey` 的集合

1. key set

  > 即所有向Selector注册的Channel的令牌信息

2. selected key set

  > 当有Channel向Selector注册的IO事件就绪的时候并且有select操作时对应的SelectionKey会被放到该集合中

3. cancelled-key

  > 当已注册的Channel取消后放到该集合

### Buffer

缓冲区是一个对象, 对数组进行了包装, 提供了对数据的结构化访问，而且还可以跟踪系统的读/写进程

#### 状态变量

* position

	追踪已经已经从缓冲区中读取/写入的值

* limit

	表明还有多少数据需要取出(在从缓冲区写入通道时)，或者还有多少空间可以放入数据(在从通道读入缓冲区时)

* capacity

	可以储存在缓冲区中的最大数据容量

**关系: position总是小于等于limit, 而limit不能大于capacity**

#### flip方法

由于缓冲区可以支持读写操作, `flip()` 方法通过修改状态变量的值来变更读/写模式

### Channel

`Channel` 是一个对象, 通过它读取或者写入数据, 就像标准IO中的流一样

在网络IO方面由 _ServerSocketChannel_ 和 _SocketChannel_ 实现, 分别表示服务端和客户端

#### 服务端ServerSocketChannel的一般流程

1. 获取一个ServerSocketChannel
2. 设置网络操作，这些参数主要是和TCP协议有关
3. 将ServerSocketChannel注册到Selector（多路复用器）
4. 将ServerSocketChannel和某个具体的地址绑定
5. 用户向多路复用器设置感兴趣的IO事件
6. 用户线程以阻塞或非阻塞方式轮询Selector来查看是否有就绪的IO事件
7. 用户针对不同的IO事件对Channel进行具体的IO操作

#### 客户端SocketChannel的一般流程

1. 获取一个SocketChannel
2. 设置Channel为非阻塞方式
3. 获取Selector
4. 将channel注册到Selector，并监听CONNECT事件
5. 调用channel的connect方法连接指定的服务器和端口
6. 如果连接成功则进行IO操作，如果没成功则轮询Selector处理CONNECT事件

服务端代码示例:

```java

public class NIOEchoServer {

  private static final ByteBuffer BUFFER = ByteBuffer.allocate(1024);

  private final Selector selector;

  public NIOEchoServer(int port) throws IOException {
    // 打开一个selector
    this.selector = Selector.open();
    // 打开一个socket channel
    ServerSocketChannel serverChannel = ServerSocketChannel.open();
    // 设置为非阻塞
    serverChannel.configureBlocking(false);
    // 获取一个socketServer
    ServerSocket serverSocket = serverChannel.socket();
    // 绑定端口
    serverSocket.bind(new InetSocketAddress(port));
    // 绑定channel到selector上
    // 将serverChannel注册到selector上并监听accept事件
    // 该事件在有客户端连接时被触发
    serverChannel.register(selector, SelectionKey.OP_ACCEPT);
    System.out.println(String.format("server start at %d", port));
  }

  public void listen() throws IOException {
    SocketChannel client;
    while (true) {
      // 阻塞直到有事件到来
      // num 表示监听到的事件总数
      int num = selector.select();
      // 响应的事件集合
      Set<SelectionKey> selectionKeys = selector.selectedKeys();
      Iterator<SelectionKey> iterator = selectionKeys.iterator();
      while (iterator.hasNext()) {
        SelectionKey key = iterator.next();
        // 响应连接事件
        if (key.isAcceptable()) {
          // 接受连接请求
          client = ((ServerSocketChannel) key.channel()).accept();
          System.out.println(String.format("accept from %s", client.getRemoteAddress()));
          // 异步处理
          client.configureBlocking(false);
          // 关键代码, 将该Channel注册到selector, 并且监听读事件
          client.register(selector, SelectionKey.OP_READ);
          // 移除原来的selectionKey
        } else if (key.isReadable()) {
          // 响应读事件
          SocketChannel sc = (SocketChannel) key.channel();
          while (true) {
            BUFFER.clear();
            if (sc.read(BUFFER) <= 0) {
              break;
            }
            String message = new String(BUFFER.array(), Charset.defaultCharset())
                .trim();
            System.out.println(message);
            BUFFER.flip();
            sc.write(BUFFER);
          }
        }
        // 处理完毕后从迭代器中移除
        iterator.remove();
      }
    }
  }

  public static void main(String[] args) throws IOException {
    new NIOEchoServer(8080).listen();
  }

}

```

客户端代码:

```java
public class NIOEchoClient {

  private final Selector selector;

  private final String sendMessage = "hello world\r\n";

  private ByteBuffer readBuffer = ByteBuffer.allocate(100);

  public NIOEchoClient(int port) throws IOException {
    // 创建一个selector
    this.selector = Selector.open();
    // 打开一个客户端channel
    SocketChannel clientChannel = SocketChannel.open();
    // 设置为非阻塞模式
    clientChannel.configureBlocking(false);
    // 尝试连接端口
    clientChannel.connect(new InetSocketAddress(port));
    // 注册并监听连接和读取事件
    clientChannel.register(selector, SelectionKey.OP_CONNECT);
  }

  public void send() throws IOException, InterruptedException {
    SocketChannel channel;
    while (true) {
      // 轮询监听
      selector.select();
      Set<SelectionKey> keys = selector.selectedKeys();
      Iterator<SelectionKey> iterator = keys.iterator();
      while (iterator.hasNext()) {
        SelectionKey key = iterator.next();
        // 因为这个selector中没有注册其他channel
        // 所以这个channel其实就是之前注册的clientChannel
        channel = (SocketChannel) key.channel();
        if (key.isConnectable()) {
          // 完成连接
          System.out.println(String.format("connect to %s", channel.getRemoteAddress()));
          channel.finishConnect();
          key.interestOps(OP_WRITE);
        } else if (key.isWritable()) {
          // 每隔两秒往服务器发送一条消息
          TimeUnit.SECONDS.sleep(2);
          channel.write(ByteBuffer.wrap(sendMessage.getBytes()));
          // 重新设置感兴趣的事件为读事件, 从服务器读取回显消息
          key.interestOps(OP_READ);
        } else if (key.isReadable()) {
          readBuffer.clear();
          channel.read(readBuffer);
          System.out.println(
              "echo from server: " + new String(readBuffer.array(), Charset.defaultCharset())
                  .trim());
          // 重新设置感兴趣的事件为写事件, 可以继续写入消息
          key.interestOps(OP_WRITE);
        }
        iterator.remove();
      }
    }

  }

  public static void main(String[] args) throws IOException, InterruptedException {
    new NIOEchoClient(8080).send();
  }
}
```

参考
----

* [Netty精粹之JAVA NIO开发需要知道的](https://my.oschina.net/andylucc/blog/614295)

* [Getting started with new I/O (NIO)](https://www.ibm.com/developerworks/java/tutorials/j-nio/j-nio.html)
