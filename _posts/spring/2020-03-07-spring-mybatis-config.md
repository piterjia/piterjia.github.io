---
layout: post
title: 【spring boot系列】 MyBatis Generator 配置与集成使用
categories: Spring
description: Spring boot 中 MyBatis Generator 配置与使用
keywords: SpringBoot, MyBatis, Generator
---

MyBatis Generator 是 MyBatis 提供的一个代码生成工具。可以帮我们生成 表对应的持久化对象(po)、操作数据库的接口(dao)、CRUD sql的xml(mapper)。
MyBatis Generator 是一个独立工具，你可以下载它的jar包来运行、也可以在 Ant 和 maven 运行。

## pom 整体配置

```
<build>
    <plugins>
        <plugin>
            <groupId>org.mybatis.generator</groupId>
            <artifactId>mybatis-generator-maven-plugin</artifactId>
            <version>1.3.7</version>
            <configuration>
                <!--mybatis的代码生成器的配置文件-->
                <configurationFile>src/main/resources/mybatis-generator-config.xml</configurationFile>
                <!--允许覆盖生成的文件-->
                <overwrite>true</overwrite>
                <!--将当前pom的依赖项添加到生成器的类路径中-->
                <!--<includeCompileDependencies>true</includeCompileDependencies>-->
            </configuration>
            <dependencies>
                <!--mybatis-generator插件的依赖包-->
                <!--<dependency>-->
                    <!--<groupId>org.mybatis.generator</groupId>-->
                    <!--<artifactId>mybatis-generator-core</artifactId>-->
                    <!--<version>1.3.7</version>-->
                <!--</dependency>-->
                <!-- mysql的JDBC驱动 -->
                <dependency>
                    <groupId>mysql</groupId>
                    <artifactId>mysql-connector-java</artifactId>
                    <version>8.0.17</version>
                </dependency>
            </dependencies>
        </plugin>
    </plugins>
<build>    
```

## MyBatis Generator Config 整体配置
```
<?xml version="1.0" encoding="UTF-8" ?>
<!--mybatis的代码生成器相关配置-->
<!DOCTYPE generatorConfiguration
        PUBLIC "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"
        "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">

<generatorConfiguration>
    <!-- 引入配置文件 -->
    <properties resource="application-dev.properties"/>

    <!-- 一个数据库一个context,context的子元素必须按照它给出的顺序
        property*,plugin*,commentGenerator?,jdbcConnection,javaTypeResolver?,
        javaModelGenerator,sqlMapGenerator?,javaClientGenerator?,table+
    -->
    <context id="myContext" targetRuntime="MyBatis3" defaultModelType="flat">

        <!-- 这个插件给生成的Java模型对象增加了equals和hashCode方法 -->
        <!--<plugin type="org.mybatis.generator.plugins.EqualsHashCodePlugin"/>-->

        <!-- 注释 -->
        <commentGenerator>
            <!-- 是否不生成注释 -->
            <property name="suppressAllComments" value="true"/>
            <!-- 不希望生成的注释中包含时间戳 -->
            <!--<property name="suppressDate" value="true"/>-->
            <!-- 添加 db 表中字段的注释，只有suppressAllComments为false时才生效-->
            <!--<property name="addRemarkComments" value="true"/>-->
        </commentGenerator>


        <!-- jdbc连接 -->
        <jdbcConnection driverClass="${spring.datasource.driverClassName}"
                        connectionURL="${spring.datasource.url}"
                        userId="${spring.datasource.username}"
                        password="${spring.datasource.password}">
            <!--高版本的 mysql-connector-java 需要设置 nullCatalogMeansCurrent=true-->
            <property name="nullCatalogMeansCurrent" value="true"/>
        </jdbcConnection>

        <!-- 类型转换 -->
        <javaTypeResolver>
            <!--是否使用bigDecimal，默认false。
                false，把JDBC DECIMAL 和 NUMERIC 类型解析为 Integer
                true，把JDBC DECIMAL 和 NUMERIC 类型解析为java.math.BigDecimal-->
            <property name="forceBigDecimals" value="true"/>
            <!--默认false
                false，将所有 JDBC 的时间类型解析为 java.util.Date
                true，将 JDBC 的时间类型按如下规则解析
                    DATE	                -> java.time.LocalDate
                    TIME	                -> java.time.LocalTime
                    TIMESTAMP               -> java.time.LocalDateTime
                    TIME_WITH_TIMEZONE  	-> java.time.OffsetTime
                    TIMESTAMP_WITH_TIMEZONE	-> java.time.OffsetDateTime
                -->
            <!--<property name="useJSR310Types" value="false"/>-->
        </javaTypeResolver>

        <!-- 生成实体类地址 -->
        <javaModelGenerator targetPackage="com.piter.mall.user.po" targetProject="src/main/java">
            <!-- 是否让 schema 作为包的后缀，默认为false -->
            <!--<property name="enableSubPackages" value="false"/>-->
            <!-- 是否针对string类型的字段在set方法中进行修剪，默认false -->
            <property name="trimStrings" value="true"/>
        </javaModelGenerator>


        <!-- 生成Mapper.xml文件 -->
        <sqlMapGenerator targetPackage="mapper" targetProject="src/main/resources">
            <!--<property name="enableSubPackages" value="false"/>-->
        </sqlMapGenerator>

        <!-- 生成 XxxMapper.java 接口-->
        <javaClientGenerator targetPackage="com.piter.mall.user.dao" targetProject="src/main/java" type="XMLMAPPER">
            <!--<property name="enableSubPackages" value="false"/>-->
        </javaClientGenerator>


        <!-- schema为数据库名，oracle需要配置，mysql不需要配置。
             tableName为对应的数据库表名
             domainObjectName 是要生成的实体类名(可以不指定，默认按帕斯卡命名法将表名转换成类名)
             enableXXXByExample 默认为 true， 为 true 会生成一个对应Example帮助类，帮助你进行条件查询，不想要可以设为false
             -->
        <table schema="" tableName="user" domainObjectName="User"
               enableCountByExample="false" enableDeleteByExample="false" enableSelectByExample="false"
               enableUpdateByExample="false" selectByExampleQueryId="false">
            <!--是否使用实际列名,默认为false-->
            <!--<property name="useActualColumnNames" value="false" />-->
        </table>
    </context>
</generatorConfiguration>
```

## 外部配置文件整体配置

```
# mysql
spring.datasource.driverClassName=com.mysql.cj.jdbc.Driver
spring.datasource.url=jdbc:mysql://10.211.55.5:3306/mall?useUnicode=true&characterEncoding=utf-8&useSSL=false&serverTimezone=Asia/Shanghai
spring.datasource.username=root
spring.datasource.password=123456
```

## 使用 MyBatis Generator
配置好后，双击 maven 中的 MyBatis Generator 运行

![](/images/posts/spring/mybatis.png)



## 使用 MyBatis 注意事项

### 项目最终架构

项目最终的目录结构如下
![](/images/posts/spring/mybatis1.png)

### 启动类添加 mapper 扫描
```
@SpringBootApplication
@MapperScan(value = "com.piter.mall.user.dao")
public class MallApplication {

	public static void main(String[] args) {
		SpringApplication.run(MallApplication.class, args);
	}

}
```

### 配置文件设置

application-dev.properties 中添加 mapper location 配置

```
# mysql
spring.datasource.driverClassName=com.mysql.cj.jdbc.Driver
spring.datasource.url=jdbc:mysql://10.211.55.5:3306/mall?useUnicode=true&characterEncoding=utf-8&useSSL=false&serverTimezone=Asia/Shanghai
spring.datasource.username=root
spring.datasource.password=123456

mybatis.mapper-locations=classpath:mapper/*.xml
```

### 启动自测试

检查是否可以正常操作数据库


## 源码地址
[springboot-mybatis-demo](https://github.com/piterjia/springboot-mybatis-demo)



## 参考：
https://juejin.im/post/5db694e3e51d452a2e25ba45#heading-20
http://mybatis.org/generator/index.html

