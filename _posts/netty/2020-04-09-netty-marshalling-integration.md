---
layout: post
title: 【Netty 系列】基于 netty 的 Marshalling 序列化机制
categories: Netty
description: 基于 netty 的 Marshalling 序列化机制
keywords: Netty, 高性能, Marshalling
---

JBoss Marshalling 是一个 Java 对象序列化包，对 JDK 默认的序列化框架进行了优化，但又保持跟 Java.io.Serializable 接口的兼容

同时增加了一些可调的参数和附件的特性，这些参数和附加的特性， 这些参数和特性可通过工厂类进行配置。

## 导入相关jar包

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
<dependency>
    <groupId>org.jboss.marshalling</groupId>
    <artifactId>jboss-marshalling</artifactId>
    <version>1.3.0.CR9</version>
</dependency>        	
<dependency>
    <groupId>org.jboss.marshalling</groupId>
    <artifactId>jboss-marshalling-serial</artifactId>
    <version>1.3.0.CR9</version>
</dependency>			
```


## 创建序列化传输的类

#### RequestData 请求对象

RequestData 为发动给服务器的数据

```
import java.io.Serializable;

public class RequestData implements Serializable {

	private static final long serialVersionUID = 7359175860641122157L;

	private String id;
	
	private String name;
	
	private String requestMessage;
	
	private byte[] attachment;

	public String getId() {
		return id;
	}

	public void setId(String id) {
		this.id = id;
	}

	public String getName() {
		return name;
	}

	public void setName(String name) {
		this.name = name;
	}

	public String getRequestMessage() {
		return requestMessage;
	}

	public void setRequestMessage(String requestMessage) {
		this.requestMessage = requestMessage;
	}

	public byte[] getAttachment() {
		return attachment;
	}

	public void setAttachment(byte[] attachment) {
		this.attachment = attachment;
	}
	
}
```

#### ResponseData 响应对象

ResponseData 为服务器返回给客户端的数据

```
import java.io.Serializable;

public class ResponseData implements Serializable {

	private static final long serialVersionUID = -6231852018644360658L;

	private String id;
	
	private String name;
	
	private String responseMessage;

	public String getId() {
		return id;
	}

	public void setId(String id) {
		this.id = id;
	}

	public String getName() {
		return name;
	}

	public void setName(String name) {
		this.name = name;
	}

	public String getResponseMessage() {
		return responseMessage;
	}

	public void setResponseMessage(String responseMessage) {
		this.responseMessage = responseMessage;
	}
	
}
```


## 创建 MarshallingEncoder 和 MarshallingDecoder 的工厂类

已经添加相应的注释信息

```
public final class MarshallingCodeCFactory {

    /**
     * 创建Jboss Marshalling解码器MarshallingDecoder
     * @return MarshallingDecoder
     */
    public static MarshallingDecoder buildMarshallingDecoder() {
    	//首先通过Marshalling工具类的方法获取Marshalling实例对象 参数serial标识创建的是java序列化工厂对象。
		final MarshallerFactory marshallerFactory = Marshalling.getProvidedMarshallerFactory("serial");
		//创建了MarshallingConfiguration对象，配置了版本号为5 
		final MarshallingConfiguration configuration = new MarshallingConfiguration();
		configuration.setVersion(5);
		//根据marshallerFactory和configuration创建provider
		UnmarshallerProvider provider = new DefaultUnmarshallerProvider(marshallerFactory, configuration);
		//构建Netty的MarshallingDecoder对象，俩个参数分别为provider和单个消息序列化后的最大长度
		MarshallingDecoder decoder = new MarshallingDecoder(provider, 1024 * 1024 * 3);
		return decoder;
    }

    /**
     * 创建Jboss Marshalling编码器MarshallingEncoder
     * @return MarshallingEncoder
     */
    public static MarshallingEncoder buildMarshallingEncoder() {
		final MarshallerFactory marshallerFactory = Marshalling.getProvidedMarshallerFactory("serial");
		final MarshallingConfiguration configuration = new MarshallingConfiguration();
		configuration.setVersion(5);
		MarshallerProvider provider = new DefaultMarshallerProvider(marshallerFactory, configuration);
		//构建Netty的MarshallingEncoder对象，MarshallingEncoder用于实现序列化接口的POJO对象序列化为二进制数组
		MarshallingEncoder encoder = new MarshallingEncoder(provider);
		return encoder;
    }
}

```

## Server端

```
public class Server {

	public static void main(String[] args) throws InterruptedException {
		
		EventLoopGroup bGroup = new NioEventLoopGroup(1);
		EventLoopGroup wGroup = new NioEventLoopGroup();
		
		ServerBootstrap sb = new ServerBootstrap();
		sb.group(bGroup, wGroup)
		.channel(NioServerSocketChannel.class)
		.option(ChannelOption.SO_BACKLOG, 1024)
		.childHandler(new ChannelInitializer<SocketChannel>() {
			@Override
			protected void initChannel(SocketChannel sc) throws Exception {
				sc.pipeline().addLast(MarshallingCodeCFactory.buildMarshallingDecoder());
				sc.pipeline().addLast(MarshallingCodeCFactory.buildMarshallingEncoder());
				sc.pipeline().addLast(new ServerHandler());
			}
		});
		
		ChannelFuture cf = sb.bind(8765).sync();
		
		cf.channel().closeFuture().sync();
		bGroup.shutdownGracefully();
		wGroup.shutdownGracefully();
	}
}
```

对应的 ServerHandler 如下：

```
public class ServerHandler extends ChannelInboundHandlerAdapter {

	@Override
	public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
		// 接受request请求 并进行业务处理
		RequestData rd = (RequestData)msg;
		System.err.println("id: " + rd.getId() + ", name: " + rd.getName() + ", requestMessage: " + rd.getRequestMessage());
		
		byte[] attachment = GzipUtils.ungzip(rd.getAttachment());
		
		String path = System.getProperty("user.dir")+ File.separatorChar + "receive" + File.separatorChar + "001.jpg";
		
		FileOutputStream fos = new FileOutputStream(path);
		fos.write(attachment);
		fos.close();
		
		//	回送相应数据
		ResponseData responseData = new ResponseData();
		responseData.setId("response " + rd.getId());
		responseData.setId("response " + rd.getName());
		responseData.setResponseMessage("响应信息");
		
		ctx.writeAndFlush(responseData);
	}
}
```

## Client端

```
public class Client {

	public static void main(String[] args) throws Exception {
		
		EventLoopGroup wGroup = new NioEventLoopGroup();
		
		Bootstrap b = new Bootstrap();
		b.group(wGroup)
		.channel(NioSocketChannel.class)
		.handler(new ChannelInitializer<SocketChannel>() {
			@Override
			protected void initChannel(SocketChannel sc) throws Exception {
				sc.pipeline().addLast(MarshallingCodeCFactory.buildMarshallingDecoder());
				sc.pipeline().addLast(MarshallingCodeCFactory.buildMarshallingEncoder());
				sc.pipeline().addLast(new ClientHandler());
			}
		});
		
		ChannelFuture cf = b.connect("127.0.0.1", 8765).sync();
		Channel channel = cf.channel();
		
		for(int i =0; i < 10; i++) {
			RequestData rd = new RequestData();
			rd.setId("" + i);
			rd.setName("我是消息" + i);
			rd.setRequestMessage("内容" + i);
			String path = System.getProperty("user.dir")
					+ File.separatorChar + "source" + File.separatorChar + "001.jpg";
			File file = new File(path);
			FileInputStream fis = new FileInputStream(file);
			byte[] data = new byte[fis.available()];
			fis.read(data);
			fis.close();
			rd.setAttachment(GzipUtils.gzip(data));
			channel.writeAndFlush(rd);
		}
		
		cf.channel().closeFuture().sync();
		wGroup.shutdownGracefully();
		
	}
}
```

对应的clienthandler代码：

```
public class ClientHandler extends ChannelInboundHandlerAdapter {

	@Override
	public void channelRead(ChannelHandlerContext ctx, Object msg) {
		try {
			ResponseData rd = (ResponseData)msg;
			System.err.println("输出服务器端相应内容: " + rd.getId());
		} finally {
			ReferenceCountUtil.release(msg);
		}
	}
}
```


## 运行

先启动服务端，再启动客户端：

服务端打印的消息如下：

```
id: 0, name: 我是消息0, requestMessage: 内容0
id: 1, name: 我是消息1, requestMessage: 内容1
id: 2, name: 我是消息2, requestMessage: 内容2
id: 3, name: 我是消息3, requestMessage: 内容3
id: 4, name: 我是消息4, requestMessage: 内容4
id: 5, name: 我是消息5, requestMessage: 内容5
id: 6, name: 我是消息6, requestMessage: 内容6
id: 7, name: 我是消息7, requestMessage: 内容7
id: 8, name: 我是消息8, requestMessage: 内容8
id: 9, name: 我是消息9, requestMessage: 内容9
```


客户端打印的消息如下：

```
输出服务器端相应内容: response 我是消息0
输出服务器端相应内容: response 我是消息1
输出服务器端相应内容: response 我是消息2
输出服务器端相应内容: response 我是消息3
输出服务器端相应内容: response 我是消息4
输出服务器端相应内容: response 我是消息5
输出服务器端相应内容: response 我是消息6
输出服务器端相应内容: response 我是消息7
输出服务器端相应内容: response 我是消息8
```


说明我们成功在 netty 上 集成了 Marshalling。


