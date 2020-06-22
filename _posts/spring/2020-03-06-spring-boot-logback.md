---
layout: post
title: 【spring boot系列】Java 日志实现框架 Logback 配置详解
categories: Spring
description: Java 日志实现框架 Logback 配置详解
keywords: SpringBoot, MyBatis, Generator
---

Log4j 的早期作者 Ceki Gülcü 于 2006年开发了 SLF4J(The Simple Logging Facade for Java) 门面框架作为 JCL(Jakarta Commons Logging) 的更可靠替代方案。之后 Ceki Gülcü 又开发了新一代日志实现框架 Logback 作为 Slf4j 的实现。

它总共包含三个模块
- logback-core：其它两个模块的基础模块
- logback-classic：它是 log4j 的一个改良版本，同时它完整实现了 slf4j API 使你可以很方便地更换成其它日志系统如 log4j 或 JDK14 Logging
- logback-access：访问模块与 Servlet 容器集成提供通过 Http 来访问日志的功能

## 配置方式

Logback 提供两种配置方式：
1. java编程
2. 配置文件
   - XML 配置文件
   - Groovy 配置文件

## 配置文件的加载顺序

Logback 会按照以下顺序查找配置文件

1. 首先在 classpath 中查找 logback-test.xml，如果找不到
2. Logback 会在 classpath 中查找 logback.groovy，如果找不到
3. Logback 会在 classpath 中查找 logback.xml，如果找不到
4. Logback 将进行自动配置

## 默认配置

即使你不进行任何配置，Logback 也会进行默认配置，那么默认配置配置了那些东西呢？

如果用 xml 来表达默认配置的话，结果如下

```
<configuration>
  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <!-- encoders are assigned the type ch.qos.logback.classic.encoder.PatternLayoutEncoder by default -->
    <encoder>
      <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
    </encoder>
  </appender>

  <root level="debug">
    <appender-ref ref="STDOUT" />
  </root>
</configuration>
```

## Logback 配置文件结构

Logback 配置文件的语法非常灵活。配置文件的最基本结构如下：
![](/images/posts/spring/logback-arch.png)


如上，根节点为 <configuration> 元素，包含零个或多个 <appender> 元素，然后是零个或多个 <logger> 元素，然后是最多一个 <root> 元素。

- logger、root 作为日志的记录器，把它关联到应用的对应的 context 上后，主要用于存放日志对象，也可以定义日志类型、级别。
- appender 主要用于指定日志输出的目的地，目的地可以是控制台、文件、远程套接字服务器、 MySQL、PostreSQL、 Oracle 和其他数据库、 JMS等。

### <configuration> 节点

configuration 节点示例：
```
<configuration debug="true" scan="true" scanPeriod="120 seconds">
  ...
</configuration>
```

#### 开启 Logback 诊断功能

如果你觉得 Logback 有问题，可以开启 Logback 诊断功能。开启方法以 xml 配置为例：

```
<configuration debug="true">
  ...
</configuration>
```

或者

```
<configuration>
  <statusListener class="ch.qos.logback.core.status.OnConsoleStatusListener" />  
  ...
</configuration>
```

#### 修改后自动重新加载配置文件

Logback 支持“热加载” ，即在项目运行过程中更改了配置会自动重新加载

开启方法以 xml 配置为例

```
<configuration scan="true">
  ...
</configuration>
```

默认情况下，每分钟扫描一次配置文件是否有更改。你也可以手动设置频率
```
<configuration scan="true" scanPeriod="120 seconds">
  ...
</configuration>
```

>注意 scanPeriod 值的写法，数值英文空格时间单位。 如果未指定时间单位，则默认时间单位为毫秒

当你开启“热加载”后，Logback 会在后台启动一个 ReconfigureOnChangeTask 的 task，此 task 在单独的线程中运行，并按照设置的频率，检查您的配置文件是否已更改


### 变量 property

#### 自定义变量
configuration 下，可以通过 property 来定义一个变量，属性 name 是变量的名称，属性 value 是变量的值

```
<configuration>
  <property name="APP_Name" value="myAppName" />
  ...
</configuration>
```

#### 使用自定义变量

通过 ${变量名} 来引用变量
```
<configuration>
    <property name="LOG_PATTERN" value="%date %level [%thread] %logger{10} [%file : %line] %msg%n" />
    
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>${LOG_PATTERN}</pattern>
        </encoder>
    </appender>
  ...
</configuration>
```


### 日志输出器 appender

appender (日志输出器)，用于将日志按照一定的格式输出到控制台、文件、数据库等地方，logger(日志记录器) 需要使用 appender(日志输出器) 将记录器中的日志输出！

![](/images/posts/spring/logback-appender.png)

appender 有两个必填属性：
1. name ：appender 的名称，任意填写，不要重名就行
2. calss ：某个具体的 appender 的完全类名，它是 appender 的具体实现，Logback 自带常用的几个 appender。
   - ch.qos.logback.core.ConsoleAppender ：将日志输出到控制台的 appender
   - ch.qos.logback.core.rolling.RollingFileAppender ：将日志输出到文件，并按照条件切换输出到新建文件(滚动输出，自动切割)

下面我们一一介绍下 appender 的子节点

#### 编码器 encoder

encoder 负责将事件(日志)转换为字节数组，并将该字节数组写出为 OutputStream。
encoder 是 appender 的子节点，在 encoder 节点中，最重要的是配置 pattern ，它是用来格式化日志输出格式，

```
<appender name="" class="">
    <encoder>
        <pattern></pattern>
        <charset></charset> 
    </encoder>
</appender>
```

#### 1、 encoder 的子节点 pattern
pattern 用于定义日志的输出格式，通过 Logback 中的转换说明符(Conversion specifier)（其实就是一些预定义变量），可以方便的组合出我们想要的日志格式
在这里列举出常用的转换说明符(Conversion specifier)，更多请参考官方文档

##### pattern之 日期、时间
假设系统时间是 2006-10-20 14:06:49,812

![](/images/posts/spring/logback-appender1.png)

##### pattern 之 日志记录器
![](/images/posts/spring/logback-appender2.png)

##### pattern 之 类名、方法名、行号
![](/images/posts/spring/logback-appender3.png)

##### pattern 之 日志信息
![](/images/posts/spring/logback-appender4.png)

##### pattern 之 换行
![](/images/posts/spring/logback-appender5.png)
##### pattern 之 消息请求级别
![](/images/posts/spring/logback-appender6.png)

##### pattern 之 线程
![](/images/posts/spring/logback-appender7.png)


```
<appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
        <pattern>%date{yyyy-MM-dd HH:mm:ss.SSS, Asia/Shanghai} %red(%-5level) --- [%thread] %cyan(%class.%method/%line) : %msg%n</pattern>
    </encoder>
</appender>
```

##### pattern 之 颜色

对于控制台的输出，Logback 支持填充颜色，支持的颜色如下

- %black()
- %red()
- %green()
- %yellow()
- %blue()
- %magenta()
- %cyan()
- %white()
- %gray()
- %boldRed()
- %boldGreen()
- %boldYellow()
- %boldBlue()
- %boldMagenta()
- %boldCyan()
- %boldWhite()
- %highlight()

使用方法是用颜色的代码把消息包起来，比如想将日志级别设置成红色 %red(%-5level)

#### encoder 的子节点 charset

charset 是 encoder 的子节点，用于设置输出字符集

```
<appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
        <pattern>%date{yyyy-MM-dd HH:mm:ss.SSS, Asia/Shanghai} %red(%-5level) --- [%thread] %cyan(%class.%method/%line) : %msg%n</pattern>
        <charset>UTF-8</charset> 
    </encoder>
</appender>
```

#### ConsoleAppender

ch.qos.logback.core.ConsoleAppender 是 Logback 自带的 appender，用于将日志输出到控制台。

对应控制台，只需要重点配置 encoder 节点

```
<configuration scan="true" scanPeriod="60 seconds" debug="false">
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%date %red(%-5level) --- [%thread] %cyan(%class.%method/%line) : %msg%n</pattern>
            <charset>UTF-8</charset> 
        </encoder>
    </appender>

    <root level="info">
        <appender-ref ref="STDOUT" />
    </root>
</configuration>
```

#### RollingFileAppender
ch.qos.logback.core.rolling.RollingFileAppender 是 Logback 自带的 Appender，用于将日志输出到文件，并按照条件切换输出到新建文件(滚动输出，自动切割)

![](../../images/posts/spring/logback-appender8.png)

要使用 RollingFileAppender , 这两个策略 RollingPolicy (滚动策略)和 TriggeringPolicy(触发策略)必不可少。

但是，如果 RollingPolicy 实现了 TriggeringPolicy 接口，则仅需要指定 RollingPolicy。


##### 滚动策略 —— RollingPolicy
RollingPolicy 负责日志文件的移动和重命名的过程。
Logback 提供了多种 RollingPolicy 的实现

- TimeBasedRollingPolicy ：按时间滚动，比如按天或按月
- SizeAndTimeBasedRollingPolicy ：按时间和大小滚动，比如说在按天滚动的基础上设置大小，防止某天的日志文件过大！比如双11

##### TimeBasedRollingPolicy

ch.qos.logback.core.rolling.TimeBasedRollingPolicy 这个滚动策略，同时也实现了 TriggeringPolicy(触发策略)
因此配置好 TimeBasedRollingPolicy 后，就不需要配置 TriggeringPolicy

TimeBasedRollingPolicy 有几个重要的属性

- maxHistory ：删除n个滚动周期之前的日志文件(最多保留前n个滚动周期的历史记录)。假设每分钟发送滚动，并将maxHistory设置为3，则每次发送滚动时，删除3分钟之前的归档文件。
举例： 
  * 12:00 ：归档文件0
  * 12:01 ：归档文件1
  * 12:02 ：无归档文件(比如这一分钟内没有产生log)
  * 12:03 ：无归档文件(比如这一分钟内没有产生log)
  * 12:04 ：触发滚动，删除3个滚动周期之前(也就是3分钟之前)的日志文件，因此，归档文件0 被删除，只剩下归档文件1

>注意，由于删除了旧的归档日志文件，将适当删除为日志文件归档而创建的所有文件夹

- totalSizeCap：在 maxHistory 限制的基础上，进一步限制所有存档文件的总大小。当超过总大小上限时，将异步删除最旧的存档。totalSizeCap 依赖于 maxHistory ，如果没有 maxHistory ，单独设置 totalSizeCap 是不生效的
- cleanHistoryOnStart：默认的删除操作是发送在触发滚动的过渡期间。但是，某些应用程序的生命周期可能不足以触发滚动，对于这种短暂的应用程序，归档删除可能永远都没有执行的机会。通过将 cleanHistoryOnStart 设置为 true，可以在 Appender 启动时执行归档删除。cleanHistoryOnStart 默认值为 false。
- fileNamePattern 是最为复杂也最为重要的属性，同时也是必填项，单独拿出来说

###### fileNamePattern 属性

RollingFileAppender (TimeBasedRollingPolicy的上级) 中的 file 属性可以设置也可以不设置。建议设置，因为通过设置包含 FileAppender 的 file 属性，可以解耦当前活动中日志文件的位置和归档日志文件的位置。

fileNamePattern 的值为日志文件的路径名，支持绝对和相对路径和日期转换符——%d。

举例说明

- 绝对路径: /Users/wqlm/work/project/test/logs/logback/application.log
- 相对路径: logs/logback/application.log
> 提醒：windows 环境注意目录分隔符

**日期转换符——%d**
%d 表示系统当前的时间，默认格式为 %d{yyyy-MM-dd}，通过 %d{日期格式} 来自定义。

**时区**
%d 默认采用主机时区，也可以自定义%d{yyyy-MM-dd,UTC+8}

举例说明
- 例1: logs/logback/%d/application.log
- 例2: logs/logback/application_%d{yyyy-MM-dd,UTC+8}.log

除了以上作用，日期转换符还被用于推断触发滚动的时机！！！

比如，
- 例1 会在每天凌晨触发滚动，创建一个新日期的文件夹和文件。
- 例2 会在每天凌晨触发滚动，创建一个新日期的文件。
  
**多个%d**

使用多个%d,只有有一个用于推断触发滚动的时机，其余的必须加上 aux 参数来标记为非推断条件；

例如 logs/logback/%d{yyyy-MM,aux}/application_%d.log

**文件压缩**

TimeBasedRollingPolicy 支持自动文件压缩。如果 fileNamePattern 属性值以.gz 或.zip结尾，则启用此功能。

情况1: fileNamePattern 启动了压缩，但 appender 没有设置 file

```
<appender name="RFA" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <!--<file>logs/logback/application.log</file>-->
    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
    	<fileNamePattern>logs/logback/application_%d.zip</fileNamePattern>
        ...
    </rollingPolicy>
    ...
</appender>
```

由于没有设置 file，所以 logback 会根据 fileNamePattern 中内容生成一个叫 logs/logback/application_2019-12-31  的无后缀名的日志文件，并将当天日志写入该文件，到了第二天会将 logs/logback/application_2019-12-31 压缩成 logs/logback/application_2019-12-31.zip 并生成 logs/logback/application_2020-01-01 新日志文件


情况2: fileNamePattern启动了压缩， appender 也设置了 file

```
<appender name="RFA" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>logs/logback/application.log</file>
    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
    	<fileNamePattern>logs/logback/application_%d.zip</fileNamePattern>
        ...
    </rollingPolicy>
    ...
</appender>
```

由于设置了 file，所以 logback 会将日志输出到 logs/logback/application.log ，到了第二天 logback 会根据 fileNamePattern 中的设置将 logs/logback/application.log 压缩成 logs/logback/application_2019-12-31.zip 并生成 logs/logback/application.log 新日志文件


**触发滚动的时机**

由于各种技术原因，滚动不是时钟驱动的，而是取决于日志事件的到来。

例如，在2002年3月8日，假设 fileNamePattern 设置为 yyyy-MM-dd （每日滚动），则在 2002-3-9 00:00:00 之后第一个事件的到来将触发滚动。
如果等到 2002-3-9 00:23:47 时，还没有收到日志事件，将不在继续等待，立即触发滚动

因此，根据事件的到达率，可能会有一定的延迟，但不会超过 00:23:47

**总结**
fileNamePattern 属性集多种功能于一身，包括
- 设置动态文件名
- 用于推断触发滚动的时机
- 用于开启压缩策略
- 影响 maxHistory 的单位(滚动周期)
- 本身也描述了如何进行滚动

```
<configuration>
    <appender name="TimeFile" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/app1/app1.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
          <fileNamePattern>logs/app1/app1_%d.zip</fileNamePattern>
           <!--最多保留60个滚动周期的日志文件-->
           <maxHistory>60</maxHistory>
           <!--在 maxHistory 的前提下，所以日志文件总大小不能超过1GB-->
           <totalSizeCap>1GB</totalSizeCap>
        </rollingPolicy>
        <encoder>
            <pattern>%date %-5level --- [%thread] %class.%method/%line : %msg%n</pattern>
        </encoder>
    </appender>
    
    <root level="DEBUG">
        <appender-ref ref="TimeFile" />
    </root>
</configuration>
```

#### SizeAndTimeBasedRollingPolicy

有时候，想在按照时间归档的基础上限制日志文件的大小，这时候就需要使用 SizeAndTimeBasedRollingPolicy。

比如按照每天归档的基础上，限制每个归档日志不能超过 50M ，防止像双 11 这样的特殊日期，导致当日归档日志文件过大

SizeAndTimeBasedRollingPolicy 继承了 TimeBasedRollingPolicy ，因此 TimeBasedRollingPolicy 的功能它全都有,SizeAndTimeBasedRollingPolicy 仅比 TimeBasedRollingPolicy 多出一个 maxFileSize 属性

maxFileSize 属性表示单个归档日志文件的大小，单位有 KB、MB、GB

```
<configuration>
    <appender name="SizeAndTimeFile" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/app1/app1.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
          <fileNamePattern>logs/app1/backup/app1_%d_%i.zip</fileNamePattern>
           <!--单个日志文件最大10MB-->
           <maxFileSize>10MB</maxFileSize>
           <!--删除n个滚动周期之前的日志文件(最多保留前n个滚动周期的历史记录)件-->
           <maxHistory>60</maxHistory>
           <!--在 maxHistory 限制的基础上，进一步限制所有存档文件的总大小-->
           <totalSizeCap>1GB</totalSizeCap>
        </rollingPolicy>
        <encoder>
            <pattern>%date %-5level --- [%thread] %class.%method/%line : %msg%n</pattern>
        </encoder>
    </appender>
    
    <root level="DEBUG">
        <appender-ref ref="SizeAndTimeFile" />
    </root>
</configuration>
```

> %i 是一个滚动周期内的归档序列号，从0开始

>注意：maxHistory 是配置了 n 个滚动周期的日志文件，而不是 n 个的日志文件，在设置了 maxFileSize 后，一个滚动周期内可能有多个日志文件




### 日志记录器 —— logger

```
<logger name="" level="" additivity="" >
    <appender-ref ref="" />
</logger>
```

logger 有三个属性:

- name : 包名或类名。即，将该记录器作用于哪个类或包下。必填
- level : 该记录器的级别，低于该级别的日志消息不记录。可选级别从小到大为 TRACE、DEBUG、INFO、WARN、ERROR、ALL、OFF(不区分大小写)。选填，不填则默认从父记录器继承level
- additivity : 是否追加父 Logger 的输出源(appender)，默认为true，选填。如果只想输出到自己的输出源(appender)，需要设置为 false

logger 还可以包含 0 或多个 appender-ref 元素, 将记录的日志使用 appender 进行输出


#### level 的继承

```
<?xml version="1.0" encoding="UTF-8"?>
<configuration scan="true" scanPeriod="30 seconds">
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%date %level [%thread] %logger{10} [%file : %line] %msg%n</pattern>
        </encoder>
    </appender>

    <logger name="com.wqlm.boot.user.controller.UserController" ></logger>
    <logger name="com.wqlm.boot.user.controller" ></logger>

    <root level="info">
        <appender-ref ref="STDOUT" />
    </root>
</configuration>
```

如上，name="com.wqlm.boot.user.controller.UserController"的 logger 没有定义 level，因此它会继承父 logger 的 level。

那它的父 logger 是谁呢？
其实就是下面的 name="com.wqlm.boot.user.controller"的 logger!

Logback 是根据包或类的层次结构来确定 logger 的父子关系的

继续往下说，name="com.wqlm.boot.user.controller"的 logger 也没有定义 level，因此它也会继承它的父 logger 的 level。

但是，在该 xml 中，已经没有比它层次还“浅”的 logger 了，因此，它的父 logger 为 root。也就是说它会继承 root logger 的 level，也就是 info。

刚才是从上层往下层走，直到找到显示标注 level 的 logger。

现在要从下层往上层走，把继承到的 level 一层一层往上传，直到遇到显示标注 level 的 logger 才停止
因此，name="com.wqlm.boot.user.controller.UserController"的 logger 的 level 也是 info。


以下各 logger 的 level 是什么？
```
<?xml version="1.0" encoding="UTF-8"?>
<configuration scan="true" scanPeriod="30 seconds">
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%date %level [%thread] %logger{10} [%file : %line] %msg%n</pattern>
        </encoder>
    </appender>

    <logger name="com.wqlm.boot.user.controller.UserController" ></logger>
    <logger name="com.wqlm.boot.user.controller" level="warn"></logger>
    <logger name="com.wqlm.boot.user" ></logger>

    <root level="info">
        <appender-ref ref="STDOUT" />
    </root>
</configuration>
```


按照上面讲的继承规则
- name="com.wqlm.boot.user.controller.UserController"的 logger 的 level 是 warn
- name="com.wqlm.boot.user"的 logger 的 level 是 info

#### additivity 的追加策略

```
<?xml version="1.0" encoding="UTF-8"?>
<configuration scan="true" scanPeriod="30 seconds">
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%date %level [%thread] %logger{10} [%file : %line] %msg%n</pattern>
        </encoder>
    </appender>

    <logger name="com.wqlm.boot.user.controller.UserController" ></logger>
    <logger name="com.wqlm.boot.user.controller" ></logger>

    <root level="info">
        <appender-ref ref="STDOUT" />
    </root>
</configuration>
```

如上，name="com.wqlm.boot.user.controller.UserController"的 logger 没有设置 additivity，因此 additivity 采用默认值 true，即，追加它父 logger 中的 appender 到自己的 logger 中。

在看它的父 logger —— name="com.wqlm.boot.user.controller" logger， additivity 也采用默认值，因此，它也会追加它父 logger 中的 appender 到自己的 logger 中。

在该 xml 中 name="com.wqlm.boot.user.controller" logger 的父 logger 为 root logger！，因此，它会追加 名为 STDOUT 的 appender 到自己的 logger 中。

刚才是从上层往下层走，直到找到显示标注 appender="false" 的 logger 或 root logger 才停止追加。

现在要从下层往上层走，把层层追加的 appender 往上传，直到遇到显示标注  appender="false" 的 logger 才停止。

因此，name="com.wqlm.boot.user.controller.UserController"的 logger 中，也有一个名为 STDOUT 的 appender，这个 appender 是从它的父 logger 中来的。

所以，一般情况下，只需要给 root logger 添加 appender 即可！！！其他 logger 都会直接或间接获取到 root logger 的 appender

在看这种情况，每一个 logger 都有一个自己的 appender

```
<?xml version="1.0" encoding="UTF-8"?>
<configuration scan="true" scanPeriod="30 seconds">
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%date %level [%thread] %logger{10} [%file : %line] %msg%n</pattern>
        </encoder>
    </appender>

    <appender name="STDOUT2" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%msg%n</pattern>
        </encoder>
    </appender>
    
    <appender name="STDOUT3" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%msg%n</pattern>
        </encoder>
    </appender>

    <logger name="com.wqlm.boot.user.controller.UserController" additivity="true">
        <appender-ref ref="STDOUT3" />
    </logger>

    <logger name="com.wqlm.boot.user.controller">
        <appender-ref ref="STDOUT2" />
    </logger>

    <root level="info">
        <appender-ref ref="STDOUT" />
    </root>
</configuration>
```

上面的这个例子：
- 首先 name="com.wqlm.boot.user.controller" 的 additivity 没填，默认为 true，因此会追加 root logger 的 appender，这样一来，它就有两个 appender 的了，即 STDOUT 和 STDOUT2
- 同理，name="com.wqlm.boot.user.controller.UserController" 会追加 name="com.wqlm.boot.user.controller" 的 appender，因此，它有三个 appender，STDOUT 、 STDOUT2 、STDOUT3

### 根记录器 —— <root>

```
<root level="">
    <appender-ref ref="" />
</root>
```

只有 level 是必须填写的

level 的可选值有 TRACE、DEBUG、INFO、WARN、ERROR、ALL、OFF，大小写都可以识别

root 也可以包含 0 或多个 appender-ref 元素



## 参考文档：

http://logback.qos.ch/manual/configuration.html



