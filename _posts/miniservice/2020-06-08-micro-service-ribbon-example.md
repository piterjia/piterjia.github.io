---
layout: post
title: 【Spring Cloud系列】Ribbon 使用案例
categories: 微服务
description: Ribbon 使用案例
keywords: Ribbon, spring Cloud, distributed, micro service
---

本文通过举一个简单的例子的方式，介绍 Ribbon 相关使用。


## 1. pom 配置

导入eureka-client、ribbon、web 等相关的包。

```
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
    </dependencies>
```

## 2. application.properties 配置

通过配置文件的方式来为 Ribbon 指定对应的配置：

```
spring.application.name=ribbon-consumer

server.port=31000

eureka.client.serviceUrl.defaultZone=http://localhost:20000/eureka/
# 采用配置文件方式，进行ribbon 负载均衡的方式
eureka-client.ribbon.NFLoadBalancerRuleClassName=com.netflix.loadbalancer.RoundRobinRule
```

Ribbon 相关的配置主要包括以下几点，需要的时候可以补充
```
<clientName>.ribbon.NFLoadBalancerClassName: Should implement ILoadBalancer(负载均衡器操作接口)
<clientName>.ribbon.NFLoadBalancerRuleClassName: Should implement IRule(负载均衡算法)
<clientName>.ribbon.NFLoadBalancerPingClassName: Should implement IPing(服务可用性检查)
<clientName>.ribbon.NIWSServerListClassName: Should implement ServerList(服务列表获取)
<clientName>.ribbon.NIWSServerListFilterClassName: Should implement ServerList­Filter(服务列表的过滤)
```

## 3. 启动类：Ribbon Consumer Application 

```
@SpringBootApplication
@EnableDiscoveryClient
public class RibbonConsumerApplication {

    @Bean
    @LoadBalanced
    public RestTemplate template() {
        return new RestTemplate();
    }

    public static void main(String[] args) {
        new SpringApplicationBuilder(RibbonConsumerApplication.class)
                .web(WebApplicationType.SERVLET)
                .run(args);
    }

}
```

注意：需要在 RestTemplate 上面，添加 LoadBalanced 注解，指名这个需要 ribbon 来实现负载均衡。

## 4. 新建配置文件
通过代码或者配置文件（ application.properties ）配置负载均衡二选一即可

配置文件也可以通过注解的方式，指定该策略针对的服务，或者切换为自定义的负载均衡策略。

```
@Configuration
//@RibbonClient(name = "eureka-client", configuration = MyRule.class)
public class RibbonConfiguration {

//    @Bean
//    public IRule defaultLBStrategy() {
//        return new RandomRule();
//    }

}
```

## 5. 新建 控制器，发起远程调用

注意：通过 ribbon 发起服务调用的时候，不需要添加端口号，直接通过 **服务名** 即可。

```
@RestController
public class Controller {

    @Autowired
    private RestTemplate restTemplate;

    @GetMapping("/sayHi")
    public String sayHi() {
        return restTemplate.getForObject(
                "http://eureka-client/sayHi",
                String.class);
    }

}
```


