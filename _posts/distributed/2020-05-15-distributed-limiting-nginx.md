---
layout: post
title: Nginx 限流配置
categories: Distributed
description: Nginx 限流配置
keywords: Nginx, distributed, limiting
---

限流算法简介，以及 Nginx 限流的配置介绍

## 限流算法

### 令牌桶算法

算法思想是：

- 令牌以固定速率产生，并缓存到令牌桶中；
- 令牌桶放满时，多余的令牌被丢弃；
- 请求要消耗等比例的令牌才能被处理；
- 令牌不够时，请求被缓存。

![](/images/posts/distributed/令牌桶算法1.png)


### 漏桶算法

算法思想是：

- 水（请求）从上方倒入水桶，从水桶下方流出（被处理）；
- 来不及流出的水存在水桶中（缓冲），以固定速率流出；
- 水桶满后水溢出（丢弃）。

这个算法的核心是：缓存请求、匀速处理、多余的请求直接丢弃。
相比漏桶算法，令牌桶算法不同之处在于它不但有一只“桶”，还有个队列，这个桶是用来存放令牌的，队列才是用来存放请求的。

![](/images/posts/distributed/漏桶算法1.png)


**从作用上来说，漏桶和令牌桶算法最明显的区别就是是否允许突发流量(burst)的处理，漏桶算法能够强行限制数据的实时传输（处理）速率，对突发流量不做额外处理；而令牌桶算法能够在限制数据的平均传输速率的同时允许某种程度的突发传输。**



## Nginx 限流配置说明

### limit_req_zone 参数配置

```
Syntax:	limit_req zone=name [burst=number] [nodelay];
Default:	—
Context:	http, server, location
```

Nginx 限制IP的连接和并发分别有两个模块：
- limit_req_zone 用来限制单位时间内的请求数，即速率限制,采用的漏桶算法 "leaky bucket"。
- limit_req_conn 用来限制同一时间连接数，即并发限制。

#### 参数说明
```
limit_req_zone $binary_remote_addr zone=one:10m rate=1r/s;
```
- 第一个参数：$binary_remote_addr 表示通过remote_addr这个标识来做限制，“binary_”的目的是缩写内存占用量，是限制同一客户端ip地址。
- 第二个参数：zone=one:10m表示生成一个大小为10M，名字为one的内存区域，用来存储访问的频次信息。
- 第三个参数：rate=1r/s表示允许相同标识的客户端的访问频次，这里限制的是每秒1次，还可以有比如30r/m的。


```
limit_req zone=one burst=5 nodelay;
```
- 第一个参数：zone=one 设置使用哪个配置区域来做限制，与上面limit_req_zone 里的name对应。
- 第二个参数：burst=5，重点说明一下这个配置，burst爆发的意思，这个配置的意思是设置一个大小为5的缓冲区当有大量请求（爆发）过来时，超过了访问频次限制的请求可以先放到这个缓冲区内。
- 第三个参数：nodelay，如果设置，超过访问频次而且缓冲区也满了的时候就会直接返回503，如果没有设置，则所有请求会等待排队。


#### 一个简单的例子：
```
http {
    limit_req_zone $binary_remote_addr zone=one:10m rate=1r/s;
    server {
        location /search/ {
            limit_req zone=one burst=5 nodelay;
        }
} 
```


#### 其他参数

1）当服务器由于limit被限速或缓存时，配置写入日志。延迟的记录比拒绝的记录低一个级别。例子：limit_req_log_level notice延迟的的基本是info。

```
Syntax:	limit_req_log_level info | notice | warn | error;
Default:	
limit_req_log_level error;
Context:	http, server, location
```
2）设置拒绝请求的返回值。值只能设置 400 到 599 之间。

```
Syntax:	limit_req_status code;
Default:	
limit_req_status 503;
Context:	http, server, location
```



### limit_conn_module 参数配置

```
Syntax:	limit_conn zone number;
Default:	—
Context:	http, server, location
```

这个模块用来限制单个IP的请求数。并非所有的连接都被计数。只有在服务器处理了请求并且已经读取了整个请求头时，连接才被计数。

#### 一个简单的例子

一次只允许每个IP地址一个连接。
```
limit_conn_zone $binary_remote_addr zone=addr:10m;

server {
    location /download/ {
        limit_conn addr 1;
    }
```


#### 可以配置多个limit_conn指令。

例如，下面的配置将限制每个客户端IP连接到服务器的数量，同时限制连接到虚拟服务器的总数。
```
limit_conn_zone $binary_remote_addr zone=perip:10m;
limit_conn_zone $server_name zone=perserver:10m;

server {
    ...
    limit_conn perip 10;
    limit_conn perserver 100;
}
```

#### 其他配置信息

1）当服务器限制连接数时，设置所需的日志记录级别。
```
Syntax:	limit_conn_log_level info | notice | warn | error;
Default:	
limit_conn_log_level error;
Context:	http, server, location
```

2）设置拒绝请求的返回值
```
Syntax:	limit_conn_status code;
Default:	
limit_conn_status 503;
Context:	http, server, location
```




## Nginx 完整的示例
```
# 根据IP地址限制速度
# 1） 第一个参数 $binary_remote_addr
#    binary_目的是缩写内存占用，remote_addr表示通过IP地址来限流
# 2） 第二个参数 zone=iplimit:20m
#    iplimit是一块内存区域（记录访问频率信息），20m是指这块内存区域的大小
# 3） 第三个参数 rate=1r/s
#    比如100r/m，标识访问的限流频率
limit_req_zone $binary_remote_addr zone=iplimit:20m rate=1r/s;

# 根据服务器级别做限流
limit_req_zone $server_name zone=serverlimit:10m rate=100r/s;

# 基于连接数的配置
limit_conn_zone $binary_remote_addr zone=perip:20m;
limit_conn_zone $server_name zone=perserver:20m;


    server {
        server_name www.imooc-training.com;
        location /access-limit/ {
            proxy_pass http://127.0.0.1:10086/;

            # 基于IP地址的限制
            # 1） 第一个参数zone=iplimit => 引用limit_req_zone中的zone变量
            # 2） 第二个参数burst=2，设置一个大小为2的缓冲区域，当大量请求到来。
            #     请求数量超过限流频率时，将其放入缓冲区域
            # 3) 第三个参数nodelay=> 缓冲区满了以后，直接返回503异常
            limit_req zone=iplimit burst=2 nodelay;

            # 基于服务器级别的限制
            # 通常情况下，server级别的限流速率是最大的
            limit_req zone=serverlimit burst=100 nodelay;

            # 每个server最多保持100个连接
            limit_conn perserver 100;
            # 每个IP地址最多保持1个连接
            limit_conn perip 5;

            # 异常情况，返回504（默认是503）
            limit_req_status 504;
            limit_conn_status 504;
        }


        location /download/ {
            limit_rate_after 100m;
            limit_rate 256k;
        }
    }
```




参考资料：

https://www.cnblogs.com/biglittleant/p/8979915.html
