# MyBatisPlus采坑记录

### 问题出现

今天运行的好好地项目，前端突然告诉我她们获取不到数据，但是程序是正常运行的没有出错，并且状态码返回的是200只不过返回的json中，数据为空，很迷。

### 开始解决

先根据日志定位到了问题出现在哪行代码，然后定位到了一条根据多个条件查询的SQL语句。大概代码如下：

```java
try {
    QueryWrapper<User> queryWrapper = new QueryWrapper<>();
    queryWrapper.eq("status", userQuery.getStatus())
    				.like("user_name", userQuery.getUserName());
    				.like("mobile", userQuery.getMobile());
            .last("order by CONVERT(user_name USING gbk) limit " + userQuery.getStartIndex() + "," + userQuery.getPageSize());
    results = userDao.selectList(queryWrapper);
} catch (Exception e) {
    LOGGER.error("queryListWithPageNew has some problems", e);
}
return results;
```

具体每行代码的含义就不多说了，可以看到构造的wrapper里面有两个`like()`方法的调用，这是执行模糊查询使用到的。

但是为什么没有数据呢，看了一下请求过来的参数才知道，前端并没有传这两个模糊查询所需要的值。

难道问题出在这？马上debug一下，最后看到控制台打印出的SQL语句一切真相大白！

```txt
Creating a new SqlSession
SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@1f901e6b] was not registered for synchronization because synchronization is not active
JDBC Connection [com.mysql.jdbc.JDBC4Connection@53c014e0] will not be managed by Spring
==>  Preparing: SELECT id， user_name， mobile FROM auth_login_user WHERE status = ? AND user_name LIKE ? AND mobile LIKE ? order by CONVERT(user_name USING gbk) limit 0,10 
==> Parameters: 1(Integer), %null%(String), %null%(String)
<==      Total: 0
Closing non transactional SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@1f901e6b]
```

没错，当我们没有传值时，我这里就是默认为空，它的like()方法并没有和其他方法一样当值为空就不使用此字段作为条件查询，而是把”null“本身当做了字符串添加到了条件中，就变成了`%null%`，那么知道了问题出在哪就好解决了，只需要加入两个判断，当字段不为空才将其添加进查询条件中即可。所以改为：

```java
try {
    QueryWrapper<AuthLoginUser> queryWrapper = new QueryWrapper<>();
    queryWrapper.eq("status", userQuery.getStatus())
            .last("order by CONVERT(user_name USING gbk) limit " + userQuery.getStartIndex() + "," + userQuery.getPageSize());
    if (null != userQuery.getUserName()) {//添加判断
        queryWrapper.like("user_name", userQuery.getUserName());
    }
    if (null != userQuery.getMobile()) {//添加判断
        queryWrapper.like("mobile", userQuery.getMobile());
    }
    results = userQuery.selectList(queryWrapper);
} catch (Exception e) {
    LOGGER.error("queryListWithPageNew has some problems", e);
}
return results;
```

### 更进一步

到mybatis-plus的源码中一探究竟

#### like()方法

```java
default Children like(R column, Object val) {
    return like(true, column, val);
}

Children like(boolean condition, R column, Object val);
```

#### like()方法的实现

```java
@Override
public Children like(boolean condition, R column, Object val) {
    getWrapper().like(condition, column, val);
    return typedThis;
}

@Override
public Children like(boolean condition, R column, Object val) {
    return doIt(condition, () -> columnToString(column), LIKE, () -> formatSql("{0}", StringPool.PERCENT + val + StringPool.PERCENT));//问题出在这
}
```

问题的根本在于本来传入的是NULL，但是经过`StringPool.PERCENT + val + StringPool.PERCENT`这一句处理之后，本来的null，就变为了`%null%`这个字符串，然后被添加到SQL语句中执行。而数据库表中当然没有一项数据为%null%的数据了！所以不论之前的条件再怎么对，最后也不会有任何匹配结果！

#### columnToString()

解析字段为字符串

```java
protected String columnToString(R column) {
    if (column instanceof String) {//这里会将条件字段变成字符串,就是代码中的"mobile"、"user_name"等
        return (String) column;
    }
    throw ExceptionUtils.mpe("not support this column !");
}
```

#### formatSql()

格式化传入的字段值

```java
protected final String formatSql(String sqlStr, Object... params) {
    return formatSqlIfNeed(true, sqlStr, params);
}

protected final String formatSqlIfNeed(boolean need, String sqlStr, Object... params) {
    if (!need || StringUtils.isEmpty(sqlStr)) {
        return null;
    }
    if (ArrayUtils.isNotEmpty(params)) {
        for (int i = 0; i < params.length; ++i) {
            String genParamName = MP_GENERAL_PARAMNAME + paramNameSeq.incrementAndGet();
            sqlStr = sqlStr.replace(String.format(PLACE_HOLDER, i),
                String.format(MYBATIS_PLUS_TOKEN, getParamAlias(), genParamName));
            paramNameValuePairs.put(genParamName, params[i]);
        }
    }
    return sqlStr;
}
```



### 总结

多学、多看、多做。