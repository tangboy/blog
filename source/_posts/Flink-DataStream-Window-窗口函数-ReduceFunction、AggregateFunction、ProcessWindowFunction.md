---
title: >-
  Flink DataStream Window 窗口函数
  ReduceFunction、AggregateFunction、ProcessWindowFunction
date: 2020-02-25 15:10:18
tags:
  - Flink
  - Window
  - ReduceFunction
  - AggregateFunction
  - ProcessWindowFunction
---

Window Function在窗口触发后，负责对窗口内的元素进行计算。Window Function分为两类: 增量聚合和全量聚合。

- 增量聚合: 窗口不维护原始数据，只维护中间结果，每次基于中间结果和增量数据进行聚合。如: `ReduceFunction`、`AggregateFunction`
- 全量聚合: 窗口需要维护全部原始数据，窗口触发进行全量聚合。如:`ProcessWindowFunction`

本文总结增量聚合函数(`ReduceFunction`、`AggregateFunction`)和全量聚合函数(`ProcessWindowFunction`)的使用。


注意:

1. `FoldFunction`也是增量聚合函数，但在Flink 1.9.0中已被标为过时(可用`AggregateFunction`代替)，这里不做总结。
2. `WindowFunction`也是全量聚合函数，已被更高级的`ProcessWindowFunction`逐渐代替，这里也不做总结

## 增量聚合

### ReduceFunction

```java
// 测试数据: 某个用户在某个时刻浏览了某个商品，以及商品的价值
// {"userID": "user_4", "eventTime": "2019-11-09 10:41:32", "eventType": "browse", "productID": "product_1", "productPrice": 10}

// API
// T: 输入输出元素类型
public interface ReduceFunction<T> extends Function, Serializable {
	T reduce(T value1, T value2) throws Exception;
}

// 示例: 获取一段时间内(Window Size)每个用户(KeyBy)浏览的商品的最大价值的那条记录(ReduceFunction)
kafkaStream
    // 将从Kafka获取的JSON数据解析成Java Bean
    .process(new KafkaProcessFunction())
    // 提取时间戳生成水印
    .assignTimestampsAndWatermarks(new MyCustomBoundedOutOfOrdernessTimestampExtractor(Time.seconds(maxOutOfOrdernessSeconds)))
    // 按用户分组
    .keyBy((KeySelector<UserActionLog, String>) UserActionLog::getUserID)
    // 构造TimeWindow
    .timeWindow(Time.seconds(windowLengthSeconds))
    // 窗口函数: 获取这段窗口时间内每个用户浏览的商品的最大价值对应的那条记录
    .reduce(new ReduceFunction<UserActionLog>() {
        @Override
        public UserActionLog reduce(UserActionLog value1, UserActionLog value2) throws Exception {
            return value1.getProductPrice() > value2.getProductPrice() ? value1 : value2;
        }
    })
    .print();
     
# 结果
UserActionLog{userID='user_4', eventTime='2019-11-09 12:51:25', eventType='browse', productID='product_3', productPrice=30}
UserActionLog{userID='user_2', eventTime='2019-11-09 12:51:29', eventType='browse', productID='product_2', productPrice=20}
UserActionLog{userID='user_1', eventTime='2019-11-09 12:51:22', eventType='browse', productID='product_3', productPrice=30}
UserActionLog{userID='user_5', eventTime='2019-11-09 12:51:21', eventType='browse', productID='product_3', productPrice=30}
```

注意: `ReduceFunction`输入输出元素类型相同。

<!--more-->
### AggregateFunction

```java
// 测试数据: 某个用户在某个时刻浏览了某个商品，以及商品的价值
// {"userID": "user_4", "eventTime": "2019-11-09 10:41:32", "eventType": "browse", "productID": "product_1", "productPrice": 10}

// API
// IN:  输入元素类型
// ACC: 累加器类型
// OUT: 输出元素类型
public interface AggregateFunction<IN, ACC, OUT> extends Function, Serializable {

    // 初始化累加器
	ACC createAccumulator();

    // 累加
	ACC add(IN value, ACC accumulator);

    // 累加器合并
	ACC merge(ACC a, ACC b);
	
	// 输出
	OUT getResult(ACC accumulator);
}

// 示例: 获取一段时间内(Window Size)每个用户(KeyBy)浏览的平均价值(AggregateFunction)
kafkaStream
   // 将从Kafka获取的JSON数据解析成Java Bean
   .process(new KafkaProcessFunction())
   // 提取时间戳生成水印
   .assignTimestampsAndWatermarks(new MyCustomBoundedOutOfOrdernessTimestampExtractor(Time.seconds(maxOutOfOrdernessSeconds)))
   // 按用户分组
   .keyBy((KeySelector<UserActionLog, String>) UserActionLog::getUserID)
   // 构造TimeWindow
   .timeWindow(Time.seconds(windowLengthSeconds))
   // 窗口函数: 获取这段窗口时间内，每个用户浏览的平均价值
   .aggregate(new AggregateFunction<UserActionLog, Tuple2<Long,Long>, Double>() {

       // 1、初始值
       // 定义累加器初始值
       @Override
       public Tuple2<Long, Long> createAccumulator() {
           return new Tuple2<>(0L,0L);
       }

       // 2、累加
       // 定义累加器如何基于输入数据进行累加
       @Override
       public Tuple2<Long, Long> add(UserActionLog value, Tuple2<Long, Long> accumulator) {
           accumulator.f0 += 1;
           accumulator.f1 += value.getProductPrice();
           return accumulator;
       }

       // 3、合并
       // 定义累加器如何和State中的累加器进行合并
       @Override
       public Tuple2<Long, Long> merge(Tuple2<Long, Long> acc1, Tuple2<Long, Long> acc2) {
           acc1.f0+=acc2.f0;
           acc1.f1+=acc2.f1;
           return acc1;
       }

       // 4、输出
       // 定义如何输出数据
       @Override
       public Double getResult(Tuple2<Long, Long> accumulator) {
           return accumulator.f1 / (accumulator.f0 * 1.0);
       }

   })
   .print(); 
   
#结果
20.0
10.0
30.0
25.0
20.0
```

## 全量聚合

### ProcessWindowFunction

```java
// 测试数据: 某个用户在某个时刻浏览了某个商品，以及商品的价值
// {"userID": "user_4", "eventTime": "2019-11-09 10:41:32", "eventType": "browse", "productID": "product_1", "productPrice": 10}

// API
// IN:  输入元素类型
// OUT: 输出元素类型
// KEY: Key类型
// W: Window类型
public abstract class ProcessWindowFunction<IN, OUT, KEY, W extends Window> extends AbstractRichFunction {
    ......
    
	public abstract void process(KEY key, Context context, Iterable<IN> elements, Collector<OUT> out) throws Exception;
    
    ........
}

// 示例: 获取一段时间内(Window Size)每个用户(KeyBy)浏览的商品总价值(ProcessWindowFunction)
kafkaStream
    // 将从Kafka获取的JSON数据解析成Java Bean
    .process(new KafkaProcessFunction())
    // 提取时间戳生成水印
    .assignTimestampsAndWatermarks(new MyCustomBoundedOutOfOrdernessTimestampExtractor(Time.seconds(maxOutOfOrdernessSeconds)))
    // 按用户分组
    .keyBy((KeySelector<UserActionLog, String>) UserActionLog::getUserID)
    // 构造TimeWindow
    .timeWindow(Time.seconds(windowLengthSeconds))
    // 窗口函数: 用ProcessWindowFunction计算这段时间内每个用户浏览的商品总价值
    .process(new ProcessWindowFunction<UserActionLog, String, String, TimeWindow>() {
        @Override
        public void process(String key, Context context, Iterable<UserActionLog> elements, Collector<String> out) throws Exception {

            int sum=0;
            for (UserActionLog element : elements) {
                sum += element.getProductPrice();
            }

            String windowStart=new DateTime(context.window().getStart(), DateTimeZone.forID("+08:00")).toString("yyyy-MM-dd HH:mm:ss");
            String windowEnd=new DateTime(context.window().getEnd(), DateTimeZone.forID("+08:00")).toString("yyyy-MM-dd HH:mm:ss");

            String record="Key: "+key+" 窗口开始时间: "+windowStart+" 窗口结束时间: "+windowEnd+" 浏览的商品总价值: "+sum;
            out.collect(record);

        }
    })
    .print();

// 结果
Key: user_1 窗口开始时间: 2019-11-09 13:32:00 窗口结束时间: 2019-11-09 13:32:10 浏览的商品总价值: 60
Key: user_5 窗口开始时间: 2019-11-09 13:32:00 窗口结束时间: 2019-11-09 13:32:10 浏览的商品总价值: 30
Key: user_5 窗口开始时间: 2019-11-09 13:32:10 窗口结束时间: 2019-11-09 13:32:20 浏览的商品总价值: 80
Key: user_3 窗口开始时间: 2019-11-09 13:32:10 窗口结束时间: 2019-11-09 13:32:20 浏览的商品总价值: 40
Key: user_4 窗口开始时间: 2019-11-09 13:32:10 窗口结束时间: 2019-11-09 13:32:20 浏览的商品总价值: 70
```


## ProcessWindowFunction与增量聚合结合

1. 可将`ProcessWindowFunction`与增量聚合函数`ReduceFunction`、`AggregateFunction结合`。
2. 元素到达窗口时增量聚合，当窗口关闭时对增量聚合的结果用`ProcessWindowFunction`再进行全量聚合。
3. 可以增量聚合，也可以访问窗口的元数据信息(如开始结束时间、状态等)


### ProcessWindowFunction与ReduceFunction结合

```java
// 测试数据: 某个用户在某个时刻浏览了某个商品，以及商品的价值
// {"userID": "user_4", "eventTime": "2019-11-09 10:41:32", "eventType": "browse", "productID": "product_1", "productPrice": 10}

// API: 如上ReduceFunction与ProcessWindowFunction

// 示例: 获取一段时间内(Window Size)每个用户(KeyBy)浏览的商品的最大价值的那条记录(ReduceFunction),并获得Key和Window信息。
kafkaStream
    // 将从Kafka获取的JSON数据解析成Java Bean
    .process(new KafkaProcessFunction())
    // 提取时间戳生成水印
    .assignTimestampsAndWatermarks(new MyCustomBoundedOutOfOrdernessTimestampExtractor(Time.seconds(maxOutOfOrdernessSeconds)))
    // 按用户分组
    .keyBy((KeySelector<UserActionLog, String>) UserActionLog::getUserID)
    // 构造TimeWindow
    .timeWindow(Time.seconds(windowLengthSeconds))
    // 窗口函数: 获取这段窗口时间内每个用户浏览的商品的最大价值对应的那条记录
    .reduce(
            new ReduceFunction<UserActionLog>() {
                @Override
                public UserActionLog reduce(UserActionLog value1, UserActionLog value2) throws Exception {
                    return value1.getProductPrice() > value2.getProductPrice() ? value1 : value2;
                }
            },
            new ProcessWindowFunction<UserActionLog, String, String, TimeWindow>() {
                @Override
                public void process(String key, Context context, Iterable<UserActionLog> elements, Collector<String> out) throws Exception {
                    
                    UserActionLog max = elements.iterator().next();
    
                    String windowStart=new DateTime(context.window().getStart(), DateTimeZone.forID("+08:00")).toString("yyyy-MM-dd HH:mm:ss");
                    String windowEnd=new DateTime(context.window().getEnd(), DateTimeZone.forID("+08:00")).toString("yyyy-MM-dd HH:mm:ss");
    
                    String record="Key: "+key+" 窗口开始时间: "+windowStart+" 窗口结束时间: "+windowEnd+" 浏览的商品的最大价值对应的那条记录: "+max;
                    out.collect(record);
    
                }
            }
    )
    .print();
    
// 结果
Key: user_2 窗口开始时间: 2019-11-09 13:54:10 窗口结束时间: 2019-11-09 13:54:20 浏览的商品的最大价值对应的那条记录: UserActionLog{userID='user_2', eventTime='2019-11-09 13:54:10', eventType='browse', productID='product_3', productPrice=30}
Key: user_4 窗口开始时间: 2019-11-09 13:54:10 窗口结束时间: 2019-11-09 13:54:20 浏览的商品的最大价值对应的那条记录: UserActionLog{userID='user_4', eventTime='2019-11-09 13:54:15', eventType='browse', productID='product_3', productPrice=30}
Key: user_3 窗口开始时间: 2019-11-09 13:54:10 窗口结束时间: 2019-11-09 13:54:20 浏览的商品的最大价值对应的那条记录: UserActionLog{userID='user_3', eventTime='2019-11-09 13:54:12', eventType='browse', productID='product_2', productPrice=20}
Key: user_5 窗口开始时间: 2019-11-09 13:54:10 窗口结束时间: 2019-11-09 13:54:20 浏览的商品的最大价值对应的那条记录: UserActionLog{userID='user_5', eventTime='2019-11-09 13:54:17', eventType='browse', productID='product_2', productPrice=20}
```

### ProcessWindowFunction与AggregateFunction结合

```java
// 测试数据: 某个用户在某个时刻浏览了某个商品，以及商品的价值
// {"userID": "user_4", "eventTime": "2019-11-09 10:41:32", "eventType": "browse", "productID": "product_1", "productPrice": 10}

// API: 如上AggregateFunction与ProcessWindowFunction

// 示例: 获取一段时间内(Window Size)每个用户(KeyBy)浏览的平均价值(AggregateFunction),并获得Key和Window信息。
kafkaStream
    // 将从Kafka获取的JSON数据解析成Java Bean
    .process(new KafkaProcessFunction())
    // 提取时间戳生成水印
    .assignTimestampsAndWatermarks(new MyCustomBoundedOutOfOrdernessTimestampExtractor(Time.seconds(maxOutOfOrdernessSeconds)))
    // 按用户分组
    .keyBy((KeySelector<UserActionLog, String>) UserActionLog::getUserID)
    // 构造TimeWindow
    .timeWindow(Time.seconds(windowLengthSeconds))
    // 窗口函数: 获取这段窗口时间内，每个用户浏览的商品的平均价值，并发出Key和Window信息
    .aggregate(
         new AggregateFunction<UserActionLog, Tuple2<Long, Long>, Double>() {

             // 1、初始值
             // 定义累加器初始值
             @Override
             public Tuple2<Long, Long> createAccumulator() {
                 return new Tuple2<>(0L, 0L);
             }

             // 2、累加
             // 定义累加器如何基于输入数据进行累加
             @Override
             public Tuple2<Long, Long> add(UserActionLog value, Tuple2<Long, Long> accumulator) {
                 accumulator.f0 += 1;
                 accumulator.f1 += value.getProductPrice();
                 return accumulator;
             }

             // 3、合并
             // 定义累加器如何和State中的累加器进行合并
             @Override
             public Tuple2<Long, Long> merge(Tuple2<Long, Long> acc1, Tuple2<Long, Long> acc2) {
                 acc1.f0 += acc2.f0;
                 acc1.f1 += acc2.f1;
                 return acc1;
             }

             // 4、输出
             // 定义如何输出数据
             @Override
             public Double getResult(Tuple2<Long, Long> accumulator) {
                 return accumulator.f1 / (accumulator.f0 * 1.0);
             }
         },
         new ProcessWindowFunction<Double, String, String, TimeWindow>() {
             @Override
             public void process(String key, Context context, Iterable<Double> elements, Collector<String> out) throws Exception {

                 Double avg = elements.iterator().next();

                 String windowStart=new DateTime(context.window().getStart(), DateTimeZone.forID("+08:00")).toString("yyyy-MM-dd HH:mm:ss");
                 String windowEnd=new DateTime(context.window().getEnd(), DateTimeZone.forID("+08:00")).toString("yyyy-MM-dd HH:mm:ss");

                 String record="Key: "+key+" 窗口开始时间: "+windowStart+" 窗口结束时间: "+windowEnd+" 浏览的商品的平均价值: "+String.format("%.2f",avg);
                 out.collect(record);

             }
         }

    )
    .print();
    
//结果
Key: user_2 窗口开始时间: 2019-11-09 14:05:40 窗口结束时间: 2019-11-09 14:05:50 浏览的商品的平均价值: 13.33
Key: user_3 窗口开始时间: 2019-11-09 14:05:50 窗口结束时间: 2019-11-09 14:06:00 浏览的商品的平均价值: 25.00
Key: user_4 窗口开始时间: 2019-11-09 14:05:50 窗口结束时间: 2019-11-09 14:06:00 浏览的商品的平均价值: 20.00
Key: user_2 窗口开始时间: 2019-11-09 14:05:50 窗口结束时间: 2019-11-09 14:06:00 浏览的商品的平均价值: 30.00
Key: user_5 窗口开始时间: 2019-11-09 14:05:50 窗口结束时间: 2019-11-09 14:06:00 浏览的商品的平均价值: 20.00
Key: user_1 窗口开始时间: 2019-11-09 14:05:50 窗口结束时间: 2019-11-09 14:06:00 浏览的商品的平均价值: 23.33
```