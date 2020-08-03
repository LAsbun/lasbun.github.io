---
title: '聊聊Mybatis[Exector核心原理]'
tags:
  - java
  - mysql
  - mybatis
date: 2020-08-03 09:07:52
---


本文深入分析下Executor一些核心原理

<!-- more -->

SqlSession对数据库的操作都是委托给Executor.下面我们剖析下Executor的核心原理。

#### 前述
Executor 上承SqlSession 下接StatementHandler。处于Mybatis体系的核心，深入理解Executor的实现原理，对实际开发会更有帮助。

#### Executor接口
```
public interface Executor {

  ResultHandler NO_RESULT_HANDLER = null;

  int update(MappedStatement ms, Object parameter) throws SQLException;

  <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey cacheKey, BoundSql boundSql) throws SQLException;

  <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException;

  <E> Cursor<E> queryCursor(MappedStatement ms, Object parameter, RowBounds rowBounds) throws SQLException;
	// 这个是为了批量查询
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
	// 这个wrapper实际是只会用到CacheExecutor. 用到SelectKeyGenerator.processGeneratedKeys()函数 提供对应的事务。我理解为了保证同一个Session的事务都是一个事务管理器。
  void setExecutorWrapper(Executor executor);

}

```

##### 类关系图
实际只有五种:
- SimpleExecutor
- ReuseExecutor
- ClosedExecutor
- CachingExecutor
- BatchExecutor

{% asset_img mybatis-executor.png %}

注意:**以上执行器的生命周期都是在SqlSession生命周期内**

##### SimpleExecutor
执行update/select 每次都创建一个StatementHandler,使用完就关闭。

##### CloseExecutor
私有类，没有什么实质实现.

##### ReuseExecutor
执行update/select 创建了StatementHandler之后，使用完，不关闭，缓存起来。

##### BatchExecutor
执行update. 不支持批量select.  所有的sql都会调用StatementHandler.addBatch()，等待executeBatch().   
这里特别要注意下: 每个statement都可能是一批sql. BatchExecutor是维护了多个statement。借用大佬的解释`BatchExecutor 维护了多个桶，每个桶里面有多个自己的SQL，就像苹果蓝里装了很多苹果，番茄蓝里装了很多番茄，最后，再统一倒进仓库(统一执行)。`

##### CacheExecutor 
基于装饰器模式,如果有缓存就直接返回缓存，如果没有缓存就委托给`SimpleExecutor`|`ReuseExecutor`|`BatchExecutor`.这三种执行器执行。

#### 部分源码分析
##### BaseExecutor 抽象类
部分代码
```
@Override
  public int update(MappedStatement ms, Object parameter) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing an update").object(ms.getId());
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
    clearLocalCache();
    return doUpdate(ms, parameter);
  }
  protected abstract int doUpdate(MappedStatement ms, Object parameter)
      throws SQLException;

  protected abstract List<BatchResult> doFlushStatements(boolean isRollback)
      throws SQLException;

  protected abstract <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql)
      throws SQLException;

  protected abstract <E> Cursor<E> doQueryCursor(MappedStatement ms, Object parameter, RowBounds rowBounds, BoundSql boundSql)
      throws SQLException;
```
典型的模板模式，具体的实现交给下面的子类来实现.

##### SimpleExecutor
很简单，就是创建Statement,然后执行完后，就关闭掉。 其他方法可以看下具体实现，原理都一样.
```
@Override
  public int doUpdate(MappedStatement ms, Object parameter) throws SQLException {
    Statement stmt = null;
    try {
      Configuration configuration = ms.getConfiguration();
      StatementHandler handler = configuration.newStatementHandler(this, ms, parameter, RowBounds.DEFAULT, null, null);
      stmt = prepareStatement(handler, ms.getStatementLog());
      return handler.update(stmt);
    } finally {
      closeStatement(stmt);
    }
  }
```
注意：SimpleExecutor不支持doFlushStatements(批量刷新statement),所以是直接返回的一个空列表.
```
  @Override
  public List<BatchResult> doFlushStatements(boolean isRollback) throws SQLException {
    return Collections.emptyList();
  }
```

##### ReuseExecutor
```
@Override
  public int doUpdate(MappedStatement ms, Object parameter) throws SQLException {
    Configuration configuration = ms.getConfiguration();
    StatementHandler handler = configuration.newStatementHandler(this, ms, parameter, RowBounds.DEFAULT, null, null);
    // 这里做缓存，如果存在就直接返回，不存在直接创建，再返回
    Statement stmt = prepareStatement(handler, ms.getStatementLog());
    return handler.update(stmt);
  }
 private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
    Statement stmt;
    BoundSql boundSql = handler.getBoundSql();
    String sql = boundSql.getSql();
    if (hasStatementFor(sql)) {
      stmt = getStatement(sql);
      applyTransactionTimeout(stmt);
    } else {
      Connection connection = getConnection(statementLog);
      stmt = handler.prepare(connection, transaction.getTimeout());
      putStatement(sql, stmt);
    }
    handler.parameterize(stmt);
    return stmt;
```
注意: 因为ReuseExecutor 缓存了StatementHandler，所以doFlushStatements, 会关闭之前缓存的StatementHandler，然后返回一个空列表
```
  @Override
  public List<BatchResult> doFlushStatements(boolean isRollback) throws SQLException {
    for (Statement stmt : statementMap.values()) {
      closeStatement(stmt);
    }
    statementMap.clear();
    return Collections.emptyList();
  }
```
##### BatchExecutor
```
// 保存需要执行的statementList, 为之后批次执行做准备
 private final List<Statement> statementList = new ArrayList<Statement>();
 // 结果
  private final List<BatchResult> batchResultList = new ArrayList<BatchResult>();
  // 当前编译过的Sql
  private String currentSql;
  // 当前的statement
  private MappedStatement currentStatement;


@Override
  public int doUpdate(MappedStatement ms, Object parameterObject) throws SQLException {
    final Configuration configuration = ms.getConfiguration();
    final StatementHandler handler = configuration.newStatementHandler(this, ms, parameterObject, RowBounds.DEFAULT, null, null);
    final BoundSql boundSql = handler.getBoundSql();
    final String sql = boundSql.getSql();
    final Statement stmt;
    // 如果当前SQL与之前SQL不一致，那么久需要重新编译SQL
    if (sql.equals(currentSql) && ms.equals(currentStatement)) {
      int last = statementList.size() - 1;
      stmt = statementList.get(last);
      applyTransactionTimeout(stmt);
     handler.parameterize(stmt);//fix Issues 322
      BatchResult batchResult = batchResultList.get(last);
      batchResult.addParameterObject(parameterObject);
    } else {
      Connection connection = getConnection(ms.getStatementLog());
      stmt = handler.prepare(connection, transaction.getTimeout());
      handler.parameterize(stmt);    //fix Issues 322
      currentSql = sql;
      currentStatement = ms;
      // 增加进待执行list
      statementList.add(stmt);
      batchResultList.add(new BatchResult(ms, sql, parameterObject));
    }
  	// 将sql添加进statement. 由statementhandler 内部执行.（jdbc.xxstatementHandler）
    handler.batch(stmt);
    return BATCH_UPDATE_RETURN_VALUE;
  }
  
  // commit/rollback/close-》 都会调用doFlushSatements,然后会执行stmt.executeBatch()。
  @Override
  public List<BatchResult> doFlushStatements(boolean isRollback) throws SQLException {
    try {
      List<BatchResult> results = new ArrayList<BatchResult>();
      if (isRollback) {
        return Collections.emptyList();
      }
      // 多个桶，一次执行每个桶
      for (int i = 0, n = statementList.size(); i < n; i++) {
        Statement stmt = statementList.get(i);
        applyTransactionTimeout(stmt);
        BatchResult batchResult = batchResultList.get(i);
        try {
        // 这里是会实际执行
          batchResult.setUpdateCounts(stmt.executeBatch());
          MappedStatement ms = batchResult.getMappedStatement();
          List<Object> parameterObjects = batchResult.getParameterObjects();
          // 主键
          KeyGenerator keyGenerator = ms.getKeyGenerator();
          if (Jdbc3KeyGenerator.class.equals(keyGenerator.getClass())) {
            Jdbc3KeyGenerator jdbc3KeyGenerator = (Jdbc3KeyGenerator) keyGenerator;
            jdbc3KeyGenerator.processBatch(ms, stmt, parameterObjects);
          } else if (!NoKeyGenerator.class.equals(keyGenerator.getClass())) { //issue #141
            for (Object parameter : parameterObjects) {
              keyGenerator.processAfter(this, ms, stmt, parameter);
            }
          }
          // Close statement to close cursor #1109
          closeStatement(stmt);
        } catch (BatchUpdateException e) {
          StringBuilder message = new StringBuilder();
          message.append(batchResult.getMappedStatement().getId())
              .append(" (batch index #")
              .append(i + 1)
              .append(")")
              .append(" failed.");
          if (i > 0) {
            message.append(" ")
                .append(i)
                .append(" prior sub executor(s) completed successfully, but will be rolled back.");
          }
          throw new BatchExecutorException(message.toString(), e, results, batchResult);
        }
        // 结果
        results.add(batchResult);
      }
      return results;
    } finally {
      for (Statement stmt : statementList) {
        closeStatement(stmt);
      }
      // 初始化掉成员变量
      currentSql = null;
      statementList.clear();
      batchResultList.clear();
    }
  }
```
需要注意的是: 如果SQL是AABBAA->那么就会有三个statement.
调用时序图

{% asset_img image-20200803154241375.png %}

##### CachingExecutor
装饰模式，实际是使用的SimpleExecutor, ReusedExecutor, BatchExecutor 三种之一。

#### tips
- Executor什么时候生成 
	- 在创建SqlSession的时候，随之生成。
- Executor 默认方式是什么？ 可否指定
	- 默认是SimpleExecutor
	- 可以指定，有下面两种方式
		- 通过SqlSessionFactory.openSession() 传入ExecutorType
		- 通过xml <seetting name="defaultExecutorType" value="REUSE"> 来指定  	


#### 参考大佬
[Mybatis3.3.x技术内幕（四）：五鼠闹东京之执行器Executor设计原本 ](https://my.oschina.net/zudajun/blog/667214)


