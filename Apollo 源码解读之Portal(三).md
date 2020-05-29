# Apollo 源码解读之Portal(三)

### 创建Namespace

![](http://static.iocoder.cn/images/Apollo/2018_03_10/01.png)

主要涉及到Portal和AdminService两个项目。在apollo中共涉及到两种namespace：`AppNamespace`和`Namespace`

##### AppNamespace

```java
@Entity
@Table(name = "AppNamespace")
@SQLDelete(sql = "Update AppNamespace set isDeleted = 1 where id = ?")
@Where(clause = "isDeleted = 0")
public class AppNamespace extends BaseEntity {

  @NotBlank(message = "AppNamespace Name cannot be blank")
  @Pattern(
      regexp = InputValidator.CLUSTER_NAMESPACE_VALIDATOR,
      message = "Invalid Namespace format: " + InputValidator.INVALID_CLUSTER_NAMESPACE_MESSAGE + " & " + InputValidator.INVALID_NAMESPACE_NAMESPACE_MESSAGE
  )
  @Column(name = "Name", nullable = false)
  private String name;

  @NotBlank(message = "AppId cannot be blank")
  @Column(name = "AppId", nullable = false)
  private String appId;

  @Column(name = "Format", nullable = false)
  private String format;

  @Column(name = "IsPublic", columnDefinition = "Bit default '0'")
  private boolean isPublic = false;

  @Column(name = "Comment")
  private String comment;

  //Setter、getter
  public String toString() {
    return toStringHelper().add("name", name).add("appId", appId).add("comment", comment)
        .add("format", format).add("isPublic", isPublic).toString();
  }
}
```

关于实体类中的`format`属性，共5种：

```java
public enum ConfigFileFormat {
  Properties("properties"), XML("xml"), JSON("json"), YML("yml"), YAML("yaml"), TXT("txt");
}
```

而`isPublic`属性只有两种：`public`和`private`，这个可以在apollo的管理页面的创建Namespace页面找到

![](http://static.iocoder.cn/images/Apollo/2018_03_10/02.png)

##### Namespace

```java
@Entity
@Table(name = "Namespace")
@SQLDelete(sql = "Update Namespace set isDeleted = 1 where id = ?")
@Where(clause = "isDeleted = 0")
public class Namespace extends BaseEntity {

  @Column(name = "appId", nullable = false)
  private String appId;

  @Column(name = "ClusterName", nullable = false)
  private String clusterName;

  @Column(name = "NamespaceName", nullable = false)
  private String namespaceName;

  //Constructor、getter、setter和toString
}
```

#### Portal侧

##### NamespaceController

```java
@PreAuthorize(value = "@permissionValidator.hasCreateAppNamespacePermission(#appId, #appNamespace)")
@PostMapping("/apps/{appId}/appnamespaces")
public AppNamespace createAppNamespace(@PathVariable String appId,
    @RequestParam(defaultValue = "true") boolean appendNamespacePrefix,
    @Valid @RequestBody AppNamespace appNamespace) {
  if (!InputValidator.isValidAppNamespace(appNamespace.getName())) {
    throw new BadRequestException(String.format("Invalid Namespace format: %s",
        InputValidator.INVALID_CLUSTER_NAMESPACE_MESSAGE + " & " + InputValidator.INVALID_NAMESPACE_NAMESPACE_MESSAGE));
  }

  AppNamespace createdAppNamespace = appNamespaceService.createAppNamespaceInLocal(appNamespace, appendNamespacePrefix);

  if (portalConfig.canAppAdminCreatePrivateNamespace() || createdAppNamespace.isPublic()) {
    namespaceService.assignNamespaceRoleToOperator(appId, appNamespace.getName(),
        userInfoHolder.getUser().getUserId());
  }

  publisher.publishEvent(new AppNamespaceCreationEvent(createdAppNamespace));

  return createdAppNamespace;
}
```

- 第一步调用`InputValidator.isValidAppNamespace(appNamespace.getName())`通过正则表达式校验appNamespace的合法性
- 第二步调用`createAppNamespaceInLocal()`方法，将appNamespace保存到Portal库的对应表中
- 第三步根据appNamespace的公有与私有授予角色
- 第四步发布创建appNamespace事件

##### createAppNamespaceInLocal

```java
@Transactional
public AppNamespace createAppNamespaceInLocal(AppNamespace appNamespace, boolean appendNamespacePrefix) {
  String appId = appNamespace.getAppId();
  App app = appService.load(appId);
  if (app == null) {
    throw new BadRequestException("App not exist. AppId = " + appId);
  }

  StringBuilder appNamespaceName = new StringBuilder();
  //add prefix postfix
  appNamespaceName
      .append(appNamespace.isPublic() && appendNamespacePrefix ? app.getOrgId() + "." : "")
      .append(appNamespace.getName())
      .append(appNamespace.formatAsEnum() == ConfigFileFormat.Properties ? "" : "." + appNamespace.getFormat());
  appNamespace.setName(appNamespaceName.toString());

  if (appNamespace.getComment() == null) {
    appNamespace.setComment("");
  }

  if (!ConfigFileFormat.isValidFormat(appNamespace.getFormat())) {
   throw new BadRequestException("Invalid namespace format. format must be properties、json、yaml、yml、xml");
  }

  String operator = appNamespace.getDataChangeCreatedBy();
  if (StringUtils.isEmpty(operator)) {
    operator = userInfoHolder.getUser().getUserId();
    appNamespace.setDataChangeCreatedBy(operator);
  }

  appNamespace.setDataChangeLastModifiedBy(operator);

  // globally uniqueness check for public app namespace
  if (appNamespace.isPublic()) {
    checkAppNamespaceGlobalUniqueness(appNamespace);
  } else {
    // check private app namespace
    if (appNamespaceRepository.findByAppIdAndName(appNamespace.getAppId(), appNamespace.getName()) != null) {
      throw new BadRequestException("Private AppNamespace " + appNamespace.getName() + " already exists!");
    }
    // should not have the same with public app namespace
    checkPublicAppNamespaceGlobalUniqueness(appNamespace);
  }

  AppNamespace createdAppNamespace = appNamespaceRepository.save(appNamespace);

  roleInitializationService.initNamespaceRoles(appNamespace.getAppId(), appNamespace.getName(), operator);
  roleInitializationService.initNamespaceEnvRoles(appNamespace.getAppId(), appNamespace.getName(), operator);

  return createdAppNamespace;
}
```

- 第一步拿出appId检查app是否存在(到Portal库中查)，如果不存在则抛出异常
- 第二步拼接appNamespace的name属性
- 第三步设置comment，如果为空则置为空串
- 第四步检验format属性是否合法，如果不合法则抛出异常
- 第五步设置创建者和修改者
- 第六步检查appNamespace的唯一性
- 第七步保存appNamespace到Portal库中
- 第八步初始化和appNamespace角色相关的信息

##### checkAppNamespaceGlobalUniqueness

```java
private void checkAppNamespaceGlobalUniqueness(AppNamespace appNamespace) {
  checkPublicAppNamespaceGlobalUniqueness(appNamespace);

  List<AppNamespace> privateAppNamespaces = findAllPrivateAppNamespaces(appNamespace.getName());

  if (!CollectionUtils.isEmpty(privateAppNamespaces)) {
    Set<String> appIds = Sets.newHashSet();
    for (AppNamespace ans : privateAppNamespaces) {
      appIds.add(ans.getAppId());
      if (appIds.size() == PRIVATE_APP_NAMESPACE_NOTIFICATION_COUNT) {
        break;
      }
    }

    throw new BadRequestException(
        "Public AppNamespace " + appNamespace.getName() + " already exists as private AppNamespace in appId: "
            + APP_NAMESPACE_JOINER.join(appIds) + ", etc. Please select another name!");
  }
}

private void checkPublicAppNamespaceGlobalUniqueness(AppNamespace appNamespace) {
  AppNamespace publicAppNamespace = findPublicAppNamespace(appNamespace.getName());
  if (publicAppNamespace != null) {
    throw new BadRequestException("AppNamespace " + appNamespace.getName() + " already exists as public namespace in appId: " + publicAppNamespace.getAppId() + "!");
  }
}

public AppNamespace findPublicAppNamespace(String namespaceName) {
  List<AppNamespace> appNamespaces = appNamespaceRepository.findByNameAndIsPublic(namespaceName, true);

  if (CollectionUtils.isEmpty(appNamespaces)) {
    return null;
  }

  return appNamespaces.get(0);
}
```

- 第一步根据appNamespace的name判断(Portal库中)是否存在**公共的(public)**的appNamespace，如果有则抛出异常
- 第二步根据appNamespace的name获取(Portal库中)所有**私有的(private)的**appNamespace的集合，如果存在则抛出异常

##### onAppNamespaceCreationEvent

```java
@EventListener
public void onAppNamespaceCreationEvent(AppNamespaceCreationEvent event) {
  AppNamespaceDTO appNamespace = BeanUtils.transform(AppNamespaceDTO.class, event.getAppNamespace());
  List<Env> envs = portalSettings.getActiveEnvs();
  for (Env env : envs) {
    try {
      namespaceAPI.createAppNamespace(env, appNamespace);
    } catch (Throwable e) {
      logger.error("Create appNamespace failed. appId = {}, env = {}", appNamespace.getAppId(), env, e);
      Tracer.logError(String.format("Create appNamespace failed. appId = %s, env = %s", appNamespace.getAppId(), env), e);
    }
  }
}
```

- 第一步拿到所有”活跃着的“环境
- 第二步通过HTTP请求调用Admin service来创建appNamespace

#### Admin Service侧

##### AppNamespaceController

```java
@PostMapping("/apps/{appId}/appnamespaces")
public AppNamespaceDTO create(@RequestBody AppNamespaceDTO appNamespace,
                              @RequestParam(defaultValue = "false") boolean silentCreation) {

  AppNamespace entity = BeanUtils.transform(AppNamespace.class, appNamespace);
  AppNamespace managedEntity = appNamespaceService.findOne(entity.getAppId(), entity.getName());

  if (managedEntity == null) {
    if (StringUtils.isEmpty(entity.getFormat())){
      entity.setFormat(ConfigFileFormat.Properties.getValue());
    }

    entity = appNamespaceService.createAppNamespace(entity);
  } else if (silentCreation) {
    appNamespaceService.createNamespaceForAppNamespaceInAllCluster(appNamespace.getAppId(), appNamespace.getName(),
        appNamespace.getDataChangeCreatedBy());

    entity = managedEntity;
  } else {
    throw new BadRequestException("app namespaces already exist.");
  }

  return BeanUtils.transform(AppNamespaceDTO.class, entity);
}
```

- 第一步根据appId和appNamespace查找`ApolloConfigDB`库中是否存在此appNamespace
- 第二步若为null给appNamespace添加format为”properties“
- 第三步保存appNamespace到`ApolloConfigDB`库的对应表中
- 第四步将appNamespace包装成appNamespaceDTO返回

##### createAppNamespace

```java
@Transactional
public AppNamespace createAppNamespace(AppNamespace appNamespace) {
  String createBy = appNamespace.getDataChangeCreatedBy();
  if (!isAppNamespaceNameUnique(appNamespace.getAppId(), appNamespace.getName())) {
    throw new ServiceException("appnamespace not unique");
  }
  appNamespace.setId(0);//protection
  appNamespace.setDataChangeCreatedBy(createBy);
  appNamespace.setDataChangeLastModifiedBy(createBy);

  appNamespace = appNamespaceRepository.save(appNamespace);

  createNamespaceForAppNamespaceInAllCluster(appNamespace.getAppId(), appNamespace.getName(), createBy);

  auditService.audit(AppNamespace.class.getSimpleName(), appNamespace.getId(), Audit.OP.INSERT, createBy);
  return appNamespace;
}
```

- 第一步查看是否存在此appNamespace
- 第二步设置好appNamespace的值，然后将其存储到`ApolloConfigDB`库的对应表中
- 第三步为cluster创建namespace
- 第四步记录日志审计信息

##### isAppNamespaceUnique

```java
public boolean isAppNamespaceNameUnique(String appId, String namespaceName) {//判断appNamespace是否存在
  Objects.requireNonNull(appId, "AppId must not be null");
  Objects.requireNonNull(namespaceName, "Namespace must not be null");
  return Objects.isNull(appNamespaceRepository.findByAppIdAndName(appId, namespaceName));
}
```

##### createNamespaceForAppNamespaceInAllCluster

```java
public void createNamespaceForAppNamespaceInAllCluster(String appId, String namespaceName, String createBy) {
  List<Cluster> clusters = clusterService.findParentClusters(appId);

  for (Cluster cluster : clusters) {

    // in case there is some dirty data, e.g. public namespace deleted in other app and now created in this app
    if (!namespaceService.isNamespaceUnique(appId, cluster.getName(), namespaceName)) {
      continue;
    }

    Namespace namespace = new Namespace();
    namespace.setClusterName(cluster.getName());
    namespace.setAppId(appId);
    namespace.setNamespaceName(namespaceName);
    namespace.setDataChangeCreatedBy(createBy);
    namespace.setDataChangeLastModifiedBy(createBy);

    namespaceService.save(namespace);
  }
}
```

- 第一步根据appId查找cluster列表
- 第二步遍历cluster，先查看是否存在此namespace，然后再存储namespace

### 总结

当apollo管理页面创建appNamespace时会先存到Portal库中一份，然后再异步的发送HTTP到admin service来存储namespace和appNamespace