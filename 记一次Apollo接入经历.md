# 记一次Apollo接入经历

### 准备

当接到开发需求的时候其实还是有些担心的，因为之前对Apollo只有简单的了解，并没有实际上手的经验，所以有些担心是否能够在期限内完成接入。

先是简单的看了一下部门里一个大佬写的Apollo接入文档，简单浏览下来并不难，具体分为以下几步：

#### 第一步，引入依赖

```xml
	<dependency>
        <groupId>com.czb.framework</groupId>
        <artifactId>apollo-spring-boot-starter</artifactId>
        <version>1.0.11</version>
    </dependency>
```

这个是必须的，从groupId可以看出这是公司又进行了一次封装之后的jar包，简单的从Idea的maven工具查看了一下内部所包含的包，大概包含了日志和一些工具类的封装。

#### 第二步，添加properties配置

```yml
### app.id唯一指定了一个在apollo创建的项目
app.id: xxx

#下面这项是本地访问所需配置
apollo.meta: xxx
```

#### 第三步，整理项目配置

我这的情况是都被写在了`pom.xml`文件中，分为`local`、`test`和`prod`三个环境，我把这些配置都整理成了properties文件，方便后续在Apollo的配置中直接粘贴复制。

#### 第四步，申请Apollo账号，创建项目

找了相关的小伙伴申请到了Apollo的账后，就开始创建项目，我在参考了高巍哥在gitlab上提交的记录后，打算是创建一个项目，在Apollo默认的TEST环境列表中创建两个集群：DEV和TEST来分别存放本地配置和测试环境配置，在PROD的默认集群中配置生产环境配置。对于一些公共的配置，就是`application.yml`的配置，我计划创建一个公共的namespace，然后每个集群都关联这个公共的namespace来完成配置。

然而一开始就出了问题，不小心把管理员给了部门主管华哥，尴尬了。。。权限转不回来，华哥主动帮我配置了集群当要创建namespace时华哥提出**新建一个项目**作为数据库的聚合层，专门用来存储各个环境下数据库的配置，并在此项目中创建一个公共的namespace来代替我之前的方案，并提醒我在编写代码时注意namespace是**大小写敏感的**，仔细想想这样做确实可以，按照组件来维护嘛，而且后来也发现部门的其他项目也可以使用此数据库配置，单独抽取出来合理！

#### 第五步，编写代码

```java
@Component
@EnableApolloConfig
@Slf4j
public class ApolloConfigCenter {

    @ApolloConfig
    private Config applicationConfig;

    @ApolloConfigChangeListener
    private void applicationOnChange(ConfigChangeEvent event) {
        Set<String> keys = event.changedKeys();
        keys.forEach(key -> log.info("[ApolloConfigChange] application keys:" + key + "change, new value:{}", applicationConfig.getProperty(key, "")));
    }
}
```

这是我Apollo的配置类

#### 第六步，运行

后面因为多了一个项目的配置需要引进，我就修改了`application.yml`文件，想当然的将app.id的配置改成了两个

```yml
app.id: xxx,yyy
```

运行项目，结果是运行失败，后来产生了疑问，是不是app.id不支持配置成两个？我把app.id设置成一个运行项目，项目是启动成功的，但是是肯定读取不到数据库配置的，然而应该是springboot对数据库有默认设置什么的，没有报错。

又咨询了华哥，华哥说新项目配置的namespace本就是公共的，代码里默认只会获取app.id定义的那个项目中指定集群下的application namespace的配置，其他的namespace需要手动指定。于是上网搜索后发现可以通过`apollo.bootstrap.namespaces`参数进行配置，并且它是支持多个namespace的，我便添加了如下配置到yml文件：

```yml
apollo.bootstrap.namespaces: applicaiton,datasource.xxx
```

项目成功的启动起来了，我参照高巍哥的代码，我在其中一个Controller中写了个接口来测试从Apollo拉取的配置，代码如下：

```java
	@ApolloConfig
    private Config config;

	@RequestMapping(value = "testApollo", method = RequestMethod.GET)
    public String testApollo() {
        String apollo = config.getProperty("test-apollo", "");
        return apollo;
    }
```

问题又来了，这个接口可以正常的获取到**application**这个namespace下的配置，然而无法获取到**datasource**这个namespace的配置。又是在网上一顿搜索也没搜出个结果，这里顺便吐槽一下网上相似的文章是真多。但是找到了另一种配置namespace的方法那就是在`@EnableApolloConfig`主街上下功夫，可以`@EnableApolloConfig({"application","datasource.XXX"})`这样配置。改变了配置方案之后依然还是不行。

最后的最后在高巍哥的帮助下找到了问题的所在：能够拿到application配置却拿不到datasource.xxx的原因是因为我的接口里面配置的config变量使用的`@ApolloConfig`注解没有绑定namespace，而没有绑定的情况下默认是application的namespace，所以问题就这样产生了。在我将注解改为：`@ApolloConfig("datasource.xxx")`后，再次去拿application的配置就拿不到了，然而datasource下的配置就可以拿到了。

至此问题就明了了。最后将该namespace的配置加入到Apollo的配置类中：

```java
@Component
@EnableApolloConfig
@Slf4j
public class ApolloConfigCenter {

    @ApolloConfig
    private Config applicationConfig;

    @ApolloConfig("datasource.xxx")
    private Config datasourceConfig;

    @ApolloConfigChangeListener
    private void applicationOnChange(ConfigChangeEvent event) {
        Set<String> keys = event.changedKeys();
        keys.forEach(key -> log.info("[ApolloConfigChange] application keys:" + key + "change, new value:{}", applicationConfig.getProperty(key, "")));
    }

    @ApolloConfigChangeListener
    private void datasourceOnChange(ConfigChangeEvent event) {
        Set<String> keys = event.changedKeys();
        keys.forEach(key -> log.info("[ApolloConfigChange] datasource keys:" + key + "change, new value:{}", datasourceConfig.getProperty(key, "")));
    }
}
```

这样就搞完了，没想到Apollo是这样的机制，这几个小时折腾下来也算是有点收获了，但是感觉对Apollo的理解还不够，有时间再整整~