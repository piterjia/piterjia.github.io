---
layout: post
title: 在 Centos7 上安装 Rabbitmq 与配置
categories: RabbitMQ
description: 在Centos7上安装 Rabbitmq 与配置
keywords: rabbitmq, Linux
---

## 1 安装 Erlang 依赖

创建文件/etc/yum.repos.d/rabbitmq-erlang.repo

```
vim /etc/yum.repos.d/rabbitmq-erlang.repo
```

添加如下内容，参考文档[erlang-rpm](https://github.com/rabbitmq/erlang-rpm)：

```
[rabbitmq_erlang]
name=rabbitmq_erlang
baseurl=https://packagecloud.io/rabbitmq/erlang/el/7/$basearch
repo_gpgcheck=1
gpgcheck=1
enabled=1
# PackageCloud's repository key and RabbitMQ package signing key
gpgkey=https://packagecloud.io/rabbitmq/erlang/gpgkey
       https://dl.bintray.com/rabbitmq/Keys/rabbitmq-release-signing-key.asc
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
metadata_expire=300

[rabbitmq_erlang-source]
name=rabbitmq_erlang-source
baseurl=https://packagecloud.io/rabbitmq/erlang/el/7/SRPMS
repo_gpgcheck=1
gpgcheck=0
enabled=1
# PackageCloud's repository key and RabbitMQ package signing key
gpgkey=https://packagecloud.io/rabbitmq/erlang/gpgkey
       https://dl.bintray.com/rabbitmq/Keys/rabbitmq-release-signing-key.asc
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
metadata_expire=300
```

安装 Erlang package:
```
yum install erlang
```

![](/images/posts/rabbitmq/rabbitmq1.png)


## 2 安装 RabbitMQ

官方文档有两种安装方式，官方建议第一种。那我们就采用第一种 Package Cloud 的方式安装。

![](/images/posts/rabbitmq/rabbitmq2.png)

详细的信息大家参考：
[package-cloud](https://www.rabbitmq.com/install-rpm.html#package-cloud)：

### 添加软件源


我们使用 Package Cloud-provided script 进行快速安装。


包云使用自己的GPG密钥标记分发的包。截至2018年底，包云正在进行一个关键的迁移。项目将不再依赖于“主键”，而是使用特定于存储库的签名键。在迁移完成之前，为了向前兼容性，新旧键都必须导入:

```
# import the new PackageCloud key that will be used starting December 1st, 2018 (GMT)
rpm --import https://packagecloud.io/rabbitmq/rabbitmq-server/gpgkey

# import the old PackageCloud key that will be discontinued on December 1st, 2018 (GMT)
rpm --import https://packagecloud.io/gpg.key
```


导入两个密钥后，执行脚本：

curl -s https://packagecloud.io/install/repositories/rabbitmq/rabbitmq-server/script.rpm.sh | sudo bash

执行结果如下：

![](/images/posts/rabbitmq/rabbitmq3.png)

详细的信息见官方资料：
[rabbitmq server install](https://packagecloud.io/rabbitmq/rabbitmq-server/install)：


### 安装 RabbitMQ

检查安装包：

```
[hongjunjia@centos-linux ~]$ yum list |grep rabbit
librabbitmq.i686                            0.8.0-2.el7                base     
librabbitmq.x86_64                          0.8.0-2.el7                base     
librabbitmq-devel.i686                      0.8.0-2.el7                base     
librabbitmq-devel.x86_64                    0.8.0-2.el7                base     
librabbitmq-examples.x86_64                 0.8.0-2.el7                base     
rabbitmq-server.noarch                      3.8.5-1.el7                rabbitmq_rabbitmq-server
```

一切准备就绪之后，开始安装 RabbitMQ：

```
yum install -y rabbitmq-server
```

![](/images/posts/rabbitmq/rabbitmq4.png)



### 生成本地服务

```
[hongjunjia@centos-linux ~]$ chkconfig rabbitmq-server on
Note: Forwarding request to 'systemctl enable rabbitmq-server.service'.
Created symlink from /etc/systemd/system/multi-user.target.wants/rabbitmq-server.service to /usr/lib/systemd/system/rabbitmq-server.service.
```

### 启动

```
[hongjunjia@centos-linux ~]$ sudo rabbitmq-server start
[sudo] password for hongjunjia: 

  ##  ##      RabbitMQ 3.8.5
  ##  ##
  ##########  Copyright (c) 2007-2020 VMware, Inc. or its affiliates.
  ######  ##
  ##########  Licensed under the MPL 1.1. Website: https://rabbitmq.com

  Doc guides: https://rabbitmq.com/documentation.html
  Support:    https://rabbitmq.com/contact.html
  Tutorials:  https://rabbitmq.com/getstarted.html
  Monitoring: https://rabbitmq.com/monitoring.html

  Logs: /var/log/rabbitmq/rabbit@centos-linux.log
        /var/log/rabbitmq/rabbit@centos-linux_upgrade.log

  Config file(s): (none)

  Starting broker... completed with 0 plugins.
```


### 查看状态
```
sudo rabbitmqctl status
```

![](/images/posts/rabbitmq/rabbitmq5.png)

## Rabbitmq 配置

### 开启防火墙

切换到root用户 打开iptables的配置文件：
```
vi /etc/sysconfig/iptables
```

添加 
-A INPUT -m state --state NEW -m tcp -p tcp --dport 5672 -j ACCEPT
-A INPUT -m state --state NEW -m tcp -p tcp --dport 15672 -j ACCEPT

打开对应该端口

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
-A INPUT -m state --state NEW -m tcp -p tcp --dport 5672 -j ACCEPT
-A INPUT -m state --state NEW -m tcp -p tcp --dport 15672 -j ACCEPT
-A INPUT -j REJECT --reject-with icmp-host-prohibited
-A FORWARD -j REJECT --reject-with icmp-host-prohibited
COMMIT
```


### 通过运行以下命令将RabbitMQ文件的所有权提供给RabbitMQ用户。
```
chown -R rabbitmq:rabbitmq /var/lib/rabbitmq/
```


### 开启web管理控制台插件

```
[hongjunjia@centos-linux ~]$ sudo rabbitmq-plugins enable rabbitmq_management
Enabling plugins on node rabbit@centos-linux:
rabbitmq_management
The following plugins have been configured:
  rabbitmq_management
  rabbitmq_management_agent
  rabbitmq_web_dispatch
Applying plugin configuration to rabbit@centos-linux...
The following plugins have been enabled:
  rabbitmq_management
  rabbitmq_management_agent
  rabbitmq_web_dispatch

started 3 plugins.
```

### 创建rabbitmq账号

```
[hongjunjia@centos-linux ~]$ sudo rabbitmqctl add_user admin admin
Adding user "admin" ...
[hongjunjia@centos-linux ~]$ sudo rabbitmqctl set_user_tags admin administrator
Setting tags for user "admin" to [administrator] ...
[hongjunjia@centos-linux ~]$ 
[hongjunjia@centos-linux ~]$ sudo rabbitmqctl set_permissions -p / admin ".*" ".*" ".*"
Setting permissions for user "admin" in vhost "/" ...
```


### 进入开启web管理控制台

在浏览器中输入如下地址即可登录rabbitmq管理控制台系统：

http://rabbitmq-server:15672/

比如我这里是：
http://10.211.55.5:15672/

![](/images/posts/rabbitmq/rabbitmq6.png)


系统有一个默认账号用户名是guest，密码也是guest，但是这个账号只能在 rabbitmq-server 所在主机上的浏览器中登录，在其他主机登录，可以用上面创建的账号登录，即用户名admin，密码是admin。


![](/images/posts/rabbitmq/rabbitmq7.png)
