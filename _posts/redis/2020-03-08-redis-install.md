---
layout: post
title: linux 安装 redis 
categories: Redis
description: linux 安装 redis 
keywords: linux, redis
---


## 安装：

### 1. 获取redis资源

```
wget http://download.redis.io/releases/redis-5.0.8.tar.gz
```

### 2. 解压

```
tar xzvf redis-5.0.8.tar.gz
```

### 3. 安装

```
cd redis-5.0.8

make

cd src

make install PREFIX=/usr/local/redis
```

### 4. 移动配置文件到安装目录下
```
cd ../

mkdir /usr/local/redis/etc

mv redis.conf /usr/local/redis/etc
```
### 5. 配置redis为后台启动

```
vi /usr/local/redis/etc/redis.conf //将daemonize no 改成daemonize yes
```

### 6. 将redis加入到开机启动

```
vi /etc/rc.local 
```

在里面添加内容：
```
/usr/local/redis/bin/redis-server /usr/local/redis/etc/redis.conf 
```

### 7. 开启redis

```
/usr/local/redis/bin/redis-server /usr/local/redis/etc/redis.conf 
```
 
### 8. 将redis-cli,redis-server拷贝到bin下，让redis-cli指令可以在任意目录下直接使用

```
cp /usr/local/redis/bin/redis-server /usr/local/bin/
cp /usr/local/redis/bin/redis-cli /usr/local/bin/
```

### 9. 设置redis密码

找到redis的配置文件— redis.conf 文件，然后修改里面的 requirepass，这个本来是注释起来了的，将注释去掉，并将后面对应的字段设置成自己想要的密码，保存退出。

重启redis服务，即可。

### 10. 让外网能够访问redis

切换到root用户 打开iptables的配置文件：

```
vi /etc/sysconfig/iptables
```

添加 6379 端口号

```
# Firewall configuration written by system-config-firewall
# Manual customization of this file is not recommended.
*filter
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
-A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
-A INPUT -p icmp -j ACCEPT
-A INPUT -i lo -j ACCEPT
-A INPUT -m state --state NEW -m tcp -p tcp --dport 22 -j ACCEPT
-A INPUT -m state --state NEW -m tcp -p tcp --dport 3306 -j ACCEPT
-A INPUT -m state --state NEW -m tcp -p tcp --dport 6379 -j ACCEPT
-A INPUT -j REJECT --reject-with icmp-host-prohibited
-A FORWARD -j REJECT --reject-with icmp-host-prohibited
COMMIT
```

全部修改完之后重启
```
iptables:service iptables restart
```
