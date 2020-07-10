---
layout: post
title: Linux 系统 RocketMQ 与控制台的安装与启动
categories: RocketMQ
description: Linux 系统 RocketMQ 与控制台的安装与启动
keywords: RocketMQ, Linux
---

##  一 下载并解压

```
#1. 进入 /usr/local 目录，下载rocketmq 包
wget https://mirrors.tuna.tsinghua.edu.cn/apache/rocketmq/4.7.1/rocketmq-all-4.7.1-source-release.zip
 
#2.解压
unzip rocketmq-all-4.7.1-bin-release.zip
 
#3.重命名
mv rocketmq-all-4.7.1-bin-release rocketmq-all-4.7.1
```


## 二 修改 broker 和n ameserver 的 启动配置

### 1. 修改runbroker.sh(因为默认是8G，虚拟机我只给了1G运行内存)
```
vim /usr/local/rocketmq/bin/runbroker.sh
```

修改对应行如下：

```
JAVA_OPT="${JAVA_OPT} -server -Xms512m -Xmx512m -Xmn256m -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=320m"
```


### 2. 修改runbroker.sh(因为默认是4G，虚拟机我只给了1G运行内存)
```
vim /usr/local/rocketmq/bin/runserver.sh
```

修改对应行如下：

```
JAVA_OPT="${JAVA_OPT} -server -Xms512m -Xmx512m -Xmn256m"
```


## 三 启动 name server

```
> nohup sh bin/mqnamesrv &
> tail -f ~/logs/rocketmqlogs/namesrv.log
  The Name Server boot success...
```


## 四 Start Broker

```
> nohup sh bin/mqbroker -n localhost:9876 &
> tail -f ~/logs/rocketmqlogs/broker.log 
The broker[%s, 172.30.30.233:10911] boot success...
```

## 五 Shutdown Servers 相关命令

```
> sh bin/mqshutdown broker
The mqbroker(36695) is running...
Send shutdown request to mqbroker(36695) OK

> sh bin/mqshutdown namesrv
The mqnamesrv(36664) is running...
Send shutdown request to mqnamesrv(36664) OK

```

## rocketmq 可视化控制台

rocketmq默认不带可视化控制台，需要去单独编译一个工具 https://github.com/apache/rocketmq-externals

> 我的并没有打成 jar 包部署运行，而是直接用 IDEA 启动运行。需要改动 application.properties 配置文件中的rocketmq.config.namesrvAddr 的值，改成自己 rocketmq 部署服务器的 ip 地址。 server.port 可改可不改。

![](/images/posts/rocketmq/rocketmq-consule-1.png)


注意，一定要 **记得在防火墙中，打开 9876 端口** ，否则控制台无法正常连接rocketmq 。

具体如何操作，可以参考我的这篇文章

[linux-firewalld-cmd](../linux/2020-02-27-linux-firewalld-cmd.md)



## 参考资料

[rocketmq 官方文档](http://rocketmq.apache.org/docs/quick-start/)

[rocketmq 官方中文文档](https://rocketmq-1.gitbook.io/rocketmq-connector/quick-start/qian-qi-zhun-bei/dan-ji-huan-jing)











