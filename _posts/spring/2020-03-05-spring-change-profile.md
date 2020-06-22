---
layout: post
title: 【spring boot系列】使用 Spring Boot Profile 实现多环境配置
categories: Spring
description: 使用 Spring Boot Profile实现多环境配置
keywords: SpringBoot, Profile
---

## 引言
通过 Spring Boot 的 Profile 可以实现多场景下的配置切换，方便开发中进行测试和部署生产环境。 本文将介绍一下如何使用 Spring Boot Profile 配置文件，配置不同环境的配置文件。

## Spring Boot配置步骤

### 1 在 Spring Boot项目下的 application.yml 或 application.properties 中添加配置项

application.yml
```
spring:
  profiles:
    active: dev
```

application.properties
```
spring.profiles.active: dev
```

spring.profiles.active: dev 表示默认加载的就是开发环境的配置，如果dev换成test，则会加载测试环境的属性，以此类推。

*注意： 如果spring.profiles.active 没有指定值，那么只会使用没有指定 spring.profiles 文件的值，Spring Boot 只会加载 application.yml或 application.properties 中的配置。*


### 2 创建不同环境下的配置文件

例如环境分为开发环境、测试环境和生产环境，创建如下文件：

- 开发环境 : application-dev.yml 或 application-dev.properties
- 测试环境：application-test.yml 或 application-test.properties
- 生产环境：application-prod.yml 或 application-prod.properties


application.yml文件分为四部分,使用 --- 来作为分隔符，第一部分通用配置部分，表示三个环境都通用的属性，用spring.profiles指定了一个值(开发为dev，测试为test，生产为prod)，这个值表示该段配置应该用在哪个profile里面。

例如， 我们在某个项目中根据不同的环境配置不同的数据库信息：

开发环境
```
spring:
  datasource:
     name: druidDataSource
    type: com.alibaba.druid.pool.DruidDataSource
    druid:
      driver-class-name: com.mysql.jdbc.Driver
      url: jdbc:mysql://192.168.1.2:3306/myDB?useUnicode=true&characterEncoding=UTF-8&autoReconnect=true&failOverReadOnly=false&useSSL=false
      username: user1
      password: 123456
```


测试环境
```
spring:
  datasource:
     name: druidDataSource
    type: com.alibaba.druid.pool.DruidDataSource
    druid:
      driver-class-name: com.mysql.jdbc.Driver
      url: jdbc:mysql://192.168.20.2:3306/myTestDB?useUnicode=true&characterEncoding=UTF-8&autoReconnect=true&failOverReadOnly=false&useSSL=false
      username: user1
      password: 123456
```


生产环境
```
spring:
  datasource:
     name: druidDataSource
    type: com.alibaba.druid.pool.DruidDataSource
    druid:
      driver-class-name: com.mysql.jdbc.Driver
      url: jdbc:mysql://172.1.16.2:3306/myProdDB?useUnicode=true&characterEncoding=UTF-8&autoReconnect=true&failOverReadOnly=false&useSSL=false
      username: prod1
      password: prod1234!@#
```

### 3 对应用进行打包操作后， 启动应用

如果是部署到服务器的话,我们正常打成jar包，启动应用时Spring Boot通过application.yml或applcation.properties文件中的spring.profiles.active配置项加载相关环境的配置信息。除此之外， 我们可以通过 --spring.profiles.active=xxx来控制加载哪个环境的配置，参考命令如下:
```
# 表示使用开发环境的配置
java -jar xxx.jar --spring.profiles.active=dev 

# 表示使用测试环境的配置
java -jar xxx.jar --spring.profiles.active=test 

# 表示使用生产环境的配置
java -jar xxx.jar --spring.profiles.active=prod 
```

### 4 添加扩展的配置文件信息

在复杂项目中，我们有可能需要添加一些额外的扩展配置信息， Spring Boot支持项目添加扩展的配置文件， 假设我们在某个功能模块hurricane-ldap中配置需要进行LDAP认证的配置文件application-ldap.yml，我们可以在application.yml或application.properties文件中修改spring.profiles.active配置项， 如下所示：
```
 spring:
     profiles:
        active: dev,ldap
```


## 通过Mavan实现多环境配置打包

### 1 在pom.xml文件中添加多环境配置
```
<!-- Application Environment Setting -->
<profiles>
     <profile>
         <id>dev</id>
         <activation>
             <!-- Default Active Without Assign Parameter -->
             <activeByDefault>true</activeByDefault>
         </activation>
         <properties>
             <profileActive>dev</profileActive>
         </properties>
     </profile>
     <profile>
         <id>test</id>
         <properties>
             <profileActive>test</profileActive>
         </properties>
     </profile>
     <profile>
         <id>prod</id>
         <properties>
             <profileActive>prod</profileActive>
         </properties>
     </profile>
</profiles>
```

*注： 配置文件中添加开发、测试和生产三个环境的配置， 其中应注意profileActive自定义配置项， 该配置项指明应用配置文件的名称， 此配置项将在application.yml或application.properties中应用。*

### 2 修改applcation.yml或application.properties配置项

修改 applcation.yml 或 application.properties 配置项，替换 spring.profies.active 配置项，如下所示：
```
spring:
    profiles:
       active: @profileActive@
```

*注： @profileActive@为上一步骤中pom.xml文件配置的自定义配置项， 该参数可以根据开发人员自身的习惯进行命名和配置。*


### 3 使用maven命令打包成相应环境的应用程序包

- 生产环境
```
mvn clean package -Pprod -U  
# 或者
mvn clean package -DprofileActive=prod -U
```

- 测试环境
```
mvn clean package -Ptest -U  
# 或者
mvn clean package -DprofileActive=test -U
```


- 开发环境
```
mvn clean package -Pdev -U  
# 或者
mvn clean package -DprofileActive=dev -U
```

