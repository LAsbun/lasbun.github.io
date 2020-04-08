---
title: spring-boot 运行源码分析
tags: [java,spring boot]
---

spring-boot 运行源码分析
<!-- more -->

spring-boot
----

  - 一个简单的demo。
    ```java

    @SpringBootApplication
    public class Main {

      public static void main(String[] args) {
        
        // 主要运行类
        SpringApplication.run(Main.class, args);
      }

    }
    ```

  - 可以看到源码

    ```
    public static ConfigurableApplicationContext run(Object[] sources, String[] args) {
      // 加载资源，
      // 再运行.
      return new SpringApplication(sources).run(args);
    }

    ```

      - 初始化资源
        ```
        public SpringApplication(Object... sources) {
          // 初始化资源。
          initialize(sources);
        }
        ```

        ```
        // 加载资源。主要初始化上下文，以及监听器
        @SuppressWarnings({ "unchecked", "rawtypes" })
        private void initialize(Object[] sources) {
          if (sources != null && sources.length > 0) {
            this.sources.addAll(Arrays.asList(sources));
          }
          // deduceWebEnvironment 判断是不是web环境
          this.webEnvironment = deduceWebEnvironment();
          // 获取ContextInitializer 应用程序初始化器
          /* 
            getSpringFactoriesInstances 
            会读取spring-core-xxx . META-INF 中的spring.factory中的文件。
          */
          setInitializers((Collection) getSpringFactoriesInstances(
              ApplicationContextInitializer.class));
          // listener 应用程序监听器 
          setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
          // 找出main类，这里是MyApplication类
          this.mainApplicationClass = deduceMainApplicationClass();
        }
        ```
        - spring.factory 主要包含了以下几个。
        ```
          # PropertySource Loaders
          org.springframework.boot.env.PropertySourceLoader

          # Failure Analyzers
          org.springframework.boot.diagnostics.FailureAnalyzer

          # Run Listeners
          org.springframework.boot.SpringApplicationRunListener

          # Environment Post Processors
          org.springframework.boot.env.EnvironmentPostProcessor

          # Application Listeners
          org.springframework.context.ApplicationListener=

          # FailureAnalysisReporters
          org.springframework.boot.diagnostics.FailureAnalysisReporter=
        ```