---
title: JAVA String vs StringBuffer vs StringBuilder
date: 2019-11-25 09:29:40
tags: java
---

通过本篇文章，你可以了解到StringBuffer与StringBuilder的区别。

<!-- more -->

String 不可变
--- 
内部数据是由final 修饰的
```
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {
    /** The value is used for character storage. */
    private final char value[];
```

StringBuffer vs StringBuilder 均可变
---
StringBuffer 和 StringBuilder 都是继承AbstractStringBuilder。
```
abstract class AbstractStringBuilder implements Appendable, CharSequence {
    /**
     * The value is used for character storage.
     */
    char[] value;

    /**
     * The count is the number of characters used.
     */
    int count;
```

- 线程安全性
    - StringBuffer 是线程安全的 使用synchronized 修饰了函数
    - StringBuilder 没有用synchronized修饰,是非线程安全的 

- 数据修改
    - StringBuffer 每次都是对原有对象的引用修改
    ```
    @Override
    public synchronized String toString() {
        if (toStringCache == null) {
            toStringCache = Arrays.copyOfRange(value, 0, count);
        }
        return new String(toStringCache, true);
    }
    ```
    - StringBuilder 每次都是创建一个新的对象
    ```
    @Override
    public String toString() {
        // Create a copy, don't share the array
        return new String(value, 0, count);
    }
    ```

- 性能
    相比之下 StringBuilder要比StringBuffer快，但是有线程不安全