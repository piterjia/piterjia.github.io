---
layout: post
title: ZooKeeper 集群环境搭建与配置
categories: Zookeeper
description: ZooKeeper 集群环境搭建与配置
keywords: Zookeeper, 中间件, Apache
---

ZooKeeper 是一个 分布式协调服务框架。本文介绍ZooKeeper 集群环境搭建与配置

## 1. 准备工作：
- 准备3个节点，要求配置好主机名称，服务器之间系统时间保持一致

```
注意 /etc/hostname 和 /etc/hosts 配置主机名称
```

## 2. 上传zk到三台服务器节点，并安装
```
注意我这里解压到/usr/local下
```


### 2.1 进行解压： 
tar zookeeper-3.4.6.tar.gz



### 2.2 重命名： 
mv zookeeper-3.4.6 zookeeper



### 2.3 修改环境变量： 

vim /etc/profile 

#### 添加zookeeper的全局变量

- export ZOOKEEPER_HOME=/usr/local/zookeeper
- export PATH=.:$ZOOKEEPER_HOME/bin


### 2.4 刷新环境变量： 
source /etc/profile



### 2.5 到zookeeper下修改配置文件： 
- 1、首先到指定目录： cd /usr/local/zookeeper/conf
- 2、然后复制zoo_sample.cfg文件，复制后为zoo.cfg： mv zoo_sample.cfg zoo.cfg
- 3、然后修改两处地方, 最后保存退出：
    - (1) 修改数据的dir    
        dataDir=/usr/local/zookeeper/data
    - (2) 修改集群地址
        server.0=bhz125:2888:3888
        server.1=bhz126:2888:3888
        server.2=bhz127:2888:3888



### 2.6 增加服务器标识配置，需要2步骤，第一是创建文件夹和文件，第二是添加配置内容： 
- 1、创建文件夹： mkdir /usr/local/zookeeper/data
- 2、创建文件myid 
  
  路径应该创建在/usr/local/zookeeper/data下面，如下：
	vim /usr/local/zookeeper/data/myid

	注意这里每一台服务器的myid文件内容不同，分别修改里面的值为0，1，2；
    
    与我们之前的zoo.cfg配置文件里：server.0，server.1，server.2 顺序相对应，然后保存退出；




### 2.7 到此为止，Zookeeper集群环境大功告成！
启动zookeeper命令
- 启动路径：/usr/local/zookeeper/bin（也可在任意目录，因为配置了环境变量）
- 执行命令：zkServer.sh start (注意这里3台机器都要进行启动，启动之后可以查看状态)
- 查看状态：zkServer.sh status (在三个节点上检验zk的mode, 会看到一个leader和俩个follower)




## Zookeeper客户端操作：

zkCli.sh 进入zookeeper客户端

根据提示命令进行操作： 
- 查找：ls /   ls /zookeeper
- 创建并赋值： create /imooc zookeeper
- 获取： get /imooc 
- 设值： set /imooc zookeeper1314 
  
```
任意节点都可以看到zookeeper集群的数据一致性
创建节点有俩种类型：短暂（ephemeral） 持久（persistent）
```




## Zookeeper核心配置详解：（zoo.cfg配置文件，扩展内容）

1、tickTime：基本事件单元，以毫秒为单位。

这个时间是作为 Zookeeper 服务器之间或客户端与服务器之间维持心跳的时间间隔，也就是每隔 tickTime时间就会发送一个心跳。
				
2、initLimit：这个配置项是用来配置 Zookeeper 接受客户端初始化连接时最长能忍受多少个心跳时间间隔数，当已经超过 10 个心跳的时间（也就是 tickTime）长度后 Zookeeper 服务器还没有收到客户端的返回信息，那么表明这个客户端连接失败。总的时间长度就是 10*2000=20 秒。
		
3、syncLimit：这个配置项标识 Leader 与 Follower 之间发送消息，请求和应答时间长度，最长不能超过多少个 tickTime 的时间长度，总的时间长度就是 5*2000=10 秒
				
4、dataDir：存储内存中数据库快照的位置，顾名思义就是 Zookeeper 保存数据的目录，默认情况下，Zookeeper 将写数据的日志文件也保存在这个目录里。
   
5、clientPort： 这个端口就是客户端连接 Zookeeper 服务器的端口，Zookeeper 会监听这个端口，接受客户端的访问请求。

6、至于最后的配置项：server.A = B:C:D: 
- A表示这个是第几号服务器,
- B 是这个服务器的 ip 地址；
- C 表示的是这个服务器与集群中的 Leader 服务器交换信息的端口；
- D 表示的是万一集群中的 Leader 服务器挂了，需要一个端口来重新进行选举，选出一个新的 Leader