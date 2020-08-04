---
title: '聊聊Mybatis[StatementHandler核心原理]'
tags:
  - java
  - mybatis
  - mysql
date: 2020-08-04 12:00:09
---


我们前面将SqlSession将SQL执行委托给Executor,Executor又最终委托给了StatementHandler. 本篇我们来剖析下StatementHandler是如何做到实质性的SQL执行。

<!-- more -->

#### 一些概念
##### JDBC 中的Statement/PrepareStatement/CallableStatement接口是啥
简单讲：`*Statement`是在我们获取到数据库连接conn之后，可以让我们发送SQL命令或者PL/SQL(过程化SQL)命令到数据库，并可以从数据库获取到数据。
**就是一个中介，我们只需要通过`*Statement`执行SQL,获取结果，其他的我们就可以不用关心了。**
* 三种Statement接口功能表
{% asset_img image-20200804080928005.png %}

#### 一次Query的时序图
我们通过一次Query的时序图来大致看下SqlSession,Executor,StatementHandler, `*Statement`之间的关系.

{% asset_img  image-20200804083037367.png %}

观看上图，其实StatementHandler也不是最终的执行者，而是委托给了`*Statement`.但是由于`*Statement`是JDBC的东西(Mybatis本身就是针对jdbc-connector的二次封装)，所以对于Mybatis来讲，StatementHandler就是SQL执行前的最后一棒。

#### StatementHandler 部分源码分析

##### 先来一个类关系图

{% asset_img image-20200804083602612.png %}
* BaseStatementHandler 抽象基类。类似BaseExecutor. 提供通用方法支持，需要定制化，使用模板模式由子类实现.
* SimpleStatementHandler 继承BaseStatementHandler. 操作JDBC.Statement接口。
* PreparedStatementHandler 继承BaseStatementHandler. 操作JDBC.PreParedStatement接口。
* CallableStatementHandler 继承BaseStatementHandler. 操作JDBC.CallableStatement接口。
* RoutingStatementHandler 策略类，根据MappedStatement类型创建上面三种StatementHandler.

##### BaseStetementHandler
主要是对prepare封装了一层，实际实现还是在子类的`instantiateStatement`
```
@Override
  public Statement prepare(Connection connection, Integer transactionTimeout) throws SQLException {
    ErrorContext.instance().sql(boundSql.getSql());
    Statement statement = null;
    try {
      statement = instantiateStatement(connection);
      setStatementTimeout(statement, transactionTimeout);
      setFetchSize(statement);
      return statement;
    } catch (SQLException e) {
      closeStatement(statement);
      throw e;
    } catch (Exception e) {
      closeStatement(statement);
      throw new ExecutorException("Error preparing statement.  Cause: " + e, e);
    }
  }

  protected abstract Statement instantiateStatement(Connection connection) throws SQLException;

```
#### SimpleStatementHandler
简单不解释
```
@Override
  public int update(Statement statement) throws SQLException {
    String sql = boundSql.getSql();
    Object parameterObject = boundSql.getParameterObject();
    KeyGenerator keyGenerator = mappedStatement.getKeyGenerator();
    int rows;
    if (keyGenerator instanceof Jdbc3KeyGenerator) {
      statement.execute(sql, Statement.RETURN_GENERATED_KEYS);
      rows = statement.getUpdateCount();
      keyGenerator.processAfter(executor, mappedStatement, statement, parameterObject);
    } else if (keyGenerator instanceof SelectKeyGenerator) {
      statement.execute(sql);
      rows = statement.getUpdateCount();
      keyGenerator.processAfter(executor, mappedStatement, statement, parameterObject);
    } else {
      statement.execute(sql);
      rows = statement.getUpdateCount();
    }
    return rows;
  }

  @Override
  public void batch(Statement statement) throws SQLException {
    String sql = boundSql.getSql();
    statement.addBatch(sql);
  }

  @Override
  public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
    String sql = boundSql.getSql();
    statement.execute(sql);
    return resultSetHandler.<E>handleResultSets(statement);
  }

  @Override
  public <E> Cursor<E> queryCursor(Statement statement) throws SQLException {
    String sql = boundSql.getSql();
    statement.execute(sql);
    return resultSetHandler.<E>handleCursorResultSets(statement);
  }

// 这里就是创建Statement.
  @Override
  protected Statement instantiateStatement(Connection connection) throws SQLException {
    if (mappedStatement.getResultSetType() != null) {
      return connection.createStatement(mappedStatement.getResultSetType().getValue(), ResultSet.CONCUR_READ_ONLY);
    } else {
      return connection.createStatement();
    }
  }

// Statement不支持传参
  @Override
  public void parameterize(Statement statement) throws SQLException {
    // N/A
  }
```
##### CallableStatementHandler
创建CallableStatement。使用ParameterHandler.setParameters 设置传参. 
```
@Override
  protected Statement instantiateStatement(Connection connection) throws SQLException {
    String sql = boundSql.getSql();
    if (mappedStatement.getResultSetType() != null) {
      return connection.prepareCall(sql, mappedStatement.getResultSetType().getValue(), ResultSet.CONCUR_READ_ONLY);
    } else {
      return connection.prepareCall(sql);
    }
  }

  @Override
  public void parameterize(Statement statement) throws SQLException {
  	// 这里特别的对于ReturnOutType 做了一次转换
    registerOutputParameters((CallableStatement) statement);
    parameterHandler.setParameters((CallableStatement) statement);
  }
```
#####PreParedStatementHandler
创建PreParedStatement。使用ParameterHandler.setParameters 设置传参. 
```
@Override
  protected Statement instantiateStatement(Connection connection) throws SQLException {
    String sql = boundSql.getSql();
    if (mappedStatement.getKeyGenerator() instanceof Jdbc3KeyGenerator) {
      String[] keyColumnNames = mappedStatement.getKeyColumns();
      if (keyColumnNames == null) {
        return connection.prepareStatement(sql, PreparedStatement.RETURN_GENERATED_KEYS);
      } else {
        return connection.prepareStatement(sql, keyColumnNames);
      }
    } else if (mappedStatement.getResultSetType() != null) {
      return connection.prepareStatement(sql, mappedStatement.getResultSetType().getValue(), ResultSet.CONCUR_READ_ONLY);
    } else {
      return connection.prepareStatement(sql);
    }
  }

  @Override
  public void parameterize(Statement statement) throws SQLException {
    parameterHandler.setParameters((PreparedStatement) statement);
  }
```
##### ParameterHandler
mybatis里面只有一种实现.`DefaultParameterHandler`
```
// 入参是PreparedStatement， 是因为CallableStatment是继承PreparedStatement接口的
@Override
  public void setParameters(PreparedStatement ps) {
    ErrorContext.instance().activity("setting parameters").object(mappedStatement.getParameterMap().getId());
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    if (parameterMappings != null) {
      for (int i = 0; i < parameterMappings.size(); i++) {
        ParameterMapping parameterMapping = parameterMappings.get(i);
        if (parameterMapping.getMode() != ParameterMode.OUT) {
          Object value;
          String propertyName = parameterMapping.getProperty();
          if (boundSql.hasAdditionalParameter(propertyName)) { // issue #448 ask first for additional params
            value = boundSql.getAdditionalParameter(propertyName);
          } else if (parameterObject == null) {
            value = null;
          } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
            value = parameterObject;
          } else {
            MetaObject metaObject = configuration.newMetaObject(parameterObject);
            value = metaObject.getValue(propertyName);
          }
          // 这里是指定的typeHandler
          TypeHandler typeHandler = parameterMapping.getTypeHandler();
          JdbcType jdbcType = parameterMapping.getJdbcType();
          if (value == null && jdbcType == null) {
            jdbcType = configuration.getJdbcTypeForNull();
          }
          try {
            typeHandler.setParameter(ps, i + 1, value, jdbcType);
          } catch (TypeException e) {
            throw new TypeException("Could not set parameters for mapping: " + parameterMapping + ". Cause: " + e, e);
          } catch (SQLException e) {
            throw new TypeException("Could not set parameters for mapping: " + parameterMapping + ". Cause: " + e, e);
          }
        }
      }
    }
  }
```
##### StatementHandler的创建时机和创建策略控制
Executor每次执行update或者是query都是会创建一个新的StatementHandler
```
@Override
  public int doUpdate(MappedStatement ms, Object parameter) throws SQLException {
    Statement stmt = null;
    try {
      Configuration configuration = ms.getConfiguration();
      // 这里new
      StatementHandler handler = configuration.newStatementHandler(this, ms, parameter, RowBounds.DEFAULT, null, null);
      stmt = prepareStatement(handler, ms.getStatementLog());
      return handler.update(stmt);
    } finally {
      closeStatement(stmt);
    }
  }

  @Override
  public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
    Statement stmt = null;
    try {
      Configuration configuration = ms.getConfiguration();
      // 这里new
      StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
      stmt = prepareStatement(handler, ms.getStatementLog());
      return handler.<E>query(stmt, resultHandler);
    } finally {
      closeStatement(stmt);
    }
  }
```
创建策略有三种.  默认为PREPARED.
```
public enum StatementType {
  STATEMENT, PREPARED, CALLABLE
}
```
需要指定的话，可以在mapper里面跟随sql来指定
```
    <select id="findOne" resultMap="articleResult" statementType="PREPARED"> // 这里指定
        SELECT
            id, author_id, title, content, create_time
        FROM
            article
        WHERE
            id = #{id}
    </select>
```

#### 参考大佬
[Mybatis3.3.x技术内幕（六）：StatementHandler（Box stop here） 
祖大俊](https://my.oschina.net/zudajun/blog/668378)