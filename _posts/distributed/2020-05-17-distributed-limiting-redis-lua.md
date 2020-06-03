---
layout: post
title: Redis + Lua 实现分布式限流器
categories: Distributed
description: Redis + Lua实现分布式限流器
keywords: Redis, Lua, distributed, limiting
---

依靠 redis + lua 将多条命令合并在一起作为一个原子操作，来实现限流器, 无需过多考虑并发，就可以实现分布式限流。


## 模式一：计数器模式限流器

### 计数器模式原理
计数器算法是指在一段窗口时间内允许通过的固定数量的请求, 比如10次/秒, 500次/30秒.

如果设置的时间粒度越细, 那么限流会更平滑.

### 计数器模式 Lua 脚本
```
-- 计数器限流
-- 此处支持的最小单位时间是秒, 若将 expire 改成 pexpire 则可支持毫秒粒度.
-- KEYS[1]  string  限流的key
-- ARGV[1]  int     限流数
-- ARGV[2]  int     单位时间(秒)

local cnt = tonumber(redis.call("incr", KEYS[1]))

if (cnt == 1) then
    -- cnt 值为1说明之前不存在该值, 因此需要设置其过期时间
    redis.call("expire", KEYS[1], tonumber(ARGV[2]))
elseif (cnt > tonumber(ARGV[1])) then
    return -1
end 

return cnt
```

返回 -1 表示超过限流, 否则返回当前单位时间已通过的请求数


### key值建议 
可以但不限于以下的情况，根据自己的业务场景定义：
* ip + 接口
* user_id + 接口


优点：实现简单

缺点：粒度不够细的情况下, 会出现在同一个窗口时间内出现双倍请求。所以我们需要尽量保持时间粒度精细，限流才会更平滑。



## 模式二：令牌桶模式

### 令牌桶模式的原理
如图所示：

![](/images/posts/distributed/redis_lua1.png)

- 令牌桶中保存有令牌, 存在上限, 且一开始是满的。
- 每次请求都要消耗令牌（可根据不同请求消耗不同数量的令牌）。
- 每隔一段时间（比如固定速率）会往桶中放令牌。


桶的实现还分为:
- 可预消费：提前预支令牌数: 前人挖坑, 后人跳
- 不可预消费：令牌数不够直接拒绝


### 令牌桶模式 Lua 脚本
此处实现的不可预消费的令牌桶, 具体Lua代码:

```
-- 令牌桶限流: 不支持预消费, 初始桶是满的
-- KEYS[1]  string  限流的key

-- ARGV[1]  int     桶最大容量
-- ARGV[2]  int     每次添加令牌数
-- ARGV[3]  int     令牌添加间隔(秒)
-- ARGV[4]  int     当前时间戳

local bucket_capacity = tonumber(ARGV[1])
local add_token = tonumber(ARGV[2])
local add_interval = tonumber(ARGV[3])
local now = tonumber(ARGV[4])

-- 保存上一次更新桶的时间的key
local LAST_TIME_KEY = KEYS[1].."_time";         
-- 获取当前桶中令牌数
local token_cnt = redis.call("get", KEYS[1])    
-- 桶完全恢复需要的最大时长
local reset_time = math.ceil(bucket_capacity / add_token) * add_interval;

if token_cnt then   -- 令牌桶存在
    -- 上一次更新桶的时间
    local last_time = redis.call('get', LAST_TIME_KEY)
    -- 恢复倍数
    local multiple = math.floor((now - last_time) / add_interval)
    -- 恢复令牌数
    local recovery_cnt = multiple * add_token
    -- 确保不超过桶容量
    local token_cnt = math.min(bucket_capacity, token_cnt + recovery_cnt) - 1
    
    if token_cnt < 0 then
        return -1;
    end
    
    -- 重新设置过期时间, 避免key过期
    redis.call('set', KEYS[1], token_cnt, 'EX', reset_time)                     
    redis.call('set', LAST_TIME_KEY, last_time + multiple * add_interval, 'EX', reset_time)
    return token_cnt
    
else    -- 令牌桶不存在
    token_cnt = bucket_capacity - 1
    -- 设置过期时间避免key一直存在
    redis.call('set', KEYS[1], token_cnt, 'EX', reset_time);
    redis.call('set', LAST_TIME_KEY, now, 'EX', reset_time + 1);    
    return token_cnt    
end
```

令牌桶的关键是以下几个参数:
- 桶最大容量
- 每次放入的令牌数
- 放入令牌的间隔时间

令牌桶的实现不会出现计数器模式中单位时间内双倍流量的问题.


## Java 调用 Lua 脚本

写完的 lua 脚本。如何在 java 中调用呢？此处提供一个Demo，供参考。

```
public static void main(String[] args) throws IOException {
        String luaScript = Files.toString(new File("E:\\limit.lua"), Charset.defaultCharset());
        Jedis jedis = new Jedis("*******", 6379);
        jedis.auth("**********");
        String key = "ip" + System.currentTimeMillis() / 1000;
        String limit = "3";
        for (int i = 0; i < 5 ;i ++) {
            Object eval = jedis.eval(luaScript, Lists.newArrayList(key), Lists.newArrayList(limit));
            System.out.println(eval);
        }
    }

```


## 总结：

相比Redis事务来说，Lua脚本有以下优点：
- 减少网络开销: 不使用 Lua 脚本的话，需要向 Redis 发送多次请求, 而脚本只需一次即可, 减少网络传输;
- 原子操作: Redis 将整个脚本作为一个原子执行, 无需担心并发, 也就无需事务;
- 复用: 脚本会永久保存 Redis 中, 其他客户端可继续使用.


分布式限流的关键是要将限流服务原子化，而解决方案可以使使用 redis+lua 来实现高并发和高性能。

首先我们来使用 redis+lua 实现时间窗内某个接口的请求数限流，实现了该功能后可以改造为限流总并发/请求数和限制总资源数。 Lua 本身就是一种编程语言，也可以使用它实现复杂的令牌桶或漏桶算法。

由于在一个 Lua 脚本中的操作相当于原子操作，而且 Redis 是单线程模型，因此这种限流方式也是线程安全的。