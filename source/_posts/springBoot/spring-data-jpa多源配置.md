---
title: spring-data-jpa多源配置 隐式命名规则（驼峰转蛇形）失效 解决
tags: [java,spring boot]
date: 2019-11-27
---

spring-data-jpa多源配置 隐式命名规则（驼峰转蛇形）失效 解决
<!-- more -->


spring-data-jpa多源配置 隐式命名规则（驼峰转蛇形）失效 
----
- Q 配置好多源之后，隐式命名策略失效
- A 要单独写一个配置JpaProperties的函数，在生成LocalContainerEntityManagerFactoryBean的时候，初始化
  ```java

  //注入JPA配置实体
  @Autowired
  private JpaProperties jpaProperties;

  //获取jpa配置信息
  private Map<String, String> getVendorProperties(DataSource dataSource) {
    // 添加自己要选择的命名策略
    jpaProperties.getHibernate().getNaming().setPhysicalStrategy("org.springframework.boot.orm.jpa.hibernate.SpringPhysicalNamingStrategy");
    return jpaProperties.getHibernateProperties(dataSource);
  }

  @Primary
  @Bean(name = "entityManagerFactorySpiderShard")
  public LocalContainerEntityManagerFactoryBean entityManagerFactorySpiderSharding(
      EntityManagerFactoryBuilder builder) {

    return builder
        .dataSource(primaryDataSource)
        .packages("xx")
        // 在这里初始化每个源自己的命名规则
        .properties(getVendorProperties(primaryDataSource))
        .persistenceUnit("spiderShardPersistenceUnit")
        .build();
  }

  ```