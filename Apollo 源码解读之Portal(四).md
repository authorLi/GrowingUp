# Apollo 源码解读之Portal(四)

### 创建Item

#### Item

Item是配置项的意思，它是Namespace中最细粒度的单位。在Namespace中分为5种类型：`properties`、`xml`、`yml`、`ymal`和`json`。

其中如果是`properties`类型，那么配置文件中的每一项配置均对应着一个Item；如果是其他四种类型，那么整个配置文件对应着一个Item。

#### 流程

![](http://static.iocoder.cn/images/Apollo/2018_03_20/01.png)

整个流程涉及到Portal和Admin Service

#### Item

```java
@Entity//对应ApolloConfigDB的Item表，用来记录配置项
@Table(name = "Item")
@SQLDelete(sql = "Update Item set isDeleted = 1 where id = ?")
@Where(clause = "isDeleted = 0")
public class Item extends BaseEntity {

  @Column(name = "NamespaceId", nullable = false)
  private long namespaceId;

  @Column(name = "key", nullable = false)
  private String key;

  @Column(name = "value")
  @Lob
  private String value;

  @Column(name = "comment")
  private String comment;

  /** 行号；例如在Properties中如果含有多个配置项，那么每个配置项对应一行配置 */
  @Column(name = "LineNum")
  private Integer lineNum;

  //getter、setter、toString
}
```

#### Commit

```java
@Entity//对应ApolloConfigDB的Commit表，用来记录KV的变更历史
@Table(name = "Commit")
@SQLDelete(sql = "Update Commit set isDeleted = 1 where id = ?")
@Where(clause = "isDeleted = 0")
public class Commit extends BaseEntity {

  /** 修改的变更集，数据库中为longtext类型 */
  @Lob
  @Column(name = "ChangeSets", nullable = false)
  private String changeSets;

  @Column(name = "AppId", nullable = false)
  private String appId;

  @Column(name = "ClusterName", nullable = false)
  private String clusterName;

  @Column(name = "NamespaceName", nullable = false)
  private String namespaceName;

  @Column(name = "Comment")
  private String comment;

  //getter、setter、toString
}
```

#### 创建操作

![](http://static.iocoder.cn/images/Apollo/2018_03_20/02.png)

### Portal侧

#### ItemController

```java
@PreAuthorize(value = "@permissionValidator.hasModifyNamespacePermission(#appId, #namespaceName, #env)")
@PostMapping("/apps/{appId}/envs/{env}/clusters/{clusterName}/namespaces/{namespaceName}/item")
public ItemDTO createItem(@PathVariable String appId, @PathVariable String env,
                          @PathVariable String clusterName, @PathVariable String namespaceName,
                          @RequestBody ItemDTO item) {
  checkModel(isValidItem(item));

  //protect
  item.setLineNum(0);
  item.setId(0);
  String userId = userInfoHolder.getUser().getUserId();
  item.setDataChangeCreatedBy(userId);
  item.setDataChangeLastModifiedBy(userId);
  item.setDataChangeCreatedTime(null);
  item.setDataChangeLastModifiedTime(null);

  return configService.createItem(appId, Env.valueOf(env), clusterName, namespaceName, item);
}

private boolean isValidItem(ItemDTO item) {
  return Objects.nonNull(item) && !StringUtils.isContainEmpty(item.getKey());
}
```

- 第一步：检验item是否是空值(对象和内容不为空或空串)
- 第二步：完善item的信息
- 第三步：

#### checkModel()

```java
private static String ILLEGAL_MODEL = "request model is invalid";

public static void checkModel(boolean valid){
  checkArguments(valid, ILLEGAL_MODEL);
}

public static void checkArguments(boolean expression, Object errorMessage) {
  if (!expression) {
    throw new BadRequestException(String.valueOf(errorMessage));
  }
}
```

#### createItem()

```java
public ItemDTO createItem(String appId, Env env, String clusterName, String namespaceName, ItemDTO item) {
  NamespaceDTO namespace = namespaceAPI.loadNamespace(appId, env, clusterName, namespaceName);
  if (namespace == null) {
    throw new BadRequestException(
        "namespace:" + namespaceName + " not exist in env:" + env + ", cluster:" + clusterName);
  }
  item.setNamespaceId(namespace.getId());

  ItemDTO itemDTO = itemAPI.createItem(appId, env, clusterName, namespaceName, item);
  Tracer.logEvent(TracerEventType.MODIFY_NAMESPACE, String.format("%s+%s+%s+%s", appId, env, clusterName, namespaceName));
  return itemDTO;
}
```

- 第一步：发送HTTP请求到AdminService，查看是否存在对应的Namespace

  - ```java
    @GetMapping("/apps/{appId}/clusters/{clusterName}/namespaces/{namespaceName:.+}")
    public NamespaceDTO get(@PathVariable("appId") String appId,
                            @PathVariable("clusterName") String clusterName,
                            @PathVariable("namespaceName") String namespaceName) {
      Namespace namespace = namespaceService.findOne(appId, clusterName, namespaceName);
      if (namespace == null) {
          throw new NotFoundException(
                  String.format("namespace not found for %s %s %s", appId, clusterName, namespaceName));
      }
      return BeanUtils.transform(NamespaceDTO.class, namespace);
    }
    ```

  - ```java
    public Namespace findOne(String appId, String clusterName, String namespaceName) {
      return namespaceRepository.findByAppIdAndClusterNameAndNamespaceName(appId, clusterName,
                                                                           namespaceName);
    }
    ```

- 第二步：为item设置namespace的Id

- 第三步：发送HTTP请求到Admin Service创建item

  - ```java
    public ItemDTO createItem(String appId, Env env, String clusterName, String namespace, ItemDTO item) {
      return restTemplate.post(env, "apps/{appId}/clusters/{clusterName}/namespaces/{namespaceName}/items",
          item, ItemDTO.class, appId, clusterName, namespace);
    }
    ```

- 第四步：记录Tracer日志信息：`Namespace.Modify`

### Admin Service侧

#### ItemController

```java
@PreAcquireNamespaceLock
@PostMapping("/apps/{appId}/clusters/{clusterName}/namespaces/{namespaceName}/items")
public ItemDTO create(@PathVariable("appId") String appId,
                      @PathVariable("clusterName") String clusterName,
                      @PathVariable("namespaceName") String namespaceName, @RequestBody ItemDTO dto) {
  Item entity = BeanUtils.transform(Item.class, dto);

  ConfigChangeContentBuilder builder = new ConfigChangeContentBuilder();
  Item managedEntity = itemService.findOne(appId, clusterName, namespaceName, entity.getKey());
  if (managedEntity != null) {
    throw new BadRequestException("item already exists");
  }
  entity = itemService.save(entity);
  builder.createItem(entity);
  dto = BeanUtils.transform(ItemDTO.class, entity);

  Commit commit = new Commit();
  commit.setAppId(appId);
  commit.setClusterName(clusterName);
  commit.setNamespaceName(namespaceName);
  commit.setChangeSets(builder.build());
  commit.setDataChangeCreatedBy(dto.getDataChangeLastModifiedBy());
  commit.setDataChangeLastModifiedBy(dto.getDataChangeLastModifiedBy());
  commitService.save(commit);

  return dto;
}
```

- 第一步：将itemDTO转成item对象

- 第二步：创建一个`ConfigChangeContentBuilder`对象，用以加入到Commit对象的changeSets中

- 第三步：到Admin Service的库中寻找对应的Item，以此来检验是否存在对应的Item(如果存在就抛出异常)

  - ```java
    public Item findOne(String appId, String clusterName, String namespaceName, String key) {
      Namespace namespace = namespaceService.findOne(appId, clusterName, namespaceName);
      if (namespace == null) {
        throw new NotFoundException(
            String.format("namespace not found for %s %s %s", appId, clusterName, namespaceName));
      }
      Item item = itemRepository.findByNamespaceIdAndKey(namespace.getId(), key);
      return item;
    }
    ```

- 第四步：保存Item到Admin Service的库中

  - ```java
    @Transactional
    public Item save(Item entity) {
      checkItemKeyLength(entity.getKey());
      checkItemValueLength(entity.getNamespaceId(), entity.getValue());
    
      entity.setId(0);//protection
    
      if (entity.getLineNum() == 0) {
        Item lastItem = findLastOne(entity.getNamespaceId());
        int lineNum = lastItem == null ? 1 : lastItem.getLineNum() + 1;
        entity.setLineNum(lineNum);
      }
    
      Item item = itemRepository.save(entity);
    
      auditService.audit(Item.class.getSimpleName(), item.getId(), Audit.OP.INSERT,
                         item.getDataChangeCreatedBy());
    
      return item;
    }
    ```

  - 第一步：检验Key(128)与Value(20000)的长度是否符合标准

  - 第二步：设置行号

  - 第三步：保存Item到Admin Service库中

  - 第四步：添加日志审计信息

- 第五步：添加item到”添加列表“中

  - ```java
    public ConfigChangeContentBuilder createItem(Item item) {
      if (!StringUtils.isEmpty(item.getKey())){
        createItems.add(cloneItem(item));//这里并没有直接传入item，而是复制了一份，这是一种保护方式？
      }
      return this;
    }
    
    Item cloneItem(Item source) {
      Item target = new Item();
      BeanUtils.copyProperties(source, target);
      return target;
    }
    ```

- 第六步：将Item对象转为ItemDTO对象

- 第七步：构建Commit对象，并将其添加到Admin Service库中

### 总结

本文简单梳理了apollo中创建item的流程，即在apollo中添加配置项时其内部的操作。

整体看下来涉及到Portal和Admin Service两个项目。先是在管理页面添加了item，portal中主要是对item做一些校验然后利用HTTP发请求到Admin Service。在Admin Service中不仅仅将item添加到数据库，还添加了一条commit信息到数据库，里面包含了修改信息

