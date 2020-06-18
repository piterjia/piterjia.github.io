---
layout: post
title:【Spring Boot系列】Spring Boot 集成 redis
categories: Spring
description: Spring Boot 集成 redis
keywords: SpringBoot, redis, Jedis, Lettuce
---



从 Spring Boot 2.x 开始 Lettuce 已取代 Jedis 成为首选 Redis 的客户端。当然 Spring Boot 2.x 仍然支持 Jedis，并且你可以任意切换客户端。

本文不讲解 redis 的安装，具体的安装见这一篇博客：

[redis 安装](../redis/2020-03-08-redis-install.md)


## Lettuce

Lettuce 是一个可伸缩的线程安全的 Redis 客户端，支持同步、异步和响应式模式。多个线程可以共享一个连接实例，而不必担心多线程并发问题。它基于优秀 Netty NIO 框架构建，支持 Redis 的高级功能，如 Sentinel、集群、流水线、自动重新连接和 Redis 数据模型。

Jedis 在实现上是直接连接的 redis server，如果在多线程环境下是非线程安全的，这个时候只有使用连接池，为每个 Jedis 实例增加物理连接。

Lettuce 的连接是基于 Netty 的，连接实例 (StatefulRedisConnection) 可以在多个线程间并发访问，应为 StatefulRedisConnection 是线程安全的，所以一个连接实例  (StatefulRedisConnection) 就可以满足多线程环境下的并发访问，当然这个也是可伸缩的设计，一个连接实例不够的情况也可以按需增加连接实例。
 
## Spring Boot 2.0 集成 redis

一般需要4步
1. 引入依赖
2. 配置 redis
3. 自定义 RedisTemplate (推荐)
4. 自定义 redis 操作类 (推荐)

### 引入依赖

引入 lettuce pool 缓存连接池 、redis 以及 一些常用的依赖。

```
	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-devtools</artifactId>
			<scope>runtime</scope>
			<optional>true</optional>
		</dependency>
		<dependency>
			<groupId>org.projectlombok</groupId>
			<artifactId>lombok</artifactId>
			<optional>true</optional>
		</dependency>

		<!--redis-->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-data-redis</artifactId>
		</dependency>

		<!-- lettuce pool 缓存连接池 -->
		<dependency>
			<groupId>org.apache.commons</groupId>
			<artifactId>commons-pool2</artifactId>
		</dependency>

		<!--jackson-->
		<dependency>
			<groupId>com.fasterxml.jackson.core</groupId>
			<artifactId>jackson-databind</artifactId>
		</dependency>

	</dependencies>
```

- 如果用的是 lettuce 客户端，需要引入 commons-pool2 连接池。
- 如果想用 json 序列化 redis 的 value 值，需要引入 jackson

### 配置 redis

详细的配置如下，已经添加备注。

```
spring.application.name=redis-demo
server.port=8090

# redis 服务端相关配置
# 服务器地址
spring.redis.host=10.211.55.5
# 端口号
spring.redis.port=6379
# 密码，默认为 null
spring.redis.password=redis123456
# 使用的数据库，默认选择下标为0的数据库
spring.redis.database=0
# 客户端超时时间,默认是2000ms
spring.redis.timeout=2000ms


## jedis 客户端配置(从 Spring Boot 2.x 开始，不再推荐使用 jedis 客户端)
## 建立连接最大等待时间，默认1ms，超出该时间会抛异常。设为-1表示无限等待，直到分配成功。
#spring.redis.jedis.pool.max-wait=1ms
## 最大连连接数，默认为8，负值表示没有限制
#spring.redis.jedis.pool.max-active=8
## 最大空闲连接数,默认8。负值表示没有限制
#spring.redis.jedis.pool.max-idle=8
## 最小空闲连接数,默认0。
#spring.redis.jedis.pool.min-idle=0


# lettuce 客户端配置(从 Spring Boot 2.x 开始，推荐使用 lettuce 客户端)
# 建立连接最大等待时间，默认1ms，超出该时间会抛异常。设为-1表示无限等待，直到分配成功。
spring.redis.lettuce.pool.max-wait=1ms
# 最大连连接数，默认为8，负值表示没有限制
spring.redis.lettuce.pool.max-active=8
# 最大空闲连接数,默认8。负值表示没有限制
spring.redis.lettuce.pool.max-idle=8
# 最小空闲连接数,默认0。
spring.redis.lettuce.pool.min-idle=0
# 设置关闭连接的超时时间
spring.redis.lettuce.shutdown-timeout=100ms
```

### 自定义 RedisTemplate

RedisTemplate 是 spring 为我们提供的 redis 操作类，通过它我们可以完成大部分 redis 操作。

只要我们引入了 redis 依赖，并将 redis 的连接信息配置正确，springboot 就会根据我们的配置会给我们生成默认 RedisTemplate。

但是默认生成的 RedisTemplate 有两个地方不是很符合日常开发中的使用习惯
- 默认生成的 RedisTemplate<K, V>  接收的key和value为泛型，经常需要类型转换，直接使用不是很方便。
- 默认生成的 RedisTemplate 序列化时，使用的是 JdkSerializationRedisSerializer ，存储到 redis 中后，内容为二进制字节，不利于查看原始内容

对于第一个问题，一般习惯将 RedisTemplate<K, V> 改为 RedisTemplate<String, Object>，即接收的 key 为 String 类型,接收的 value 为 Object 类型。

对于第二个问题,一般会把数据序列化为 json 格式，然后存储到 redis 中，序列化成 json 格式还有一个好处就是跨语言，其他语言也可以读取你存储在 redis 中的内容。

为了实现上面两个目的，我们需要自定义自己的 RedisTemplate。

如下，创建一个 config 类，在里面配置 自定义的 RedisTemplate：

```
package com.piter.config;

import com.fasterxml.jackson.annotation.JsonAutoDetect;
import com.fasterxml.jackson.annotation.PropertyAccessor;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.boot.autoconfigure.AutoConfigureAfter;
import org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.Jackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.StringRedisSerializer;

@Configuration
// 控制配置类的加载顺序,先加载 RedisAutoConfiguration.class 再加载该类,这样才能覆盖默认的 RedisTemplate
@AutoConfigureAfter(RedisAutoConfiguration.class)
public class RedisConfig {
    /**
     * 自定义 redisTemplate （方法名一定要叫 redisTemplate 因为 @Bean 是根据方法名配置这个bean的name的）
     * 默认的 RedisTemplate<K,V> 为泛型，使用时不太方便，自定义为 <String, Object>
     * 默认序列化方式为 JdkSerializationRedisSerializer 序列化后的内容不方便阅读，改为序列化成 json
     *
     * @param redisConnectionFactory
     * @return
     */
    @Bean
    RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) {
        // 配置 json 序列化器 - Jackson2JsonRedisSerializer
        Jackson2JsonRedisSerializer jacksonSerializer = new Jackson2JsonRedisSerializer<>(Object.class);
        ObjectMapper objectMapper = new ObjectMapper();
        objectMapper.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        objectMapper.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL);
        jacksonSerializer.setObjectMapper(objectMapper);

        // 创建并配置自定义 RedisTemplateRedisOperator
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory);
        // 将 key 序列化成字符串
        template.setKeySerializer(new StringRedisSerializer());
        // 将 hash 的 key 序列化成字符串
        template.setHashKeySerializer(new StringRedisSerializer());
        // 将 value 序列化成 json
        template.setValueSerializer(jacksonSerializer);
        // 将 hash 的 value 序列化成 json
        template.setHashValueSerializer(jacksonSerializer);
        template.afterPropertiesSet();
        return template;
    }
}
```



### 自定义 Redis 操作类

虽然 RedisTemplate 已经对 redis 的操作进行了一定程度的封装，但是直接使用还是有些不方便，实际开发中，一般会对 RedisTemplate 做进一步封装，形成方便使用的Redis 操作类。

#### BaseRedisOperator

该类的设计思路是，与 Redis 命令语法及格式保持一致，以达到使用该类与直接使用 Redis 命令相似的体验。

由于 Redis 命令众多，并没有实现全部的命令，而是实现了常用的大部分命令。

```
package com.piter.util;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.connection.DataType;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Component;

import java.util.Date;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.concurrent.TimeUnit;
import java.util.stream.Collectors;
import java.util.stream.Stream;


@Component
public class BaseRedisOperator {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;


    enum Position {
        BEFORE, AFTER
    }


    /**************************************************** key 相关操作 *************************************************/

    /**
     * 实现命令 : KEYS pattern
     * 查找所有符合 pattern 模式的 key
     * ?     匹配单个字符
     * *     匹配0到多个字符
     * [a-c] 匹配a和c
     * [ac]  匹配a到c
     * [^a]  匹配除了a以外的字符
     *
     * @param pattern redis pattern 表达式
     * @return
     */
    public Set<String> keys(String pattern) {
        return redisTemplate.keys(pattern);
    }


    /**
     * 实现命令 : DEL key1 [key2 ...]
     * 删除一个或多个key
     *
     * @param keys
     * @return
     */
    public Long del(String... keys) {
        Set<String> keySet = Stream.of(keys).collect(Collectors.toSet());
        return redisTemplate.delete(keySet);
    }

    /**
     * 实现命令 : UNLINK key1 [key2 ...]
     * 删除一个或多个key
     *
     * @param keys
     * @return
     */
    public Long unlink(String... keys) {
        Set<String> keySet = Stream.of(keys).collect(Collectors.toSet());
        return redisTemplate.unlink(keySet);
    }


    /**
     * 实现命令 : EXISTS key1 [key2 ...]
     * 查看 key 是否存在，返回存在 key 的个数
     *
     * @param keys
     * @return
     */
    public Long exists(String... keys) {
        Set<String> keySet = Stream.of(keys).collect(Collectors.toSet());
        return redisTemplate.countExistingKeys(keySet);
    }


    /**
     * 实现命令 : TYPE key
     * 查看 key 的 value 的类型
     *
     * @param key
     * @return
     */
    public String type(String key) {
        DataType dataType = redisTemplate.type(key);
        if (dataType == null) {
            return "";
        }
        return dataType.code();
    }


    /**
     * 实现命令 : PERSIST key
     * 取消 key 的超时时间，持久化 key
     *
     * @param key
     * @return
     */
    public boolean persist(String key) {
        Boolean result = redisTemplate.persist(key);
        if (null == result) {
            return false;
        }
        return result;
    }


    /**
     * 实现命令 : TTL key
     * 返回给定 key 的剩余生存时间,key不存在返回 null
     * 单位: 秒
     *
     * @param key
     * @return
     */
    public Long ttl(String key) {
        return redisTemplate.getExpire(key);
    }

    /**
     * 实现命令 : PTTL key
     * 返回给定 key 的剩余生存时间,key不存在返回 null
     * 单位: 毫秒
     *
     * @param key
     * @return
     */
    public Long pTtl(String key) {
        return redisTemplate.getExpire(key, TimeUnit.MILLISECONDS);
    }

    /**
     * 实现命令 : EXPIRE key 秒
     * 设置key 的生存时间
     * 单位 : 秒
     *
     * @param key
     * @return
     */
    public boolean expire(String key, int ttl) {
        Boolean result = redisTemplate.expire(key, ttl, TimeUnit.SECONDS);
        if (null == result) {
            return false;
        }
        return result;
    }

    /**
     * 实现命令 : PEXPIRE key 毫秒
     * 设置key 的生存时间
     * 单位 : 毫秒
     *
     * @param key
     * @return
     */
    public boolean pExpire(String key, Long ttl) {
        Boolean result = redisTemplate.expire(key, ttl, TimeUnit.MILLISECONDS);
        if (null == result) {
            return false;
        }
        return result;
    }

    /**
     * 实现命令 : EXPIREAT key Unix时间戳(自1970年1月1日以来的秒数)
     * 设置key 的过期时间
     *
     * @param key
     * @param date
     * @return
     */
    public boolean expireAt(String key, Date date) {
        Boolean result = redisTemplate.expireAt(key, date);
        if (null == result) {
            return false;
        }
        return result;
    }


    /**
     * 实现命令 : RENAME key newkey
     * 重命名key，如果newKey已经存在，则newKey的原值被覆盖
     *
     * @param oldKey
     * @param newKey
     */
    public void rename(String oldKey, String newKey) {
        redisTemplate.rename(oldKey, newKey);
    }

    /**
     * 实现命令 : RENAMENX key newkey
     * 安全重命名key，newKey不存在时才重命名
     *
     * @param oldKey
     * @param newKey
     * @return
     */
    public boolean renameNx(String oldKey, String newKey) {
        Boolean result = redisTemplate.renameIfAbsent(oldKey, newKey);
        if (null == result) {
            return false;
        }
        return result;
    }

    /**************************************************** key 相关操作 *************************************************/


    /************************************************* String 相关操作 *************************************************/

    /**
     * 实现命令 : SET key value
     * 添加一个持久化的 String 类型的键值对
     *
     * @param key
     * @param value
     */
    public void set(String key, Object value) {
        redisTemplate.opsForValue().set(key, value);
    }

    /**
     * 实现命令 : SET key value EX 秒、 setex  key value 秒
     * 添加一个 String 类型的键值对，并设置生存时间
     *
     * @param key
     * @param value
     * @param ttl   key 的生存时间，单位:秒
     */
    public void set(String key, Object value, int ttl) {
        redisTemplate.opsForValue().set(key, value, ttl, TimeUnit.SECONDS);
    }


    /**
     * 实现命令 : SET key value PX 毫秒 、 psetex key value 毫秒
     * 添加一个 String 类型的键值对，并设置生存时间
     *
     * @param key
     * @param value
     * @param ttl   key 的生存时间，单位:毫秒
     */
    public void set(String key, Object value, long ttl) {
        redisTemplate.opsForValue().set(key, value, ttl, TimeUnit.MILLISECONDS);
    }


    /**
     * 实现命令 : SET key value [EX 秒|PX 毫秒] [NX|XX]
     * 添加一个 String 类型的键值对，
     * ttl、timeUnit 不为 null 时设置生存时间
     * keyIfExist 不为 null 时，设置 NX 或 XX 模式
     *
     * @param key
     * @param value
     * @param ttl        生存时间
     * @param timeUnit   生存时间的单位
     * @param keyIfExist true 表示 xx,key 存在时才添加. false 表示 nx，key 不存在时才添加
     */
    public boolean set(String key, Object value, Long ttl, TimeUnit timeUnit, Boolean keyIfExist) {
        Boolean result = false;

        if ((ttl == null || timeUnit == null) && (keyIfExist == null)) {
            // SET key value
            redisTemplate.opsForValue().set(key, value);
            result = true;
        }

        if (ttl != null && timeUnit != null && keyIfExist == null) {
            // SET key value [EX 秒|PX 毫秒]
            redisTemplate.opsForValue().set(key, value, ttl, timeUnit);
            result = true;
        }

        if ((ttl == null || timeUnit == null) && (keyIfExist != null) && keyIfExist) {
            // SET key value XX
            result = redisTemplate.opsForValue().setIfPresent(key, value);
        }

        if (ttl != null && timeUnit != null && keyIfExist != null && keyIfExist) {
            // SET key value [EX 秒|PX 毫秒] XX
            result = redisTemplate.opsForValue().setIfPresent(key, value, ttl, timeUnit);
        }

        if ((ttl == null || timeUnit == null) && (keyIfExist != null) && (!keyIfExist)) {
            // SET key value NX
            result = redisTemplate.opsForValue().setIfAbsent(key, value);
        }

        if (ttl != null && timeUnit != null && keyIfExist != null && (!keyIfExist)) {
            // SET key value [EX 秒|PX 毫秒] NX
            result = redisTemplate.opsForValue().setIfAbsent(key, value, ttl, timeUnit);

        }

        if (result == null) {
            return false;
        }
        return result;
    }


    /**
     * 实现命令 : MSET key1 value1 [key2 value2...]
     * 安全批量添加键值对,只要有一个 key 已存在，所有的键值对都不会插入
     *
     * @param keyValueMap
     */
    public void mSet(Map<String, Object> keyValueMap) {
        redisTemplate.opsForValue().multiSetIfAbsent(keyValueMap);
    }


    /**
     * 实现命令 : MSETNX key1 value1 [key2 value2...]
     * 批量添加键值对
     *
     * @param keyValueMap
     */
    public void mSetNx(Map<String, Object> keyValueMap) {
        redisTemplate.opsForValue().multiSet(keyValueMap);
    }


    /**
     * 实现命令 : SETRANGE key 下标 str
     * 覆盖 原始 value 的一部分，从指定的下标开始覆盖， 覆盖的长度为指定的字符串的长度。
     *
     * @param key
     * @param str    字符串
     * @param offset 开始覆盖的位置，包括开始位置，下标从0开始
     */
    public void setRange(String key, Object str, int offset) {
        redisTemplate.opsForValue().set(key, str, offset);
    }


    /**
     * 实现命令 : APPEND key value
     * 在原始 value 末尾追加字符串
     *
     * @param key
     * @param str 要追加的字符串
     */
    public void append(String key, String str) {
        redisTemplate.opsForValue().append(key, str);
    }


    /**
     * 实现命令 : GETSET key value
     * 设置 key 的 value 并返回旧 value
     *
     * @param key
     * @param value
     */
    public Object getSet(String key, Object value) {
        return redisTemplate.opsForValue().getAndSet(key, value);
    }


    /**
     * 实现命令 : GET key
     * 获取一个key的value
     *
     * @param key
     * @return value
     */
    public Object get(String key) {
        return redisTemplate.opsForValue().get(key);
    }

    /**
     * 实现命令 : MGET key1 [key2...]
     * 获取多个key的value
     *
     * @param keys
     * @return value
     */
    public List<Object> mGet(String... keys) {
        Set<String> keySet = Stream.of(keys).collect(Collectors.toSet());
        return redisTemplate.opsForValue().multiGet(keySet);
    }


    /**
     * 实现命令 : GETRANGE key 开始下标 结束下标
     * 获取指定key的value的子串，下标从0开始，包括开始下标，也包括结束下标。
     *
     * @param key
     * @param start 开始下标
     * @param end   结束下标
     * @return
     */
    public String getRange(String key, int start, int end) {
        return redisTemplate.opsForValue().get(key, start, end);
    }


    /**
     * 实现命令 : STRLEN key
     * 获取 key 对应 value 的字符串长度
     *
     * @param key
     * @return
     */
    public Long strLen(String key) {
        return redisTemplate.opsForValue().size(key);
    }


    /**
     * 实现命令 : INCR key
     * 给 value 加 1,value 必须是整数
     *
     * @param key
     * @return
     */
    public Long inCr(String key) {
        return redisTemplate.opsForValue().increment(key);
    }

    /**
     * 实现命令 : INCRBY key 整数
     * 给 value 加 上一个整数,value 必须是整数
     *
     * @param key
     * @return
     */
    public Long inCrBy(String key, Long number) {
        return redisTemplate.opsForValue().increment(key, number);
    }

    /**
     * 实现命令 : INCRBYFLOAT key 数
     * 给 value 加上一个小数,value 必须是数
     *
     * @param key
     * @return
     */
    public Double inCrByFloat(String key, double number) {
        return redisTemplate.opsForValue().increment(key, number);
    }


    /**
     * 实现命令 : DECR key
     * 给 value 减去 1,value 必须是整数
     *
     * @param key
     * @return
     */
    public Long deCr(String key) {
        return redisTemplate.opsForValue().decrement(key);
    }


    /**
     * 实现命令 : DECRBY key 整数
     * 给 value 减去一个整数,value 必须是整数
     *
     * @param key
     * @return
     */
    public Long deCcrBy(String key, Long number) {
        return redisTemplate.opsForValue().decrement(key, number);
    }


    /************************************************* String 相关操作 *************************************************/


    /************************************************* Hash 相关操作 ***************************************************/

    /**
     * 实现命令 : HSET key field value
     * 添加 hash 类型的键值对，如果字段已经存在，则将其覆盖。
     *
     * @param key
     * @param field
     * @param value
     */
    public void hSet(String key, String field, Object value) {
        redisTemplate.opsForHash().put(key, field, value);
    }


    /**
     * 实现命令 : HSET key field1 value1 [field2 value2 ...]
     * 添加 hash 类型的键值对，如果字段已经存在，则将其覆盖。
     *
     * @param key
     * @param map
     */
    public void hSet(String key, Map<String, Object> map) {
        redisTemplate.opsForHash().putAll(key, map);
    }


    /**
     * 实现命令 : HSETNX key field value
     * 添加 hash 类型的键值对，如果字段不存在，才添加
     *
     * @param key
     * @param field
     * @param value
     */
    public boolean hSetNx(String key, String field, Object value) {
        Boolean result = redisTemplate.opsForHash().putIfAbsent(key, field, value);
        if (result == null) {
            return false;
        }
        return result;
    }


    /**
     * 实现命令 : HGET key field
     * 返回 field 对应的值
     *
     * @param key
     * @param field
     * @return
     */
    public Object hGet(String key, String field) {
        return redisTemplate.opsForHash().get(key, field);
    }


    /**
     * 实现命令 : HMGET key field1 [field2 ...]
     * 返回 多个 field 对应的值
     *
     * @param key
     * @param fields
     * @return
     */
    public List<Object> hGet(String key, String... fields) {
        Set<Object> fieldSet = Stream.of(fields).collect(Collectors.toSet());
        return redisTemplate.opsForHash().multiGet(key, fieldSet);
    }


    /**
     * 实现命令 : HGETALL key
     * 返回所以的键值对
     *
     * @param key
     * @return
     */
    public Map<Object, Object> hGetAll(String key) {
        return redisTemplate.opsForHash().entries(key);
    }


    /**
     * 实现命令 : HKEYS key
     * 获取所有的 field
     *
     * @param key
     * @return
     */
    public Set<Object> hKeys(String key) {
        return redisTemplate.opsForHash().keys(key);
    }


    /**
     * 实现命令 : HVALS key
     * 获取所有的 value
     *
     * @param key
     * @return
     */
    public List<Object> hValue(String key) {
        return redisTemplate.opsForHash().values(key);
    }


    /**
     * 实现命令 : HDEL key field [field ...]
     * 删除哈希表 key 中的一个或多个指定域，不存在的域将被忽略。
     *
     * @param key
     * @param fields
     */
    public Long hDel(String key, Object... fields) {
        return redisTemplate.opsForHash().delete(key, fields);
    }


    /**
     * 实现命令 : HEXISTS key field
     * 删除哈希表 key 中的一个或多个指定域，不存在的域将被忽略。
     *
     * @param key
     * @param field
     */
    public boolean hExists(String key, String field) {
        Boolean result = redisTemplate.opsForHash().hasKey(key, field);
        if (result == null) {
            return false;
        }
        return result;
    }


    /**
     * 实现命令 : HLEN key
     * 获取 hash 中字段:值对的数量
     *
     * @param key
     */
    public Long hLen(String key) {
        return redisTemplate.opsForHash().size(key);
    }


    /**
     * 实现命令 : HSTRLEN key field
     * 获取字段对应值的长度
     *
     * @param key
     * @param field
     */
    public Long hStrLen(String key, String field) {
        return redisTemplate.opsForHash().lengthOfValue(key, field);
    }


    /**
     * 实现命令 : HINCRBY key field 整数
     * 给字段的值加上一个整数
     *
     * @param key
     * @param field
     * @return 运算后的值
     */
    public Long hInCrBy(String key, String field, long number) {
        return redisTemplate.opsForHash().increment(key, field, number);
    }


    /**
     * 实现命令 : HINCRBYFLOAT key field 浮点数
     * 给字段的值加上一个浮点数
     *
     * @param key
     * @param field
     * @return 运算后的值
     */
    public Double hInCrByFloat(String key, String field, Double number) {
        return redisTemplate.opsForHash().increment(key, field, number);
    }


    /************************************************* Hash 相关操作 ***************************************************/


    /************************************************* List 相关操作 ***************************************************/

    /**
     * 实现命令 : LPUSH key 元素1 [元素2 ...]
     * 在最左端推入元素
     *
     * @param key
     * @param values
     * @return 执行 LPUSH 命令后，列表的长度。
     */
    public Long lPush(String key, Object... values) {
        return redisTemplate.opsForList().leftPushAll(key, values);
    }


    /**
     * 实现命令 : RPUSH key 元素1 [元素2 ...]
     * 在最右端推入元素
     *
     * @param key
     * @param values
     * @return 执行 RPUSH 命令后，列表的长度。
     */
    public Long rPush(String key, Object... values) {
        return redisTemplate.opsForList().rightPushAll(key, values);
    }


    /**
     * 实现命令 : LPOP key
     * 弹出最左端的元素
     *
     * @param key
     * @return 弹出的元素
     */
    public Object lPop(String key) {
        return redisTemplate.opsForList().leftPop(key);
    }

    /**
     * 实现命令 : RPOP key
     * 弹出最右端的元素
     *
     * @param key
     * @return 弹出的元素
     */
    public Object rPop(String key) {
        return redisTemplate.opsForList().rightPop(key);
    }


    /**
     * 实现命令 : BLPOP key
     * (阻塞式)弹出最左端的元素，如果 key 中没有元素，将一直等待直到有元素或超时为止
     *
     * @param key
     * @param timeout 等待的时间，单位秒
     * @return 弹出的元素
     */
    public Object bLPop(String key, int timeout) {
        return redisTemplate.opsForList().leftPop(key, timeout, TimeUnit.SECONDS);
    }

    /**
     * 实现命令 : BRPOP key
     * (阻塞式)弹出最右端的元素，将一直等待直到有元素或超时为止
     *
     * @param key
     * @return 弹出的元素
     */
    public Object bRPop(String key, int timeout) {
        return redisTemplate.opsForList().rightPop(key, timeout, TimeUnit.SECONDS);
    }


    /**
     * 实现命令 : LINDEX key index
     * 返回指定下标处的元素，下标从0开始
     *
     * @param key
     * @param index
     * @return 指定下标处的元素
     */
    public Object lIndex(String key, int index) {
        return redisTemplate.opsForList().index(key, index);
    }


    /**
     * 实现命令 : LINSERT key BEFORE|AFTER 目标元素 value
     * 在目标元素前或后插入元素
     *
     * @param key
     * @param position
     * @param pivot
     * @param value
     * @return
     */
    public Long lInsert(String key, Position position, Object pivot, Object value) {
        switch (position) {
            case AFTER:
                return redisTemplate.opsForList().rightPush(key, pivot, value);
            case BEFORE:
                return redisTemplate.opsForList().rightPush(key, pivot, value);
            default:
                return null;
        }
    }


    /**
     * 实现命令 : LRANGE key 开始下标 结束下标
     * 获取指定范围的元素,下标从0开始，包括开始下标，也包括结束下标(待验证)
     *
     * @param key
     * @param start
     * @param end
     * @return
     */
    public List<Object> lRange(String key, int start, int end) {
        return redisTemplate.opsForList().range(key, start, end);
    }


    /**
     * 实现命令 : LLEN key
     * 获取 list 的长度
     *
     * @param key
     * @return
     */
    public Long lLen(String key) {
        return redisTemplate.opsForList().size(key);
    }


    /**
     * 实现命令 : LREM key count 元素
     * 删除 count 个指定元素,
     *
     * @param key
     * @param count count > 0: 从头往尾移除指定元素。count < 0: 从尾往头移除指定元素。count = 0: 移除列表所有的指定元素。(未验证)
     * @param value
     * @return
     */
    public Long lLen(String key, int count, Object value) {
        return redisTemplate.opsForList().remove(key, count, value);
    }


    /**
     * 实现命令 : LSET key index 新值
     * 更新指定下标的值,下标从 0 开始，支持负下标，-1表示最右端的元素(未验证)
     *
     * @param key
     * @param index
     * @param value
     * @return
     */
    public void lSet(String key, int index, Object value) {
        redisTemplate.opsForList().set(key, index, value);
    }


    /**
     * 实现命令 : LTRIM key 开始下标 结束下标
     * 裁剪 list。[01234] 的 `LTRIM key 1 -2` 的结果为 [123]
     *
     * @param key
     * @param start
     * @param end
     * @return
     */
    public void lTrim(String key, int start, int end) {
        redisTemplate.opsForList().trim(key, start, end);
    }


    /**
     * 实现命令 : RPOPLPUSH 源list 目标list
     * 将 源list 的最右端元素弹出，推入到 目标list 的最左端，
     *
     * @param sourceKey 源list
     * @param targetKey 目标list
     * @return 弹出的元素
     */
    public Object rPopLPush(String sourceKey, String targetKey) {
        return redisTemplate.opsForList().rightPopAndLeftPush(sourceKey, targetKey);
    }


    /**
     * 实现命令 : BRPOPLPUSH 源list 目标list timeout
     * (阻塞式)将 源list 的最右端元素弹出，推入到 目标list 的最左端，如果 源list 没有元素，将一直等待直到有元素或超时为止
     *
     * @param sourceKey 源list
     * @param targetKey 目标list
     * @param timeout   超时时间，单位秒， 0表示无限阻塞
     * @return 弹出的元素
     */
    public Object bRPopLPush(String sourceKey, String targetKey, int timeout) {
        return redisTemplate.opsForList().rightPopAndLeftPush(sourceKey, targetKey, timeout, TimeUnit.SECONDS);
    }


    /************************************************* List 相关操作 ***************************************************/


    /************************************************** SET 相关操作 ***************************************************/

    /**
     * 实现命令 : SADD key member1 [member2 ...]
     * 添加成员
     *
     * @param key
     * @param values
     * @return 添加成功的个数
     */
    public Long sAdd(String key, Object... values) {
        return redisTemplate.opsForSet().add(key, values);
    }


    /**
     * 实现命令 : SREM key member1 [member2 ...]
     * 删除指定的成员
     *
     * @param key
     * @param values
     * @return 删除成功的个数
     */
    public Long sRem(String key, Object... values) {
        return redisTemplate.opsForSet().remove(key, values);
    }


    /**
     * 实现命令 : SCARD key
     * 获取set中的成员数量
     *
     * @param key
     * @return
     */
    public Long sCard(String key) {
        return redisTemplate.opsForSet().size(key);
    }


    /**
     * 实现命令 : SISMEMBER key member
     * 查看成员是否存在
     *
     * @param key
     * @param values
     * @return
     */
    public boolean sIsMember(String key, Object... values) {
        Boolean result = redisTemplate.opsForSet().isMember(key, values);
        if (null == result) {
            return false;
        }
        return result;
    }


    /**
     * 实现命令 : SMEMBERS key
     * 获取所有的成员
     *
     * @param key
     * @return
     */
    public Set<Object> sMembers(String key) {
        return redisTemplate.opsForSet().members(key);
    }


    /**
     * 实现命令 : SMOVE 源key 目标key member
     * 移动成员到另一个集合
     *
     * @param sourceKey 源key
     * @param targetKey 目标key
     * @param value
     * @return
     */
    public boolean sMove(String sourceKey, String targetKey, Object value) {
        Boolean result = redisTemplate.opsForSet().move(sourceKey, value, targetKey);
        if (null == result) {
            return false;
        }
        return result;
    }


    /**
     * 实现命令 : SDIFF key [otherKey ...]
     * 求 key 的差集
     *
     * @param key
     * @param otherKeys
     * @return
     */
    public Set<Object> sDiff(String key, String... otherKeys) {
        List<String> otherKeyList = Stream.of(otherKeys).collect(Collectors.toList());
        return redisTemplate.opsForSet().difference(key, otherKeyList);
    }


    /**
     * 实现命令 : SDIFFSTORE 目标key key [otherKey ...]
     * 存储 key 的差集
     *
     * @param targetKey 目标key
     * @param key
     * @param otherKeys
     * @return
     */
    public Long sDiffStore(String targetKey, String key, String... otherKeys) {
        List<String> otherKeyList = Stream.of(otherKeys).collect(Collectors.toList());
        return redisTemplate.opsForSet().differenceAndStore(key, otherKeyList, targetKey);
    }


    /**
     * 实现命令 : SINTER key [otherKey ...]
     * 求 key 的交集
     *
     * @param key
     * @param otherKeys
     * @return
     */
    public Set<Object> sInter(String key, String... otherKeys) {
        List<String> otherKeyList = Stream.of(otherKeys).collect(Collectors.toList());
        return redisTemplate.opsForSet().intersect(key, otherKeyList);
    }


    /**
     * 实现命令 : SINTERSTORE 目标key key [otherKey ...]
     * 存储 key 的交集
     *
     * @param targetKey 目标key
     * @param key
     * @param otherKeys
     * @return
     */
    public Long sInterStore(String targetKey, String key, String... otherKeys) {
        List<String> otherKeyList = Stream.of(otherKeys).collect(Collectors.toList());
        return redisTemplate.opsForSet().intersectAndStore(key, otherKeyList, targetKey);
    }


    /**
     * 实现命令 : SUNION key [otherKey ...]
     * 求 key 的并集
     *
     * @param key
     * @param otherKeys
     * @return
     */
    public Set<Object> sUnion(String key, String... otherKeys) {
        List<String> otherKeyList = Stream.of(otherKeys).collect(Collectors.toList());
        return redisTemplate.opsForSet().union(key, otherKeyList);
    }


    /**
     * 实现命令 : SUNIONSTORE 目标key key [otherKey ...]
     * 存储 key 的并集
     *
     * @param targetKey 目标key
     * @param key
     * @param otherKeys
     * @return
     */
    public Long sUnionStore(String targetKey, String key, String... otherKeys) {
        List<String> otherKeyList = Stream.of(otherKeys).collect(Collectors.toList());
        return redisTemplate.opsForSet().unionAndStore(key, otherKeyList, targetKey);
    }


    /**
     * 实现命令 : SPOP key [count]
     * 随机删除(弹出)指定个数的成员
     *
     * @param key
     * @return
     */
    public Object sPop(String key) {
        return redisTemplate.opsForSet().pop(key);
    }

    /**
     * 实现命令 : SPOP key [count]
     * 随机删除(弹出)指定个数的成员
     *
     * @param key
     * @param count 个数
     * @return
     */
    public List<Object> sPop(String key, int count) {
        return redisTemplate.opsForSet().pop(key, count);
    }


    /**
     * 实现命令 : SRANDMEMBER key [count]
     * 随机返回指定个数的成员
     *
     * @param key
     * @return
     */
    public Object sRandMember(String key) {
        return redisTemplate.opsForSet().randomMember(key);
    }


    /**
     * 实现命令 : SRANDMEMBER key [count]
     * 随机返回指定个数的成员
     * 如果 count 为正数，随机返回 count 个不同成员
     * 如果 count 为负数，随机选择 1 个成员，返回 count 个
     *
     * @param key
     * @param count 个数
     * @return
     */
    public List<Object> sRandMember(String key, int count) {
        return redisTemplate.opsForSet().randomMembers(key, count);
    }


    /************************************************** SET 相关操作 ***************************************************/


    /*********************************************** Sorted SET 相关操作 ***********************************************/

    /**
     * 实现命令 : ZADD key score member
     * 添加一个 成员/分数 对
     *
     * @param key
     * @param value 成员
     * @param score 分数
     * @return
     */
    public boolean zAdd(String key, double score, Object value) {
        Boolean result = redisTemplate.opsForZSet().add(key, value, score);
        if (result == null) {
            return false;
        }
        return result;
    }


    /**
     * 实现命令 : ZREM key member [member ...]
     * 删除成员
     *
     * @param key
     * @param values
     * @return
     */
    public Long zRem(String key, Object... values) {
        return redisTemplate.opsForZSet().remove(key, values);
    }


    /**
     * 实现命令 : ZREMRANGEBYRANK key start stop
     * 删除 start下标 和 end下标间的所有成员
     * 下标从0开始，支持负下标，-1表示最右端成员，包括开始下标也包括结束下标 (未验证)
     *
     * @param key
     * @param start 开始下标
     * @param end   结束下标
     * @return
     */
    public Long zRemRangeByRank(String key, int start, int end) {
        return redisTemplate.opsForZSet().removeRange(key, start, end);
    }


    /**
     * 实现命令 : ZREMRANGEBYSCORE key start stop
     * 删除分数段内的所有成员
     * 包括min也包括max (未验证)
     *
     * @param key
     * @param min 小分数
     * @param max 大分数
     * @return
     */
    public Long zRemRangeByScore(String key, double min, double max) {
        return redisTemplate.opsForZSet().removeRangeByScore(key, min, max);
    }


    /**
     * 实现命令 : ZSCORE key member
     * 获取成员的分数
     *
     * @param key
     * @param value
     * @return
     */
    public Double zScore(String key, Object value) {
        return redisTemplate.opsForZSet().score(key, value);
    }


    /**
     * 实现命令 : ZINCRBY key 带符号的双精度浮点数 member
     * 增减成员的分数
     *
     * @param key
     * @param value
     * @param delta 带符号的双精度浮点数
     * @return
     */
    public Double zInCrBy(String key, Object value, double delta) {
        return redisTemplate.opsForZSet().incrementScore(key, value, delta);
    }


    /**
     * 实现命令 : ZCARD key
     * 获取集合中成员的个数
     *
     * @param key
     * @return
     */
    public Long zCard(String key) {
        return redisTemplate.opsForZSet().size(key);
    }


    /**
     * 实现命令 : ZCOUNT key min max
     * 获取某个分数范围内的成员个数，包括min也包括max (未验证)
     *
     * @param key
     * @param min 小分数
     * @param max 大分数
     * @return
     */
    public Long zCount(String key, double min, double max) {
        return redisTemplate.opsForZSet().count(key, min, max);
    }


    /**
     * 实现命令 : ZRANK key member
     * 按分数从小到大获取成员在有序集合中的排名
     *
     * @param key
     * @param value
     * @return
     */
    public Long zRank(String key, Object value) {
        return redisTemplate.opsForZSet().rank(key, value);
    }


    /**
     * 实现命令 : ZREVRANK key member
     * 按分数从大到小获取成员在有序集合中的排名
     *
     * @param key
     * @param value
     * @return
     */
    public Long zRevRank(String key, Object value) {
        return redisTemplate.opsForZSet().reverseRank(key, value);
    }


    /**
     * 实现命令 : ZRANGE key start end
     * 获取 start下标到 end下标之间到成员，并按分数从小到大返回
     * 下标从0开始，支持负下标，-1表示最后一个成员，包括开始下标，也包括结束下标(未验证)
     *
     * @param key
     * @param start 开始下标
     * @param end   结束下标
     * @return
     */
    public Set<Object> zRange(String key, int start, int end) {
        return redisTemplate.opsForZSet().range(key, start, end);
    }


    /**
     * 实现命令 : ZREVRANGE key start end
     * 获取 start下标到 end下标之间到成员，并按分数从小到大返回
     * 下标从0开始，支持负下标，-1表示最后一个成员，包括开始下标，也包括结束下标(未验证)
     *
     * @param key
     * @param start 开始下标
     * @param end   结束下标
     * @return
     */
    public Set<Object> zRevRange(String key, int start, int end) {
        return redisTemplate.opsForZSet().reverseRange(key, start, end);
    }


    /**
     * 实现命令 : ZRANGEBYSCORE key min max
     * 获取分数范围内的成员并按从小到大返回
     * (未验证)
     *
     * @param key
     * @param min 小分数
     * @param max 大分数
     * @return
     */
    public Set<Object> zRangeByScore(String key, double min, double max) {
        return redisTemplate.opsForZSet().rangeByScore(key, min, max);
    }


    /**
     * 实现命令 : ZRANGEBYSCORE key min max LIMIT offset count
     * 分页获取分数范围内的成员并按从小到大返回
     * 包括min也包括max(未验证)
     *
     * @param key
     * @param min    小分数
     * @param max    大分数
     * @param offset 开始下标，下标从0开始
     * @param count  取多少条
     * @return
     */
    public Set<Object> zRangeByScore(String key, double min, double max, int offset, int count) {
        return redisTemplate.opsForZSet().rangeByScore(key, min, max, offset, count);
    }


    /**
     * 实现命令 : ZREVRANGEBYSCORE key min max
     * 获取分数范围内的成员并按从大到小返回
     * (未验证)
     *
     * @param key
     * @param min 小分数
     * @param max 大分数
     * @return
     */
    public Set<Object> zRevRangeByScore(String key, double min, double max) {
        return redisTemplate.opsForZSet().reverseRangeByScore(key, min, max);
    }


    /**
     * 实现命令 : ZREVRANGEBYSCORE key min max LIMIT offset count
     * 分页获取分数范围内的成员并按从大到小返回
     * 包括min也包括max(未验证)
     *
     * @param key
     * @param min    小分数
     * @param max    大分数
     * @param offset 开始下标，下标从0开始
     * @param count  取多少条
     * @return
     */
    public Set<Object> zRevRangeByScore(String key, double min, double max, int offset, int count) {
        return redisTemplate.opsForZSet().reverseRangeByScore(key, min, max, offset, count);
    }

    /*********************************************** Sorted SET 相关操作 ***********************************************/

}
```

#### RedisOperator

RedisOperator 类在 BaseRedisOperator 的基础上补充了一下无法按照 Redis 命令 设计的类。

使用该类并不会给你带来类似直接使用Redis 命令的体验

```
package com.piter.util;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.connection.RedisZSetCommands;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.core.ZSetOperations;
import org.springframework.stereotype.Component;

import java.util.*;

@Component
public class RedisOperator extends BaseRedisOperator {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;


    /**************************************************** key 相关操作 *************************************************/

    /**
     * 实现命令：DEL key1 [key2 ...]
     * 删除任意多个 key
     *
     * @param keys
     * @return
     */
    public Long del(Collection<String> keys) {
        Set<String> keySet = new HashSet<>(keys);
        return redisTemplate.delete(keySet);
    }


    /**
     * 实现命令：UNLINK key1 [key2 ...]
     * 删除任意多个 key
     *
     * @param keys
     * @return
     */
    public Long unlink(Collection<String> keys) {
        Set<String> keySet = new HashSet<>(keys);
        return redisTemplate.unlink(keySet);
    }


    /**
     * 实现命令：EXISTS key1 [key2 ...]
     * key去重后，查看 key 是否存在，返回存在 key 的个数
     *
     * @param keys
     * @return
     */
    public Long exists(Collection<String> keys) {
        Set<String> keySet = new HashSet<>(keys);
        return redisTemplate.countExistingKeys(keySet);
    }


    /**
     * 实现命令：EXPIREAT key Unix时间戳(自1970年1月1日以来的秒数)
     * 设置key 的过期时间
     *
     * @param key
     * @param ttl 存活时间 单位秒
     * @return
     */
    public boolean expireAt(String key, int ttl) {
        Date date = new Date(System.currentTimeMillis() + ttl * 1000);
        Boolean result = redisTemplate.expireAt(key, date);
        if (null == result) {
            return false;
        }
        return result;
    }


    /**
     * 实现命令：PEXPIREAT key 自1970年1月1日以来的毫秒数
     * 设置key 的过期时间
     *
     * @param key
     * @param ttl 存活时间 单位毫秒
     * @return
     */
    public boolean pExpireAt(String key, long ttl) {
        Date date = new Date(System.currentTimeMillis() + ttl);
        Boolean result = redisTemplate.expireAt(key, date);
        if (null == result) {
            return false;
        }
        return result;
    }

    /**
     * 实现命令 : PEXPIREAT key 自1970年1月1日以来的毫秒数
     * 设置key 的过期时间
     *
     * @param key
     * @param date
     * @return
     */
    public boolean pExpireAt(String key, Date date) {
        Boolean result = redisTemplate.expireAt(key, date);
        if (null == result) {
            return false;
        }
        return result;
    }

    /**************************************************** key 相关操作 *************************************************/


    /************************************************* String 相关操作 *************************************************/

    /**
     * 实现命令 : MGET key1 [key2...]
     * key去重后，获取多个key的value
     *
     * @param keys
     * @return value
     */
    public List<Object> mGet(Collection<String> keys) {
        Set<String> keySet = new HashSet<>(keys);
        return redisTemplate.opsForValue().multiGet(keySet);
    }

    /************************************************* String 相关操作 *************************************************/


    /************************************************* Hash 相关操作 ***************************************************/

    /**
     * 实现命令 : HMGET key field1 [field2 ...]
     * 返回 多个 field 对应的值
     *
     * @param key
     * @param fields
     * @return
     */
    public List<Object> hGet(String key, Collection<Object> fields) {
        return redisTemplate.opsForHash().multiGet(key, fields);
    }


    /**
     * 实现命令 : HDEL key field [field ...]
     * 删除哈希表 key 中的一个或多个指定域，不存在的域将被忽略。
     *
     * @param key
     * @param fields
     */
    public Long hDel(String key, Collection<Object> fields) {
        Object[] objects = fields.toArray();
        return redisTemplate.opsForHash().delete(key, objects);
    }
    /************************************************* Hash 相关操作 ***************************************************/


    /************************************************* List 相关操作 ***************************************************/

    /**
     * 实现命令 : LPUSH key 元素1 [元素2 ...]
     * 在最左端推入元素
     *
     * @param key
     * @param values
     * @return 执行 LPUSH命令后，列表的长度。
     */
    public Long lPush(String key, Collection<Object> values) {
        return redisTemplate.opsForList().leftPushAll(key, values);
    }

    /**
     * 实现命令 : LPUSHX key 元素
     * key 存在时在，最左端推入元素
     *
     * @param key
     * @param value
     * @return 执行 LPUSHX 命令后，列表的长度。
     */
    public Long lPushX(String key, Object value) {
        return redisTemplate.opsForList().leftPushIfPresent(key, value);
    }


    /**
     * 实现命令 : RPUSH key 元素1 [元素2 ...]
     * 在最右端推入元素
     *
     * @param key
     * @param values
     * @return 执行 RPUSH 命令后，列表的长度。
     */
    public Long rPush(String key, Collection<Object> values) {
        return redisTemplate.opsForList().rightPushAll(key, values);
    }

    /**
     * 实现命令 : RPUSHX key 元素
     * key 存在时，在最右端推入元素
     *
     * @param key
     * @param value
     * @return 执行 RPUSHX 命令后，列表的长度。
     */
    public Long rPushX(String key, Object value) {
        return redisTemplate.opsForList().rightPushIfPresent(key, value);
    }

    /************************************************* List 相关操作 ***************************************************/


    /************************************************** SET 相关操作 ***************************************************/

    /**
     * 实现命令 : SADD key member1 [member2 ...]
     * 添加成员
     *
     * @param key
     * @param values
     * @return 添加成功的个数
     */
    public Object rPopLpush(String key, Collection<Object> values) {
        Object[] members = values.toArray();
        return redisTemplate.opsForSet().add(key, members);
    }


    /**
     * 实现命令 : SREM key member1 [member2 ...]
     * 删除指定的成员
     *
     * @param key
     * @param values
     * @return 删除成功的个数
     */
    public Long sRem(String key, Collection<Object> values) {
        Object[] members = values.toArray();
        return redisTemplate.opsForSet().remove(key, members);
    }


    /**
     * 实现命令 : SDIFF key [otherKey ...]
     * 求 key 的差集
     *
     * @param key
     * @param otherKeys
     * @return
     */
    public Set<Object> sDiff(String key, List<String> otherKeys) {
        return redisTemplate.opsForSet().difference(key, otherKeys);
    }


    /**
     * 实现命令 : SDIFFSTORE 目标key key [otherKey ...]
     * 存储 key 的差集
     *
     * @param targetKey 目标key
     * @param key
     * @param otherKeys
     * @return
     */
    public Long sDiffStore(String targetKey, String key, List<String> otherKeys) {
        return redisTemplate.opsForSet().differenceAndStore(key, otherKeys, targetKey);
    }


    /**
     * 实现命令 : SINTER key [otherKey ...]
     * 求 key 的交集
     *
     * @param key
     * @param otherKeys
     * @return
     */
    public Set<Object> sInter(String key, List<String> otherKeys) {
        return redisTemplate.opsForSet().intersect(key, otherKeys);
    }


    /**
     * 实现命令 : SINTERSTORE 目标key key [otherKey ...]
     * 存储 key 的交集
     *
     * @param targetKey 目标key
     * @param key
     * @param otherKeys
     * @return
     */
    public Long sInterStore(String targetKey, String key, List<String> otherKeys) {
        return redisTemplate.opsForSet().intersectAndStore(key, otherKeys, targetKey);
    }


    /**
     * 实现命令 : SUNION key [otherKey ...]
     * 求 key 的并集
     *
     * @param key
     * @param otherKeys
     * @return
     */
    public Set<Object> sUnion(String key, List<String> otherKeys) {
        return redisTemplate.opsForSet().union(key, otherKeys);
    }


    /**
     * 实现命令 : SUNIONSTORE 目标key key [otherKey ...]
     * 存储 key 的并集
     *
     * @param targetKey 目标key
     * @param key
     * @param otherKeys
     * @return
     */
    public Long sUnionStore(String targetKey, String key, List<String> otherKeys) {
        return redisTemplate.opsForSet().unionAndStore(key, otherKeys, targetKey);
    }

    /************************************************** SET 相关操作 ***************************************************/


    /*********************************************** Sorted SET 相关操作 ***********************************************/

    /**
     * 实现命令 : ZADD key score1 member1 [score2 member2 ...]
     * 批量添加 成员/分数 对
     *
     * @param key
     * @param tuples
     * @return
     */
    public Long zAdd(String key, Set<ZSetOperations.TypedTuple<Object>> tuples) {
        return redisTemplate.opsForZSet().add(key, tuples);
    }


    /**
     * 实现命令 : ZREM key member [member ...]
     * 删除成员
     *
     * @param key
     * @param values
     * @return
     */
    public Long zRem(String key, Collection<Object> values) {
        Object[] members = values.toArray();
        return redisTemplate.opsForZSet().remove(key, members);
    }


    /**
     * 实现命令 : ZUNIONSTORE destination numkeys key1 [key2 ...]
     * 计算指定key集合与otherKey集合的并集，并保存到targetKey集合，aggregat 默认为 SUM
     *
     * @param targetKey 目标集合
     * @param key       指定集合
     * @param otherKey  其他集合
     * @return
     */
    public Long zUnionStore(String targetKey, String key, String otherKey) {
        return redisTemplate.opsForZSet().unionAndStore(key, otherKey, targetKey);
    }


    /**
     * 实现命令 : ZUNIONSTORE destination numkeys key1 [key2 ...]
     * 计算指定key集合与otherKey集合的并集，并保存到targetKey集合，aggregat 默认为 SUM
     *
     * @param targetKey 目标集合
     * @param key       指定集合
     * @param otherKeys 其他集合
     * @return
     */
    public Long zUnionStore(String targetKey, String key, Collection<String> otherKeys) {
        return redisTemplate.opsForZSet().unionAndStore(key, otherKeys, targetKey);
    }


    /**
     * 实现命令 : ZUNIONSTORE destination numkeys key1 [key2 ...][AGGREGATE SUM | MIN | MAX]
     * 计算指定key集合与otherKey集合的并集，并保存到targetKey集合
     *
     * @param targetKey 目标集合
     * @param key       指定集合
     * @param otherKeys 其他集合
     * @param aggregat  SUM 将同一成员的分数相加, MIN 取同一成员中分数最小的, MAX 取同一成员中分数最大的
     * @return
     */
    public Long zUnionStore(String targetKey, String key, Collection<String> otherKeys, RedisZSetCommands.Aggregate aggregat) {
        return redisTemplate.opsForZSet().unionAndStore(key, otherKeys, targetKey, aggregat);
    }


    /**
     * 实现命令 : ZINTERSTORE destination numkeys key1 [key2 ...]
     * 计算指定key集合与otherKey集合的交集，并保存到targetKey集合，aggregat 默认为 SUM
     *
     * @param targetKey 目标集合
     * @param key       指定集合
     * @param otherKey  其他集合
     * @return
     */
    public Long zInterStore(String targetKey, String key, String otherKey) {
        return redisTemplate.opsForZSet().intersectAndStore(key, otherKey, targetKey);
    }


    /**
     * 实现命令 : ZINTERSTORE destination numkeys key1 [key2 ...]
     * 计算指定key集合与otherKey集合的交集，并保存到targetKey集合，aggregat 默认为 SUM
     *
     * @param targetKey 目标集合
     * @param key       指定集合
     * @param otherKeys 其他集合
     * @return
     */
    public Long zInterStore(String targetKey, String key, Collection<String> otherKeys) {
        return redisTemplate.opsForZSet().unionAndStore(key, otherKeys, targetKey);
    }


    /**
     * 实现命令 : ZINTERSTORE destination numkeys key1 [key2 ...][AGGREGATE SUM | MIN | MAX]
     * 计算指定key集合与otherKey集合的交集，并保存到targetKey集合
     *
     * @param targetKey 目标集合
     * @param key       指定集合
     * @param otherKeys 其他集合
     * @param aggregat  SUM 将同一成员的分数相加, MIN 取同一成员中分数最小的, MAX 取同一成员中分数最大的
     * @return
     */
    public Long zInterStore(String targetKey, String key, Collection<String> otherKeys, RedisZSetCommands.Aggregate aggregat) {
        return redisTemplate.opsForZSet().unionAndStore(key, otherKeys, targetKey, aggregat);
    }

    /*********************************************** Sorted SET 相关操作 ***********************************************/
}

```


## 总结

至此，我们就完成了 spring boot 集成 redis 的功能。

然后就可以编写测试代码，进行功能验证了。具体的 redis 集成与测试代码见：

[springboot-redis-integration](https://github.com/piterjia/springboot-redis-integration)

