# Apollo本地部署体验

### 背景

现有需求要重新部署线上的apollo，所以准备在本地先跑一跑看一下apollo的部署过程以及尝试了解下如果要重新部署需要改动哪些地方。

### 准备

简单看了下apollo的官方文档(https://github.com/ctripcorp/apollo ) 

准备工作包括：

1. 安装好java和MySql(java是必须的，MySql用来存储用户配置项等信息)
2. 下载好需要的代码，主要是portal、configservive和adminservice这三个起作用
3. 导入官方的sql文件(会创建两个数据库`ApolloConfigDB`和`ApolloPortalDB`)，方便测试使用(如果自己有也可以不导入)

### 开始部署(每个环境一台机器)

#### 执行构建

需要找到源码中的`scripts/build.sh`文件，执行构建。此脚本的作用是将`apollo-adminservice`、`apollo-configservice`和`apollo-portal`三个项目打包。构建时将会下载一些依赖，可以使用阿里云的maven库来加速下载。

构建完成后，源代码中的这三个项目对应的文件夹下将多出`target`文件夹，每个项目的target文件夹中将出现`apollo-adminservice-1.7.0-SNAPSHOT-github.zip`、`apollo-configservice-1.7.0-SNAPSHOT-github.zip`和`apollo-portal-1.7.0-SNAPSHOT-github.zip`三个压缩文件和一些其他文件。

#### 修改配置

将上述压缩文件复制一份，创建一个新的文件夹并放到此文件夹里(这一步可以省略，我是为了方便才这么做的)，解压。解压后将出现三个文件夹。

首先修改portal配置，进入到`apollo-portal-1.7.0-SNAPSHOT-github/config`文件夹下修改`apollo-env.properties`文件，配置好如下两行，我这里是配置了两个环境，开发和线上环境

```properties
dev.meta=http://localhost:8080
pro.meta=http://localhost:8080
```

然后修改同文件夹下的`application-github.properties`，配置好数据库，一般来说每个环境应该有自己的一套数据库(集群部署也是如此)，所以这个配置是我们对此环境下数据库的配置

```properties
# DataSource
spring.datasource.url = jdbc:mysql://localhost:3306/ApolloPortalDB?characterEncoding=utf8
spring.datasource.username = root
spring.datasource.password = ******
```

其次分别到`apollo-configservice-1.7.0-SNAPSHOT-github/config`和`apollo-adminservice-1.7.0-SNAPSHOT-github/config`中修改`application-github.properties`，配置好自己的数据库：

```properties
spring.datasource.url = jdbc:mysql://localhost:3306/ApolloConfigDB?characterEncoding=utf8
spring.datasource.username = root
spring.datasource.password = ******
```

这样就完成了配置

**注意：**这里再说明一下，根据官方文档：portal项目只需要任选一个环境配置一份即可(即，所有环境都使用一个portal)，其他每个环境(这就取决于我们定义了几个环境了)都需要配置**一套**configservice和adminservice项目来提供服务。举个例子：我在上例中共配置了两个环境：dev(开发)和pro(生产)，并且把portal项目布在了dev，configservice和adminservice项目两个环境各一个。这样我的部署结果就是**dev：portal、configservice和adminservice**，**pro：configservice和adminservice**这样的。所以我要保证dev环境的三个项目是同一个数据库的配置，pro环境的两个项目是同一个数据库的配置。这就对应了每个环境一个数据库的概念。(这里的数据库是指整个数据库，当然protal用的是数据库里面的ApolloPortalDB数据库，configservice和adminservice使用的是数据库里面的ApolloConfigDB数据库)

#### 修改数据库

到我们部署了portal项目的那台机器上登录mysql，选择ApolloPortalDB数据库，进入后修改表`ServerConfig`，找到Key为`apollo.portal.envs`的字段，将其对应的Value修改为`dev,pro`(我这里是配置了两个环境，可以配置多个环境，以”英文逗号“分隔。表里也有注释，写了”可支持的环境列表“)。这里配置了几个环境对应了之前的修改的`apollo-env.properties`文件里配置了几个环境

#### 运行服务

分别到对应的`scripts`文件夹下执行`startup.sh`脚本(我是mac系统，所以执行sh文件，linux也同样)

执行顺序为**先configservice、后adminservice再portal**

> **注意：**这里有个坑，第一次运行并不会成功，因为apollo会操作/opt文件夹，所以必须开放此文件夹的读写权限给当前用户才行，可以直接使用：chmod 777 /opt 来开放所有权限

#### 网上验证

进入浏览器，输入：`http://ip:8070/`即可(ip指的是我们部署了portal项目的那台机器的ip)，如果看到apollo的页面则表示成功！

登录后创建项目后会自动提示补全环境，那么点击补全环境就可以看到配置的环境了。

### 部署apollo集群(某个环境多台机器)

#### 执行构建

同上，使用`build.sh`脚本构建，然后解压zip压缩包。

#### 修改配置

依然同上。

然后修改portal项目的config包下的`apollo-env.properties`文件，修改`dev.meta`为`dev.meta=http://ip1:8080,http://ip2:8080`。这里解释下，因为我这个是dev环境的集群，所以我的配置文件里面只有`dev.meta`，然后ip1是我原本dev单机的ip地址，ip2为我新加入集群的机器的ip地址。注意要用英文逗号分开

#### 修改数据库

这一步先执行同上的操作，设置好共有几个环境。

然后，到我们想要构建集群的那个环境，比如我这里是将集群搭建在dev环境，那么我就需要登录dev的mysql数据库，修改`ApolloConfigDB`数据库，找到表`ServerConfig`，修改里面Key为`eureka.service.url`的那个，将其Value改为`http://ip1:8080/eureka/,http://ip2:8080/eureka/`(这里我说明下：ip1为我dev环境原本的机器的ip，ip2为我新加入的想构建为集群的机器的ip。那么dev环境的集群里将有两台机器ip分别为ip1和ip2)。所以这里可以看出两点：

1. 集群里的每个实例不在一台机器上(这也说明可以在一台机器上开放不同端口来部署多个实例，不过这就没有意义了，不能保证高可用，一台机器挂了，整个集群也挂了)
2. 通过Key可以看出来Apollo是使用Eureka来做服务发现和注册的(但是听说Eureka不再维护了？Apollo会改用其他注册中心吗？)

最后修改每个新加入集群的机器的configservice和adminservice的数据库配置为当前dev环境的配置。(保证每个环境只有一个数据库的准则)

#### 运行服务

按照上面的方法运行每台机器的configservice、adminservice，最后运行唯一的portal。

#### 网上验证

进入浏览器，登录到apollo，可以看到基本与上面相比没什么变化。然而，可以尝试停掉dev环境中的其中一台机器(我是dev里部署的集群)，发现apollo依然可用，没有报错，也可以正常修改和发布。

#### 多说一句

如果想要查看apollo有关的机器，可以登录apollo提供的默认账号，或者拥有最高权限的账号(应该可以)点击`管理员工具`再点击`系统信息`即可查看所有注册进来的configservice和adminservice，并通过点击`check`来查看他们是否正常运行。

或直接登录`http://ip:8080`查看eureka中注册了哪些机器和他的是否正常运行。

### 总结

其实这种东西确实需要去浏览像官方文档这样比较权威的文档才能真正的去了解。官方文档里有个视频的链接，很有助于快速上手apollo。还有就是不要光看，还要去实践。说实话折腾了这么多天其实只是总结了这么点东西，基本都在改部署中出现的问题，但是不是真正上手去操作可能也没法让我更了解apollo。虽然很慢但也算了解了基本的单机和集群部署方式。关于apollo还有很多要去学习……