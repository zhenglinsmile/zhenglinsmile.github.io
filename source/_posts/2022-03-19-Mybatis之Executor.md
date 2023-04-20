---
title: Mybatis 之 Executor
date: 2022-03-19 11:43:58
categories:
  - mybatis
tags:
  - mybatis
---

# Mybatis Executor

```
日常工作中，mybatis框架已经使用了很多年，然而终究只是停留在最表面。早就听闻其源码优雅，最近忍不住再一次捡起来看看，然观之仍不得要领，仅记之。
```

## Executor 接口

```java
public interface Executor {

  ResultHandler NO_RESULT_HANDLER = null;
	// 更新方法
  int update(MappedStatement ms, Object parameter) throws SQLException;

  <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey cacheKey, BoundSql boundSql) throws SQLException;

  <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException;

  <E> Cursor<E> queryCursor(MappedStatement ms, Object parameter, RowBounds rowBounds) throws SQLException;

  List<BatchResult> flushStatements() throws SQLException;

  void commit(boolean required) throws SQLException;

  void rollback(boolean required) throws SQLException;

  CacheKey createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql);

  boolean isCached(MappedStatement ms, CacheKey key);

  void clearLocalCache();

  void deferLoad(MappedStatement ms, MetaObject resultObject, String property, CacheKey key, Class<?> targetType);

  Transaction getTransaction();

  void close(boolean forceRollback);

  boolean isClosed();

  void setExecutorWrapper(Executor executor);

}
```

## **继承关系**

![image-20220319115805569](https://raw.githubusercontent.com/zhenglinsmile/picture/master/img/image-20220319115805569.png)

Executor 方法摘要

![image-20220319120704115](https://raw.githubusercontent.com/zhenglinsmile/picture/master/img/image-20220319120704115.png)

BaseExecutor 方法摘要

红框中四个抽象方法，由 BaseExecutor 子类实现，这样可以在 BaseExecutor 其中进行统一的处理，如一级缓存相关。

![image-20220319120822251](https://raw.githubusercontent.com/zhenglinsmile/picture/master/img/image-20220319120822251.png)

以 BaseExecutor 中的 query 分析执行 select 语句的过程

```java
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
		// 是否清除本地缓存
    if (queryStack == 0 && ms.isFlushCacheRequired()) {
      clearLocalCache();
    }
    List<E> list;
    try {
      queryStack++;
      // 缓存中获取数据
      list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
      if (list != null) {
        handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
      } else {
        // 执行查询处理
        list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
      }
    } finally {
      queryStack--;
    }
  ......
    return list;
  }
```

queryFromDatabase

```java
private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    List<E> list;
    localCache.putObject(key, EXECUTION_PLACEHOLDER);
    try {
      // 调用具体的实现类中的方法
      list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
    } finally {
      localCache.removeObject(key);
    }
    // 缓存处理
    localCache.putObject(key, list);
    // 存储过程调用返回值处理
    if (ms.getStatementType() == StatementType.CALLABLE) {
      localOutputParameterCache.putObject(key, parameter);
    }
    return list;
  }
```

## 对比 doQuery 方法

### SimpleExecutor

```java
public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
    Statement stmt = null;
    try {
      Configuration configuration = ms.getConfiguration();
      StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
      stmt = prepareStatement(handler, ms.getStatementLog());
      return handler.query(stmt, resultHandler);
    } finally {
      closeStatement(stmt);
    }
  }

  private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
    Statement stmt;
    Connection connection = getConnection(statementLog);
    stmt = handler.prepare(connection, transaction.getTimeout());
    handler.parameterize(stmt);
    return stmt;
  }
```

### ReuseExecutor

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
    String sql = boundSql.getSql();
    // statement 重用
    if (hasStatementFor(sql)) {
      stmt = getStatement(sql);
      applyTransactionTimeout(stmt);
    } else {
      Connection connection = getConnection(statementLog);
      stmt = handler.prepare(connection, transaction.getTimeout());
      // 添加缓存
      putStatement(sql, stmt);
    }
    handler.parameterize(stmt);
    return stmt;
  }
```

### CacheExecutor

通过 Debug SqlSession sqlSession = sqlSessionFactory.openSession() ，查看到 CachingExecutor 的由来：

```java
public class Configuration {
  	public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
      executorType = executorType == null ? defaultExecutorType : executorType;
      executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
      Executor executor;
      if (ExecutorType.BATCH == executorType) {
        executor = new BatchExecutor(this, transaction);
      } else if (ExecutorType.REUSE == executorType) {
        executor = new ReuseExecutor(this, transaction);
      } else {
        executor = new SimpleExecutor(this, transaction);
      }
      // 二级缓存 mybatis-config.xml 中配置，默认为true
      // <settings>
        // <setting name="cacheEnabled" value="true"/>
      // </settings>
      if (cacheEnabled) {
        executor = new CachingExecutor(executor);
      }
      // 责任链
      executor = (Executor) interceptorChain.pluginAll(executor);
      return executor;
    }
}
```

```java
public class InterceptorChain {

  private final List<Interceptor> interceptors = new ArrayList<>();

  public Object pluginAll(Object target) {
    for (Interceptor interceptor : interceptors) {
      target = interceptor.plugin(target);
    }
    return target;
  }

  public void addInterceptor(Interceptor interceptor) {
    interceptors.add(interceptor);
  }

  public List<Interceptor> getInterceptors() {
    return Collections.unmodifiableList(interceptors);
  }

}
```
