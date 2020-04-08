---
title: Flink之收入最高出租车司机
date: 2019-06-14 16:31:19
tags: Flink
---

本篇文章使用纽约市2009-1015年关于出租车驾驶的公共数据集，模拟现实数据流，获取一定时间内收入最高的出租车司机。

<!-- more -->

### 输入输出
输入: 详见下面数据集
输出：每个小时收入收入topN的driverId.
额外条件：
模拟丢失数据。每n条记录丢失一条数据。

### 数据集
网站[New York City Taxi & Limousine Commission](http://www.nyc.gov/html/tlc/html/home/home.shtml)提供了关于纽约市从2009-1015年关于出租车驾驶的公共数据集。

- 下载：
```
wget http://training.ververica.com/trainingData/nycTaxiRides.gz
wget http://training.ververica.com/trainingData/nycTaxiFares.gz
```
TaxiRides 行程信息。每次出行包含两条记录。分别标识为 行程开始start 和 行程结束end。
数据集结构

```
rideId         : Long      // 唯一行程id
taxiId         : Long      // 出租车唯一id 
driverId       : Long      // 出租车司机唯一id
isStart        : Boolean   // 是否是行程开始。false标识行程结束 
startTime      : DateTime  // 行程开始时间
endTime        : DateTime  // 行程结束时间 对于行程开始记录该值为 1970-01-01 00:00:00
startLon       : Float     // 开始行程的经度
startLat       : Float     // 开始行程的纬度
endLon         : Float     // 结束的经度
endLat         : Float     // 结束的纬度
passengerCnt   : Short     // 乘客数量
```

TaxiRides 数据示例

{% asset_img image-20190617122740489.png TaxiRides 数据示例 %}

TaxiFares 费用信息。 与上面行程信息对应

```
rideId         : Long      // 唯一行程id
taxiId         : Long      // 出租车唯一id
driverId       : Long      // 出租车司机唯一id
startTime      : DateTime  // 行程开始时间
paymentType    : String    // CSH or CRD
tip            : Float     // tip for this ride (消费)
tolls          : Float     // tolls for this ride
totalFare      : Float     // total fare collected
```
TaxiFares 数据示例

{% asset_img image-20190617122848797.png TaxiFares 数据示例 %}

### 分析
通过上面的数据集以及输入输出的要求，可以分析如下：
先根据上满两个数据集，生成输入流。再根据ridrId进行join，对join的结果进行窗口分割，最后对窗口内的数据入库计算收入最高的n个driverId.

### 思路
- 生成数据流。（读取上面两个数据集） 
- 模拟丢失数据 filter
- 根据routId 将两个输入流join (这里其实是过滤掉了filter过滤掉的数据的对应数据)
- 对上面join的结果划分窗口，并以driverId分组计算窗口内收入，(这里就是简单对taxiFare进行取topN)
- 选出topN
- 输出。

### 部分核心实现
新建两个class 表示ride和fare [**完整代码**](https://github.com/LAsbun/flink-learning-in-action/tree/master/src/main/java/myflink/entity)

对应source [**完整代码**](https://github.com/LAsbun/flink-learning-in-action/tree/master/src/main/java/myflink/stream/source)

主要逻辑 [**完整代码**](https://github.com/LAsbun/flink-learning-in-action/blob/master/src/main/java/myflink/stream/task/RideAndFareExercise.java)
``` java
public static void main(String[] args) throws Exception {

    // 初始化enviorment
    StreamExecutionEnvironment environment = StreamExecutionEnvironment.getExecutionEnvironment();

    // 设置事件事件
    environment.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);

    // 读取输入流
    DataStreamSource<TaxiFare> taxiFareDataStreamSource = environment
        .addSource(new TaxiFareSource());

    DataStream<TaxiFare> taxiFare = taxiFareDataStreamSource
        .keyBy(new KeySelector<TaxiFare, Long>() {
          @Override
          public Long getKey(TaxiFare value) throws Exception {
            return value.getRideId();
          }
        });

//    taxiFareDataStreamSource.print();
    DataStreamSource<TaxiRide> taxiRideDataStreamSource = environment
        .addSource(new TaxiRideSource());

//    taxiRideDataStreamSource.print();
    // 对source 过滤掉rided % 1000 == 0, 模拟现实世界丢失数据
    DataStream<TaxiRide> taxiRide = taxiRideDataStreamSource.filter(new FilterFunction<TaxiRide>() {
      @Override
      public boolean filter(TaxiRide value) throws Exception {
        return value.isStart && value.getRideId() % 1000 != 0;
      }
    })
        .keyBy(new KeySelector<TaxiRide, Long>() {
          @Override
          public Long getKey(TaxiRide value) throws Exception {
            return value.getRideId();
          }
        });

    // join 将两个输入流以rideId为key， 合并
    SingleOutputStreamOperator<Tuple2<TaxiFare, TaxiRide>> process = taxiFare.connect(taxiRide)
        .process(new ConnectProcess());

    // 设置窗口
    SingleOutputStreamOperator<Tuple3<Long, Float, Timestamp>> aggregate = process
        // 先将taxiFare 筛选出来，因为是要统计topN taxiFare
        .flatMap(new FlatMapFunction<Tuple2<TaxiFare, TaxiRide>, TaxiFare>() {
          @Override
          public void flatMap(Tuple2<TaxiFare, TaxiRide> value, Collector<TaxiFare> out)
              throws Exception {
            out.collect(value.f0);
          }
        })
        // 因为是时间递增，所以watermark 很简单
        .assignTimestampsAndWatermarks(new AscendingTimestampExtractor<TaxiFare>() {
          @Override
          public long extractAscendingTimestamp(TaxiFare element) {
//            System.out.println(element.getEventTime());
            return element.getEventTime();
          }
        })
        // 根据 driverId 分组
        .keyBy(new KeySelector<TaxiFare, Long>() {
          @Override
          public Long getKey(TaxiFare value) throws Exception {
            return value.getDriverId();
          }
        })
        // 设置时间窗口，每30min 计算一次最近1个小时的内driverId的总收入
        .timeWindow(Time.hours(1), Time.minutes(30))
        // 这个是累加函数,调用aggregate 结果会计算出同一个窗口中，每个driverId的收入总值
        .aggregate(getAggregateFunction(),
            // 这个windowFunc 是格式化输出
            new WindowFunction<Float, Tuple3<Long, Float, Timestamp>, Long, TimeWindow>() {
              @Override
              public void apply(Long driverId, TimeWindow window, Iterable<Float> input,
                  Collector<Tuple3<Long, Float, Timestamp>> out)
                  throws Exception {
                Float next = input.iterator().next();
                out.collect(new Tuple3(driverId, next, new Timestamp(window.getEnd())));
              }
            });

    // topSize N
    int topSize = 3;

    aggregate
        // 根据时间进行分窗口
        .keyBy(2)
        .timeWindow(Time.hours(1), Time.minutes(30))
        .process(new topN(topSize)).print();
    environment.execute("RideAndFareExercise");

  }
```

### 运行结果
{% asset_img image-20190620053831246.png TaxiFares 数据示例 %}

### 参考
[ververica]https://training.ververica.com/ 


