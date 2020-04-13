---
title: Flink之第一个Flink程序
tags: Flink
categories: Flink
date: 2019-06-11
---

通过本篇文章，帮助你通过使用Maven 快速实现官网demo.
<!-- more -->

### 环境
- 操作系统： mac
- java版本：1.8
- Flink版本：1.7.2
- scala版本：2.11
- maven 版本:Apache Maven 3.5.2

### 描述
输入为连续的单词，每5s对30s内的单词进行计数并输出。

### 使用Maven 创建项目
``` base
mvn archetype:generate                               \
      -DarchetypeGroupId=org.apache.Flink              \
      -DarchetypeArtifactId=Flink-quickstart-java      \
      -DarchetypeVersion=1.7.2 \
	    -DgroupId=Flink-learning-in-action \
	    -DartifactId=Flink-learning-in-action \
	    -Dversion=0.1 \
	    -Dpackage=myFlink 
```
运行成功之后会再当前目录下生成一个名为 Flink-learning-in-action的目录。
目录结构
``` bash
$ tree Flink-learning-in-action
Flink-learning-in-action
├── pom.xml
└── src
    └── main
        ├── java
        │   └── myFlink
        │       ├── BatchJob.java
        │       └── StreamingJob.java
        └── resources
            └── log4j.properties
```
### 代码思想
- 获取运行环境
- 获取输入源 (socketTextStream)
- 对输入源进行算子操作
  - flatMap (拆分成单词，并给个默认值)
  - keyby分组 
  - timeWindow 划分时间窗口
  - reduce 对每一个窗口计算相同单词出现的次数
  - print 输出。

部分代码
```java
public class SocketWindowCount {

  public static void main(String[] args) throws Exception {

    ParameterTool parameterTool = ParameterTool.fromArgs(args);

    String host = parameterTool.get("host", "localhost");
    int port = parameterTool.getInt("port", 9000);

    // 先初始化执行环境
    StreamExecutionEnvironment environment = StreamExecutionEnvironment.getExecutionEnvironment();

    // 设置并发度为1， 方便观察输出
    environment.setParallelism(1);

    // 输入流
    DataStreamSource<String> socketTextStream = environment.socketTextStream(host, port);

    // 对输入的数据进行拆分处理
    DataStream<Tuple2<String, Long>> reduce = socketTextStream.flatMap(split2Word())
        // 根据tuple2 中的第一个值分组
        .keyBy(0)
        // 设置窗口，每 30s为一个窗口，每5s计算一次
        .timeWindow(Time.seconds(30), Time.seconds(5))
        // 计算
        .reduce(CountReduce());

    // 打印到控制台 输出时间
    reduce.addSink(new RichSinkFunction<Tuple2<String, Long>>() {

      @Override
      public void invoke(Tuple2<String, Long> value, Context context) {
        System.out.println(now() + " word: " + value.f0 + " count: " + value.f1);
      }
    });

    environment.execute("SocketWindowCount");

  }

  // 统计相同值出现的次数
  private static ReduceFunction<Tuple2<String, Long>> CountReduce() {
    return new ReduceFunction<Tuple2<String, Long>>() {
      @Override
      public Tuple2<String, Long> reduce(Tuple2<String, Long> value1,
          Tuple2<String, Long> value2)
          throws Exception {
        return new Tuple2<>(value1.f0, value1.f1 + value2.f1);
      }
    };
  }

  // 将输入的一行 分割成单词，并初始化次数为1
  private static FlatMapFunction<String, Tuple2<String, Long>> split2Word() {
    return new FlatMapFunction<String, Tuple2<String, Long>>() {
      @Override
      public void flatMap(String value, Collector<Tuple2<String, Long>> out) throws Exception {

        String[] words = value.split("\\W");

        for (String word : words) {
          if (word.length() > 0) {
            out.collect(new Tuple2<>(word, 1L));
          }
        }


      }
    };
  }


}

```

### 运行
1 打开终端执行
``` base
nc -l 9000
```
2 运行代码。

结果:

{% asset_img image-20190612180013464.png [title] %}

### 完整代码
[完整代码](https://github.com/LAsbun/Flink-learning-in-action/blob/master/src/main/java/myFlink/stream/task/SocketWindowCount.java)


### 参考资料
[java_api_quickstart](https://ci.apache.org/projects/Flink/Flink-docs-stable/dev/projectsetup/java_api_quickstart.html)