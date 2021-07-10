# Netty

## 简介

Netty 是一个基于 NIO 的 client-server 框架，使用它可以快速简单的开发网络应用程序。

1. 它极大地简化并简化了 TCP 和 UDP 套接字服务器等网络编程,并且性能以及安全性等很多方面甚至都要更好。
2. 支持多种协议如 FTP，SMTP，HTTP 以及各种二进制和基于文本的传统协议。

### 架构

![image-20210226184754000](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210226184754000.png)

- 通用的传输 API （TCP/ UDP）
- 多种网络协议（HTTP/WebSocket）
- 基于事件驱动的 IO 模型
- 超高性能的零拷贝机制

## 实现 HTTP Server

### Netty 编解码器

通过 Netty 处理 HTTP 请求，需要先进行编解码。在传输数据用的 ByteBuf 和 请求、响应对象（例如HttpRequest 和 HttpContent ）之间相互转换。

Netty 自带 4 个常用的编码器：

- HttpRequestEncoder：将 HttpRequest 和 HttpContent 编码为 ByteBuf
- HttpRequestDecoder：将 ByteBuf 解码为 HttpRequest 和 HttpContent
- HttpResponseEncoder：将 HttpResponse 和 HttpContent 编码为 ByteBuf
- HttpResponseDecoder：将 ByteBuf 解码为 HttpResponse 和 HttpContent

## Netty 核心组件

![image-20210228171914370](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210228171914370.png)

### ByteBuf 

网络通信中数据始终通过字节流传输。 ByteBuf 是 Netty 提供的一个字节容器，内部是一个字节数组。可以看做是 Netty 对 Java NIO 中 ByteBuffer 的封装和抽象。

### Bootstrap 和 ServerBootstrap

Bootstrap 作为客户端的启动引导类，具体可以使用如下：

```java
        EventLoopGroup group = new NioEventLoopGroup();
        try {
            //创建客户端启动引导/辅助类：Bootstrap
            Bootstrap b = new Bootstrap();
            //指定线程模型
            b.group(group).
                    ......
            // 尝试建立连接
            ChannelFuture f = b.connect(host, port).sync();
            f.channel().closeFuture().sync();
        } finally {
            // 优雅关闭相关线程组资源
            group.shutdownGracefully();
        }
```

ServerBootstrap 作为服务器端的引导类，具体可以使用如下：

```java
        // 1.bossGroup 用于接收连接，workerGroup 用于具体的处理
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            //2.创建服务端启动引导/辅助类：ServerBootstrap
            ServerBootstrap b = new ServerBootstrap();
            //3.给引导类配置两大线程组,确定了线程模型
            b.group(bossGroup, workerGroup).
                   ......
            // 6.绑定端口
            ChannelFuture f = b.bind(port).sync();
            // 等待连接关闭
            f.channel().closeFuture().sync();
        } finally {
            //7.优雅关闭相关线程组资源
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
```

- `Bootstrap` 通常使用 `connet()` 方法连接到远程的主机和端口，作为一个 Netty TCP 协议通信中的客户端。另外，`Bootstrap` 也可以通过 `bind()` 方法绑定本地的一个端口，作为 UDP 协议通信中的一端。
- `ServerBootstrap` 通常使用 `bind()` 方法绑定本地的端口上，然后等待客户端的连接。
- `Bootstrap` 只需要配置一个线程组— `EventLoopGroup` ,而 `ServerBootstrap`需要配置两个线程组— `EventLoopGroup` ，一个用于接收连接，一个用于具体的 IO 处理。

### Channel

Channel 接口是 Netty 对网络操作的抽象类，通过 Channel 可以进行 I/O 操作。

一旦客户端成功连接服务端，就会新建一个 `Channel` 同该用户端进行绑定，示例代码如下：

```java
   //  通过 Bootstrap 的 connect 方法连接到服务端
   public Channel doConnect(InetSocketAddress inetSocketAddress) {
        CompletableFuture<Channel> completableFuture = new CompletableFuture<>();
        bootstrap.connect(inetSocketAddress).addListener((ChannelFutureListener) future -> {
            if (future.isSuccess()) {
                completableFuture.complete(future.channel());
            } else {
                throw new IllegalStateException();
            }
        });
        return completableFuture.get();
    }
```

常用的实现类为

- NioServerSocketChannel
- NioSocketChannel

可以分别于 NIO 编程模型中 ServerSocket 和 Socket 对应

### EventLoop

EventLoop 定义了 Netty 的核心抽象，用于处理连接的生命周期中发生的事件。主要作用实际在于**监听网络事件并调用事件处理器进行相关 I/O 操作**

#### Channel 与 EventLoop

Channel 为 Netty 网络读写操作的抽象类，EventLoop 负责处理注册到其上的 Channel 的I/O操作。两者配合完成 I/O

#### EventloopGroup 和 EventLoop

`EventLoopGroup` 包含多个 `EventLoop`（每一个 `EventLoop` 通常内部包含一个线程），它管理着所有的 `EventLoop` 的生命周期。并且，**`EventLoop` 处理的 I/O 事件都将在它专有的 `Thread` 上被处理，即 `Thread` 和 `EventLoop` 属于 1 : 1 的关系，从而保证线程安全。**

![image-20210228191451230](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210228191451230.png)

### ChannelHandler

`ChannelHandler` 是消息的具体处理器，主要负责**处理**客户端/服务端接收和发送的数据。

当 `Channel` 被创建时，它会被自动地分配到它专属的 `ChannelPipeline`。 一个`Channel`包含一个 `ChannelPipeline`。 `ChannelPipeline` 为 `ChannelHandler` 的链，一个 pipeline 上可以有多个 `ChannelHandler`。

通过 `addLast()` 方法添加一个或者多个`ChannelHandler` （*一个数据或者事件可能会被多个 Handler 处理*） 。当一个 `ChannelHandler` 处理完之后就将数据交给下一个 `ChannelHandler` 。

`ChannelHandler` 添加到的 `ChannelPipeline` 得到一个 `ChannelHandlerContext`，它代表一个 `ChannelHandler` 和 `ChannelPipeline` 之间的“绑定”。 `ChannelPipeline` 通过 `ChannelHandlerContext`来间接管理 `ChannelHandler` 。

#### ChannelFuture

```java
public interface ChannelFuture extends Future<Void> {
    Channel channel();

    ChannelFuture addListener(GenericFutureListener<? extends Future<? super Void>> var1);
     ......

    ChannelFuture sync() throws InterruptedException;
}
```

Netty 是异步非阻塞的，所有的 I/O 操作都为异步的。通过 `ChannelFuture` 接口的 `addListener()` 方法注册一个 `ChannelFutureListener`，当操作执行成功或者失败时，监听就会自动触发返回结果。

