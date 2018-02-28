---
title: Netty笔记 (一)
date: 2018-02-20 20:53:31
tags: [Java, Netty, NIO]
---

学习了Java原生的NIO实现后阅读 《Netty In Action》的读书笔记
主要包括Netty中的各个组件

<!-- more -->

Channel接口
-----------

个人理解Netty中的Channel接口应该是对Java原生Api中的Socket封装, 因为它不仅提供了NIO的channel实现,
也提供了OIO的实现, 所以跟原生NIO中的Channel因该不是一个概念

EventLoop接口
-------------

EventLoop 是Netty的核心抽象, 用于处理连接的生命周期中发生的事情, 一下是EventLoop跟Channel及
EventLoopGroup的关系图

![EventLoop](/images/eventloop.png)

ChannelHandler接口
------------------

主要通过继承/复写ChannelHandler来实现自定义的入站, 出站逻辑

ChannelHandler有两个重要的子接口实现, ChannelInboundHandler和ChannelOutboundHandler, 
他们分别作为程序中数据的入站和出站的抽象

ChannelPipeline接口
-------------------

ChannelPipeline主要作为ChannelHandler的 **链式容器** , 即将多个ChannelHandler通过pipeline链接到一起

当ChannelHandler被添加到ChannelPipeline后会分配一个上下文对象ChannelHandlerContext, 通过它可以进行socket的读写操作

BootStrap引导
------------

BootStrap引导类主要是为网络层配置提供容器, 主要需要注意的是服务端使用 `ServerBootstrap`,
而客户端使用 `BootStrap` 类, 有点类似Socket中的 `ServerSocket` 和 `Socket` 的区别

因为服务端需要绑定(bind)端口, 而客户端则是连接(connect)服务端打开的端口, 所有有不同的实现

第二点区别是服务器通常会有两个EventLoopGroup(也可以只使用一个), 因为服务器不同于客户端, 它需要面向对个连接,
第一个EventLoopGroup监听端口, 并将接收到的连接交给第二个EventLoopGroup处理, 第二个EventLoopGroup则会为它分配
一个EventLoop来处理

![eventloopgroup](/images/eventloopgroup.png)

Echo Server示例代码:
---------------------------

```java
public class NettyServer {

  private final int port;

  // 入站handler
  private final ChannelInboundHandlerAdapter adapter;

  public NettyServer(int port, ChannelInboundHandlerAdapter adapter) {
    checkNotNull(adapter, "adapter can't be null");
    this.port = port;
    this.adapter = adapter;
  }

  public void run() throws InterruptedException {
    EventLoopGroup boosGroup = new NioEventLoopGroup();
    EventLoopGroup workerGroup = new NioEventLoopGroup();
    try {
      ServerBootstrap b = new ServerBootstrap();
      // 设置两个group
      b.group(boosGroup, workerGroup)
          .channel(NioServerSocketChannel.class)
          .childHandler(new ChannelInitializer<SocketChannel>() {
            @Override
            protected void initChannel(SocketChannel ch) {
              // 将自己实现的ChannelHandler添加到链式容器中
              // 该方法初始化后会将自己从容器中移除
              ch.pipeline().addLast(adapter);
            }
          })
          .option(ChannelOption.SO_BACKLOG, 128)
          .childOption(ChannelOption.SO_KEEPALIVE, true);

      ChannelFuture f = b.bind(port).sync();
      f.channel().closeFuture().sync();
    } finally {
      boosGroup.shutdownGracefully().sync();
      workerGroup.shutdownGracefully().sync();
    }
  }

  public static void main(String[] args) throws InterruptedException {
    new NettyServer(8080, new EchoServerHandler()).run();
  }

}

@Sharable
// 通过Sharable注解标注的表示该handler可以重用
class EchoServerHandler extends ChannelInboundHandlerAdapter {

  @Override
  public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    ByteBuf buf = (ByteBuf) msg;
    System.out.print("receive: " + (buf.toString(Charset.defaultCharset())));
    ctx.write(buf);
  }

  @Override
  public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
    ctx.writeAndFlush(Unpooled.EMPTY_BUFFER)
        // 添加关闭的监听器, 以便连接断开时关闭
        .addListener(ChannelFutureListener.CLOSE);
  }

  @Override
  public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
    cause.printStackTrace();
    ctx.close();
  }
}
```

Echo Client示例代码:
---------------------------

```java
public class NettyClient {

  private final int port;

  public NettyClient(int port) {
    this.port = port;
  }

  public void start() throws InterruptedException {
    EventLoopGroup group = new NioEventLoopGroup();
    try {
      Bootstrap b = new Bootstrap();
      // 客户端只需一个goup
      b.group(group)
          .channel(NioSocketChannel.class)
          .remoteAddress(new InetSocketAddress(port))
          .handler(new ChannelInitializer<SocketChannel>() {
            @Override
            protected void initChannel(SocketChannel ch) {
              ch.pipeline().addLast(new EchoClientHandler());
            }
          });
      ChannelFuture f = b.connect().sync();
      f.channel().closeFuture().sync();
    } finally {
      group.shutdownGracefully().sync();
    }
  }

  public static void main(String[] args) throws InterruptedException {
    new NettyClient(8080).start();
  }
}

class EchoClientHandler extends SimpleChannelInboundHandler<ByteBuf> {

  @Override
  public void channelActive(ChannelHandlerContext ctx) {
    // 建立连接成功后向服务器发送消息
    ctx.writeAndFlush(Unpooled.copiedBuffer("hello world", Charset.defaultCharset()));
  }

  @Override
  protected void channelRead0(ChannelHandlerContext ctx, ByteBuf msg) {
    // 回显服务器echo回来的消息
    System.out.println("Client Received: " + msg.toString(Charset.defaultCharset()));
  }

  @Override
  public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
    cause.printStackTrace();
    ctx.close();
  }
}
```

