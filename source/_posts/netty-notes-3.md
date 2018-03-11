---
title: netty笔记 (三)
date: 2018-03-10 21:42:24
tags:
---

Netty笔记第三部分

<!-- more -->

编解码器
---------------

在处理基于流的传输(如TCP/IP)时需要注意netty获取到的数据会暂存在buffer中, 但是在buffer中存储的顺序
不一定是真实传输的顺序

netty可以通过实现编码器/解码器来处理字节数组, 将它们编码/解码成想要的数据类型

如以下的解码器将字节编码为时间对象

```java
package io.netty.example.time;

public class TimeDecoder extends ByteToMessageDecoder {
  @Override
  protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {
		if (in.readableBytes() < 4) {
			// 凑够四个字节即一个int才可以组成时间
			return;
    }
    out.add(new UnixTime(in.readUnsignedInt()));
  }
}
```

Http服务器
-----------

通过netty提供的http编解码器, 可以轻松编写出http服务器而不用关心底层实现

**例子:**

```java
public class HttpServerInitializer extends ChannelInitializer<SocketChannel> {

  private static final byte[] CONTENT = {'H', 'e', 'l', 'l', 'o', ' ', 'W', 'o', 'r', 'l', 'd'};

  private static final AsciiString CONTENT_TYPE = AsciiString.cached("Content-Type");
  private static final AsciiString CONTENT_LENGTH = AsciiString.cached("Content-Length");
  private static final AsciiString CONNECTION = AsciiString.cached("Connection");
  private static final AsciiString KEEP_ALIVE = AsciiString.cached("keep-alive");

  private static final boolean SSL = System.getProperty("ssl") != null;

  private static SslContext SSL_CONTEXT;

  static {
    SelfSignedCertificate ssc;
    if (SSL) {
      try {
        ssc = new SelfSignedCertificate();
        SSL_CONTEXT = SslContextBuilder.forServer(ssc.certificate(), ssc.privateKey()).build();
      } catch (CertificateException | SSLException e) {
        e.printStackTrace();
      }
    }
  }

  @Override
  protected void initChannel(SocketChannel ch) {
    ChannelPipeline pipeline = ch.pipeline();
    // 如果支持HTTPS 则添加https编解码器
    if (SSL) {
      pipeline.addLast(SSL_CONTEXT.newHandler(ch.alloc()));
    }
    // netty内置的http编解码器, 同时实现了
    // HttpRequestDecoder 和 HttpResponseEncoder
    pipeline.addLast(new HttpServerCodec());
    // 自己实现的服务器实现逻辑, 由于放在解码器之后, 所以处理的已经是
    // 经过解码器处理的httpObject对象
    pipeline.addLast(new HttpServerHandler());
  }

  private static class HttpServerHandler extends ChannelInboundHandlerAdapter {

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
      if (msg instanceof HttpRequest) {
        HttpRequest req = (HttpRequest) msg;

        boolean keepAlive = HttpUtil.isKeepAlive(req);
        FullHttpResponse response = new DefaultFullHttpResponse(HTTP_1_1, OK,
            Unpooled.wrappedBuffer(CONTENT));
        response.headers().set(CONTENT_TYPE, "text/plain");
        response.headers().setInt(CONTENT_LENGTH, response.content().readableBytes());

        if (!keepAlive) {
          ctx.write(response)
              .addListener(ChannelFutureListener.CLOSE);
        } else {
          response.headers().set(CONNECTION, KEEP_ALIVE);
          ctx.write(response);
        }
      }
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
      cause.printStackTrace();
      ctx.close();
    }

    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) {
      ctx.flush();
    }

  }

}
```

Http客户端
----------

同理, http客户端的例子

```java
public class HttpClientInitializer extends ChannelInitializer<SocketChannel> {

  private final String url;

  public HttpClientInitializer(String url) {
    this.url = url;
  }

  @Override
  protected void initChannel(SocketChannel ch) {
    ChannelPipeline pipeline = ch.pipeline();
    pipeline.addLast(new HttpClientCodec());
    pipeline.addLast(new HttpClientHandler(url));
  }

  private static class HttpClientHandler extends ChannelInboundHandlerAdapter {

    private final String url;

    private HttpClientHandler(String url) {
      this.url = url;
    }

    @Override
    public void channelActive(ChannelHandlerContext ctx) {
      FullHttpRequest request = new DefaultFullHttpRequest(HTTP_1_1, GET, url);

      // 不设置该行若不手动关闭context则客户端不会自动关闭
      // 相当于服务端关闭
      request.headers().set(HttpHeaderNames.CONNECTION, HttpHeaderValues.CLOSE);
      request.headers().set(HttpHeaderNames.HOST, url);
      ctx.writeAndFlush(request);
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
      if (msg instanceof HttpResponse) {
        HttpResponseStatus status = ((HttpResponse) msg).status();
        if (status.code() == 200) {
          System.out.println("request successful!");
        }
      }
      if (msg instanceof HttpContent) {
        ByteBuf buf = ((HttpContent) msg).content();
        System.out.println(buf.toString(CharsetUtil.UTF_8));
        if (msg instanceof LastHttpContent) {
          ReferenceCountUtil.release(buf);
          ctx.close();
        }
      }
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
      cause.printStackTrace();
      ctx.close();
    }
  }

}
```

遇到的问题
----------

在编写客户端的时候发现客户端在执行完后并没有关闭, 经过一下午的研究后基本搞懂了netty的关闭过程

netty的标准模式中经常会出现下面的代码

```java
ChannelFuture f = b.connect().sync();
f.channel().closeFuture().sync();
```

其中`sync()` 表示该方法同步执行, 通过阅读源码发现`sync()`中会调用当前线程的`wait()`以此来实现同步
原先http客户端运行完没有自动关闭就是因为主线程一直等待, 那么主线程何时会停止等待呢

官方文档对 `closeFuture()` 的注释如下:

```java
/**
* Returns the {@link ChannelFuture} which will be notified when this
* * channel is closed.  This method always returns the same future instance.
*/
ChannelFuture closeFuture();
```

也就是说连接结束或异常的时候, 主线程才能继续执行

### 解决方法

第一次尝试是在 `HttpClientHandler` 的 `channelRead()` 中调用 `ctx.close()`, 这确实可行, 客户端
运行完后就关闭了, 但是又想了一下为什么之前的echoClient不调用该方法也能自动关闭?之后google了一下又看了
netty的官方 `HttpSnoopClient` 示例, 才想起来服务端有这样的代码:

```java
if (!keepAlive) {
  ctx.write(response)
      .addListener(ChannelFutureListener.CLOSE);
} else {
  response.headers().set(CONNECTION, KEEP_ALIVE);
  ctx.write(response);
}
```

也就是说如果客户端是长连接那服务端不会关闭这个连接, 对比了一下自己的实现和官方的实现, 果然是自己没有设置关闭长连接,
加入了如下代码后就正常了

```java
request.headers().set(HttpHeaderNames.CONNECTION, HttpHeaderValues.CLOSE);
```
