---
layout: post
title: 【Netty 系列】netty 快速入门实战
categories: Netty
description: netty 快速入门实战
keywords: Netty, 高性能
---

Netty 是一个广受欢迎的异步事件驱动的 Java 开源网络应用程序框架，用于快速开发可维护的高性能协议服务器和客户端。

详细的介绍见：

[【Netty 系列】Netty 简介与架构原理浅析](2020-04-07-netty-introduce.md)

本文主要是从一个简单的例子开始，介绍如何快速搭建服务器和客户端的一个简单Demo。


## 依赖包导入

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency> 
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-lang3</artifactId>
</dependency>  		
<dependency>
    <groupId>io.netty</groupId>
    <artifactId>netty-all</artifactId>
    <version>4.1.12.Final</version>
</dependency>	

```

## 服务端代码

服务端代码已经添加相应的备注

```
public class Server {

	public static void main(String[] args) throws Exception {
		//1 创建线两个程组 
		//一个是用于处理服务器端接收客户端连接的
		//一个是进行网络通信的（网络读写的）
		EventLoopGroup pGroup = new NioEventLoopGroup();
		EventLoopGroup cGroup = new NioEventLoopGroup();
		
		//2 	创建辅助工具类，用于服务器通道的一系列配置
		ServerBootstrap b = new ServerBootstrap();
		b.group(pGroup, cGroup)		//	绑定俩个线程组
		.channel(NioServerSocketChannel.class)		//	指定NIO的模式
		.option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 3000)
//		.option(ChannelOption.TCP_NODELAY, true)	//	不延迟
		.option(ChannelOption.SO_BACKLOG, 32*1024)		//	设置tcp缓冲区
		.option(ChannelOption.SO_RCVBUF, 32*1024)	//	这是接收缓冲大小
//		.option(ChannelOption.RCVBUF_ALLOCATOR, AdaptiveRecvByteBufAllocator.DEFAULT) //	自动动态扩容
//		.option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT)	//	使用对象池，重用缓冲区		
		.childHandler(new ChannelInitializer<SocketChannel>() {
			@Override
			protected void initChannel(SocketChannel sc) throws Exception {
				//3 在这里配置具体数据接收方法的处理
				sc.pipeline().addLast(new ServerHandler());
			}
		});
		
		//4 进行绑定 
		ChannelFuture cf1 = b.bind(8765).sync();
		//ChannelFuture cf2 = b.bind(8764).sync();
		//5 等待关闭
		cf1.channel().closeFuture().sync();
		//cf2.channel().closeFuture().sync();
		pGroup.shutdownGracefully();
		cGroup.shutdownGracefully();
	}
}
```

对应的 ServerHandler 代码如下：

```
//ChannelHandlerAdapter
public class ServerHandler extends ChannelInboundHandlerAdapter {


	@Override
	public void channelActive(ChannelHandlerContext ctx) throws Exception {
		System.out.println("server channel active... ");
	}


	@Override
	public void channelRead(ChannelHandlerContext ctx, Object msg)
			throws Exception {
			ByteBuf buf = (ByteBuf) msg;
			byte[] req = new byte[buf.readableBytes()];
			buf.readBytes(req);
			String body = new String(req, "utf-8");
			System.out.println("Server :" + body );
			String response = "进行返回给客户端的响应：" + body ;
			
//			PooledByteBufAllocator pool = new PooledByteBufAllocator();
//			PooledByteBufAllocator.DEFAULT.heapBuffer();
//			ctx.writeAndFlush(Unpooled.wrappedBuffer(response.getBytes()));		
			
			ByteBuf directBuf = Unpooled.directBuffer(1024*10, 1024*32);
			directBuf.writeBytes(response.getBytes());
			ctx.writeAndFlush(directBuf);	

			//.addListener(ChannelFutureListener.CLOSE);
	}

	@Override
	public void channelReadComplete(ChannelHandlerContext ctx)
			throws Exception {
		System.out.println("读完了");
		ctx.flush();
	}

	@Override
	public void exceptionCaught(ChannelHandlerContext ctx, Throwable t)
			throws Exception {
		System.err.println(t);
		ctx.close();
	}

}
```


## 客户端的代码

客户端代码已经添加相应的备注

```
public class Client {

	public static void main(String[] args) throws Exception{
		
		EventLoopGroup group = new NioEventLoopGroup();
		Bootstrap b = new Bootstrap();
		b.group(group)
		.channel(NioSocketChannel.class)
//		.option(ChannelOption.TCP_NODELAY, true)	//不延迟
		.option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 3000)
//		.option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT) //内存池
//		.option(ChannelOption.RCVBUF_ALLOCATOR, AdaptiveRecvByteBufAllocator.DEFAULT)	//自动调整下一次缓冲区建立时分配的空间大小，避免内存的浪费
        
        .option(ChannelOption.SO_SNDBUF, 1024 * 1024)
        .option(ChannelOption.SO_RCVBUF, 1024 * 1024)
		.handler(new ChannelInitializer<SocketChannel>() {
			@Override
			protected void initChannel(SocketChannel sc) throws Exception {
				sc.pipeline().addLast(new ClientHandler());
			}
		});
		ChannelFuture cf1 = b.connect("127.0.0.1", 8765).syncUninterruptibly();

		// 开始测试代码
		cf1.channel().writeAndFlush(Unpooled.copiedBuffer("777".getBytes()));
		Thread.sleep(1000);
		cf1.channel().writeAndFlush(Unpooled.copiedBuffer("777".getBytes()));
//		cf1.channel().writeAndFlush(Unpooled.copiedBuffer("777".getBytes()));
//		cf1.channel().writeAndFlush(Unpooled.copiedBuffer("777".getBytes()));
		
		cf1.channel().closeFuture().sync();
		group.shutdownGracefully();
		
	}
}

```


对应的 ClientHandler 代码如下：

```
public class ClientHandler extends ChannelInboundHandlerAdapter {

	@Override
	public void channelActive(ChannelHandlerContext ctx) throws Exception {
	}

	@Override
	public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
		try {
			ByteBuf buf = (ByteBuf) msg;
			byte[] req = new byte[buf.readableBytes()];
			buf.readBytes(req);
			
			String body = new String(req, "utf-8");
			System.out.println("Client :" + body );
			String response = "收到服务器端的返回信息：" + body;
		} finally {
			ReferenceCountUtil.release(msg);
		}
	}

	@Override
	public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {

	}

	@Override
	public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause)
			throws Exception {
		ctx.close();
	}

}

```

## 自测试

分别启动服务器端和客户端：

服务器控制台输出内容如下：

```
Server :777
读完了
Server :777
读完了
```

客户端控制台输出内容如下：
```
Client :进行返回给客户端的响应：777
Client :进行返回给客户端的响应：777
```

成功实现了服务器和客户端的通讯。

