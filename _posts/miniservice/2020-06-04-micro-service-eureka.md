---
layout: post
title: 【Spring Cloud系列】搭建 Eureka 单节点注册中心
categories: 微服务
description: Eureka 使用简介与单中心示例
keywords: Eureka, spring Cloud, distributed, micro service
---

Eureka 是 spring cloud 中的一个负责服务注册与发现的组件。遵循着 CAP 理论中的 A(可用性) P(分区容错性)。

一个 Eureka 中分为 eureka server 和 eureka client。其中 eureka server 是作为服务的注册与发现中心。 eureka client 既可以作为服务的生产者，又可以作为服务的消费者。


该案例，我们采用一个项目中集成多个子 module 的方式实现。

1. 首先会启动一个 Eureka server，该 eureka server 同步保留着所有的服务信息。该子module命名为 eurekaserver 。
2. 然后我们启动第一个 eureka client，向服务端发起服务注册，他作为服务提供者。该子module命名为 eurekaprovider 。
3. 然后我们启动第二个 eureka client，向服务端发起服务注册，他作为服务消费者。该子module命名为 eurekaconsumer 。


## 父 Module 的实现

### 项目结构
spring cloud 基于 spring boot 进行开发，因此我们需要创建一个 spring boot 项目，同时在里面新建三个子 module，分别是：eurekaserver、eurekaprovider、eurekaconsumer。

项目结构如下：

![](/images/microservice/eureka-demo-1.png)

### POM 配置
**pom文件配置如下：**

```
  <packaging>pom</packaging>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>
    </properties>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>Greenwich.SR1</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>

            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-parent</artifactId>
                <version>2.1.5.RELEASE</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <dependencies>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.8</version>
        </dependency>
    </dependencies>
```

## eureka server 实现
此章节我们开发一个 Eureka server，该 eureka server 作为注册中心，同步保留所有的服务信息。

### server pom 配置
在子module eurekaserver 中的 pom.xml里添加如下的配置信息
```
    <artifactId>eurekaserver</artifactId>
    <name>eureka-server</name>
    <description>Demo project for Eureka Server</description>

    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
        </dependency>
    </dependencies>
```

### server 启动类
新建启动类，并在我们的启动类中添加注解 @EnableEurekaServer
```
@EnableEurekaServer
@SpringBootApplication
public class EurekaServicecenterApplication {

    public static void main(String[] args) {
        SpringApplication.run(EurekaServicecenterApplication.class, args);
    }
}
```

### 应用配置信息
我们在eureka server中采用yml的方式进行配置，并对每一行的作用做了详细的备注。

resource目录下新建application.yml文件，并添加一下配置信息：
```
spring:
  application:
    #服务名称
    name: eureka-server
server:
  #服务注册中心端口号
  port: 10000
eureka:
  instance:
    hostname: localhost
  client:
    #是否向服务注册中心注册自己
    registerWithEureka: false
    #是否检索服务
    fetchRegistry: false
    #服务注册中心的配置内容，指定服务注册中心位置
    #注册中心路径，如果有多个eureka server，在这里需要配置其他eureka server的地址，用","进行区分，如"http://address:8888/eureka,http://address:8887/eureka"
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
  # 以下配置信息可选
  server:
    #开启注册中心的保护机制，默认是开启
    enable-self-preservation: true
    #设置保护机制的阈值，默认是0.85。
    renewal-percent-threshold: 0.8
```


**eureka server 注册中心就准备完毕了。启动eureka server，静静等待其他服务的注册吧。**

服务启动成功：

![](/images/microservice/eureka-demo-2.png)


注册中心后台页面可以正常打开：

![](/images/microservice/eureka-demo-3.png)



## eureka provider 实现
此章节我们开发一个 eureka client，向服务端发起服务注册，他作为服务提供者对外提供服务。

### provider pom 配置
pom.xml中添加如下的配置信息
```
    <artifactId>eurekaprovider</artifactId>
    <name>eureka-provider</name>
    <description>Demo project for Eureka Provider</description>

    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
        </dependency>
    </dependencies>
```

### provider 启动类
新建启动类，并在我们的启动类中添加注解 @EnableDiscoveryClient
```
@EnableDiscoveryClient
@SpringBootApplication
public class EurekaServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

### 应用配置信息
为何熟练掌握多种配置方式，我们在 eureka provider 中采用 properties 的方式进行配置，并对每一行的作用做了详细的备注。

resource目录下新建 aapplication.properties 文件，并添加一下配置信息：
```
#服务端口
server.port=10001
#服务名称
spring.application.name=eureka-provider
#服务地址
eureka.instance.hostname=localhost
#注册中心路径，表示我们向这个注册中心注册服务，如果向多个注册中心注册，用“，”进行分隔
eureka.client.service-url.defaultZone=http://localhost:10000/eureka
#心跳间隔10s，默认30s。每一个服务配置后，心跳间隔和心跳超时时间会被保存在server端，不同服务的心跳频率可能不同，server端会根据保存的配置来分别探活
eureka.instance.lease-renewal-interval-in-seconds=10
#心跳超时时间30s，默认90s。从client端最后一次发出心跳后，达到这个时间没有再次发出心跳，表示服务不可用，将它的实例从注册中心移除
eureka.instance.lease-expiration-duration-in-seconds=30
```

### 对外提供服务

新建一个controller，并写一个简单的hello api。代码如下：

```
@RestController
public class HelloController {

    @ResponseBody
    @RequestMapping("/hello")
    public String hello(String name) {
        return name + ", Welcome to service provider";
    }

}
```


**eureka provider 服务提供者就准备完毕了。启动eureka provider ，静静等待其他client的调用吧。**

provider 启动成功：

![](/images/microservice/eureka-demo-4.png)


provider 成功注册到了注册中心：

![](/images/microservice/eureka-demo-5.png)


## eureka consumer 实现

此章节我们开发一个 eureka client ，向服务端发起服务注册，他作为服务消费者者，对 provider 进行远程调用。

### consumer pom 配置
pom.xml中添加如下的配置信息
```
    <artifactId>eurekaconsumer</artifactId>
    <name>eureka-client</name>
    <description>Demo project for Spring Boot</description>

    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
    </dependencies>
```

### consumer 启动类
新建启动类，并在我们的启动类中添加注解 @EnableDiscoveryClient
```
@SpringBootApplication
@EnableDiscoveryClient
public class EurekaClientApplication {

    public static void main(String[] args) {
        SpringApplication.run(EurekaClientApplication.class, args);
    }
}
```

### 应用配置信息
为何熟练掌握多种配置方式，我们在 eureka consumer 中采用 properties 的方式进行配置，并对每一行的作用做了详细的备注。

resource目录下新建 aapplication.properties 文件，并添加一下配置信息：
```
#服务端口
server.port=10002
#注册到Eureka Server上的服务ID
spring.application.name=eureka-client
#服务地址
eureka.instance.hostname=localhost
#Eureka Server默认地址：注册中心路径，表示我们向这个注册中心注册服务，如果向多个注册中心注册，用“，”进行分隔
eureka.client.service-url.defaultZone=http://localhost:10000/eureka
#心跳间隔10s，默认30s。每一个服务配置后，心跳间隔和心跳超时时间会被保存在server端，不同服务的心跳频率可能不同，server端会根据保存的配置来分别探活
eureka.instance.lease-renewal-interval-in-seconds=10
#心跳超时时间30s，默认90s。从client端最后一次发出心跳后，达到这个时间没有再次发出心跳，表示服务不可用，将它的实例从注册中心移除
eureka.instance.lease-expiration-duration-in-seconds=30
```

### 开启自测试

新建一个controller，并写一个简单的hello api。内部发起远程调用，代码如下：

```
@RestController
public class ConsumerController {

    @Autowired
    RestTemplate restTemplate;

    @LoadBalanced
    @Bean
    public RestTemplate rest() {
        return new RestTemplate();
    }

    @RequestMapping("/hi")
    public String hello(String name) {
        return restTemplate.getForObject("http://eureka-provider/hello?name=" + name, String.class);
    }
}
```


**eureka consumer 服务提供者就准备完毕了。启动 eureka consumer ，开始测试吧。**

consumer 成功注册到注册中心：

![](/images/microservice/eureka-demo-6.png)


consumer发起远程调用，成功获取到了 provider 提供的服务返回信息：

![](/images/microservice/eureka-demo-7.png)



## 总结：

本文简单介绍了 eureka 的单中心案例，下一个文章，我们介绍下 eureka 高可用方案。
敬请期待。

该案例源码所在地址：

[单节点 Eureka](https://github.com/piterjia/eurekademo/tree/master/stage1)






