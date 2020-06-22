---
layout: post
title: 【Spring Cloud系列】Feign 入门与案例
categories: 微服务
description: Feign 入门与案例
keywords: Feign, spring Cloud, distributed, micro service
---

Feign 是声明式的 web service 客户端，它让微服务之间的调用变得更简单了，类似 controller调用 service。Spring Cloud 集成了 Ribbon 和 Eureka，可在使用 Feign 时提供负载均衡的http 客户端。


# 编写 Feign 案例

## 创建注册中心
关于注册中心如何创建，大家参考之前的这两篇文章

[搭建 Eureka 单节点注册中心]( https://piterjia.github.io/2020/06/04/micro-service-eureka/)

[搭建 Eureka 高可用服务注册中心集群](  https://piterjia.github.io/2020/06/05/micro-service-eureka-cluster/)


## 创建公共接口子 Module
这个module的主要作用，就是提供一个协议，服务提供者和消费者之间远程交互的调用协议。

### pom 导包

采用最小原则导包，不要引入不需要的依赖。

```
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
        </dependency>

    </dependencies>
```
### 定义接口层

定义接口，并添加注解 FeignClient。注意注解的value 为提供远程服务的服务提供者的名字。

```
@FeignClient("feign-client")
public interface IService {

    @GetMapping("/sayHi")
    public String sayHi();

}
```

## 创建 服务提供者 子 Module

这个module的主要作用，就是对外提供服务，供服务消费者远程调用。

### pom文件配置
 ```
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>

        <!--上面定义的接口module-->
        <dependency>
            <groupId>com.jhj</groupId>
            <artifactId>feign-client-intf</artifactId>
            <version>${project.version}</version>
        </dependency>

    </dependencies>
```


### 添加配置文件

application.properties 中添加如下配置：

```
spring.application.name=feign-client
server.port=40006
eureka.client.serviceUrl.defaultZone=http://localhost:20000/eureka/
```

### 实现对外提供的服务

```
@RestController
@Slf4j
public class Controller implements IService {

    @Value("${server.port}")
    private String port;

    @Override
    public String sayHi() {
        return "This is " + port;
    }

}
```

### 实现启动类

注意这个类作为服务提供者，不需要添加注解 EnableFeignClients

```
@EnableDiscoveryClient
@SpringBootApplication
public class FeignClientApplication {

    public static void main(String[] args) {
        new SpringApplicationBuilder(FeignClientApplication.class)
                .web(WebApplicationType.SERVLET)
                .run(args);
    }

}
```

启动该服务，等待消费者调用。

![](/images/microservice/feign-1.png)


## 创建 服务消费者
这个module的主要作用，就是对前面创建的服务消费者，发起远程调用。测试整个链路是否是通的。

### pom 导包
 ```
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>

        <!--上面定义的接口module-->
        <dependency>
            <groupId>com.jhj</groupId>
            <artifactId>feign-client-intf</artifactId>
            <version>${project.version}</version>
        </dependency>

    </dependencies>
```

### 配置文件

application.properties   中添加如下配置：

```
spring.application.name=feign-consumer-advanced
server.port=40001

spring.main.allow-bean-definition-overriding=true

eureka.client.serviceUrl.defaultZone=http://localhost:20000/eureka/

# 以下配置可选
# 每台机器最大重试次数
feign-client.ribbon.MaxAutoRetries=2
# 可以再重试几台机器
feign-client.ribbon.MaxAutoRetriesNextServer=2
# 连接超时
feign-client.ribbon.ConnectTimeout=1000
# 业务处理超时
feign-client.ribbon.ReadTimeout=2000
# 在所有HTTP Method进行重试
feign-client.ribbon.OkToRetryOnAllOperations=true
```

### 实现消费者控制器

实现对服务提供者的远程调用
```
@RestController
@Slf4j
public class Controller {

    @Autowired
    private IService service;

    @GetMapping("/sayHi")
    public String sayHi() {
        return service.sayHi();
    }
}
```

### 实现启动类
注意这个类作为服务消费者，需要添加注解 EnableFeignClients
```
@EnableDiscoveryClient
@SpringBootApplication
@EnableFeignClients
public class FeingConsumerApp {

    public static void main(String[] args) {
        new SpringApplicationBuilder(FeingConsumerApp.class)
                .web(WebApplicationType.SERVLET)
                .run(args);
    }

}
```

启动该服务消费者，可以看到服务正常启动，并成功挂载到注册中心。

![](/images/microservice/feign-2.png)



## 联调

发起远程调用，成功返回服务提供者的端口号40006。

![](/images/microservice/feign-3.png)


## 总结
本文简单介绍了 Feign，并以一个例子介绍了 Feign 的使用方式和配置。