#  MyBatisPlus 使用心得

初衷：由于原来的项目里service层过于复杂，mybatis.xml文件写的庞大又复杂，所以想要简化整个项目的流程，将庞大的sql语句去掉，对service层进行整理等等。对于简化sql就想到了使用MyBatisPlus来实现。MyBatisPlus是在MyBatis的基础上的扩展，秉承着“只做增强不做改变”的设计原则，使得代码的开发变得更加简单和快捷。而且MybatisPlus结合了MyBatis和JPA的优点，这就使得它更加强大。

### 整合

注意：**以下都是在项目已经接入MyBatis的基础上做的改动。**

引入jar包:

```java
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-boot-starter</artifactId>
    <version>3.1.0</version>
</dependency>
```

我选择的是使用量比较多的版本，这样出错的话便于解决问题

除此之外，与SpringBoot整合的过程中要配置MyBatisPlus（以下皆称MP）的配置类，我编写的配置类如下：

```java
    @Bean
    public MybatisSqlSessionFactoryBean sqlSessionFactory(@Qualifier("dataSource")DataSource dataSource) throws Exception{
        final MybatisSqlSessionFactoryBean mybatisPlus = new MybatisSqlSessionFactoryBean();
        mybatisPlus.setDataSource(dataSource);
        ResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();
        mybatisPlus.setTypeAliasesPackage(ALIASES_PACKAGE);//实体类路径
        mybatisPlus.setConfigLocation(resolver.getResource(CONFIG_LOCATION));//mybatis配置文件路径
        mybatisPlus.setVfs(SpringBootVFS.class);
        mybatisPlus.setPlugins(new Interceptor[]{new CzbMybatisInterceptor()});
        GlobalConfig globalConfig = new GlobalConfig();
        GlobalConfig.DbConfig dbConfig = new GlobalConfig.DbConfig();
        dbConfig.setIdType(IdType.AUTO);//id为自增
        globalConfig.setDbConfig(dbConfig);
        mybatisPlus.setGlobalConfig(globalConfig);
        return mybatisPlus;
    }
```

另外如果有分页需求，还可以添加MP的分页插件：

```java
    @Bean
    public PaginationInterceptor paginationInterceptor() {
        PaginationInterceptor interceptor = new PaginationInterceptor();
        interceptor.setDialectType("mysql");
        return interceptor;
    }
```

最后还要记得在MainApplication上添加一个`@EnableTransactionManagement`注解。

然后运行，并没有运行成功，而是抛出了一个异常：

```java
Caused by: java.lang.ClassNotFoundException: org.mybatis.logging.LoggerFactory
	at java.net.URLClassLoader.findClass(URLClassLoader.java:382)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:418)
	at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:355)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:351)
	... 68 common frames omitted
```

表示缺少了`LoggerFactory`这个类。然后就开始找问题，最后才发现是因为`mybatis`与`mybatis-spring`版本不匹配的原因，而后对症下药，手动引入jar包：

```java
<dependency>
    <groupId>org.mybatis</groupId>
    <artifactId>mybatis</artifactId>
    <version>3.5.3</version>
</dependency>
<dependency>
    <groupId>org.mybatis</groupId>
    <artifactId>mybatis-spring</artifactId>
    <version>2.0.3</version>
</dependency>
```

运行，成功。

### 注解

MP有三个重要的注解`@TableName`、`@TableField`和`@TableId`。

- @TableName：主要用于实体类，指定实体类与数据库的那个表关联。默认情况下，实体类的驼峰形式对应着数据库表的下划线命名。当实体类名符合默认规则，那么不需要显示的指定@TableName
- @TableId：主要用于实体类中某一属性，指定了与数据库表的的主键的映射。默认情况下，属性的驼峰形式对应数据库表中的字段的下划线命名。当实体中属性名符合默认规则，那么不需要显示的指定@TableId
- @TableField：主要用于实体类中的属性，指定了与数据库表的字段的映射。默认情况下，属性的驼峰形式对应数据库表中的字段的下划线命名。当实体中属性名符合默认规则，那么不需要显示的指定@TableField。另外，此注解内还拥有一个属性exist，用于判断该属性是否在数据库表有对应的字段。默认为true，如果没有对应的字段(即不需要映射到数据库表)，那么就可以将其设为false。**再多说一句，如果在项目运行时实体类中的属性在数据库中没有对应的映射，那么在MP进行操作时是会报错的。**

### 进行操作

下面正式的改造DAO层代码，省去多余的sql文件。可以通过继承MP的`BaseMapper`来实现。该接口可以传入一个泛型，具体制定要操作的实体类。此接口里面定义了基本的CRUD的方法，我们自定义的接口可以通过实现此接口来实现在不写sql.xml文件的情况下完成业务！这个也可以看作是它很像JPA的地方吧。

### 关于查询

##### 普通查询

```java
 /**
     * 根据 ID 查询
     *
     * @param id 主键ID
     */
    T selectById(Serializable id);

    /**
     * 查询（根据ID 批量查询）
     *
     * @param idList 主键ID列表(不能为 null 以及 empty)
     */
    List<T> selectBatchIds(@Param(Constants.COLLECTION) Collection<? extends Serializable> idList);

    /**
     * 查询（根据 columnMap 条件）
     *
     * @param columnMap 表字段 map 对象
     */
    List<T> selectByMap(@Param(Constants.COLUMN_MAP) Map<String, Object> columnMap);

    /**
     * 根据 entity 条件，查询一条记录
     *
     * @param queryWrapper 实体对象封装操作类（可以为 null）
     */
    T selectOne(@Param(Constants.WRAPPER) Wrapper<T> queryWrapper);

    /**
     * 根据 Wrapper 条件，查询总记录数
     *
     * @param queryWrapper 实体对象封装操作类（可以为 null）
     */
    Integer selectCount(@Param(Constants.WRAPPER) Wrapper<T> queryWrapper);

    /**
     * 根据 entity 条件，查询全部记录
     *
     * @param queryWrapper 实体对象封装操作类（可以为 null）
     */
    List<T> selectList(@Param(Constants.WRAPPER) Wrapper<T> queryWrapper);

    /**
     * 根据 Wrapper 条件，查询全部记录
     *
     * @param queryWrapper 实体对象封装操作类（可以为 null）
     */
    List<Map<String, Object>> selectMaps(@Param(Constants.WRAPPER) Wrapper<T> queryWrapper);

    /**
     * 根据 Wrapper 条件，查询全部记录
     * <p>注意： 只返回第一个字段的值</p>
     *
     * @param queryWrapper 实体对象封装操作类（可以为 null）
     */
    List<Object> selectObjs(@Param(Constants.WRAPPER) Wrapper<T> queryWrapper);

    /**
     * 根据 entity 条件，查询全部记录（并翻页）
     *
     * @param page         分页查询条件（可以为 RowBounds.DEFAULT）
     * @param queryWrapper 实体对象封装操作类（可以为 null）
     */
    IPage<T> selectPage(IPage<T> page, @Param(Constants.WRAPPER) Wrapper<T> queryWrapper);

    /**
     * 根据 Wrapper 条件，查询全部记录（并翻页）
     *
     * @param page         分页查询条件
     * @param queryWrapper 实体对象封装操作类
     */
    IPage<Map<String, Object>> selectMapsPage(IPage<T> page, @Param(Constants.WRAPPER) Wrapper<T> queryWrapper);
```

提供了以上接口，甚至还有分页查询的功能。这些都可以直接调用而不用去写sql文件，也不用加sql注解，非常方便！

##### 条件构造器查询

条件构造器并不是仅仅只供查询使用，因为它有一个条件构造器的抽象类：`AbstractWrapper`,里面定义了很多条件,例如:大于、小于、大于等于、小于等于、and、or、between和like等等一系列条件。

条件构造器查询示例如下：

```java
//where name like '王%' or age >= 25 order by age desc, id asc
QueryWrapper<User> queryWrapper = new QueryWrapper<>();
queryWrapper.like("name", "王").or().ge("age", 25).orderByDesc("age").orderByAsc("id"); //查询名字里包含"王"或者年龄大于等于25岁的人并按照年龄降序，id升序排列。
List<AuthLoginUser> result = authLoginUserDao.selectList(queryWrapper);//调用BaseMapper给出的接口
```

###### `apply()`方法：

```java
//where age = 25 and manager_id in (select id from user where name like '王%') 
QueryWrapper<User> queryWrapper = new QueryWrapper<>();
queryWrapper.apply("age = {0}", 25)//使用“{0}”这样的方式可以避免sql注入，“{0}”相当于一个占位符 
  .inSql("manager_id", "select id from user where name like 王%");//查找年龄等于25的并且并且manager_id为王姓的人
List<User> userList = userMapper.selectList(queryWrapper);
```

###### `and()`方法

```java
//where name like '王%' and (age < 40 or email is not null)
QueryWrapper<User> queryWrapper = new QueryWrapper<>();
queryWrapper.likeRight("name", "王").and(qw -> qw.lt("age", 40).or().isNotNull("email"));
List<User> userList = userMapper.selectList(queryWrapper);
```

###### `nested()`方法

```java
//where (age < 40 or email is not Null) and name like '王%'
QueryWrapper<User> queryWrapper = new QueryWrapper<>();
queryWrapper.nested(qw -> qw.lt("age", 40).or().isNotNull("email")).likeRight("name", "王");
List<User> userList = userMapper.selectList(queryWrapper);
```

###### `in()`方法

```java
//where age in (30, 31, 35)
QueryWrapper<User> queryWrapper = new QueryWrapper<>();
queryWrapper.in("age", Arrays.asList(30, 31, 35));
List<User> userList = userMapper.selectList(queryWrapper);
```

###### `last()`方法

```java
//where age > 40 limit 1		注意last()这个方法是会无视优化规则直接将其内容拼接到SQL语句最后的，它是有SQL注入的风险的，要谨慎使用。如果有多个last()的调用则始终以最后一个为准 
QueryWrapper<User> queryWrapper = new QueryWrapper<>();
queryWrapper.gt("age", 30).last("limit 1");
List<User> userList = userMapper.selectList(queryWrapper);
```

###### `select()`方法

```java
//select id, name where age > 40;			select()可以帮助我们指定想查询的列 
QueryWrapper<User> queryWrapper = new QueryWrapper<>();
queryWrapper.select("id", "name").gt("age", 40);
List<User> userList = userMapper.selectList(queryWrapper);
```

另一种情况：当我们仅仅只想要去掉某个字段不去查询的话，利用上面的方法把除了那个属性的所有属性都写出来显然是不方便的，所以可以这样：

```java
QueryWrapper<User> queryWrapper = new QueryWrapper<>();
queryWrapper.select(User.class, info -> !info.getColumn().equals("create_time") && !info.getColumn().equals("manager_id"));//仅仅不查create_time和manager_id这两个列，其他的都会查出来
List<User> userList = userMapper.selectList(queryWrapper);
```

###### 关于condition

有了condition可以控制这个条件加不加入到SQL语句中

```java
//可以查看通用BaseMapper接口查找可以输入条件的方法来调用，例如下面的例子：
QueryWrapper<User> queryWrapper = new QueryWrapper<>();
queryWrapper.like(StringUtils.isNotEmpty(email), "email", email);//这里的email是前端传过来的参数
List<User> userList = userMapper.selectList(queryWrapper);
```

###### 使用实体类来查询

```java
User user = new User();
user.setName("zhangSan");
user.setAge(25);
QueryWrapper<User> queryWrapper = new QueryWrapper<>(user);//传入user对象作为查询条件
queryWrapper.like("name", "王");//查询构造器中设置的条件依然有效
List<User> userList = userMapper.selectList(queryWrapper); 
```

###### `allEq()`方法

 ```java
QueryWrapper<User> queryWrapper = new QueryWrapper<>();
Map<String, Object> params = new HashMap<>();
params.put("name", "zhangSan");
params.put("age", 25);
queryWrapper.allEq(params, false);//第二个参数为如果传入的params的某一个键值对(条件)的值为null，那么就不将其添加到SQL语句中。如果不加这第二个参数，那么当值为null时SQL语句会变为“XXX IS NULL”这样的条件 
//开启下面这句，那么会将params中键为name的条件从SQL语句中去掉。
//另外多说一句，它也是支持condition的
//queryWrapper.allEq((k, v) -> !k.equals("name"), params, false);
List<User> userList = userMapper.selectList(queryWrapper);
 ```

##### 其他条件构造器查询

###### `selectMaps()`方法

它返回的是`List<Map<String, Object>>`，也就是说有时候只会返回几个字段，而这种情况是没有必要再封装成实体类返回的，那么就直接返回想要的那几个字段就可以了。所以返回的List里面就是装着想要返回的字段。

###### `selectObjs()`方法

它返回的是`List<Object>`，它只会返回我们所查询的第一个字段，即使我们设定的SQL语句中有很多字段，它所返回的List中也只包含第一个字段，是只有第一个字段的集合。

###### `selectCount()`方法

它返回的是`Integer`，即符合查询条件的记录数。 使用这个方法就不允许我们去创建想要查询的列了，它内部会帮助我们将查询条件设置为`count(1)`

###### `selectOne()`方法

它返回的是`T`，即一个实体类型，它会找出符合条件的一条数据，这条数据的类型被封装到`T`里面。 如果根据条件查询出的结果多于一条，那么它会报错，没有符合条件的数据返回，它不会报错。

##### lambda条件构造器

允许我们使用lambda表达式来进行条件的设定，例如：

```java
LambdaQueryWrapper<User> lambdaWrapper = new LambdaQueryWrapper<>();
lambdaWrapper.like(User::getName, "王").lt(User::getAge, 40);
List<User> userList = userMapper.selectList(lambdaWrapper);
```

在`3.0.7`版本以后又出现了一种新的写法：

```java
List<User> userList = new LambdaQueryChainWrapper<>(userMapper)//需要一个实现了BaseMapper的接口
  .like(User::getName, "王").ge(User::getAge, 20)//利用Lambda表达式设置条件
  .list();//此方法直接返回一个List
```

### 关于更新

##### 使用实体类更新

可以根据实体类来更新，以id为条件查找，如果其中某个属性不为null，则将其加入SQL语句中。可以看到这样是把条件和要修改的字段都设定到一个实体类中了。

```java
User user = new User();
user.setId(123L);
user.setName("zhangSan");
user.setAge(26);
int rows = userMapper.updateById(user);//返回影像记录数
```

##### 使用更新构造器更新

###### 普通写法

```java
UpdateWrapper<User> updateWrapper = new UpdateWrapper<>();
updateWrapper.eq("name", "zhangSan").eq("age", 28);//条件
User user = new User();//设置要改成什么样
user.setEmail("abc@abc.com");
user.setAge(29);
int rows = userMapper.update(user, updateWrapper);//返回影响记录数
```

或者这么写：

```java
User whereUser = new User();//用实体类构造条件
whereUser.setName("zhangSan");
UpdateWrapper<User> updateWrapper = new UpdateWrapper<>(whereUser);//将条件实体类放入更新构造器
updateWrapper.eq("name", "zhangSan").eq("age", 29);//这里依然可以构造条件，且如果和实体类的条件相同，那么它们也都会被添加到SQL语句中，也就是说会添加两个where name = ’zhangSan‘
User user = new User();
user.setAge(30);
int rows = userMapper.update(user, updateWrapper);
```

###### `set()`方法

如果只更新少量字段，那么就可以使用set()方法

```java
UpdateWrapper<User> updateWrapper = new UpdateWrapper<>();
updateWrapper.eq("name", "zhangSan").eq("age", 28).set("email", "abc@abc.com").set("age", 29);//条件和修改
int rows = userMapper.update(user, updateWrapper);//返回影响记录数
```

##### Lambda更新构造器 

那么就可以使用Lambda来做修改：

###### 常规写法

```java
LambdaUpdateWrapper<User> lambdaWrapper = new LambdaUpdateWrapper<>();
lambdaWrapper.eq(User::getName, "zhangSan").eq(User::getAge, 28).set(User::getEmail, "abc@abc.com");
int rows = userMapper.update(null, lambdaWrapper);
```

###### 另一种写法

```java
boolean update = new LambdaUpdateChainWrapper<User>(userMapper)
  .eq(User::getName, "zhangSan").eq(User::getAge, 28).set(User::getEmail, "abc@abc.com")//条件和要改的字段内容
  .update(); //执行更新，返回更新是否成功
```

### 关于删除

##### 根据id删除

 非常简单只要调用一个方法即可

```java
int rows = userMapper.deleteById(1L);//执行根据id删除，返回结果是影响记录数  
```

##### 其他删除方法

###### 根据多个条件删除

```java
Map<String, Object> columnMap = new HashMap<>();
columnMap.put("name", "zhangSan");
columnMap.put("age", 31);
int rows = userMapper.deleteByMap(columnMap);//利用Map中的条件找到对应的数据进行删除，返回影响记录数
```

###### 根据id批量删除

```java
int rows = userMapper.deleteBatchIds(Arrays.asList(1L, 2L, 3L));//根据id批量删除，内部使用了where id in ()这样的语句
```

 ##### 使用条件构造器删除

 ```java
LambdaQueryWrapper<User> lambdaWrapper = new LambdaQueryWrapper<>();
lambdaWrapper.eq(User::getName, "zhangSan").or().gt(User::getAge, 29);
int rows = userMapper.delete(lambdaWrapper);//使用条件构造器执行删除，并返回影响记录数
 ```

### 自定义SQL

可以再Mapper中自定义SQL语句：

```java
@Select("select * from user ${ew.customSqlSegment}")//这里的ew就是Constants.WRAPPER，并且在注解SQL语句中不用加where，因为如果有条件的话MP会自动帮我们添加好where语句
List<User> selectAll(@Param(Constants.WRAPPER) Wrapper<User> wrapper); 
```

### 分页插件

##### 两个分页方法

```java
QueryWrapper<User> queryWrapper = new QueryWrapper<>();
queryWrapper.ge("age", 26);
Page<User> page = new Page<>(1, 2);//当前页current，每页显示记录数size。还有其他构造器自己看源码吧
IPage<User> iPage = userMapper.selectPage(page, queryWrapper);//执行分页查询
System.out.println("总页数：", iPage.getPages());
System.out.println("总记录数：", iPage.getTotal());
List<User> userList = iPage.getRecords();//获得数据
```

另一种：

```java
QueryWrapper<User> queryWrapper = new QueryWrapper<>();
queryWrapper.ge("age", 26);
Page<User> page = new Page<>(1, 2);
IPage<Map<String, Object>> iPage = userMapper.selectMapsPage(page, queryWrapper);
System.out.println("总页数：", iPage.getPages());
System.out.println("总记录数：", iPage.getTotal());
List<Map<String, Object>> userList = iPage.getRecords();
```

可以看到上面的两个方法返回的类型不同，第一种是返回**实体类**，另外一种是返回**包含了所有字段的Map**。而且这两种方法都会利用**两条SQL**语句来进行查询。`第一条是根据条件查询总记录数，第二条才是分页的去查数据 `。这点很重要要记住。

但是我们有时的需求是不需要查询总记录数。这时可以使用其另一个构造器：

```java
public Page(long current, long size, boolean isSearchCount) {
    this(current, size, 0, isSearchCount);
}
```

这第三个参数表示是否需要查询总记录数 。

### AR模式

AR（Active Record）,其中有一个model的概念,一个Model对应一张表，model的每个实例对应表中的一条记录。简言之就是说：支持你使用实体类来进行CRUD的操作。

首先需要将我们的实体类继承`Model<User>`，并且这个Model是带泛型的。 然后与实体类对应的Mapper必须要继承MP的`BaseMapper`。

##### 插入示例

```java
User user = new User();
user.setName("zhangSan");
user.setAge(29);
boolean insert = user.insert();//这里直接使用了实体类调用插入方法(这个insert方法是实体类中没有的，是由于继承了Model类才有的) 
```

##### 查询示例

```java
User user = new User();
User selectUser = user.selectById(1L);
System.out.println(user == selectUser);//false，并不是一个对象，说明selectById返回的是一个新对象
System.out.println(selectUser);//查询出来的结果
```

或者这样写：

```java
User user = new User();
user.setId(1L);
User selectUser = user.selectById();
System.out.println(user == selectUser);//false，依然不是同一个对象
System.out.println(selectUser);//查询出来的结果
```

##### 更新示例

```java
User user = new User();
user.setId(1L);
user.setName("liSi");
boolean update = user.updateById();//通过传入的id修改名字  
```

##### 删除示例

```java
User user = new User();
user.setId(1L);
boolean delete = user.deleteById();//根据id删除某一条数据
```

##### `insertOrUpdate()`方法

```java
User user = new User();
user.setName("wangWu");
user.setAge(29);
boolean insertOrUpdate = user.insertOrUpdate(); 
```

为什么要单独说这个方法，因为它是把插入和更新写到了一起，**如果我们没有指定id，那么它会执行插入语句，如果我们指定了id，那么它会先根据id做查询，如果有数据那么就执行更新，否则依然执行插入语句**

另外还要注意，在MP中**删除不存在的它也会返回成功**

### 主键策略

##### 简要介绍

利用枚举类`IdType`来实现

- AUTO：数据库主键自增
- NONE：未设置主键类型，默认值
- INPUT：用户输入ID
- ID_WORKER：全局唯一ID，利用雪花算法
- UUID：全局唯一ID，利用UUID字符串
- ID_WORKER_STR：全局唯一ID，ID_WORKER的字符串表示

**注意**：`后三种类型只有当插入对象ID为空才会自动填充`

##### 全局配置

可以在`application.yml`中配置：

```yml
mybatis-plus:
	global-config: 
		db-config: 
			id-type: uuid
```

或者在配置类中设置：

```java
    @Bean
    public MybatisSqlSessionFactoryBean sqlSessionFactory(@Qualifier("dataSource")DataSource dataSource) throws Exception{
        final MybatisSqlSessionFactoryBean mybatisPlus = new MybatisSqlSessionFactoryBean();
        mybatisPlus.setDataSource(dataSource);
        ResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();
        mybatisPlus.setTypeAliasesPackage(ALIASES_PACKAGE);
        mybatisPlus.setConfigLocation(resolver.getResource(CONFIG_LOCATION));
        mybatisPlus.setVfs(SpringBootVFS.class);
        mybatisPlus.setPlugins(new Interceptor[]{new CzbMybatisInterceptor()});
        GlobalConfig globalConfig = new GlobalConfig();
        GlobalConfig.DbConfig dbConfig = new GlobalConfig.DbConfig();
        dbConfig.setIdType(IdType.AUTO);//id为自增，就是这样设置
        globalConfig.setDbConfig(dbConfig);
        mybatisPlus.setGlobalConfig(globalConfig);
        return mybatisPlus;
    }
```

##### 注意

**如果局部策略和全局策略都设置了，那么局部策略是`优于`全局策略的**

### 基本配置

```yml
mybatis-plus:
	config-location: classpath:mybatis-config.xml
	type-aliases-package: com.xxx.xxx.entity
	...
```

上面只显示了部分配置，其他配置可以参考MP官方文档：`[https://mp.baomidou.com/config/#%E5%9F%BA%E6%9C%AC%E9%85%8D%E7%BD%AE](基本配置)`

注意：**定义了`configuration`之后不要再定义`config-location`否则在执行时会报错**

##### fieldStrategy

这个是`global-config`下的配置，配置举例：

```yml
global-config: 
	field-strategy: ignored
```

前面也说了，当插入删除更新时，如果传入的某个字段为空，那么就不将其加入到SQL语句中。但是如果配置了该参数，并且值为`ignored`，那么不管我们设没设置属性所有字段都会被写入到SQL语句中然后没设置的它会插入null  ，它还可以配置为`default`、`not-null`和`not-empty`。当为not-empty时值为null或者为空串的都会忽略掉也就是说不会被添加到SQL语句中。

可以直接在`@TableField`注解中配置strategy属性来实现局部的设置

##### tablePrefix

这个同样是`global-config`下的配置，配置举例：

```yml
global-config: 
	table-prefix: mp_
```

它可以指定表名的前缀，这个在开发的时候使用也是非常方便的。

##### tableUnderline

同样是`global-config`下的配置，配置举例：

```yml
global-config: 
	table-underline: true
```

它表明数据库表名是否是下划线命名的，默认为true

### 通用Service

 前面说了通用的mapper,其实MP也支持通用的Service 

首先先要继承MP的`IService<UserMapper, User>`接口，它是带泛型的，第一个是指定继承了通用Mapper的mapper接口，第二个是指定了要操作的实体类

##### `getOne()`方法

```java
User one = userService.getOne(Wrappers.<User>lambdaQuery().gt(User::getAge, 25), false);
```

第二个参数不写的话默认是true，表示如果结果大于一条程序将会报错。如果设置为false，那么在返回多条数据时只返回第一条。一般情况下getOne只适合用于返回一条或者零条数据

##### `saveBatch()`方法

```java
User user1 = new User();
user1.setName("zhangSan");
user1.setAge(29);
User user2 = new User();
user2.setName("liSi");
user2.setAge(29);
List<User> userList = Arrays.asList(user1, user2);
boolean saveBatch = userService.saveBatch(userList);//执行批量插入
```

##### `saveOrUpdateBatch()`方法

```java
User user1 = new User();
user1.setName("zhangSan");
user1.setAge(29);
User user2 = new User();
user2.setId(2L);
user2.setName("wangWu");
user2.setAge(29);
List<User> userList = Arrays.asList(user1, user2);
boolean saveOrUpdateBatch = userService.saveOrUpdateBatch(userList);//执行批量插入或更新操作，视每条数据的情况而定 
```

##### 链式调用

```java
List<User> userList = userService.lambdaQuery().gt(User::getAge, 29).like(User::getName, "zhangSan").list();
```

```java
boolean result = userService.lambdaUpdate().eq(User::getAge, 28).set(User::getAge, 29).update();
```

```java
boolean result = userService.lambdaUpdate().eq(User::getAge, 28).remove(); //可以使用update执行删除操作
```

