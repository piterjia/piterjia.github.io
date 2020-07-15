---
layout: post
title: 【spring boot 系列】SpringBoot 集成 Zookeeper
categories: Zookeeper Spring
description: SpringBoot 集成 Zookeeper
keywords: Zookeeper, 中间件, Apache, Spring, Spring boot
---

本文主要介绍 SpringBoot 集成 Zookeeper，关于 Zookeeper 相关的概念，参考我其他的几篇文章。

[ZooKeeper 原理简介与使用场景](../zookeeper/2020-03-06-introduce-zookeeper.md)

[ZooKeeper 集群环境搭建与配置](../zookeeper/2020-03-08-zookeeper-cluster-install.md)

[ZooKeeper 常用命令行操作](../zookeeper/2020-03-09-zookeeper-common-cmd.md)

## pom.xml引入 zookeeper 依赖

引入 Apache下的Zookeeper jar包

```
<!-- Apache下的Zookeeper架包-->
<dependency>
    <groupId>org.apache.zookeeper</groupId>
    <artifactId>zookeeper</artifactId>
    <version>3.4.14</version>
</dependency>
```


## 配置 application.properties

配置 zookeeper 的地址、端口，以及超时时间。
```
zookeeper.address=127.0.0.1:2181
zookeeper.timeout=4000
```

## 添加 Zookeeper Config 配置类

添加 Zookeeper Config 配置类，完成 ZooKeeper 初始化工作。

```
mport lombok.extern.slf4j.Slf4j;
import org.apache.zookeeper.WatchedEvent;
import org.apache.zookeeper.Watcher;
import org.apache.zookeeper.ZooKeeper;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.concurrent.CountDownLatch;
import java.util.logging.Logger;

@Configuration
@Slf4j
public class ZookeeperConfig {

    @Value("${zookeeper.address}")
    private String connectString;

    @Value("${zookeeper.timeout}")
    private int timeout;

    @Bean(name = "zkClient")
    public ZooKeeper zkClient() {
        ZooKeeper zooKeeper = null;
        try {
            final CountDownLatch countDownLatch = new CountDownLatch(1);
            //连接成功后，会回调watcher监听，此连接操作是异步的，执行完new语句后，直接调用后续代码
            //  可指定多台服务地址 127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183
            zooKeeper = new ZooKeeper(connectString, timeout, new Watcher() {
                @Override
                public void process(WatchedEvent event) {
                    if (Event.KeeperState.SyncConnected == event.getState()) {
                        //如果收到了服务端的响应事件,连接成功
                        countDownLatch.countDown();
                    }
                }
            });
            countDownLatch.await();
            log.info("【初始化ZooKeeper连接状态....】={}", zooKeeper.getState());

        } catch (Exception e) {
            log.error("初始化ZooKeeper连接异常....】={}", e);
        }
        return zooKeeper;
    }
}
```

## 封装 ZooKeeper 客户端工具类

封装 ZooKeeper 客户端工具类，对 ZooKeeper 进一步封装，提供更友好的API。

```
import lombok.extern.slf4j.Slf4j;
import org.apache.zookeeper.*;
import org.apache.zookeeper.data.Stat;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import javax.annotation.PostConstruct;
import java.util.List;

@Component
@Slf4j
public class ZkApi {

    @Autowired
    private ZooKeeper zkClient;

    /**
     * 判断指定节点是否存在
     *
     * @param path
     * @param needWatch 指定是否复用zookeeper中默认的Watcher
     * @return
     */
    public Stat exists(String path, boolean needWatch) {
        try {
            return zkClient.exists(path, needWatch);
        } catch (Exception e) {
            log.error("【断指定节点是否存在异常】{},{}", path, e);
            return null;
        }
    }

    /**
     * 检测结点是否存在 并设置监听事件
     * 三种监听类型： 创建，删除，更新
     *
     * @param path
     * @param watcher 传入指定的监听类
     * @return
     */
    public Stat exists(String path, Watcher watcher) {
        try {
            return zkClient.exists(path, watcher);
        } catch (Exception e) {
            log.error("【断指定节点是否存在异常】{},{}", path, e);
            return null;
        }
    }

    /**
     * 创建持久化节点
     *
     * @param path
     * @param data
     */
    public boolean createNode(String path, String data) {
        try {
            zkClient.create(path, data.getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
            return true;
        } catch (Exception e) {
            log.error("【创建持久化节点异常】{},{},{}", path, data, e);
            return false;
        }
    }

    /**
     * 修改持久化节点
     *
     * @param path
     * @param data
     */
    public boolean updateNode(String path, String data) {
        try {
            //zk的数据版本是从0开始计数的。如果客户端传入的是-1，则表示zk服务器需要基于最新的数据进行更新。如果对zk的数据节点的更新操作没有原子性要求则可以使用-1.
            //version参数指定要更新的数据的版本, 如果version和真实的版本不同, 更新操作将失败. 指定version为-1则忽略版本检查
            zkClient.setData(path, data.getBytes(), -1);
            return true;
        } catch (Exception e) {
            log.error("【修改持久化节点异常】{},{},{}", path, data, e);
            return false;
        }
    }

    /**
     * 删除持久化节点
     *
     * @param path
     */
    public boolean deleteNode(String path) {
        try {
            //version参数指定要更新的数据的版本, 如果version和真实的版本不同, 更新操作将失败. 指定version为-1则忽略版本检查
            zkClient.delete(path, -1);
            return true;
        } catch (Exception e) {
            log.error("【删除持久化节点异常】{},{}", path, e);
            return false;
        }
    }

    /**
     * 获取当前节点的子节点(不包含孙子节点)
     *
     * @param path 父节点path
     */
    public List<String> getChildren(String path) throws KeeperException, InterruptedException {
        List<String> list = zkClient.getChildren(path, false);
        return list;
    }

    /**
     * 获取指定节点的值
     *
     * @param path
     * @return
     */
    public String getData(String path, Watcher watcher) {
        try {
            Stat stat = new Stat();
            byte[] bytes = zkClient.getData(path, watcher, stat);
            return new String(bytes);
        } catch (Exception e) {
            e.printStackTrace();
            return null;
        }
    }

}
```

其中，WatcherApi 类实现 Watcher 接口，实现对 zookeeper 事件的监听。

```
import lombok.extern.slf4j.Slf4j;
import org.apache.zookeeper.WatchedEvent;
import org.apache.zookeeper.Watcher;

@Slf4j
public class WatcherApi implements Watcher {

    @Override
    public void process(WatchedEvent event) {
        log.info("【Watcher监听事件】={}", event.getState());
        log.info("【监听路径为】={}", event.getPath());
        log.info("【监听的类型为】={}", event.getType()); //  三种监听类型： 创建，删除，更新
    }
}
```


## 编写 测试 controller

```
import com.piter.zookeeper.utils.ZkApi;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@Slf4j
public class ZookeeperController {

    @Autowired
    private ZkApi zkApi;

    @GetMapping(value = "createNode")
    public boolean createNode(String path, String data) {
        log.debug("ZookeeperController create node {},{}", path, data);
        return zkApi.createNode(path, data);
    }

}
```

## 编写  启动类 

```
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class SpringBootZookeeperIntegrationApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringBootZookeeperIntegrationApplication.class, args);
    }

}
```


## 项目架构

最终项目集成结构如下：

![](/images/posts/spring/spring-zookeeper.png)


项目源码仓库：

[SpringBoot 集成 Zookeeper](https://github.com/piterjia/spring-zookeeper-integration/tree/master)


## 联调自测试

直接采用浏览器简单自测试，发起调用。

![](/images/posts/spring/spring-zookeeper-1.png)


命令行查看 zookeeper 节点信息，发现数据被成功写入：

```
[zk: 127.0.0.1:2181(CONNECTED) 89] ls /
[czk, jhj, node1, node_1, piter, zk, zookeeper]
```






