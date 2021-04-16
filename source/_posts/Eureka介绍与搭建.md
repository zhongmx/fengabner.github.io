---
title: Eureka介绍与搭建
date: 2021-04-14 14:19:45
categories: springcloud
tags:
	- eureka
	- springcloud
---

# 简介

Eureka 是 Netflix 开源的一款注册中心，它提供服务注册与服务发现功能。

官方架构图

<!-- more -->

![eureka6](http://tencent.fengabner.com/ap-guangzhoueureka6.png)

# 单体服务搭建

环境 idea+java11+maven

建立父工程 <font color='cornflowerblue'>demo-parent</font>

pom.xml文件如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>org.fengabner</groupId>
    <artifactId>cloud-parent</artifactId>
    <version>1.0-SNAPSHOT</version>
    <!--父工程打包方式为pom-->
    <packaging>pom</packaging>

    <!--spring boot 父启动器依赖-->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.6.RELEASE</version>
    </parent>
    <properties>
        <lombok.version>1.18.4</lombok.version>
        <jaxb-core.version>2.2.11</jaxb-core.version>
        <jaxb-impl.version>2.2.11</jaxb-impl.version>
        <jaxb-runtime.version>2.2.10-b140310.1920</jaxb-runtime.version>
        <activation.version>1.1.1</activation.version>
    </properties>

    <dependencyManagement>
        <dependencies>
            <!--spring cloud依赖管理，引入了Spring Cloud的版本-->
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>Greenwich.RELEASE</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>


    <dependencies>
        <!--web依赖-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <!--日志依赖-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-logging</artifactId>
        </dependency>
        <!--测试依赖-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <!--lombok工具-->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>${lombok.version}</version>
            <scope>provided</scope>
        </dependency>
        <!-- Actuator可以帮助你监控和管理Spring Boot应用-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <!--热部署-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
            <optional>true</optional>
        </dependency>

        <!--引⼊Jaxb，开始-->
        <dependency>
            <groupId>com.sun.xml.bind</groupId>
            <artifactId>jaxb-core</artifactId>
            <version>${jaxb-core.version}</version>
        </dependency>
        <dependency>
            <groupId>javax.xml.bind</groupId>
            <artifactId>jaxb-api</artifactId>
        </dependency>
        <dependency>
            <groupId>com.sun.xml.bind</groupId>
            <artifactId>jaxb-impl</artifactId>
            <version>${jaxb-impl.version}</version>
        </dependency>
        <dependency>
            <groupId>org.glassfish.jaxb</groupId>
            <artifactId>jaxb-runtime</artifactId>
            <version>${jaxb-runtime.version}</version>
        </dependency>
        <dependency>
            <groupId>javax.activation</groupId>
            <artifactId>activation</artifactId>
            <version>${activation.version}</version>
        </dependency>
        <!--引⼊Jaxb，结束-->

    </dependencies>

    <build>
        <plugins>
            <!--编译插件-->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <source>11</source>
                    <target>11</target>
                    <encoding>utf-8</encoding>
                </configuration>
            </plugin>
            <!--打包插件-->
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

基于Maven构建SpringBoot⼯程，在SpringBoot⼯程之上搭建EurekaServer服务  

子工程 <font color='red'>**demo-eureka-server1**</font>

pom.xml如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.fengabner</groupId>
        <artifactId>cloud-parent</artifactId>
        <version>1.0-SNAPSHOT</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>com.fengabner</groupId>
    <artifactId>demo-eureka-server1</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>demo-eureka-server1</name>
    <description>Demo project for Spring Boot</description>
    <properties>
        <java.version>1.8</java.version>
    </properties>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
        </dependency>
    </dependencies>
</project>
```

<font color='red'>**注意：在⽗⼯程的pom⽂件中⼿动引⼊jaxb的jar，因为Jdk9之后默认没有加载该模块，
EurekaServer使⽤到，所以需要⼿动导⼊，否则EurekaServer服务⽆法启动**  </font>

查看父工程中以下引入

```xml
<!--引⼊Jaxb，开始-->
<dependency>
    <groupId>com.sun.xml.bind</groupId>
    <artifactId>jaxb-core</artifactId>
    <version>${jaxb-core.version}</version>
</dependency>
<dependency>
    <groupId>javax.xml.bind</groupId>
    <artifactId>jaxb-api</artifactId>
</dependency>
<dependency>
    <groupId>com.sun.xml.bind</groupId>
    <artifactId>jaxb-impl</artifactId>
    <version>${jaxb-impl.version}</version>
</dependency>
<dependency>
    <groupId>org.glassfish.jaxb</groupId>
    <artifactId>jaxb-runtime</artifactId>
    <version>${jaxb-runtime.version}</version>
</dependency>
<dependency>
    <groupId>javax.activation</groupId>
    <artifactId>activation</artifactId>
    <version>${activation.version}</version>
</dependency>
<!--引⼊Jaxb，结束-->
```

配置启动类

```java
@SpringBootApplication
@EnableEurekaServer
public class DemoEurekaServer1Application {

    public static void main(String[] args) {
        SpringApplication.run(DemoEurekaServer1Application.class, args);
    }

}
```

配置配置文件

```yaml
server:
  port: 18061
eureka:
  instance:
    # 应⽤名称，会在Eureka中作为服务的id标识
    hostname: localhost
  client:
    #由于该应用为注册中心,所以设置为false,代表不向注册中心注册自己
    registerWithEureka: false
    #由于注册中心的职责就是维护服务实例,它并不需要去检索服务,所以也设置为false
    fetchRegistry: false
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
spring:
  freemarker:
    prefer-file-system-access: false
```

启动后，访问 ip:port 如 http://localhost:18601

![eureka1](http://tencent.fengabner.com/ap-guangzhoueureka1.png)

说明：

![eureka2](http://tencent.fengabner.com/ap-guangzhoueureka2.png)

# 集群服务搭建

在上面的基础上再次新建一个子工程<font color='red'> **demo-eureka-server2**</font>，配置与<font color='red'> **demo-eureka-server1**</font> 一样，设置一个不同的端口

使用本机环境测试很难模拟多主机的情况  ，需要修改host，将来正式环境使用真实的就行。

修改 hosts

```xml
127.0.0.1 eurekaServer1
127.0.0.1 eurekaServer2
```

修改 <font color='red'> **demo-eureka-server1**</font> 配置文件

```yaml
server:
  port: 18061
eureka:
  instance:
    # 应⽤名称，会在Eureka中作为服务的id标识
    hostname: eurekaServer1
  client:
    #集群模式下，需向注册中心注册自己 设置为true
    registerWithEureka: true
    #集群模式下，需要去检索服务,设置为true
    fetchRegistry: true
    serviceUrl:
      defaultZone: http://eurekaServer2:18062/eureka
  dashboard:
    enabled: true
spring:
  freemarker:
    prefer-file-system-access: false
  application:
    name: demo-eureka-server1
```

修改 <font color='red'> **demo-eureka-server2**</font> 配置文件

```yaml
server:
  port: 18062
eureka:
  instance:
    # 应⽤名称，会在Eureka中作为服务的id标识
    hostname: eurekaServer2
  client:
    #集群模式下，需向注册中心注册自己 设置为true
    registerWithEureka: true
    #集群模式下，需要去检索服务,设置为true
    fetchRegistry: true
    serviceUrl:
      defaultZone: http://eurekaServer1:18061/eureka
spring:
  freemarker:
    prefer-file-system-access: false
  application:
    name: demo-eureka-server2
```

启动两个服务，查看如下

![image-20210413142756642](http://tencent.fengabner.com/ap-guangzhoueureka3.png)

![image-20210413142841378](http://tencent.fengabner.com/ap-guangzhoueureka4.png)