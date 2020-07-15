---
layout: post
title: 【Netty 系列】基于 netty 的 Protobuf 序列化机制
categories: Netty
description: 基于 netty 的 Protobuf 序列化机制
keywords: Netty, 高性能, Marshalling
---

Protobuf 是一个高性能、易扩展的序列化框架，它的性能测试有关数据可以参看官方文档。通常在 TCP Socket通 讯（RPC调用）相关的应用中使用。Protobuf 是跨语言无歧义的 IDL。

其在效率、兼容性等方面非常出色。尤其是网络通信、通用数据交换等场景应该会优先选择 ProtoBuf。

## Protobuf 入门

ProtoBuf 是结构数据序列化方法，可简单类比于 XML，其具有以下特点：
- 语言无关、平台无关。即 ProtoBuf 支持 Java、C++、Python 等多种语言，支持多个平台
- 高效。即比 XML 更小（3 ~ 10倍）、更快（20 ~ 100倍）、更为简单
- 扩展性、兼容性好。你可以更新数据结构，而不影响和破坏原有的旧程序


官方定义：
> protocol buffers 是一种语言无关、平台无关、可扩展的序列化结构数据的方法，它可用于（数据）通信协议、数据存储等。
> Protocol Buffers 是一种灵活，高效，自动化机制的结构数据序列化方法－可类比 XML，但是比 XML 更小（3 ~ 10倍）、更快（20 ~ 100倍）、更为简单。
> 你可以定义数据的结构，然后使用特殊生成的源代码轻松的在各种数据流中使用各种语言进行编写和读取结构数据。你甚至可以更新数据结构，而不破坏由旧数据结构编译的已部署程序。

## 使用 ProtoBuf

对 ProtoBuf 的基本概念有了一定了解之后，我们来看看具体该如何使用 ProtoBuf。

第一步，创建 .proto 文件，定义数据结构，如下所示：

```
// 在 xxx.proto 文件中定义 Info message
syntax = "proto3";
package com.piter.protobuf;
option java_outer_classname = "Person";
message Info {
    string name = 1;
    int32 age = 2;
    string sex = 3;

    message Address {
      string province = 1;
      string city = 2;
      string area = 3;
    }

    Address address = 4;

    message Favorite{
      string what = 1;
    }

    repeated Favorite favorite = 5;
}
```

简单的介绍下上面的内容：
- syntax指定protobuf的版本，不写的话是2
- package就是生成文件所在的包了
- java_outer_classname就是生成的文件名了
- message一般就可以理解为类了
- 每个字段对应的数字的值就是对应数据的位置
- repeated就是重复，代表这是一个列表

第二版的字段支持写 require 和 optional，第三版好像不支持了


第二步 使用工具生成对应的类即可

```
$ protoc --java_out=${OUTPUT_DIR} path/to/your/proto/file
```

## netty 集成  protobuf

熟悉了如何使用 ProtoBuf 之后，下面我们看看如何在 netty 中集成 protobuf。

### 创建 request 以及 response proto 文件

创建 request proto 文件，添加我们想要发送的数据。

```
syntax = "proto3";

option java_package = "com.bfxy.netty.protobuf";
option java_outer_classname = "RequestModule";

message Request {

	string id = 1;
	
	int32 sequence = 2;
	
	string data = 3;
	
}
```

创建 response proto 文件，添加我们想要发送的数据。
```
syntax = "proto3";

option java_package = "com.bfxy.netty.protobuf";
option java_outer_classname = "ResponseModule";

message Response {

	string id = 1;
	
	int32 code = 2;
	
	string desc = 3;

}

```

然后采用工具，生成这两个proto文件对应的java类。这里不再列出来了。

### 创建 server 端

server 端 代码如下：

```
public class Server {
    public void bind(int port) throws Exception {
		// 配置服务端的NIO线程组
		EventLoopGroup bossGroup = new NioEventLoopGroup();
		EventLoopGroup workerGroup = new NioEventLoopGroup();
		try {
		    ServerBootstrap b = new ServerBootstrap();
		    b.group(bossGroup, workerGroup)
			    .channel(NioServerSocketChannel.class)
			    .option(ChannelOption.SO_BACKLOG, 100)
			    .handler(new LoggingHandler(LogLevel.INFO))
			    .childHandler(new ChannelInitializer<SocketChannel>() {
					@Override
					public void initChannel(SocketChannel ch) {
						ch.pipeline().addLast(new ProtobufVarint32FrameDecoder());
						ch.pipeline().addLast(new ProtobufDecoder(RequestModule.Request.getDefaultInstance()));
						ch.pipeline().addLast(new ProtobufVarint32LengthFieldPrepender());
						ch.pipeline().addLast(new ProtobufEncoder());
						ch.pipeline().addLast(new ServerHandler());
					}
			    });
	
		    // 绑定端口，同步等待成功
		    ChannelFuture f = b.bind(port).sync();
		    System.out.println("Server Start .. ");
		    // 等待服务端监听端口关闭
		    f.channel().closeFuture().sync();
		} finally {
		    // 优雅退出，释放线程池资源
		    bossGroup.shutdownGracefully();
		    workerGroup.shutdownGracefully();
		}
    }

    public static void main(String[] args) throws Exception {
		int port = 8080;
		if (args != null && args.length > 0) {
		    try {
		    	port = Integer.valueOf(args[0]);
		    } catch (NumberFormatException e) {
			// 采用默认值
		    }
		}
		new Server().bind(port);
    }
}
```

其中关键的代码如下：Server 首先向 pipeline 中添加 ProtobufVarint32FrameDecoder， 它主要用于半包处理，然后添加 ProtobufDecoder 解码器，它的参数是com.google.protobuf.MessageLite，实际上是告诉ProtobufDecoder需要解码的目标类是什么，否则仅仅从字节数组是无法知道要解码的目标类型信息的。

```
ch.pipeline().addLast(new ProtobufVarint32FrameDecoder());
ch.pipeline().addLast(new ProtobufDecoder(RequestModule.Request.getDefaultInstance()));
ch.pipeline().addLast(new ProtobufVarint32LengthFieldPrepender());
ch.pipeline().addLast(new ProtobufEncoder());
ch.pipeline().addLast(new ServerHandler());
```

服务器响应处理类 ServerHandler 代码如下，他继承 ChannelInboundHandlerAdapter ，并重写了 channelRead 函数：

```
public class ServerHandler extends ChannelInboundHandlerAdapter {

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    	RequestModule.Request request = (RequestModule.Request)msg;
    	System.err.println("服务端：" + request.getId() + "," + request.getSequence() + "," + request.getData());
    	ctx.writeAndFlush(createResponse(request.getId(), request.getSequence()));
    }
    
    private ResponseModule.Response createResponse(String id, int seq) {
		return ResponseModule.Response.newBuilder()
				.setId(id)
				.setCode(seq)
				.setDesc("响应报文")
				.build();
	}
    
	@Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
		cause.printStackTrace();
		ctx.close();
    }
}
```


### Client端 

Client端 代码如下：

```
public class Client {

    public void connect(int port, String host) throws Exception {
		// 配置客户端NIO线程组
		EventLoopGroup group = new NioEventLoopGroup();
		try {
		    Bootstrap b = new Bootstrap();
		    b.group(group).channel(NioSocketChannel.class)
			    .option(ChannelOption.TCP_NODELAY, true)
			    .handler(new ChannelInitializer<SocketChannel>() {
					@Override
					public void initChannel(SocketChannel ch) throws Exception {
						ch.pipeline().addLast(new ProtobufVarint32FrameDecoder());
						ch.pipeline().addLast(new ProtobufDecoder(ResponseModule.Response.getDefaultInstance()));
						ch.pipeline().addLast(new ProtobufVarint32LengthFieldPrepender());
						ch.pipeline().addLast(new ProtobufEncoder());
						ch.pipeline().addLast(new ClientHandler());
					}
			    });
	
		    // 发起异步连接操作
		    ChannelFuture f = b.connect(host, port).sync();
		    System.out.println("Client Start .. ");
		    // 当代客户端链路关闭
		    f.channel().closeFuture().sync();
		} finally {
		    // 优雅退出，释放NIO线程组
		    group.shutdownGracefully();
		}
    }

    /**
     * @param args
     * @throws Exception
     */
    public static void main(String[] args) throws Exception {
		int port = 8080;
		if (args != null && args.length > 0) {
		    try {
		    	port = Integer.valueOf(args[0]);
		    } catch (NumberFormatException e) {
			// 采用默认值
		    }
		}
			new Client().connect(port, "127.0.0.1");
	    }
}
```

关键代码是如下几句，Client 首先向 pipeline 中添加 ProtobufVarint32FrameDecoder， 它主要用于半包处理，然后添加 ProtobufDecoder 解码器，它的参数是需要解码的类型。

```
ch.pipeline().addLast(new ProtobufVarint32FrameDecoder());
ch.pipeline().addLast(new ProtobufDecoder(ResponseModule.Response.getDefaultInstance()));
ch.pipeline().addLast(new ProtobufVarint32LengthFieldPrepender());
ch.pipeline().addLast(new ProtobufEncoder());
ch.pipeline().addLast(new ClientHandler());
```

其中 ClientHandler 代码如下，由于ProtobufDecoder 会对消息自动解码，所以这里我们可以直接写业务逻辑：

```
public class ClientHandler extends ChannelInboundHandlerAdapter {

    /**
     * Creates a client-side handler.
     */
    public ClientHandler() {
    }

    @Override
    public void channelActive(ChannelHandlerContext ctx) {
    	System.err.println("客户端通道激活");
    	for(int i =0; i< 100; i ++) {
    		ctx.writeAndFlush(createRequest(i));
    	}
    }
    
    private Request createRequest(int i) {
    	return RequestModule.Request.newBuilder().setId("主键" + i)
    	.setSequence(i)
    	.setData("数据内容" + i)
    	.build();
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    	try {
        	ResponseModule.Response response = (ResponseModule.Response)msg;
        	System.err.println("客户端：" +  response.getId() + "," + response.getCode() + "," + response.getDesc());
		} finally {
			ReferenceCountUtil.release(msg);
		}
    }

    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
    	ctx.flush();
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
		cause.printStackTrace();
		ctx.close();
    }
}
```

## 自测试

分别启动服务器端和客户端，观察控制台输出

服务器端控制台输出内容如下：
```
服务端：主键0,0,数据内容0
服务端：主键1,1,数据内容1
服务端：主键2,2,数据内容2
服务端：主键3,3,数据内容3
服务端：主键4,4,数据内容4
服务端：主键5,5,数据内容5
服务端：主键6,6,数据内容6
服务端：主键7,7,数据内容7
服务端：主键8,8,数据内容8
服务端：主键9,9,数据内容9
```

客户端端控制台输出内容如下：
```
客户端：主键0,0,响应报文
客户端：主键1,1,响应报文
客户端：主键2,2,响应报文
客户端：主键3,3,响应报文
客户端：主键4,4,响应报文
客户端：主键5,5,响应报文
客户端：主键6,6,响应报文
客户端：主键7,7,响应报文
客户端：主键8,8,响应报文
客户端：主键9,9,响应报文
```

可见，我们成功集成了 Protobuf。


## Protobuf的使用注意事项

ProtobufDecoder仅仅负责解码，它不支持读半包。因此在ProtobufDecoder的前面，一定要有能够处理半包消息的解码器。
有3种方式可以选择：
1.使用Netty提供的ProtobufVarint32FrameDecoder，它可以处理半包消息；
2.继承Netty提供的通用半包解码器LengthFieldBasedFrameDecoder;
3.继承ByteToMessageDecoder类，自己处理半包消息。

如果只使用ProtobufDecoder解码器，而忽略对半包消息的处理，程序没法正常工作。

