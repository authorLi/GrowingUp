# Apollo 源码解析之Portal(二)

### 创建Cluster

![](http://static.iocoder.cn/images/Apollo/2018_03_07/01.png)

如上图，主要涉及到两个部分：Portal和AdminService

#### Cluster实体类

```java
@Entity
@Table(name = "Cluster")
@SQLDelete(sql = "Update Cluster set isDeleted = 1 where id = ?")
@Where(clause = "isDeleted = 0")
public class Cluster extends BaseEntity implements Comparable<Cluster> {

  @Column(name = "Name", nullable = false)
  private String name;

  @Column(name = "AppId", nullable = false)
  private String appId;

  @Column(name = "ParentClusterId", nullable = false)
  private long parentClusterId;

  public String getAppId() {
    return appId;
  }

  public String getName() {
    return name;
  }

  public void setAppId(String appId) {
    this.appId = appId;
  }

  public void setName(String name) {
    this.name = name;
  }

  public long getParentClusterId() {
    return parentClusterId;
  }

  public void setParentClusterId(long parentClusterId) {
    this.parentClusterId = parentClusterId;
  }

  public String toString() {
    return toStringHelper().add("name", name).add("appId", appId)
        .add("parentClusterId", parentClusterId).toString();
  }

  @Override
  public int compareTo(Cluster o) {
    if (o == null || getId() > o.getId()) {
      return 1;
    }

    if (getId() == o.getId()) {
      return 0;
    }

    return -1;
  }
}
```

- 使用了Hibernate框架，类似于之前介绍过的实体类
- 与Cluster表绑定，删除是逻辑删除，是将isDeleted改为1，查询也都默认查isDeleted为0的数据

#### Portal侧

##### ClusterController

```java
@PreAuthorize(value = "@permissionValidator.hasCreateClusterPermission(#appId)")
@PostMapping(value = "apps/{appId}/envs/{env}/clusters")
public ClusterDTO createCluster(@PathVariable String appId, @PathVariable String env,
                                @Valid @RequestBody ClusterDTO cluster) {
  String operator = userInfoHolder.getUser().getUserId();
  cluster.setDataChangeLastModifiedBy(operator);
  cluster.setDataChangeCreatedBy(operator);

  return clusterService.createCluster(Env.valueOf(env), cluster);
}
```

- 第一步拿到当前操作人，并将其设置到创建人和修改人这两个字段
- 调用createCluster方法创建集群

##### createCluster

```java
public ClusterDTO createCluster(Env env, ClusterDTO cluster) {
  if (!clusterAPI.isClusterUnique(cluster.getAppId(), env, cluster.getName())) {
    throw new BadRequestException(String.format("cluster %s already exists.", cluster.getName()));
  }
  ClusterDTO clusterDTO = clusterAPI.create(env, cluster);

  Tracer.logEvent(TracerEventType.CREATE_CLUSTER, cluster.getAppId(), "0", cluster.getName());

  return clusterDTO;
}
```

- 第一步调用isClusterUnique方法查询是否存在此Cluster，如果有将抛出异常
- 第二步调用clusterAPI里的方法发送HTTP请求到AdminService来创建cluster
- 第三步记录创建cluster的日志

##### isClusterUnique

```java
public boolean isClusterUnique(String appId, Env env, String clusterName) {//直接去请求AdminService
  return restTemplate
      .get(env, "apps/{appId}/cluster/{clusterName}/unique", Boolean.class,
          appId, clusterName);

}
```

##### create

```java
public ClusterDTO create(Env env, ClusterDTO cluster) {//发送HTTP请求到AdminService添加Cluster
  return restTemplate.post(env, "apps/{appId}/clusters", cluster, ClusterDTO.class,
      cluster.getAppId());
}
```

#### AdminService侧

##### ClusterController——isUniqueCluster

```java
@GetMapping("/apps/{appId}/cluster/{clusterName}/unique")
public boolean isAppIdUnique(@PathVariable("appId") String appId,
                             @PathVariable("clusterName") String clusterName) {
  return clusterService.isClusterNameUnique(appId, clusterName);//调用ClusterService的方法查询是否存在此Cluster
}
```

##### ClusterController——create

```java
@PostMapping("/apps/{appId}/clusters")
public ClusterDTO create(@PathVariable("appId") String appId,
                         @RequestParam(value = "autoCreatePrivateNamespace", defaultValue = "true") boolean autoCreatePrivateNamespace,
                         @Valid @RequestBody ClusterDTO dto) {
  Cluster entity = BeanUtils.transform(Cluster.class, dto);
  Cluster managedEntity = clusterService.findOne(appId, entity.getName());
  if (managedEntity != null) {
    throw new BadRequestException("cluster already exist.");
  }

  if (autoCreatePrivateNamespace) {
    entity = clusterService.saveWithInstanceOfAppNamespaces(entity);
  } else {
    entity = clusterService.saveWithoutInstanceOfAppNamespaces(entity);
  }

  return BeanUtils.transform(ClusterDTO.class, entity);
}
```

- 第一步使用Cluster对象代替ClusterDTO对象
- 第二步判断此AppId是否存在此Cluster，如果存在将抛出错误
- 第三步根据是否自动创建私有的namespace来判断是否应该创建私有namespace
- 第四步将Cluster转为ClusterDTO返回给请求方

##### isClusterNameUnique

```java
public boolean isClusterNameUnique(String appId, String clusterName) {
  Objects.requireNonNull(appId, "AppId must not be null");
  Objects.requireNonNull(clusterName, "ClusterName must not be null");
  return Objects.isNull(clusterRepository.findByAppIdAndName(appId, clusterName));//判断是否存在此Cluster
}
```

### 总结

对于Cluster集群的创建并不像App一样，它在Portal侧没有对应的数据库表来存储Cluster的信息，所以从上面的代码也可以看到，它在判断某个AppId是否存在某个Cluster的时候也发送了一条HTTP请求到AdminService然后到它的数据库表里面查找。