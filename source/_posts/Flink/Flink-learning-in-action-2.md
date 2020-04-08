---
title: Flink之获取topNWord
date: 2019-06-12 23:40:43
tags: Flink
categories: Flink
---

本片文章基于{% post_link Flink-learning-in-action-1 上一篇文章 %}的逻辑上增加获取topN的逻辑，可以加深对Flink的认识。

<!-- more -->


### 描述
{% post_link Flink-learning-in-action-1 上一篇文章 %}中只是统计并输出了一定时间内的相同单词的次数，这次我们更深入点，统计一定时间内的前N个word.

### 思路：
- 获取运行环境
- 获取输入源 (socketTextStream)
-对输入源进行算子操作
  - flatMap (拆分成单词，并给个默认值)
  - keyby分组
  - timeWindow 划分时间窗口
  - reduce 对每一个窗口计算相同单词出现的次数
  - 增加新窗口（里面的数据是上一步统计好次数之后的单词）
  - 对上面窗口中的数据排序，输出前topN
  - print 输出。

### 部分代码
```java
// 对输入的数据进行拆分处理
    DataStream<Tuple2<String, Long>> ret = socketTextStream.flatMap(split2Word())
        // 根据tuple2 中的第一个值分组
        .keyBy(0)
        // 设置窗口，每 30s为一个窗口，每5s计算一次
        .timeWindow(Time.seconds(30), Time.seconds(5))
        // 相同字母次数相加
        .reduce(CountReduce())
        // 滑动窗口，每 30s为一个窗口，每5s计算一次 （新增的逻辑）
        .windowAll(SlidingProcessingTimeWindows.of(Time.seconds(30), Time.seconds(5)))
        // 对同一个窗口的所有元素排序取前topSize （新增的逻辑）
        .process(new TopN(topSize));

```
- TopN class 
``` java
static class TopN extends
      ProcessAllWindowFunction<Tuple2<String, Long>, Tuple2<String, Long>, TimeWindow> {

    private final int topSize;

    TopN(int topSize) {
      this.topSize = topSize;
    }

    @Override
    public void process(Context context, Iterable<Tuple2<String, Long>> elements,
        Collector<Tuple2<String, Long>> out) throws Exception {

      /*
      1 先创建一颗有序树，
      2 依次往树里面放数据
      3 如果超过topSize 那么就去掉树的最后一个节点
       */

      TreeMap<Long, Tuple2<String, Long>> treeMap = new TreeMap<>(
          new Comparator<Long>() {
            @Override
            public int compare(Long o1, Long o2) {
              return o2 > o1 ? 1 : -1;
            }
          }
      );

      for (Tuple2<String, Long> element : elements) {

        treeMap.put(element.f1, element);
        if (treeMap.size() > this.topSize) {
          treeMap.pollLastEntry();
        }

      }

      for (Entry<Long, Tuple2<String, Long>> longTuple2Entry : treeMap.entrySet()) {
        out.collect(longTuple2Entry.getValue());
      }

    }
  }
```

### 完整代码
[完整代码请点我](https://github.com/LAsbun/Flink-learning-in-action/blob/master/src/main/java/myFlink/stream/task/SocketWindowCountTopN.java)
