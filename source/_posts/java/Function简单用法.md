---
title: java 基础学习之Java.util.function 简单使用
tags: [java]
date: 2019-12-08
---

Java.util.function 简单使用
<!-- more -->

Java.util.function 简单使用
----

- 有很多函数
  - Consumer<T> 接受一个入参，无返回值。

```java
@Data
public class TestArgumentFunction {

  private String a = "1";
  private Integer b = 0;
  private Long c = 9L;
  private boolean d = true;

  @Test
  public void test() {

    System.out.println("str " + getA());
    setValue("bb ", this::setA);
    System.out.println("str " + getA());

    System.out.println("In " + getB());
    setValue(78, this::setB);
    System.out.println("In " + getB());

    System.out.println("Lo " + getC());
    setValue(78L, this::setC);
    System.out.println("Lo " + getC());

    System.out.println("bo " + isD());
    setValue(false, this::setD);
    System.out.println("bo " + isD());

	/* result
	str 1
    str bb 
    In 0
    In 78
    Lo 9
    Lo 78
    bo true
    bo false
    */
  }

  public void set(String str, Consumer<String> function) {
    function.accept(str);
  }

  public void setA(String a) {
    this.a = a;
  }

  public String getA() {
    return a;
  }

  // 设置参数
  private <T> void setValue(T value, Consumer<T> consumer) {

    consumer.accept(value);

  }
}
```
{% asset_img image-20190214215346153.png  %}