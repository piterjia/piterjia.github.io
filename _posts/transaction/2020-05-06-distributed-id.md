---
layout: post
title: 分布式全局id
categories: Transaction
description: 分布式全局id
keywords: distributed, Transaction, mysql
---

全局唯一的 ID 几乎是所有系统都会遇到的刚需。这个 id 在搜索, 存储数据, 加快检索速度 等等很多方面都有着重要的意义。针对常见的几种场景，我在这里进行简单的总结和对比。

ID是数据的唯一标识，传统的做法是利用 UUID 和数据库的自增 ID，在互联网企业中，大部分公司使用的都是 Mysql，并且因为需要事务支持，所以通常会使用 Innodb 存储引擎，UUID 太长以及无序，所以并不适合在 Innodb 中来作为主键，自增ID比较合适。

但是随着业务发展，数据量将越来越大，需要对数据进行分库分表，而分库分表后，每个表中的数据都会按自己的节奏进行自增，很有可能出现 ID 冲突。这时就需要一个单独的机制来负责生成唯一 ID，生成的 ID 也可以叫做分布式 ID，或全局 ID。

下面来简单介绍下目前主流的各个生成分布式 ID 的方案。

## 方法一: 数据库自增ID

### 优点：
此方法使用数据库原有的功能，所以相对简单
- 能够保证唯一性
- 能够保证递增性
- id 之间的步长是固定且可自定义的

### 缺点：
- 可用性难以保证：数据库常见架构是 一主多从 + 读写分离，生成自增 ID 是写请求。主库挂了就废了。
- 扩展性差，性能有上限：因为写入是单点，数据库主库的写性能决定ID的生成性能上限，并且难扩展

## 方法二: 数据库多主模式（方法一的改进方案）：
主要的设计思路是：
1. 冗余主库，采用多主模式。避免写入单点
2. 数据水平切分，保证各主库生成的ID不重复

如图所示：

![](/images/posts/transition/sqlmasterslave.png)

如上图所述，由 1 个写库变成 3 个写库，每个写库设置不同的 auto_increment 初始值，以及相同的增长步长，以保证每个数据库生成的ID是不同的（上图中 DB 01生成0,3,6,9…，DB 02生成1,4,7,10，DB 03生成2,5,8,11…）


### 该模式的缺点
- 丧失了ID生成的“绝对递增性”
- 数据库的写压力依然很大，每次生成ID都要访问数据库
- 数据库的读压力也很大，有些业务场景，可能需要遍历读取所有的数据库
- 扩展性不太好，如果三台Mysql实例不够用，需要新增Mysql实例来提高性能时，这时就会比较麻烦


## 方案三：UUID

### UUID概念
UUID是指在一台机器在同一时间中生成的数字在所有机器中都是唯一的。按照开放软件基金会(OSF)制定的标准计算，用到了以太网卡地址、纳秒级时间、芯片ID码和许多可能的数字
UUID由以下几部分的组合：
（1）当前日期和时间。
（2）时钟序列。
（3）全局唯一的IEEE机器识别号，如果有网卡，从网卡MAC地址获得，没有网卡以其他方式获得。
标准的UUID格式为：xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx (8-4-4-4-12)，以连字号分为五段形式的36个字符，示例：550e8400-e29b-41d4-a716-446655440000

### UUID 生成方式
uuid是一种常见的本地生成ID的方法。Java标准类库中已经提供了UUID的API。

```
UUID uuid = UUID.randomUUID();  
```

### 优点：
- 本地生成ID，不需要进行远程调用
- 扩展性好，基本可以认为没有性能上限

### 缺点：
- 无法保证趋势递增
- 不易存储：UUID太长，16字节128位，通常以36长度的字符串表示，很多场景不适用。
- uuid过长，往往用字符串表示，作为主键建立索引查询效率低


## 方案4：雪花算法

snowflake 是 twitter 开源的分布式ID生成算法，其核心思想为，一个long型的ID：

![](/images/posts/transition/雪花算法.png)

- 41 bit 是标识部分，在java中由于long的最高位是符号位，正数是0，负数是1，一般生成的ID为正数，所以固定为0。
- 41 bit 作为毫秒数 ； 41位的长度可以使用69年
- 10 bit 作为机器编号 ；（5个bit是数据中心，5个bit的机器ID） - 10位的长度最多支持部署1024个节点
- 12 bit 作为毫秒内序列号 ； 12位的计数顺序号支持每个节点每毫秒产生4096个ID序号


根据这个算法的逻辑，只需要将这个算法用Java语言实现出来，封装为一个工具方法，那么各个业务应用可以直接使用该工具方法来获取分布式ID，只需保证每个业务应用有自己的工作机器id即可，而不需要单独去搭建一个获取分布式ID的应用。


### 雪花算法代码实现
```
public class SnowflakeIdGenerator {
    //================================================Algorithm's Parameter=============================================
    // 系统开始时间截 (UTC 2017-06-28 00:00:00)
    private final long startTime = 1498608000000L;
    // 机器id所占的位数
    private final long workerIdBits = 5L;
    // 数据标识id所占的位数
    private final long dataCenterIdBits = 5L;
    // 支持的最大机器id(十进制)，结果是31 (这个移位算法可以很快的计算出几位二进制数所能表示的最大十进制数)
    // -1L 左移 5位 (worker id 所占位数) 即 5位二进制所能获得的最大十进制数 - 31
    private final long maxWorkerId = -1L ^ (-1L << workerIdBits);
    // 支持的最大数据标识id - 31
    private final long maxDataCenterId = -1L ^ (-1L << dataCenterIdBits);
    // 序列在id中占的位数
    private final long sequenceBits = 12L;
    // 机器ID 左移位数 - 12 (即末 sequence 所占用的位数)
    private final long workerIdMoveBits = sequenceBits;
    // 数据标识id 左移位数 - 17(12+5)
    private final long dataCenterIdMoveBits = sequenceBits + workerIdBits;
    // 时间截向 左移位数 - 22(5+5+12)
    private final long timestampMoveBits = sequenceBits + workerIdBits + dataCenterIdBits;
    // 生成序列的掩码(12位所对应的最大整数值)，这里为4095 (0b111111111111=0xfff=4095)
    private final long sequenceMask = -1L ^ (-1L << sequenceBits);
    //=================================================Works's Parameter================================================
    /**
     * 工作机器ID(0~31)
     */
    private long workerId;
    /**
     * 数据中心ID(0~31)
     */
    private long dataCenterId;
    /**
     * 毫秒内序列(0~4095)
     */
    private long sequence = 0L;
    /**
     * 上次生成ID的时间截
     */
    private long lastTimestamp = -1L;
    //===============================================Constructors=======================================================
    /**
     * 构造函数
     *
     * @param workerId     工作ID (0~31)
     * @param dataCenterId 数据中心ID (0~31)
     */
    public SnowflakeIdGenerator(long workerId, long dataCenterId) {
        if (workerId > maxWorkerId || workerId < 0) {
            throw new IllegalArgumentException(String.format("Worker Id can't be greater than %d or less than 0", maxWorkerId));
        }
        if (dataCenterId > maxDataCenterId || dataCenterId < 0) {
            throw new IllegalArgumentException(String.format("DataCenter Id can't be greater than %d or less than 0", maxDataCenterId));
        }
        this.workerId = workerId;
        this.dataCenterId = dataCenterId;
    }
    // ==================================================Methods========================================================
    // 线程安全的获得下一个 ID 的方法
    public synchronized long nextId() {
        long timestamp = currentTime();
        //如果当前时间小于上一次ID生成的时间戳: 说明系统时钟回退过 - 这个时候应当抛出异常
        if (timestamp < lastTimestamp) {
            throw new RuntimeException(
                    String.format("Clock moved backwards.  Refusing to generate id for %d milliseconds", lastTimestamp - timestamp));
        }
        //如果是同一时间生成的，则进行毫秒内序列
        if (lastTimestamp == timestamp) {
            sequence = (sequence + 1) & sequenceMask;
            //毫秒内序列溢出 即 序列 > 4095
            if (sequence == 0) {
                //阻塞到下一个毫秒,获得新的时间戳
                timestamp = blockTillNextMillis(lastTimestamp);
            }
        }
        //时间戳改变，毫秒内序列重置
        else {
            sequence = 0L;
        }
        //上次生成ID的时间截
        lastTimestamp = timestamp;
        //移位并通过或运算拼到一起组成64位的ID
        return ((timestamp - startTime) << timestampMoveBits) //
                | (dataCenterId << dataCenterIdMoveBits) //
                | (workerId << workerIdMoveBits) //
                | sequence;
    }
    // 阻塞到下一个毫秒 即 直到获得新的时间戳
    protected long blockTillNextMillis(long lastTimestamp) {
        long timestamp = currentTime();
        while (timestamp <= lastTimestamp) {
            timestamp = currentTime();
        }
        return timestamp;
    }
    // 获得以毫秒为单位的当前时间
    protected long currentTime() {
        return System.currentTimeMillis();
    }
    //====================================================Test Case=====================================================
    public static void main(String[] args) {
        SnowflakeIdGenerator idWorker = new SnowflakeIdGenerator(0, 0);
        for (int i = 0; i < 1000; i++) {
            long id = idWorker.nextId();
            System.out.println(Long.toBinaryString(id));
            System.out.println(id);
        }
    }
}  
```

### 优点
- 简单高效，生成速度快。
- 时间戳在高位，自增序列在低位，整个ID是趋势递增的，按照时间有序递增。
- 灵活度高，可以根据业务需求，调整 bit 位的划分，满足不同的需求。


### 缺点
- 依赖机器的时钟，如果服务器时钟回拨，会导致重复 ID 生成。
- 在分布式环境上，每个服务器的时钟不可能完全同步，有时会出现不是全局递增的情况。

## 方法五：使用 Redis 来生成 id
当使用数据库来生成ID性能不够要求的时候，我们可以尝试使用Redis来生成ID。这主要依赖于Redis是单线程的，所以也可以用生成全局唯一的ID。可以用Redis的原子操作 INCR 和 INCRBY 来实现。

### 优点：
- 依赖于数据库，灵活方便，且性能优于数据库。
- 数字ID天然排序，对分页或者需要排序的结果很有帮助。

### 缺点：
- 如果系统中没有Redis，还需要引入新的组件，增加系统复杂度。
- 需要编码和配置的工作量比较大。
- Redis 异常挂掉了，重启Redis后会可能出现ID重复。


