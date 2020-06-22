---
layout: post
title: Linux 安装 Mysql
categories: Mysql
description: Linux 安装 Mysql教程
keywords: mysql, distributed, Linux
---

Mysql数据库的安装对于开发者来说，是我们必然会面对的问题，它的安装过程其实并不复杂，并且网络上的安装教程也非常多，有些比较老，有些按照步骤安装完毕也不能用。
下面记录了我在Linux环境下安装Mysql的完整过程，如有错误或遗漏，欢迎指正。


# 安装前准备
## 1.检查mysql用户组和用户是否存在，如果没有，则创建
```
> cat /etc/group | grep mysql
> cat /etc/passwd | grep mysql
> groupadd mysql
> useradd -r -g mysql -s /bin/false mysql
```

## 2.从官网下载用于linux的mysql安装包
我们选择二进制预编译版本的mysql，无需configure ，make make install 等步骤，只需配置一下即可使用，卸载也方便，直接删除即可。可以自行调整编译参数，最大化地定制安装结果。


# 安装mysql

## 1.找到mysql安装包 mysql-8.0.20-linux-glibc2.12-x86_64.tar.xz
执行解压命令
```
> tar xzvf mysql-8.0.20-linux-glibc2.12-x86_64.tar.xz
> ls
mysql-8.0.20-linux-glibc2.12-x86_64.tar.xz
mysql-8.0.20-linux-glibc2.12-x86_64
```

解压完成后，可以看到当前目录下多了个解压文件，移动文件到/usr/local/mysql 执行移动命令
```
> mv mysql-8.0.20-linux-glibc2.12-x86_64 /usr/local/mysql
```

## 2、接下来，创建到安装目录的软连接

```
> ln -s /usr/local/mysql/bin/mysql /usr/bin/mysql
```

或者：
ln命令生成指向安装目录的符号链接。这使您可以更容易地将其称为/usr/local/mysql。

为了避免在使用MySQL时总是需要键入客户端程序的路径名，可以将/usr/local/MySQL/bin目录添加到path变量中：

```
> export PATH=$PATH:/usr/local/mysql/bin
```

## 3.在/usr/local/mysql目录下创建data目录

本部分的参考资料
https://dev.mysql.com/doc/refman/8.0/en/data-directory-initialization.html

### 1）切换到mysql的安装目录 /usr/local/mysql
```
cd /usr/local/mysql
```
### 2）secure_file_priv系统变量限制对特定目录的导入和导出操作。创建一个目录，他可以用来存储变量的值。

```
> mkdir mysql-files
```

将该目录用户和组的所有权，授予mysql用户和mysql组，并适当设置目录权限:

```
> chown mysql:mysql mysql-files
> chmod 750 mysql-files
```

### 3）初始化数据目录，包括mysql模式。
例如：其中包含初始mysql授予表，用于确定如何允许用户连接到服务器等等。

```
> bin/mysqld --initialize --user=mysql
```

### 4) 如果您希望部署具有自动安全连接支持的服务器，请使用mysql_ssl_rsa_setup实用程序创建默认的SSL和RSA文件:

```
> bin/mysql_ssl_rsa_setup
```

## 4 编译安装并初始化mysql，务必记住初始化输出末尾的密码（数据库管理员临时密码）

切换到mysql的安装目录 /usr/local/mysql

```
> cd /usr/local/mysql
```

开始安装
指定安装目录或数据目录的正确位置，——basedir或——datadir。

```
> bin/mysqld --initialize --user=mysql
  --basedir=/usr/local/mysql
  --datadir=/usr/local/mysql/data
```

## 5 初始化后的根密码分配

编辑配置文件my.cnf

```
> vim /etc/my.cnf
```

添加配置如下
```
[mysqld]
datadir=/usr/local/mysql/data
basedir=/usr/local/mysql
port = 3306
# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0
```

## 6.启动mysql服务器
```
> /usr/local/mysql/support-files/mysql.server start
```

## 7.添加软链接，并重启mysql服务
```
> ln -s /usr/local/mysql/support-files/mysql.server /etc/init.d/mysql
> ln -s /usr/local/mysql/bin/mysql /usr/bin/mysql
> service mysql restart
```

## 8.登录mysql， 修改密码（密码为步骤4生成的临时密码）
```
> mysql -u root -p
Enter password:
mysql > set password for root@localhost = password('yourpass');
```

如果执行成功，输出如下

否则执行
```
> mysql -u root -p
Enter password:
mysql > alter user 'root'@'localhost' IDENTIFIED BY '123456';
```

## 10.开放远程连接
```
mysql>use mysql;
msyql>update user set user.Host='%' where user.User='root';
mysql>flush privileges;
```

## 11 解决虚拟机linux端mysql数据库无法远程访问

### 1）在控制台执行 
```
mysql -u root -p mysql
```
entOS系统提示输入数据库root用户的密码，输入完成后即进入mysql控制台

### 2）在mysql控制台执行 
```
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'MyPassword' WITH GRANT OPTION; FLUSH PRIVILEGES;
```
在mysql控制台执行命令中'root'@'%' 可以这样理解: root是用户名，%是主机名或IP地址，这里的%代表任意主机或IP地址，你也可替换成任意其它用户名或指定唯一的IP地址；'MyPassword'是给授权用户指定的登录数据库的密码；另外需要说明一点的是我这里的都是授权所有权限，可以指定部分权

### 3)切换到root用户 打开iptables的配置文件：

```
vi /etc/sysconfig/iptables
```

添加 -A INPUT -m state --state NEW -m tcp -p tcp --dport 3306 -j ACCEPT

打开 3306 端口

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

全部修改完之后重启 iptables:service iptables restart

可以验证一下是否规则都已经生效：iptables -L 这样，我们就完成了CentOS防火墙的设置修改。

