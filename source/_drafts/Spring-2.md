---
title: 聊聊Spring[Spring AOP]
tags: [java,spring,aop]
---

本篇从..

<!-- more -->


#### 什么是AOP
##### 概念	
AOP(Aspect Oriented Programming) 面向切面编程。是OOP(Object oriented Programming)的延伸。
##### 举个栗子
假设我们现在有一个简易的商城项目，你现在负责的模块有三个，用户订单管理，用户物流查询，用户基本信息查询。
这三个模块的共同点都是需要用户登录，即需要进行用户鉴权。通常的解决办法是在业务请求前调用额外代码来进行.如下
```
	@Component
	public class OrderManager {
	
	    @Autowired
	    AuthManager authManager;
	
	    public void selectOrder() {
	
	        // 鉴权
	        authManager.auth();
	
	        System.out.println("查询到订单");
	    }
	}
```

这种模式带来的问题是，如果后续增加调用日志，又会改动原有代码逻辑。(**当然这种情况也有其他的解决方式比如抽出一个公共网关服务做前置调度也可以**)
```
  @Component
  public class OrderManager {

      @Autowired
      AuthManager authManager;

      @Autowired
      LogManager logManager;

      public void selectOrder() {
					//增加一行日志
          logManager.log();

          // 鉴权
          authManager.auth();

          System.out.println("查询到订单");
      }
  }
```
AOP这个时候就派上用场了。不需要额外改动原有逻辑,只需要一个注解，就可以在不影响原有业务逻辑情况下进行额外的功能。改造后的代码如下：
```
  @Component
  public class OrderManager {

      @Autowired
      AuthManager authManager;

      @Autowired
      LogManager logManager;

      @AuthAspect
      public void selectOrder() {

  //        logManager.log();

          // 鉴权
  //        authManager.auth();

          System.out.println("查询到订单");
      }
  }
```
##### AOP解决了什么问题
通过上面的示例，我们可以得出AOP主要用来解决：在不改变原有业务逻辑情况下，增加横切逻辑(鉴权，日志等与实际业务逻辑无关的逻辑)。或者是说，AOP实现的是在我们原来写的代码基础上，进行一定的包装(类似python中的修饰器)，做进一步的拦截或者是增强。

#### 什么是Spring AOP
Spring AOP 是AOP编程中的一种实现. **其致力于企业级开发中普遍的AOP需求(方法植入)**。另外一种AOP编程的实现`AspectJ`是AOP编程的完全解决方案，可以干很多Spring AOP干不了的事情。
关于`AspectJ` 后续有机会整理下。

#### Spring AOP 的使用方式

#### Spring AOP 源码

#### 总结

#### 参考大佬

```

```