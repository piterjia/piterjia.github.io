---
layout: post
title: Linux 防火墙常用命令
categories: Linux
description: Linux防火墙开启、查看端口等常用命令
keywords: Linux, 防护墙, 端口
---

## firewall命令:
```
systemctl status firewalld		 			#查看firewall防火墙状态
firewall-cmd --list-ports					#查看firewall防火墙开放端口
systemctl start firewalld.service			#打开firewall防火墙
systemctl stop firewalld.service			#关闭firewall防火墙
firewall -cmd --reload						#重启firewal防火墙
systemctl disable firewalld.service			#禁止firewall开机启动  

#开放firewall防火墙端口，需重启防火墙生效
firewall-cmd --zone=public --add-port=80/tcp --permanent 	

命令含义:
–zone #作用域
–add-port=80/tcp #添加端口，格式为：端口/通讯协议
–permanent #永久生效，没有此参数重启后失效
```

## iptable防火墙：

```
service iptables status 	#查看iptable防火墙状态
iptables -L -n -v			#查看iptable防火墙规则
systemctl start iptables	#打开iptable防火墙
systemctl stop iptables	    #关闭iptable防火墙
yum install  iptables -y	#安装iptable防火墙
systemctl enable iptables	#开机自启iptable防火墙
systemctl disable firewalld	#开机自动关闭iptable防火墙
iptables -F					#清空iptable的规则
service iptables save  		#保存iptable防火墙规则

iptables -A INPUT -p tcp --dport 80 -j REJECT #禁止来自80端口访问的数据包
iptables -A INPUT -p tcp --dport 80 -j ACCEPT #允许来自80端口访问的数据包

iptables -A OUTPUT -p tcp --sport 80 -j REJECT #禁止从80端口出去的数据包
iptables -A OUTPUT -p tcp --sport 80 -j ACCEPT #允许从80端口出去的数据包
```

### 打开或者禁止端口号，也可以采用修改文件的方式

切换到root用户 打开iptables的配置文件：
```
vi /etc/sysconfig/iptables
```

添加 -A INPUT -m state --state NEW -m tcp -p tcp --dport 端口号 -j ACCEPT

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
-A INPUT -j REJECT --reject-with icmp-host-prohibited
-A FORWARD -j REJECT --reject-with icmp-host-prohibited
COMMIT
```


## 总结

firewall和iptable都是Linux的防火墙，firewall调用了iptable的command去执行内核的netfilter，也就是底层还是使用 iptables 对内核命令动态通信包过滤，firewall是Centos7里的新防火墙命令，相当于iptables 的孩子。

这两种方式，二选一使用即可。



