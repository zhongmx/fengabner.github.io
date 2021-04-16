---
title: 'tomcat总体架构'
date: 2021-04-12 11:29:05
categories: tomcat
tags:
	- tomcat
---

# 总体架构设计

Tomcat既按照 Servlet 规范的要求去实现了 Servlet 容器，同时它也具有HTTP服务器的功能。

Tomcat 具备两个身份 「`Http` 服务器 + `Servlet` 容器」

<!-- more -->

![tomcat02](http://tencent.fengabner.com/ap-guangzhoutomcat02.png)

## Tomcat 总体结构图

![image-20210406181308096](http://tencent.fengabner.com/ap-guangzhoutomcat01.png)

+ <font color='red'>Server</font> 是最顶级的组件，代表Tomcat的运行实例，在一个jvm中止会包含一个Server。
+ <font color='red'>Service</font> 一个Server可以包含至少一个Service，用于具体提供服务。
+ <font color='red'>Connector</font> 一个Service 可以包含多个Connector。用于接受请求并将请求封装成Request和Response来具体处理。（作用是监听客户端请求，并将请求封装提交container处理，然后将处理结果返回客户端）
+ <font color='red'>Container</font> 容器。负责处理用户的Servlet请求。

![tomcat03](http://tencent.fengabner.com/ap-guangzhoutomcat03.png)

+ <font color='red'>Engine</font> 表示一个虚拟主机的引擎。引擎管理多个站点(Host)，一个Service一个Engine。但是⼀个引擎可包含多个Host。
+ <font color='red'>Host</font>  代表一个站点，也可以叫虚拟主机，通过配置Host就可以添加站点。一个Engine可以有多个虚拟主机，每个主机都有对应的域名，在 Tomcat 中，一个 webapps 就代表一个虚拟主机， webapps 也可以配置多个。
+ <font color='red'>Context</font> 表示⼀个Web应⽤程序， ⼀个Web应⽤可包含多个Wrapper。
+ <font color='red'>Wrapper </font>可以有多个Wrapper，每一Wrapper封装着一个Servlet。它负责管理一个 Servlet，包括的 Servlet 的装载、初始化、执行以及资源回收。Wrapper 作为容器中的最底层，不能包含⼦容器。

## Catalina

Tomcat是⼀个由⼀系列可配置（conf/server.xml）的组件构成的Web容器，⽽Catalina是Tomcat的servlet容器。

从另⼀个⻆度来说，**Tomcat** **本质上就是⼀款** **Servlet** **容器**， 因为 Catalina 才是 Tomcat 的核⼼ ， 其他模块都是为Catalina 提供⽀撑的。

如 ： 通过 Coyote 模块提供链接通信，Jasper 模块提供 JSP 引擎，Naming 提供JNDI 服务，Juli 提供⽇志服务。

分层示意图

![tomcat04](http://tencent.fengabner.com/ap-guangzhoutomcat04.png)



## Coyote

### 概述

Coyote 是Tomcat链接器框架的名称。是Tomcat对外的接口，客户端通过Coyote与服务器建立链接、发送请求并接收响应。

![tomcat05](http://tencent.fengabner.com/ap-guangzhoutomcat05.png)

+ Coyote 封装了底层的⽹络通信（Socket 请求及响应处理）

+ Coyote 使Catalina 容器（容器组件）与具体的请求协议及IO操作⽅式完全解耦

+ Coyote 将Socket 输⼊转换封装为 Request 对象，进⼀步封装后交由Catalina 容器进⾏处理，处理请求完成后, Catalina 通过Coyote 提供的Response 对象将结果写⼊输出流

+ Coyote 负责的**是具体协议（应⽤层）和****IO****（传输层）相关内容**

### IO/协议

`Tomcat`支持的 `I/O` 模型有（传输层）：

- `NIO`：非阻塞 `I/O`，采用 `Java NIO` 类库实现。
- `NIO2`：异步`I/O`，采用 `JDK 7` 最新的 `NIO2` 类库实现。
- `APR`：采用 `Apache`可移植运行库实现，是 `C/C++` 编写的本地库。

>  在 8.0 之前 ，Tomcat 默认采⽤的I/O⽅式为 BIO，之后改为 NIO。 ⽆论 NIO、NIO2 还是 APR， 在性
>
>  能⽅⾯均优于以往的BIO。 如果采⽤APR， 甚⾄可以达到 Apache HTTP Server 的影响性能

`Tomcat`支持的`应用层`协议有：

- `HTTP/1.1`：这是大部分 Web 应用采用的访问协议。
- `AJP`：用于和 Web 服务器集成（如 Apache）。以实现对静态资源的优化以及集群部署，当前支持AJP/1.3。
- `HTTP/2`：HTTP 2.0 大幅度的提升了 Web 性能。下一代HTTP协议 ， 自8.5以及9.0版本之后支持。

### 内部组件及流程

![tomcat06](http://tencent.fengabner.com/ap-guangzhoutomcat06.png)

<font color='red'>**EndPoint**</font>

EndPoint 是 Coyote 通信端点，即通信监听的接⼝，是具体Socket接收和发送处理器，是对传输层的抽象，因此EndPoint⽤来实现TCP/IP协议的。

<font color='red'>**Processor**</font>

Processor 是Coyote 协议处理接⼝ ，如果说EndPoint是⽤来实现TCP/IP协议的，那么Processor⽤来实现HTTP协议，Processor接收来⾃EndPoint的Socket，读取字节流解析成Tomcat Request和Response对象，并通过Adapter将其提交到容器处理，Processor是对应⽤层协议的抽象。

<font color='red'>**ProtocolHandler**</font>

Coyote 协议接⼝， 通过Endpoint 和 Processor ， 实现针对具体协议的处理能⼒。Tomcat 按照协议和I/O 提供了6个实现类 ： AjpNioProtocol ，AjpAprProtocol， AjpNio2Protocol ， Http11NioProtocol ，Http11Nio2Protocol ，Http11AprProtocol。

<font color='red'>**Adapter** </font>

由于协议不同，客户端发过来的请求信息也不尽相同，Tomcat定义了⾃⼰的Request类来封装这些请求信息。ProtocolHandler接⼝负责解析请求并⽣成Tomcat Request类。但是这个Request对象不是标准的ServletRequest，不能⽤Tomcat Request作为参数来调⽤容器。Tomcat设计者的解决⽅案是引⼊CoyoteAdapter，这是适配器模式的经典运⽤，连接器调⽤CoyoteAdapter的Sevice⽅法，传⼊的是Tomcat Request对象，CoyoteAdapter负责将Tomcat Request转成ServletRequest，再调⽤容器。

## http 请求处理

![tomcat07](http://tencent.fengabner.com/ap-guangzhoutomcat07.jpeg)

<font color='cornflowerblue'>假设 请求为 http://localhost:8080/test/index.jsp 请求被发送到本机端口8080</font>

- Connector监听到请求，把该请求交给它所在的Service的Engine来处理，并等待Engine的回应
- Engine获得请求localhost:8080/test/index.jsp，匹配它所有的虚拟主机Host
- Engine匹配到名为localhost的Host(即使匹配不到也把请求交给该Host处理，因为该Host被定义为该Engine的默认主机)
- localhost Host获得请求/test/index.jsp，匹配它所拥有的所有Context
- Host匹配到路径为/test的Context(如果匹配不到就把该请求交给路径名为""的Context去处理)
- path="/test"的Context获得请求/index.jsp，在它的mapping table中寻找对应的servlet
- Context匹配到URL PATTERN为*.jsp的servlet，对应于JspServlet类，构造HttpServletRequest对象和HttpServletResponse对象，作为参数调用JspServlet的doGet或doPost方法
- Context把执行完了之后的HttpServletResponse对象返回给Host
- Host把HttpServletResponse对象返回给Engine
- Engine把HttpServletResponse对象返回给Connector
- Connector把HttpServletResponse对象返回给客户browser