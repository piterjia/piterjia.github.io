---
layout: post
title: 布隆过滤器原理与 spring 集成
categories: Redis
description: 布隆过滤器原理与 spring 集成
keywords: 布隆过滤器, redis, spring
---

布隆过滤器(Bloom Filter) 是一种专门用来解决去重问题的高级数据结构。它有有那么一点点不精确，存在一定的误判概率，但它能在解决去重的同时，在空间上能节省至少 90% 以上。

## 布隆过滤器是什么

布隆过滤器是一个叫“布隆”的人提出的，它本身是一个很长的二进制向量，既然是二进制的向量，那么显而易见的，存放的不是0，就是1。实际上你也可以把它 简单理解 为一个不怎么精确的 set 结构，当你使用它的 contains 方法判断某个对象是否存在时，它可能会误判。但是布隆过滤器也不是特别不精确，只要参数设置的合理，它的精确度可以控制的相对足够精确，只会有小小的误判概率。

现在我们新建一个长度为16的布隆过滤器，默认值都是0，就像下面这样：

![](/images/posts/redis/bloom1.png)

现在需要添加一个数据：

我们通过某种计算方式，比如 Hash 算法，计算出了 Hash value = 5，我们就把下标为 5 的格子改成 1 ，就像下面这样：

![](/images/posts/redis/bloom2.png)

我们又要新添一个数据，计算出了 Hash value = 9，我们就把下标为9的格子改成 1，就像下面这样：

![](/images/posts/redis/bloom3.png)

我们再次要新添一个数据，计算出了 Hash value = 2，我们就把下标为2的格子改成 1，就像下面这样：

![](/images/posts/redis/bloom4.png)


这样，刚才添加的数据就占据了布隆过滤器“5”，“9”，“2”三个格子。

可以看出，仅仅从布隆过滤器本身而言，根本没有存放完整的数据，只是运用一系列随机映射函数计算出位置，然后填充二进制向量。

这有什么用呢？比如现在再给你一个数据，你要判断这个数据是否重复，你怎么做？

你只需利用上面固定的计算方式，计算出这个数据占据哪些格子，然后看看这些格子里面放置的是否都是1，如果有一个格子不为1，那么就代表这个数字不在其中。

>注意，某个数值通过哈希算法计算出对应的格子为 1，不一定代表给定的数据一定重复，也许其他数据经过计算可能也是对应这个格子。比如我们需要判断对象是否相等，是不可以仅仅判断他们的哈希值是否相等的。

也就是说布隆过滤器只能判断数据是否一定不存在，而无法判断数据是否一定存在。

布隆过滤器很难做到删除数据的。比如你要删除刚才给你的数据，你把“5”，“9”，“2”三个格子都改成了0，但是可能其他的数据也映射到了“5”，“9”，“2”三个格子啊，这不就乱套了吗？

布隆过滤器的优缺点：
- 优点：由于存放的不是完整的数据，所以占用的内存很少，而且新增，查询速度够快；
- 缺点：随着数据的增加，误判率随之增加；无法做到删除数据；只能判断数据是否一定不存在，而无法判断数据是否一定存在。

可以看到，布隆过滤器的优点和缺点一样明显。

在上文中，我举的例子二进制向量长度为16，在实际开发中，如果你要添加大量的数据，仅仅16位是远远不够的，为了让误判率降低，我们还可以用更多的随机映射函数、更长的二进制向量去计算位置。

## 布隆过滤器的使用场景

基于上述的功能，我们大致可以把布隆过滤器用于以下的场景之中：

- 大数据判断是否存在：这就可以实现出上述的去重功能，如果你的服务器内存足够大的话，那么使用 HashMap 可能是一个不错的解决方案，但是当数据量起来之后，还是只能考虑布隆过滤器。
- 解决缓存穿透：我们经常会把一些热点数据放在 Redis 中当作缓存，例如产品详情。 通常一个请求过来之后我们会先查询缓存，而不用直接读取数据库，这是提升性能最简单也是最普遍的做法，但是如果一直请求一个不存在的缓存，那么此时一定不存在缓存，那就会有大量请求直接打到数据库上，造成缓存穿透，布隆过滤器也可以用来解决此类问题。


## 布隆过滤器的使用

Redis 官方提供的布隆过滤器到了 Redis 4.0 提供了插件功能之后才正式登场。布隆过滤器作为一个插件加载到 Redis Server 中，给 Redis 提供了强大的布隆去重功能。下面我们来体验一下 Redis 的布隆过滤器。

### 布隆过滤器的基本用法

布隆过滤器有两个基本指令，bf.add 添加元素，bf.exists 查询元素是否存在，它的用法和 set 集合的 sadd 和 sismember 差不多。注意 bf.add 只能一次添加一个元素，如果想要一次添加多个，就需要用到 bf.madd 指令。同样如果需要一次查询多个元素是否存在，就需要用到 bf.mexists 指令。

```
127.0.0.1:6379> bf.add codehole user1
(integer) 1
127.0.0.1:6379> bf.add codehole user2
(integer) 1
127.0.0.1:6379> bf.add codehole user3
(integer) 1
127.0.0.1:6379> bf.exists codehole user1
(integer) 1
127.0.0.1:6379> bf.exists codehole user2
(integer) 1
127.0.0.1:6379> bf.exists codehole user3
(integer) 1
127.0.0.1:6379> bf.exists codehole user4
(integer) 0
127.0.0.1:6379> bf.madd codehole user4 user5 user6
1) (integer) 1
2) (integer) 1
3) (integer) 1
127.0.0.1:6379> bf.mexists codehole user4 user5 user6 user7
1) (integer) 1
2) (integer) 1
3) (integer) 1
4) (integer) 0
```

上面使用的布隆过过滤器只是默认参数的布隆过滤器，它在我们第一次 add 的时候自动创建。Redis 也提供了可以自定义参数的布隆过滤器，只需要在 add 之前使用 bf.reserve 指令显式创建就好了。如果对应的 key 已经存在，bf.reserve 会报错。

bf.reserve 有三个参数，分别是 key、error_rate (错误率) 和 initial_size：
- error_rate 越低，需要的空间越大，对于不需要过于精确的场合，设置稍大一些也没有关系。
- initial_size 表示预计放入的元素数量，当实际数量超过这个值时，误判率就会提升，所以需要提前设置一个较大的数值避免超出导致误判率升高；

如果不适用 bf.reserve，默认的 error_rate 是 0.01，默认的 initial_size 是 100。

## guava实现布隆过滤器

首先我们需要在项目中引入 Guava 的依赖：
```
<dependency>
    <groupId>com.google.guava</groupId>
    <artifactId>guava</artifactId>
    <version>28.0-jre</version>
</dependency>
```

实际使用如下：

我们创建了一个最多存放 最多 1500 个整数的布隆过滤器，并且我们可以容忍误判的概率为百分之（0.01）

```
// 创建布隆过滤器对象
BloomFilter<Integer> filter = BloomFilter.create(
        Funnels.integerFunnel(),
        1500,
        0.01);
// 判断指定元素是否存在
System.out.println(filter.mightContain(1));
System.out.println(filter.mightContain(2));
// 将元素添加进布隆过滤器
filter.put(1);
filter.put(2);
System.out.println(filter.mightContain(1));
System.out.println(filter.mightContain(2));
```

在我们的示例中，当 mightContain() 方法返回 true 时，我们可以 99％ 确定该元素在过滤器中，当过滤器返回 false 时，我们可以 100％ 确定该元素不存在于过滤器中。

Guava 提供的布隆过滤器有一个重大的缺陷就是只能单机使用 （另外，容量扩展也不容易），而现在互联网一般都是分布式的场景。为了解决这个问题，我们就需要用到 Redis 中的布隆过滤器了。


## 结合SpringBoot使用 redis 布隆过滤器

redis 环境搭建  以及 spring boot 工程配置，不是本文的重点，这里不再详细讲述。

可以参考我之前的文章
[redis 安装](2020-03-08-redis-install.md)

[spring boot 集成 redis ](../spring/2020-03-10-spring-boot-redis-integration.md)


### 定义布隆算法

布隆算法实现如下：

```
package org.sky.platform.util;
 
import com.google.common.base.Preconditions;
import com.google.common.hash.Funnel;
import com.google.common.hash.Hashing;
 
public class BloomFilterHelper<T> {
	private int numHashFunctions;
 
	private int bitSize;
 
	private Funnel<T> funnel;
 
	public BloomFilterHelper(Funnel<T> funnel, int expectedInsertions, double fpp) {
		Preconditions.checkArgument(funnel != null, "funnel不能为空");
		this.funnel = funnel;
		bitSize = optimalNumOfBits(expectedInsertions, fpp);
		numHashFunctions = optimalNumOfHashFunctions(expectedInsertions, bitSize);
	}
 
	int[] murmurHashOffset(T value) {
		int[] offset = new int[numHashFunctions];
 
		long hash64 = Hashing.murmur3_128().hashObject(value, funnel).asLong();
		int hash1 = (int) hash64;
		int hash2 = (int) (hash64 >>> 32);
		for (int i = 1; i <= numHashFunctions; i++) {
			int nextHash = hash1 + i * hash2;
			if (nextHash < 0) {
				nextHash = ~nextHash;
			}
			offset[i - 1] = nextHash % bitSize;
		}
 
		return offset;
	}
 
	/**
	 * 计算bit数组的长度
	 */
	private int optimalNumOfBits(long n, double p) {
		if (p == 0) {
			p = Double.MIN_VALUE;
		}
		return (int) (-n * Math.log(p) / (Math.log(2) * Math.log(2)));
	}
 
	/**
	 * 计算hash方法执行次数
	 */
	private int optimalNumOfHashFunctions(long n, long m) {
		return Math.max(1, (int) Math.round((double) m / n * Math.log(2)));
	}
}
```


### 定义 redis 工具类

其中就包括 根据给定的布隆过滤器添加值，以及根据给定的布隆过滤器判断值是否存在。

然后我们就可以愉快的使用了。

```
package org.sky.platform.util;
 
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Component;
 
import java.util.Collection;
import java.util.Date;
import java.util.Set;
import java.util.concurrent.TimeUnit;
import java.util.stream.Collectors;
import java.util.stream.Stream;
import com.google.common.base.Preconditions;
import org.springframework.data.redis.core.RedisTemplate;
 
@Component
public class RedisUtil {
	@Autowired
	private RedisTemplate<String, String> redisTemplate;
 
	/**
	 * 默认过期时长，单位：秒
	 */
	public static final long DEFAULT_EXPIRE = 60 * 60 * 24;
 
	/**
	 * 不设置过期时长
	 */
	public static final long NOT_EXPIRE = -1;
 
	public boolean existsKey(String key) {
		return redisTemplate.hasKey(key);
	}
 
	/**
	 * 重名名key，如果newKey已经存在，则newKey的原值被覆盖
	 *
	 * @param oldKey
	 * @param newKey
	 */
	public void renameKey(String oldKey, String newKey) {
		redisTemplate.rename(oldKey, newKey);
	}
 
	/**
	 * newKey不存在时才重命名
	 *
	 * @param oldKey
	 * @param newKey
	 * @return 修改成功返回true
	 */
	public boolean renameKeyNotExist(String oldKey, String newKey) {
		return redisTemplate.renameIfAbsent(oldKey, newKey);
	}
 
	/**
	 * 删除key
	 *
	 * @param key
	 */
	public void deleteKey(String key) {
		redisTemplate.delete(key);
	}
 
	/**
	 * 删除多个key
	 *
	 * @param keys
	 */
	public void deleteKey(String... keys) {
		Set<String> kSet = Stream.of(keys).map(k -> k).collect(Collectors.toSet());
		redisTemplate.delete(kSet);
	}
 
	/**
	 * 删除Key的集合
	 *
	 * @param keys
	 */
	public void deleteKey(Collection<String> keys) {
		Set<String> kSet = keys.stream().map(k -> k).collect(Collectors.toSet());
		redisTemplate.delete(kSet);
	}
 
	/**
	 * 设置key的生命周期
	 *
	 * @param key
	 * @param time
	 * @param timeUnit
	 */
	public void expireKey(String key, long time, TimeUnit timeUnit) {
		redisTemplate.expire(key, time, timeUnit);
	}
 
	/**
	 * 指定key在指定的日期过期
	 *
	 * @param key
	 * @param date
	 */
	public void expireKeyAt(String key, Date date) {
		redisTemplate.expireAt(key, date);
	}
 
	/**
	 * 查询key的生命周期
	 *
	 * @param key
	 * @param timeUnit
	 * @return
	 */
	public long getKeyExpire(String key, TimeUnit timeUnit) {
		return redisTemplate.getExpire(key, timeUnit);
	}
 
	/**
	 * 将key设置为永久有效
	 *
	 * @param key
	 */
	public void persistKey(String key) {
		redisTemplate.persist(key);
	}
 
	/**
	 * 根据给定的布隆过滤器添加值
	 */
	public <T> void addByBloomFilter(BloomFilterHelper<T> bloomFilterHelper, String key, T value) {
		Preconditions.checkArgument(bloomFilterHelper != null, "bloomFilterHelper不能为空");
		int[] offset = bloomFilterHelper.murmurHashOffset(value);
		for (int i : offset) {
			redisTemplate.opsForValue().setBit(key, i, true);
		}
	}
 
	/**
	 * 根据给定的布隆过滤器判断值是否存在
	 */
	public <T> boolean includeByBloomFilter(BloomFilterHelper<T> bloomFilterHelper, String key, T value) {
		Preconditions.checkArgument(bloomFilterHelper != null, "bloomFilterHelper不能为空");
		int[] offset = bloomFilterHelper.murmurHashOffset(value);
		for (int i : offset) {
			if (!redisTemplate.opsForValue().getBit(key, i)) {
				return false;
			}
		}
 
		return true;
	}
}

```


## 参考文档：

https://blog.csdn.net/lifetragedy/article/details/103945885

https://snailclimb.gitee.io/javaguide/#/docs/database/Redis/redis-collection/Redis(5)%E2%80%94%E2%80%94%E4%BA%BF%E7%BA%A7%E6%95%B0%E6%8D%AE%E8%BF%87%E6%BB%A4%E5%92%8C%E5%B8%83%E9%9A%86%E8%BF%87%E6%BB%A4%E5%99%A8



