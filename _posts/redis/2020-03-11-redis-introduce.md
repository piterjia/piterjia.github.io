---
layout: post
title: redis 介绍与数据结构
categories: Redis
description: redis 介绍与数据结构
keywords: redis, spring
---

## 一、Redis 简介

Redis 是一个开源，高级的键值存储和一个适用的解决方案，用于构建高性能，可扩展的 Web 应用程序。在 Redis 中都采用键值对的方式，存储数据。

### Redis 的优点
以下是 Redis 的一些优点：

- 异常快 - Redis 非常快；每秒可执行大约 110000 次的设置(SET)操作，每秒大约可执行 81000 次的读取/获取(GET)操作。
- 支持丰富的数据类型；Redis 支持开发人员常用的大多数数据类型，例如列表，集合，排序集和散列等等。这使得 Redis 很容易被用来解决各种问题，因为我们知道哪些问题可以更好使用地哪些数据类型来处理解决。
- 操作具有原子性；所有 Redis 操作都是原子操作，这确保如果两个客户端并发访问，Redis 服务器能接收更新的值。
- 多实用工具； Redis 是一个多实用工具，可用于多种用例，如：缓存，消息队列(Redis 本地支持发布/订阅)，应用程序中的任何短期数据，例如，web应用程序中的会话，网页命中计数等。

### Redis 的安装

参考我的这篇文章
[Redis 安装](2020-03-08-redis-install.md)


## 二、Redis 五种基本数据结构

Redis 有 5 种基础数据结构，它们分别是：string(字符串)、list(列表)、hash(字典)、set(集合) 和 zset(有序集合)。这 5 种是 Redis 相关知识中最基础、最重要的部分，下面我们通过一些实践来给大家分别讲解一下。

### 1 字符串 string

Redis 中的字符串是一种 动态字符串，这意味着使用者可以修改，它的底层实现有点类似于 Java 中的 ArrayList，有一个字符数组。

**对字符串的基本操作**

安装好 Redis，我们可以使用 redis-cli 来对 Redis 进行命令行的操作，当然 Redis 官方也提供了在线的调试器，你也可以在里面敲入命令进行操作：http://try.redis.io/#run

#### 设置和获取键值对

```
> SET key value
OK
> GET key
"value"
```

通常使用 SET 和 GET 来设置和获取字符串值。

当 key 存在时，SET 命令会覆盖掉你上一次设置的值：

```
> SET key newValue
OK
> GET key
"newValue"
```

另外你还可以使用 EXISTS 和 DEL 关键字来查询是否存在和删除键值对：

```
> EXISTS key
(integer) 1
> DEL key
(integer) 1
> GET key
(nil)
```

#### 批量设置键值对

```
> SET key1 value1
OK
> SET key2 value2
OK
> MGET key1 key2 key3    # 返回一个列表
1) "value1"
2) "value2"
3) (nil)
> MSET key1 value1 key2 value2
> MGET key1 key2
1) "value1"
2) "value2"
```

#### 过期和 SET 命令扩展

可以对 key 设置过期时间，到时间会被自动删除，这个功能常用来控制缓存的失效时间。(过期可以是任意数据结构)

```
> SET key value1
> GET key
"value1"
> EXPIRE key 5    # 5s 后过期，key为键
...                # 等待 5s
> GET key
(nil)
```

等价于 SET + EXPIRE 的 SETEX 命令：

```
> SETEX key 5 value1
...                # 等待 5s 后获取
> GET key
(nil)

> SETNX key value1  # 如果 key 不存在则 SET 成功
(integer) 1
> SETNX key value2  # 如果 key 存在则 SET 失败
(integer) 0
> GET key
"value"             # 没有改变 
```

#### 计数

如果 value 是一个整数，还可以对它使用 INCR 命令进行 原子性 的自增操作，这意味着及时多个客户端对同一个 key 进行操作，也决不会导致竞争的情况：

```
> SET counter 100
> INCR counter
(integer) 101
> INCRBY counter 50
(integer) 151
```

#### 返回原值的 GETSET 命令

它的功能跟它名字一样：为 key 设置一个值并返回原值：

```
> SET key value
> GETSET key value1
"value"
```

这可以对于某一些需要隔一段时间就统计的 key 很方便的设置和查看。

例如：系统每当由用户进入的时候你就是用 INCR 命令操作一个 key，当需要统计时候你就把这个 key 使用 GETSET 命令重新赋值为 0，这样就达到了统计的目的。


### 2）列表 list

Redis 的列表相当于 Java 语言中的 LinkedList，注意它是链表而不是数组。这意味着 list 的插入和删除操作非常快，时间复杂度为 O(1)，但是索引定位很慢，时间复杂度为 O(n)。

#### 链表的基本操作
- LPUSH 和 RPUSH 分别可以向 list 的左边（头部）和右边（尾部）添加一个新元素；
- LRANGE 命令可以从 list 中取出一定范围的元素；
- LINDEX 命令可以从 list 中取出指定下表的元素，相当于 Java 链表操作中的 get(int index) 操作；

示范：

```
> rpush mylist A
(integer) 1
> rpush mylist B
(integer) 2
> lpush mylist first
(integer) 3
> lrange mylist 0 -1    # -1 表示倒数第一个元素, 这里表示从第一个元素到最后一个元素，即所有
1) "first"
2) "A"
3) "B"
```

#### list 实现队列

队列是先进先出的数据结构，常用于消息排队和异步逻辑处理，它会确保元素的访问顺序：

```
> RPUSH books python java golang
(integer) 3
> LPOP books
"python"
> LPOP books
"java"
> LPOP books
"golang"
> LPOP books
(nil)
```


#### list 实现栈

栈是先进后出的数据结构，跟队列正好相反：

```
> RPUSH books python java golang
> RPOP books
"golang"
> RPOP books
"java"
> RPOP books
"python"
> RPOP books
(nil)
```

### 3）字典 hash

Redis 中的字典相当于 Java 中的 HashMap，内部实现也差不多类似，都是通过 "数组 + 链表" 的链地址法来解决部分哈希冲突。

#### 字典的基本操作

hash 结构的存储消耗要高于单个字符串，所以到底该使用 hash 还是字符串，需要根据实际情况决定：

```
> HSET books java "think in java"    # 命令行的字符串如果包含空格则需要使用引号包裹
(integer) 1
> HSET books python "python cookbook"
(integer) 1
> HGETALL books    # key 和 value 间隔出现
1) "java"
2) "think in java"
3) "python"
4) "python cookbook"
> HGET books java
"think in java"
> HSET books java "head first java"  
(integer) 0        # 因为是更新操作，所以返回 0
> HMSET books java "effetive  java" python "learning python"    # 批量操作
OK
```

### 4）集合 set

Redis 的集合相当于 Java 语言中的 HashSet，它内部的键值对是无序、唯一的。它的内部实现相当于一个特殊的字典，字典中所有的 value 都是一个值 NULL。

#### 集合 set 的基本使用

```
> SADD books java
(integer) 1
> SADD books java    # 重复
(integer) 0
> SADD books python golang
(integer) 2
> SMEMBERS books    # 注意顺序，set 是无序的 
1) "java"
2) "python"
3) "golang"
> SISMEMBER books java    # 查询某个 value 是否存在，相当于 contains
(integer) 1
> SCARD books    # 获取长度
(integer) 3
> SPOP books     # 弹出一个
"java"
```


### 5）有序列表 zset

这可能使 Redis 最具特色的一个数据结构了，它类似于 Java 中 SortedSet 和 HashMap 的结合体，一方面它是一个 set，保证了内部 value 的唯一性，另一方面它可以为每个 value 赋予一个 score 值，用来代表排序的权重。

它的内部实现用的是一种叫做 「跳跃表」 的数据结构实现的。


#### 有序列表 zset 基础操作

```
> ZADD books 9.0 "think in java"
> ZADD books 8.9 "java concurrency"
> ZADD books 8.6 "java cookbook"

> ZRANGE books 0 -1     # 按 score 排序列出，参数区间为排名范围
1) "java cookbook"
2) "java concurrency"
3) "think in java"

> ZREVRANGE books 0 -1  # 按 score 逆序列出，参数区间为排名范围
1) "think in java"
2) "java concurrency"
3) "java cookbook"

> ZCARD books           # 相当于 count()
(integer) 3

> ZSCORE books "java concurrency"   # 获取指定 value 的 score
"8.9000000000000004"                # 内部 score 使用 double 类型进行存储，所以存在小数点精度问题

> ZRANK books "java concurrency"    # 排名
(integer) 1

> ZRANGEBYSCORE books 0 8.91        # 根据分值区间遍历 zset
1) "java cookbook"
2) "java concurrency"

> ZRANGEBYSCORE books -inf 8.91 withscores  # 根据分值区间 (-∞, 8.91] 遍历 zset，同时返回分值。inf 代表 infinite，无穷大的意思。
1) "java cookbook"
2) "8.5999999999999996"
3) "java concurrency"
4) "8.9000000000000004"

> ZREM books "java concurrency"             # 删除 value
(integer) 1
> ZRANGE books 0 -1
1) "java cookbook"
2) "think in java"
```


## 参考资料

Redis 数据类型介绍 ： http://www.redis.cn/topics/data-types-intro.html

阿里云 Redis 开发规范 ： https://www.infoq.cn/article/K7dB5AFKI9mr5Ugbs_px