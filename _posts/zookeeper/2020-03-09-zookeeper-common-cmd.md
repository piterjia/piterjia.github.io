---
layout: post
title: ZooKeeper 常用命令行操作
categories: Zookeeper
description: ZooKeeper 常用命令行操作
keywords: Zookeeper, 中间件, Apache
---

通过./zkCli.sh 打开 zk 的客户端进入命令行后台

## ZooKeeper服务命令:
1. 启动ZK服务: bin/zkServer.sh start
2. 查看ZK服务状态: bin/zkServer.sh status
3. 停止ZK服务: bin/zkServer.sh stop
4. 重启ZK服务: bin/zkServer.sh restart
5. 连接服务器: zkCli.sh -server 127.0.0.1:2181


## zk客户端命令
ZooKeeper命令行工具类似于 Linux 的 shell 环境，使用它我们可以简单的对ZooKeeper进行访问，数据创建，数据修改等操作.  使用 zkCli.sh -server 127.0.0.1:2181 连接到 ZooKeeper 服务，连接成功后，系统会输出 ZooKeeper 的相关环境以及配置信息。

### 命令行工具的一些简单操作如下：

#### ls 列出当前节点下的子节点
```
[zk: 127.0.0.1:2181(CONNECTED) 28] ls -R /
/
/node1
/zk
/zookeeper
/node1/node1.1
/node1/node1.2
/zookeeper/config
/zookeeper/quota
[zk: 127.0.0.1:2181(CONNECTED) 29] 
```

#### stat 列出节点状态

```
[zk: 127.0.0.1:2181(CONNECTED) 29] stat /
cZxid = 0x0
ctime = Thu Jan 01 08:00:00 CST 1970
mZxid = 0x0
mtime = Thu Jan 01 08:00:00 CST 1970
pZxid = 0xc
cversion = 1
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 0
numChildren = 3
```


#### get命令 获取节点内容和节点状态

```
[zk: 127.0.0.1:2181(CONNECTED) 35] get /zk
jiang1234
```

#### create命令

创建永久节点

```
[zk: 127.0.0.1:2181(CONNECTED) 39] create /node_1 node-data
Created /node_1
```

创建临时节点 &session过期临时节点删除

```
[zk: 127.0.0.1:2181(CONNECTED) 40] create -e /czk/tmp temp-data
Created /czk/tmp
```

创建顺序节点

```
[zk: 127.0.0.1:2181(CONNECTED) 46] ls /czk
[tmp]
[zk: 127.0.0.1:2181(CONNECTED) 47] create -s /czk/sec seq
Created /czk/sec0000000001
[zk: 127.0.0.1:2181(CONNECTED) 48] create -s /czk/sec seq
Created /czk/sec0000000002
[zk: 127.0.0.1:2181(CONNECTED) 49] ls /czk
[sec0000000001, sec0000000002, tmp]
[zk: 127.0.0.1:2181(CONNECTED) 50] 

```


####  set命令:
```
 set path data [version] # set 路径 数据 [版本号]       []内为可选参数
```

示例如下：

```
[zk: 127.0.0.1:2181(CONNECTED) 50] get /czk
czk-data
[zk: 127.0.0.1:2181(CONNECTED) 51] set /czk czk-data2
[zk: 127.0.0.1:2181(CONNECTED) 52] get /czk
czk-data2
```


#### delete 命令: 
```
delete path [version]
```

示例如下：

```
[zk: 127.0.0.1:2181(CONNECTED) 57] ls /czk
[sec0000000001, sec0000000002, tmp]
[zk: 127.0.0.1:2181(CONNECTED) 58] delete /czk/sec000000001
Node does not exist: /czk/sec000000001
[zk: 127.0.0.1:2181(CONNECTED) 59] ls /czk
[sec0000000001, sec0000000002, tmp]
[zk: 127.0.0.1:2181(CONNECTED) 60] delete /czk/sec0000000001
[zk: 127.0.0.1:2181(CONNECTED) 61] ls /czk
[sec0000000002, tmp]
```



