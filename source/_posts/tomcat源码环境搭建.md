---
title: tomcat源码环境搭建
date: 2021-04-12 14:05:14
categories: tomcat
tags:
	- tomcat
---

# tomcat 源码环境搭建

## 源码下载

[下载地址](https://tomcat.apache.org/)

<!-- more -->

![tomcat08](http://tencent.fengabner.com/ap-guangzhoutomcat08-1617797507419.png)

## 创建resource目录

![tomcat09](http://tencent.fengabner.com/ap-guangzhoutomcat09.png)

<font color='red'>将conf 目录和 webapps 目录放到 resource 目录下</font>

## 创建pom.xml

<font color='red'>在根目录创建 pom.xml</font>

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>org.apache.tomcat</groupId>
    <artifactId>apache-tomcat-8.5.50-src</artifactId>
    <name>Tomcat8.5</name>
    <version>8.5</version>
    <build>
        <!--指定源⽬录-->
        <finalName>Tomcat8.5</finalName>
        <sourceDirectory>java</sourceDirectory>
        <resources>
            <resource>
                <directory>java</directory>
            </resource>
        </resources>
        <plugins>
            <!--引⼊编译插件-->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.1</version>
                <configuration>
                    <encoding>UTF-8</encoding>
                    <source>11</source>
                    <target>11</target>
                </configuration>
            </plugin>
        </plugins>
    </build>
    <!--tomcat 依赖的基础包-->
    <dependencies>
        <dependency>
            <groupId>org.easymock</groupId>
            <artifactId>easymock</artifactId>
            <version>3.4</version>
        </dependency>
        <dependency>
            <groupId>ant</groupId>
            <artifactId>ant</artifactId>
            <version>1.7.0</version>
        </dependency>
        <dependency>
            <groupId>wsdl4j</groupId>
            <artifactId>wsdl4j</artifactId>
            <version>1.6.2</version>
        </dependency>
        <dependency>
            <groupId>javax.xml</groupId>
            <artifactId>jaxrpc</artifactId>
            <version>1.1</version>
        </dependency>
        <dependency>
            <groupId>org.eclipse.jdt.core.compiler</groupId>
            <artifactId>ecj</artifactId>
            <version>4.5.1</version>
        </dependency>
        <dependency>
            <groupId>javax.xml.soap</groupId>
            <artifactId>javax.xml.soap-api</artifactId>
            <version>1.4.0</version>
        </dependency>
    </dependencies>
</project>
```

## 导入IDEA

使用idea打开项目，配置启动类 Bootstrap 

![tomcat11](http://tencent.fengabner.com/ap-guangzhoutomcat11.png)

配置VM启动参数

```xml
-Dcatalina.home=D:/develop/github/apache-tomcat-8.5.64/resource
-Dcatalina.base=D:/develop/github/apache-tomcat-8.5.64/resource
-Djava.util.logging.manager=org.apache.juli.ClassLoaderLogManager
-Djava.util.logging.config.file=D:/develop/github/apache-tomcat-8.5.64/resource/conf/logging.properties
```

## 错误解决

###  HTTP 状态500

![tomcat10](http://tencent.fengabner.com/ap-guangzhoutomcat10.png)

解决方案：

在org.apache.catalina.startup.ContextConfig#configureStart方法中加入以下代码

```java
context.addServletContainerInitializer(new JasperInitializer(), null);
```

### 程序包问题

如报错： java: 程序包 sun.rmi.registry 不可见

解决方案：按照idea提示操作即可