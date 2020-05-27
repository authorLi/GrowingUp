# Apollo 源码解析之Portal(一)

### 创建App

![](http://static.iocoder.cn/images/Apollo/2018_03_05/01.png)

整体涉及到portal与admin service

#### App实体类

```java
@Entity
@Table(name = "App")
@SQLDelete(sql = "Update App set isDeleted = 1 where id = ?")
@Where(clause = "isDeleted = 0")
public class App extends BaseEntity {

  @NotBlank(message = "Name cannot be blank")
  @Column(name = "Name", nullable = false)
  private String name;

  @NotBlank(message = "AppId cannot be blank")
  @Pattern(
      regexp = InputValidator.CLUSTER_NAMESPACE_VALIDATOR,
      message = InputValidator.INVALID_CLUSTER_NAMESPACE_MESSAGE
  )
  @Column(name = "AppId", nullable = false)
  private String appId;

  @Column(name = "OrgId", nullable = false)
  private String orgId;

  @Column(name = "OrgName", nullable = false)
  private String orgName;

  @NotBlank(message = "OwnerName cannot be blank")
  @Column(name = "OwnerName", nullable = false)
  private String ownerName;

  @NotBlank(message = "OwnerEmail cannot be blank")
  @Column(name = "OwnerEmail", nullable = false)
  private String ownerEmail;

  //getter/setter……

  public String toString() {
    return toStringHelper().add("name", name).add("appId", appId)
        .add("orgId", orgId)
        .add("orgName", orgName)
        .add("ownerName", ownerName)
        .add("ownerEmail", ownerEmail).toString();
  }

  public static class Builder {
		//builder……
  }

  public static Builder builder() {
    return new Builder();
  }
}
```

- 可以看到此实体类是由Hibernate框架支持的，直接对应了数据库`ApolloPortalDB`的表`App`
- 其内部还定义了Builder，方便构造App对象
- `@SQLDelete(sql = "Update App set isDeleted = 1 where id = ?")`表示删除是逻辑删除，仅仅将`isDeleted`标识为1
- `@Where(clause = "isDeleted = 0")`表示查找时仅查找没有被逻辑删除的数据

#### BaseEntity基本实体类

```java
@MappedSuperclass
@Inheritance(strategy = InheritanceType.TABLE_PER_CLASS)
public abstract class BaseEntity {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  @Column(name = "Id")
  private long id;

  @Column(name = "IsDeleted", columnDefinition = "Bit default '0'")
  protected boolean isDeleted = false;

  @Column(name = "DataChange_CreatedBy", nullable = false)
  private String dataChangeCreatedBy;

  @Column(name = "DataChange_CreatedTime", nullable = false)
  private Date dataChangeCreatedTime;

  @Column(name = "DataChange_LastModifiedBy")
  private String dataChangeLastModifiedBy;

  @Column(name = "DataChange_LastTime")
  private Date dataChangeLastModifiedTime;

  //getter/setter

  @PrePersist
  protected void prePersist() {
    if (this.dataChangeCreatedTime == null) {
        dataChangeCreatedTime = new Date();
    }
    if (this.dataChangeLastModifiedTime == null) {
        dataChangeLastModifiedTime = new Date();
    }
  }

  @PreUpdate
  protected void preUpdate() {
    this.dataChangeLastModifiedTime = new Date();
  }

  @PreRemove
  protected void preRemove() {
    this.dataChangeLastModifiedTime = new Date();
  }

  protected ToStringHelper toStringHelper() {
    return MoreObjects.toStringHelper(this).omitNullValues().add("id", id)
        .add("dataChangeCreatedBy", dataChangeCreatedBy)
        .add("dataChangeCreatedTime", dataChangeCreatedTime)
        .add("dataChangeLastModifiedBy", dataChangeLastModifiedBy)
        .add("dataChangeLastModifiedTime", dataChangeLastModifiedTime);
  }

  public String toString(){
    return toStringHelper().toString();
  }
}
```

- 定义了一些实体类基本的属性：是否被删除、数据创建人、数据创建时间、数据最新更新操作人和数据最新更新操作时间
- `@PrePersist`、`@PreUpdate`和`@PreRemove`这三个注解是Hibernate的注解，分别在添加修改和删除前执行一些操作，这里是设置时间
- 在apollo中所有的实体类都会继承此基础实体类，这种将一些公用字段统一定义的设计是很有意义的，而且这些字段对排查问题很方便

#### Portal侧

##### AppController

```java
@RestController
@RequestMapping("/apps")
public class AppController {
  @PreAuthorize(value = "@permissionValidator.hasCreateApplicationPermission()")
  @PostMapping
  public App create(@Valid @RequestBody AppModel appModel) {

    App app = transformToApp(appModel);

    App createdApp = appService.createAppInLocal(app);

    publisher.publishEvent(new AppCreationEvent(createdApp));

    Set<String> admins = appModel.getAdmins();
    if (!CollectionUtils.isEmpty(admins)) {
      rolePermissionService
          .assignRoleToUsers(RoleUtils.buildAppMasterRoleName(createdApp.getAppId()),
              admins, userInfoHolder.getUser().getUserId());
    }

    return createdApp;
  }
}
```

- 第一步处理传入的AppModel，将数据“复制到”一个App对象中
- 第二步调用appService的createAppInLocal方法，将app添加到protal库的app表
- 第三步发布创建App事件，将App异步的同步至admin service
- 第四步授予App管理员的角色。具体以后再看。

##### AppModel

```java
public class AppModel {

  @NotBlank(message = "name cannot be blank")
  private String name;

  @NotBlank(message = "appId cannot be blank")
  @Pattern(
      regexp = InputValidator.CLUSTER_NAMESPACE_VALIDATOR,
      message = "Invalid AppId format: " + InputValidator.INVALID_CLUSTER_NAMESPACE_MESSAGE
  )
  private String appId;

  @NotBlank(message = "orgId cannot be blank")
  private String orgId;

  @NotBlank(message = "orgName cannot be blank")
  private String orgName;

  @NotBlank(message = "ownerName cannot be blank")
  private String ownerName;

  private Set<String> admins;
  
  //getter/setter
}
```

- 可以看到这个类和App实体类相差无几，只不过此类用于接收从前端传来的数据，而App是在后端处理数据使用的

##### createAppInLocal

```java
@Transactional
public App createAppInLocal(App app) {
  String appId = app.getAppId();
  App managedApp = appRepository.findByAppId(appId);

  if (managedApp != null) {
    throw new BadRequestException(String.format("App already exists. AppId = %s", appId));
  }

  UserInfo owner = userService.findByUserId(app.getOwnerName());
  if (owner == null) {
    throw new BadRequestException("Application's owner not exist.");
  }
  app.setOwnerEmail(owner.getEmail());

  String operator = userInfoHolder.getUser().getUserId();
  app.setDataChangeCreatedBy(operator);
  app.setDataChangeLastModifiedBy(operator);

  App createdApp = appRepository.save(app);

  appNamespaceService.createDefaultAppNamespace(appId);
  roleInitializationService.initAppRoles(createdApp);

  Tracer.logEvent(TracerEventType.CREATE_APP, appId);

  return createdApp;
}
```

- 第一步根据appId查找portal库的App表中是否有数据，如果有直接抛出存在app的异常而导致创建失败
- 第二步根据拥有者名称查询是否有此人，如果没有此人直接抛出异常
- 给app设置拥有者邮箱、当前操作人和当前修改人(创建时间和修改时间都会自动添加，之前讲过根据那个注解可以……)，并添加到数据库
- 第四步调用appNamespaceService的方法为此app添加默认的appNamespace
- 第五步初始化App角色，这个放在后面看
- 第六步打印tracer日志

##### createDefaultAppNamespace

```java
@Transactional
public void createDefaultAppNamespace(String appId) {
  if (!isAppNamespaceNameUnique(appId, ConfigConsts.NAMESPACE_APPLICATION)) {
    throw new BadRequestException(String.format("App already has application namespace. AppId = %s", appId));
  }

  AppNamespace appNs = new AppNamespace();
  appNs.setAppId(appId);
  appNs.setName(ConfigConsts.NAMESPACE_APPLICATION);
  appNs.setComment("default app namespace");
  appNs.setFormat(ConfigFileFormat.Properties.getValue());
  String userId = userInfoHolder.getUser().getUserId();
  appNs.setDataChangeCreatedBy(userId);
  appNs.setDataChangeLastModifiedBy(userId);

  appNamespaceRepository.save(appNs);
}

public boolean isAppNamespaceNameUnique(String appId, String namespaceName) {
  Objects.requireNonNull(appId, "AppId must not be null");
  Objects.requireNonNull(namespaceName, "Namespace must not be null");
  return Objects.isNull(appNamespaceRepository.findByAppIdAndName(appId, namespaceName));//验证此appNamespace是否存在
}
```

- 第一步验证appNamespace是否存在，如果存在将抛出异常
- 第二步创建默认的appNamespace，name为`application`
- 第三步将默认的appNamespace保存至portal库的表

##### AppCreationEvent

```java
public class AppCreationEvent extends ApplicationEvent {//继承了ApplicationEvent

  public AppCreationEvent(Object source) {
    super(source);
  }

  public App getApp() {
    Preconditions.checkState(source != null);
    return (App) this.source;
  }
}
```

##### onAppCreationEvent

```java
@EventListener
public void onAppCreationEvent(AppCreationEvent event) {
  AppDTO appDTO = BeanUtils.transform(AppDTO.class, event.getApp());
  List<Env> envs = portalSettings.getActiveEnvs();
  for (Env env : envs) {
    try {
      appAPI.createApp(env, appDTO);
    } catch (Throwable e) {
      logger.error("Create app failed. appId = {}, env = {})", appDTO.getAppId(), env, e);
      Tracer.logError(String.format("Create app failed. appId = %s, env = %s", appDTO.getAppId(), env), e);
    }
  }
}

public AppDTO createApp(Env env, AppDTO app) {
  return restTemplate.post(env, "apps", app, AppDTO.class);
}
```

- 此方法用来监听创建app的事件
- 第一步取出app的内容放到appDTO中
- 第二步获取所有“有效的”环境
- 第三步遍历每个环境发送http请求至admin service，这里体现出异步的将App同步至admin service

#### Admin Service侧

##### AppController

```java
@PostMapping("/apps")
public AppDTO create(@Valid @RequestBody AppDTO dto) {
  App entity = BeanUtils.transform(App.class, dto);
  App managedEntity = appService.findOne(entity.getAppId());
  if (managedEntity != null) {
    throw new BadRequestException("app already exist.");
  }

  entity = adminService.createNewApp(entity);

  return BeanUtils.transform(AppDTO.class, entity);
}
```

- 第一步将传入的AppDTO转换成app对象
- 第二步查找是否存在此app的信息，如果存在则抛出异常
- 第三步调用adminService的方法创建新的App
- 第四步将添加的App转换成AppDTO传递回去

##### createNewApp

```java
@Transactional
public App createNewApp(App app) {
  String createBy = app.getDataChangeCreatedBy();
  App createdApp = appService.save(app);

  String appId = createdApp.getAppId();

  appNamespaceService.createDefaultAppNamespace(appId, createBy);

  clusterService.createDefaultCluster(appId, createBy);

  namespaceService.instanceOfAppNamespaces(appId, ConfigConsts.CLUSTER_NAME_DEFAULT, createBy);

  return app;
}
```

- 第一步存储app到`ApolloConfigDB`的库的`App`表，此操作内部也会添加一条日志审计信息到`Audit`表
- 第二步为app创建默认的AppNamespace，name为`application`，内部也会添加一条日志审计信息
- 第三步为app创建默认的Cluster，name为`default`，内部也会添加一条日志审计信息
- 第四步根据appId获取所有的AppNamespace，遍历之，为Cluster的每个节点创建默认的命名空间，内部会添加一条日志审计信息

### 总结

当我们从apollo管理页面创建新的App时，请求会首先打到portal项目的`AppController`，然后在将数据保存到portal库的同时也会发送http请求来实现异步的同步新创建的App给admin service，此时admin service也会将新创建的App信息存到ApolloConfigDB库的App表里

