## Mybatis主要类模型

![mybatis-1](./images/mybatis-1.png)

## SqlSession 会话



## Executor 执行器

执行器主要负责SQL的执行逻辑。在Mybatis中一共有三种类型：SIMPLE、REUSE和BATCH。这三种执行器都继承了BaseExecutor，而BaseExector实现了Exector接口。在Excetor接口中包含各个执行器公共的方法，比如update query commit rollback close等，BaseExector实现了这些方法的同时，提供一些事务管理、一级缓存等功能。而SIMPLE、REUSE、BATCH三种执行器主要实现了BaseExector中的一些抽象方法，比如doQuery、doUpdate等。其中，SimpleExecutor是默认执行器，实现了BaseExecutor的抽象方法，没有额外的增加功能。ReUseExecutor在SimpleExecutor的基础之上，增加了一个HashMap结构，用来缓存解析之后的Statement。BatchEexcutor则增加了List集合用于存储多个预编译之后的Statement，BatchExecutor只有对修改操作才会减少预编译操作，并且批处理必须手动提交。

![executor](./images/executor.png)

三种执行器类型：SIMPLE、REUSE、BATCH

```java
// 三种执行器类型
public enum ExecutorType {
  SIMPLE, REUSE, BATCH
}
```

### SIMPLE

1. 核心逻辑

```java
 @Override
  public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
    Statement stmt = null;
    try {
      Configuration configuration = ms.getConfiguration();
      StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
      // 生成Statement, 并且将参数赋值到sql中
      stmt = prepareStatement(handler, ms.getStatementLog());
      return handler.query(stmt, resultHandler);
    } finally {
      closeStatement(stmt);
    }
  }
private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
    Statement stmt;
    // 获取连接
    Connection connection = getConnection(statementLog);
    // 预编译, 生成statement
    stmt = handler.prepare(connection, transaction.getTimeout());
    // 参数赋值
    handler.parameterize(stmt);
    return stmt;
  }
```

2. 测试代码

   ```java
     @Test
     public void testSimpleExecutor() throws SQLException {
       Executor executor = new SimpleExecutor(configuration, transaction);
       MappedStatement mappedStatement = configuration.getMappedStatement("com.finalch.mybatis.dao.UserMapper.getUserById");
       RowBounds rowBounds = new RowBounds();
       List<User> query = executor.query(mappedStatement, 1, rowBounds, null);
       List<User> query1 = executor.query(mappedStatement, 2, rowBounds, null);
       System.out.println(query.get(0).getName());
       System.out.println(query1.get(0).getName());
     }
   ```

   ![simpleExecutorTestResult-1](.\images\simpleExecutorTestResult-1.png)

   一共执行了两次预编译、参数设置

### REUSE

ReuseExecutor类中有一个HashMap，以sql为key，statement为value，缓存了预编译之后生成的statement。

1. 核心逻辑

```java
@Override
  public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
    Configuration configuration = ms.getConfiguration();
    StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
    Statement stmt = prepareStatement(handler, ms.getStatementLog());
    return handler.query(stmt, resultHandler);
  }
private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
    Statement stmt;
    BoundSql boundSql = handler.getBoundSql();
    // 例如sql: SELECT * FROM `user` WHERE id = ?
    String sql = boundSql.getSql();
    // 根据预编译的sql查找对应的statement, 如果有就复用
    if (hasStatementFor(sql)) {
      stmt = getStatement(sql);
      applyTransactionTimeout(stmt);
    } else {
      // 获取连接
      Connection connection = getConnection(statementLog);
      // 预编译
      stmt = handler.prepare(connection, transaction.getTimeout());
      // 将预编译之后的sql--statement进行缓存
      putStatement(sql, stmt);
    }
    // 设置参数
    handler.parameterize(stmt);
    return stmt;
  }
```

2. 测试代码

   ```java
     @Test
     public void testReuseExecutor() throws SQLException {
       Executor executor = new ReuseExecutor(configuration, transaction);
       MappedStatement mappedStatement = configuration.getMappedStatement("com.finalch.mybatis.dao.UserMapper.getUserById");
       RowBounds rowBounds = new RowBounds();
       List<User> query = executor.query(mappedStatement, 1, rowBounds, null);
       List<User> query1 = executor.query(mappedStatement, 2, rowBounds, null);
       System.out.println(query.get(0).getName());
       System.out.println(query1.get(0).getName());
     }
   ```

   ![reuseExecutorTestResult-1](.\images\reuseExecutorTestResult-1.png)

   一共执行了1次预编译，2次参数设置。

### BATCH

1. 只有更新操作才会进行批处理
2. 只有手动执行了doFlushStatements方法才会真正执行sql，并提交到数据库。doUpdate方法只是将需要执行的statement存储起来了，并没有真正执行。

```java
@Override
  public int doUpdate(MappedStatement ms, Object parameterObject) throws SQLException {
    final Configuration configuration = ms.getConfiguration();
    final StatementHandler handler = configuration.newStatementHandler(this, ms, parameterObject, RowBounds.DEFAULT, null, null);
    final BoundSql boundSql = handler.getBoundSql();
    final String sql = boundSql.getSql();
    final Statement stmt;
    if (sql.equals(currentSql) && ms.equals(currentStatement)) {
      int last = statementList.size() - 1;
      stmt = statementList.get(last);
      applyTransactionTimeout(stmt);
      // 设置参数
      handler.parameterize(stmt);// fix Issues 322
      BatchResult batchResult = batchResultList.get(last);
      batchResult.addParameterObject(parameterObject);
    } else {
      // 获取连接
      Connection connection = getConnection(ms.getStatementLog());
      // 预编译
      stmt = handler.prepare(connection, transaction.getTimeout());
      // 设置参数
      handler.parameterize(stmt);    // fix Issues 322
      // 记录当前sql和MappedStatement
      currentSql = sql;
      currentStatement = ms;
      statementList.add(stmt);
      batchResultList.add(new BatchResult(ms, sql, parameterObject));
    }
    handler.batch(stmt);
    return BATCH_UPDATE_RETURN_VALUE;
  }
```



## 缓存

### 一级缓存

在BaseExecutor类中实现了一级缓存的逻辑。一级缓存是基于SqlSession的，所以一级缓存的生命周期和会话的生命周期一致。在Mybatis中, 一级缓存采用的是PerpetualCache（pərˈpetʃuəl）永不过期的机制。

一级缓存失效或者一级缓存无法命中的场景：

- 不是同一个session（一级缓存是session级别的，如果session都不相同，那一级缓存肯定不相同）

- 不是同一个sql语句、查询参数不同、分页参数不同、statementId不同（一级缓存中key的生成规则涉及这几项）

- 查询之前，执行了update操作(删除/更新/插入)；

- 查询之前，执行了配置了flushCache为true的Statement；

  ```xml
  <select id="getUserById" parameterType="INTEGER" resultMap="BaseResultMap" flushCache="true">
          SELECT * FROM `user` WHERE  id = #{id}
  </select>
  ```

- 查询之前，执行了commit操作；

- 查询之前，执行了rollback操作；

- 在mybatis配置文件中配置了localCacheScope为statement。

  ```xml
  <settings>
      <setting name="logImpl" value="LOG4J"/>
      <!-- 一级缓存作用域-->
      <setting name="localCacheScope" value="STATEMENT"/>
   </settings>
  ```

  

### 二级缓存

二级缓存的逻辑在CachingExecutor中实现，默认采用的是PerputalCache，永不过期的机制，并且默认是开启的，作用域是namespace。在mapper.xml文件中可用通过<cache>标签进行配置。

```xml
 <cache eviction="LRU" flushInterval="100000" readOnly="true" size="1024"/>
```

二级缓存失效或者二级缓存未命中的场景：

- session未手动提交
- 不是同一个sql语句、查询参数不同、分页参数不同、statementId不同
- 全局缓存开关cacheEnabled不是true，默认是true
- statement配置了useCache不是true，默认是true
- 未通过cache或者cache-ref标签配置缓存空间

## 主键生成

## #{}和${}的区别

处理#{}是通过字符串替换，将#{}部分替换为参数；

处理${}是通过预编译生成SQL，使用？替换掉${}部分，然后通过PreparedStatement的set方法进行参数赋值，这样可以有效防止SQL注入。

## DAO原理

## spring+mybatis集成的原理

mybatis处理$是进行字符串替换，处理#时是对sql进行预编译，将#部分替换为?，然后通过PreparedStatement的set方法进行参数赋值，可以防止SQL注入。

## 插件

## 分页

## 延迟加载

