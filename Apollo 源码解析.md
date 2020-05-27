# Apollo 源码解析之概述

### 介绍

具体介绍官方也有，都知道它是携程框架开发部开发的一款应用于管理项目配置的**分布式配置中心**，适用于微服务配置管理场景

它的服务端是基于SpringBoot和SpringCloud开发，这就使得它不需要额外安装tomcat，而可以直接运行

Java客户端不依赖任何框架，并可以轻松地整合到SpringBoot/Spring环境中

### 单机部署示意图

![](http://www.iocoder.cn/images/Apollo/2017-01-01/01.png)

这里就涉及到最重要的三个模块，之前也说过，就是`Configservice`、`AdminService`和`Portal`，其中：

- ConfigService：为apollo客户端，即我们的Java应用程序提供服务，主要是配置的读取、推送等功能。基本不太会更新、添加新的接口
- AdminService：为apollo的portal，即apollo的管理界面提供服务，主要是配置的修改、发布等功能。后期将不断地添加新的API接口提供给AdminService
- Portal：apollo的管理界面，主要是用来做配置项的修改，即管理应用的配置

**注意：**在我们的应用中读取apollo的配置时，建议使用`@Value`而非`@ConfigurationProperties`因为当apollo的配置刷新时后者无法获取到新的配置，而前者可以。具体原因需要看源码。

