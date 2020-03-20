---
title: 'Flink Table & SQL 用户自定义函数: UDF、UDAF、UDTF'
date: 2020-02-25 16:28:27
tags:
    - Flink
    - UDF
    - UDAF
    - UDTF
---

本文总结Flink Table & SQL中的用户自定义函数: UDF、UDAF、UDTF。

- **UDF**: 自定义标量函数(User Defined Scalar Function)。一行输入一行输出。
- **UDAF**: 自定义聚合函数。多行输入一行输出。
- **UDTF**: 自定义表函数。一行输入多行输出或一列输入多列输出。

## 测试数据

```json
// 某个用户在某个时刻浏览了某个商品，以及商品的价值
// eventTime: 北京时间，方便测试。如下，乱序数据:
{"userID": "user_5", "eventTime": "2019-12-01 10:02:00", "eventType": "browse", "productID": "product_5", "productPrice": 20}
{"userID": "user_4", "eventTime": "2019-12-01 10:02:02", "eventType": "browse", "productID": "product_5", "productPrice": 20}
{"userID": "user_5", "eventTime": "2019-12-01 10:02:06", "eventType": "browse", "productID": "product_5", "productPrice": 20}
{"userID": "user_4", "eventTime": "2019-12-01 10:02:10", "eventType": "browse", "productID": "product_5", "productPrice": 20}
{"userID": "user_5", "eventTime": "2019-12-01 10:02:06", "eventType": "browse", "productID": "product_5", "productPrice": 20}
{"userID": "user_5", "eventTime": "2019-12-01 10:02:06", "eventType": "browse", "productID": "product_5", "productPrice": 20}
{"userID": "user_4", "eventTime": "2019-12-01 10:02:12", "eventType": "browse", "productID": "product_5", "productPrice": 20}
{"userID": "user_5", "eventTime": "2019-12-01 10:02:06", "eventType": "browse", "productID": "product_5", "productPrice": 20}
{"userID": "user_5", "eventTime": "2019-12-01 10:02:06", "eventType": "browse", "productID": "product_5", "productPrice": 20}
{"userID": "user_4", "eventTime": "2019-12-01 10:02:15", "eventType": "browse", "productID": "product_5", "productPrice": 20}
{"userID": "user_4", "eventTime": "2019-12-01 10:02:16", "eventType": "browse", "productID": "product_5", "productPrice": 20}
```

<!--more-->

## UDF时间转换

UDF需要继承`ScalarFunction`抽象类，主要实现eval方法。

自定义UDF，实现将Flink Window Start/End Timestamp类型时间转换为指定时区时间。

### 示例

```java
package com.bigdata.flink.tableSqlUDF.udf;

import com.alibaba.fastjson.JSON;
import com.bigdata.flink.beans.table.UserBrowseLog;
import lombok.extern.slf4j.Slf4j;
import org.apache.flink.api.common.serialization.SimpleStringSchema;
import org.apache.flink.api.java.utils.ParameterTool;
import org.apache.flink.streaming.api.TimeCharacteristic;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.ProcessFunction;
import org.apache.flink.streaming.api.functions.timestamps.BoundedOutOfOrdernessTimestampExtractor;
import org.apache.flink.streaming.api.windowing.time.Time;
import org.apache.flink.streaming.connectors.kafka.FlinkKafkaConsumer010;
import org.apache.flink.table.api.EnvironmentSettings;
import org.apache.flink.table.api.Table;
import org.apache.flink.table.api.java.StreamTableEnvironment;
import org.apache.flink.table.functions.ScalarFunction;
import org.apache.flink.types.Row;
import org.apache.flink.util.Collector;

import java.sql.Timestamp;
import java.time.*;
import java.time.format.DateTimeFormatter;
import java.util.Properties;


/**
 * Summary:
 *  UDF
 */
@Slf4j
public class Test {
    public static void main(String[] args) throws Exception{

        //args=new String[]{"--application","flink/src/main/java/com/bigdata/flink/tableSqlUDF/application.properties"};

        //1、解析命令行参数
        ParameterTool fromArgs = ParameterTool.fromArgs(args);
        ParameterTool parameterTool = ParameterTool.fromPropertiesFile(fromArgs.getRequired("application"));
        String kafkaBootstrapServers = parameterTool.getRequired("kafkaBootstrapServers");
        String browseTopic = parameterTool.getRequired("browseTopic");
        String browseTopicGroupID = parameterTool.getRequired("browseTopicGroupID");

        //2、设置运行环境
        EnvironmentSettings settings = EnvironmentSettings.newInstance().inStreamingMode().useBlinkPlanner().build();
        StreamExecutionEnvironment streamEnv = StreamExecutionEnvironment.getExecutionEnvironment();
        streamEnv.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);
        StreamTableEnvironment tableEnv = StreamTableEnvironment.create(streamEnv, settings);
        streamEnv.setParallelism(1);

        //3、注册Kafka数据源
        Properties browseProperties = new Properties();
        browseProperties.put("bootstrap.servers",kafkaBootstrapServers);
        browseProperties.put("group.id",browseTopicGroupID);
        DataStream<UserBrowseLog> browseStream=streamEnv
                .addSource(new FlinkKafkaConsumer010<>(browseTopic, new SimpleStringSchema(), browseProperties))
                .process(new BrowseKafkaProcessFunction())
                .assignTimestampsAndWatermarks(new BrowseBoundedOutOfOrdernessTimestampExtractor(Time.seconds(5)));

        // 增加一个额外的字段rowtime为事件时间属性
        tableEnv.registerDataStream("source_kafka",browseStream,"userID,eventTime,eventTimeTimestamp,eventType,productID,productPrice,rowtime.rowtime");

        //4、注册UDF
        //日期转换函数: 将Flink Window Start/End Timestamp转换为指定时区时间(默认转换为北京时间)
        tableEnv.registerFunction("UDFTimestampConverter", new UDFTimestampConverter());

        //5、运行SQL
        //基于事件时间，maxOutOfOrderness为5秒，滚动窗口，计算10秒内每个商品被浏览的PV
        String sql = ""
                + "	select "
                + "		UDFTimestampConverter(TUMBLE_START(rowtime, INTERVAL '10' SECOND),'YYYY-MM-dd HH:mm:ss') as window_start, "
                + "		UDFTimestampConverter(TUMBLE_END(rowtime, INTERVAL '10' SECOND),'YYYY-MM-dd HH:mm:ss','+08:00') as window_end, "
                + "		productID, "
                + "		count(1) as browsePV"
                + "	from source_kafka "
                + "	group by productID,TUMBLE(rowtime, INTERVAL '10' SECOND)";

        Table table = tableEnv.sqlQuery(sql);
        tableEnv.toAppendStream(table,Row.class).print();

        //6、开始执行
        tableEnv.execute(Test.class.getSimpleName());


    }

    /**
     * 自定义UDF
     */
    public static class UDFTimestampConverter extends ScalarFunction{

        /**
         * 默认转换为北京时间
         * @param timestamp Flink Timestamp 格式时间
         * @param format 目标格式,如"YYYY-MM-dd HH:mm:ss"
         * @return 目标时区的时间
         */
        public String eval(Timestamp timestamp,String format){

            LocalDateTime noZoneDateTime = timestamp.toLocalDateTime();
            ZonedDateTime utcZoneDateTime = ZonedDateTime.of(noZoneDateTime, ZoneId.of("UTC"));

            ZonedDateTime targetZoneDateTime = utcZoneDateTime.withZoneSameInstant(ZoneId.of("+08:00"));

            return targetZoneDateTime.format(DateTimeFormatter.ofPattern(format));
        }

        /**
         * 转换为指定时区时间
         * @param timestamp Flink Timestamp 格式时间
         * @param format 目标格式,如"YYYY-MM-dd HH:mm:ss"
         * @param zoneOffset 目标时区偏移量
         * @return 目标时区的时间
         */
        public String eval(Timestamp timestamp,String format,String zoneOffset){

            LocalDateTime noZoneDateTime = timestamp.toLocalDateTime();
            ZonedDateTime utcZoneDateTime = ZonedDateTime.of(noZoneDateTime, ZoneId.of("UTC"));

            ZonedDateTime targetZoneDateTime = utcZoneDateTime.withZoneSameInstant(ZoneId.of(zoneOffset));

            return targetZoneDateTime.format(DateTimeFormatter.ofPattern(format));
        }
    }




    /**
     * 解析Kafka数据
     */
    static class BrowseKafkaProcessFunction extends ProcessFunction<String, UserBrowseLog> {
        @Override
        public void processElement(String value, Context ctx, Collector<UserBrowseLog> out) throws Exception {
            try {

                UserBrowseLog log = JSON.parseObject(value, UserBrowseLog.class);

                // 增加一个long类型的时间戳
                // 指定eventTime为yyyy-MM-dd HH:mm:ss格式的北京时间
                DateTimeFormatter format = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
                OffsetDateTime eventTime = LocalDateTime.parse(log.getEventTime(), format).atOffset(ZoneOffset.of("+08:00"));
                // 转换成毫秒时间戳
                long eventTimeTimestamp = eventTime.toInstant().toEpochMilli();
                log.setEventTimeTimestamp(eventTimeTimestamp);

                out.collect(log);
            }catch (Exception ex){
                log.error("解析Kafka数据异常...",ex);
            }
        }
    }

    /**
     * 提取时间戳生成水印
     */
    static class BrowseBoundedOutOfOrdernessTimestampExtractor extends BoundedOutOfOrdernessTimestampExtractor<UserBrowseLog> {

        BrowseBoundedOutOfOrdernessTimestampExtractor(Time maxOutOfOrderness) {
            super(maxOutOfOrderness);
        }

        @Override
        public long extractTimestamp(UserBrowseLog element) {
            return element.getEventTimeTimestamp();
        }
    }
}
```

### 结果

```bash
2019-12-01 10:02:00,2019-12-01 10:02:10,product_5,7
```

## UDAF求Sum

UDAF，自定义聚合函数，需要继承`AggregateFunction`抽象类，实现一系列方法。`AggregateFunction`抽象类如下:

```java
abstract class AggregateFunction<T, ACC> extends UserDefinedAggregateFunction<T, ACC>
T: UDAF输出的结果类型
ACC: UDAF存放中间结果的类型
```

最基本的UDAF至少需要实现如下三个方法:

- `createAccumulator`: UDAF是聚合操作，需要定义一个存放中间结果的数据结构(即Accumulator)。一般，在这里，初始化时，定义这个Accumulator
- `accumulate`: 定义如何根据输入更新Accumulator
- `getValue`: 定义如何返回Accumulator中存储的中间结果作为UDAF的最终结果


除了三个基本方法外，在一些特殊的场景，可能还需要以下三个方法:

- `retract`: 和accumulate操作相反,定义如何Restract，即减少Accumulator中的值
- `merge`: 定义如何merge多个Accumulator
- `resetAccumulator`: 定义如何重置Accumulator

### 示例

```java
package com.bigdata.flink.tableSqlUDF.udaf;

import com.alibaba.fastjson.JSON;
import com.bigdata.flink.beans.table.UserBrowseLog;
import lombok.extern.slf4j.Slf4j;
import org.apache.flink.api.common.serialization.SimpleStringSchema;
import org.apache.flink.api.java.utils.ParameterTool;
import org.apache.flink.streaming.api.TimeCharacteristic;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.ProcessFunction;
import org.apache.flink.streaming.api.functions.timestamps.BoundedOutOfOrdernessTimestampExtractor;
import org.apache.flink.streaming.api.windowing.time.Time;
import org.apache.flink.streaming.connectors.kafka.FlinkKafkaConsumer010;
import org.apache.flink.table.api.EnvironmentSettings;
import org.apache.flink.table.api.Table;
import org.apache.flink.table.api.java.StreamTableEnvironment;
import org.apache.flink.table.functions.AggregateFunction;
import org.apache.flink.table.functions.ScalarFunction;
import org.apache.flink.types.Row;
import org.apache.flink.util.Collector;

import java.sql.Timestamp;
import java.time.*;
import java.time.format.DateTimeFormatter;
import java.util.Properties;


/**
 * Summary:
 *   UDAF
 */
@Slf4j
public class Test {
    public static void main(String[] args) throws Exception{

        //args=new String[]{"--application","flink/src/main/java/com/bigdata/flink/tableSqlUDF/application.properties"};

        //1、解析命令行参数
        ParameterTool fromArgs = ParameterTool.fromArgs(args);
        ParameterTool parameterTool = ParameterTool.fromPropertiesFile(fromArgs.getRequired("application"));
        String kafkaBootstrapServers = parameterTool.getRequired("kafkaBootstrapServers");
        String browseTopic = parameterTool.getRequired("browseTopic");
        String browseTopicGroupID = parameterTool.getRequired("browseTopicGroupID");

        //2、设置运行环境
        EnvironmentSettings settings = EnvironmentSettings.newInstance().inStreamingMode().useBlinkPlanner().build();
        StreamExecutionEnvironment streamEnv = StreamExecutionEnvironment.getExecutionEnvironment();
        streamEnv.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);
        StreamTableEnvironment tableEnv = StreamTableEnvironment.create(streamEnv, settings);
        streamEnv.setParallelism(1);

        //3、注册Kafka数据源
        Properties browseProperties = new Properties();
        browseProperties.put("bootstrap.servers",kafkaBootstrapServers);
        browseProperties.put("group.id",browseTopicGroupID);
        DataStream<UserBrowseLog> browseStream=streamEnv
                .addSource(new FlinkKafkaConsumer010<>(browseTopic, new SimpleStringSchema(), browseProperties))
                .process(new BrowseKafkaProcessFunction())
                .assignTimestampsAndWatermarks(new BrowseBoundedOutOfOrdernessTimestampExtractor(Time.seconds(5)));

        // 增加一个额外的字段rowtime为事件时间属性
        tableEnv.registerDataStream("source_kafka",browseStream,"userID,eventTime,eventTimeTimestamp,eventType,productID,productPrice,rowtime.rowtime");

        //4、注册自定义函数
        //UDF: 时间转换
        tableEnv.registerFunction("UDFTimestampConverter", new UDFTimestampConverter());
        //UDAF: 求Sum
        tableEnv.registerFunction("UDAFSum", new UDAFSum());

        //5、运行SQL
        //基于事件时间，maxOutOfOrderness为5秒，滚动窗口，计算10秒内每个商品被浏览的总价值
        String sql = ""
                + "	select "
                + "		UDFTimestampConverter(TUMBLE_START(rowtime, INTERVAL '10' SECOND),'YYYY-MM-dd HH:mm:ss') as window_start, "
                + "		UDFTimestampConverter(TUMBLE_END(rowtime, INTERVAL '10' SECOND),'YYYY-MM-dd HH:mm:ss','+08:00') as window_end, "
                + "		productID, "
                + "		UDAFSum(productPrice) as sumPrice"
                + "	from source_kafka "
                + "	group by productID,TUMBLE(rowtime, INTERVAL '10' SECOND)";

        Table table = tableEnv.sqlQuery(sql);
        tableEnv.toAppendStream(table,Row.class).print();

        //6、开始执行
        tableEnv.execute(Test.class.getSimpleName());


    }

    /**
     * 自定义UDF
     */
    public static class UDFTimestampConverter extends ScalarFunction{

        /**
         * 默认转换为北京时间
         * @param timestamp Flink Timestamp 格式时间
         * @param format 目标格式,如"YYYY-MM-dd HH:mm:ss"
         * @return 目标时区的时间
         */
        public String eval(Timestamp timestamp,String format){

            LocalDateTime noZoneDateTime = timestamp.toLocalDateTime();
            ZonedDateTime utcZoneDateTime = ZonedDateTime.of(noZoneDateTime, ZoneId.of("UTC"));

            ZonedDateTime targetZoneDateTime = utcZoneDateTime.withZoneSameInstant(ZoneId.of("+08:00"));

            return targetZoneDateTime.format(DateTimeFormatter.ofPattern(format));
        }

        /**
         * 转换为指定时区时间
         * @param timestamp Flink Timestamp 格式时间
         * @param format 目标格式,如"YYYY-MM-dd HH:mm:ss"
         * @param zoneOffset 目标时区偏移量
         * @return 目标时区的时间
         */
        public String eval(Timestamp timestamp,String format,String zoneOffset){

            LocalDateTime noZoneDateTime = timestamp.toLocalDateTime();
            ZonedDateTime utcZoneDateTime = ZonedDateTime.of(noZoneDateTime, ZoneId.of("UTC"));

            ZonedDateTime targetZoneDateTime = utcZoneDateTime.withZoneSameInstant(ZoneId.of(zoneOffset));

            return targetZoneDateTime.format(DateTimeFormatter.ofPattern(format));
        }
    }


    /**
     * 自定义UDAF
     */
    public static class UDAFSum extends AggregateFunction<Long, UDAFSum.SumAccumulator>{

        /**
         * 定义一个Accumulator，存放聚合的中间结果
         */
        public static class SumAccumulator{
            public long sumPrice;
        }

        /**
         * 初始化Accumulator
         * @return
         */
        @Override
        public SumAccumulator createAccumulator() {
            SumAccumulator sumAccumulator = new SumAccumulator();
            sumAccumulator.sumPrice=0;
            return sumAccumulator;
        }

        /**
         * 定义如何根据输入更新Accumulator
         * @param accumulator  Accumulator
         * @param productPrice 输入
         */
        public void accumulate(SumAccumulator accumulator,int productPrice){
            accumulator.sumPrice += productPrice;
        }

        /**
         * 返回聚合的最终结果
         * @param accumulator Accumulator
         * @return
         */
        @Override
        public Long getValue(SumAccumulator accumulator) {
            return accumulator.sumPrice;
        }
    }

    /**
     * 解析Kafka数据
     */
    static class BrowseKafkaProcessFunction extends ProcessFunction<String, UserBrowseLog> {
        @Override
        public void processElement(String value, Context ctx, Collector<UserBrowseLog> out) throws Exception {
            try {

                UserBrowseLog log = JSON.parseObject(value, UserBrowseLog.class);

                // 增加一个long类型的时间戳
                // 指定eventTime为yyyy-MM-dd HH:mm:ss格式的北京时间
                DateTimeFormatter format = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
                OffsetDateTime eventTime = LocalDateTime.parse(log.getEventTime(), format).atOffset(ZoneOffset.of("+08:00"));
                // 转换成毫秒时间戳
                long eventTimeTimestamp = eventTime.toInstant().toEpochMilli();
                log.setEventTimeTimestamp(eventTimeTimestamp);

                out.collect(log);
            }catch (Exception ex){
                log.error("解析Kafka数据异常...",ex);
            }
        }
    }

    /**
     * 提取时间戳生成水印
     */
    static class BrowseBoundedOutOfOrdernessTimestampExtractor extends BoundedOutOfOrdernessTimestampExtractor<UserBrowseLog> {

        BrowseBoundedOutOfOrdernessTimestampExtractor(Time maxOutOfOrderness) {
            super(maxOutOfOrderness);
        }

        @Override
        public long extractTimestamp(UserBrowseLog element) {
            return element.getEventTimeTimestamp();
        }
    }
}
```

### 结果

```bash
2019-12-01 10:02:00,2019-12-01 10:02:10,product_5,140
```

## UDTF一列转多列

UDTF，自定义表函数，继承TableFunction抽象类，主要实现eval方法。TableFunction抽象类如下:

```java
abstract class TableFunction<T> extends UserDefinedFunction
T: 输出的数据类型
```

注意:

1. 如果需要UDTF返回多列，只需要将返回值类型声明为`Row`或`Tuple`即可。若返回Row,需要重写`getResultType`方法，显示声明返回的`Row`的字段类型。如下，示例。
2. 在使用UDTF时，需要带上`LATERAL`和`TABLE`两个关键字。
3. UDTF支持`CROSS JOIN`和`LEFT JOIN`。
    1. `CROSS JOIN`: 对于左侧表的每一行，右侧UDTF不输出，则这一行不输出。
    2. `LEFT JOIN`: 对于左侧表的每一行，右侧UDTF不输出，则这一行会输出，右侧UDTF字段为Null。
    
### 示例

```java
package com.bigdata.flink.tableSqlUDF.udtf;

import com.alibaba.fastjson.JSON;
import com.bigdata.flink.beans.table.UserBrowseLog;
import lombok.extern.slf4j.Slf4j;
import org.apache.flink.api.common.serialization.SimpleStringSchema;
import org.apache.flink.api.common.typeinfo.TypeInformation;
import org.apache.flink.api.common.typeinfo.Types;
import org.apache.flink.api.java.typeutils.RowTypeInfo;
import org.apache.flink.api.java.utils.ParameterTool;
import org.apache.flink.streaming.api.TimeCharacteristic;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.ProcessFunction;
import org.apache.flink.streaming.api.functions.timestamps.BoundedOutOfOrdernessTimestampExtractor;
import org.apache.flink.streaming.api.windowing.time.Time;
import org.apache.flink.streaming.connectors.kafka.FlinkKafkaConsumer010;
import org.apache.flink.table.api.EnvironmentSettings;
import org.apache.flink.table.api.Table;
import org.apache.flink.table.api.java.StreamTableEnvironment;
import org.apache.flink.table.functions.AggregateFunction;
import org.apache.flink.table.functions.ScalarFunction;
import org.apache.flink.table.functions.TableFunction;
import org.apache.flink.types.Row;
import org.apache.flink.util.Collector;

import java.sql.Timestamp;
import java.time.*;
import java.time.format.DateTimeFormatter;
import java.util.Properties;


/**
 * Summary:
 *    UDTF
 */
@Slf4j
public class Test {
    public static void main(String[] args) throws Exception{

        //args=new String[]{"--application","flink/src/main/java/com/bigdata/flink/tableSqlUDF/application.properties"};

        //1、解析命令行参数
        ParameterTool fromArgs = ParameterTool.fromArgs(args);
        ParameterTool parameterTool = ParameterTool.fromPropertiesFile(fromArgs.getRequired("application"));
        String kafkaBootstrapServers = parameterTool.getRequired("kafkaBootstrapServers");
        String browseTopic = parameterTool.getRequired("browseTopic");
        String browseTopicGroupID = parameterTool.getRequired("browseTopicGroupID");

        //2、设置运行环境
        EnvironmentSettings settings = EnvironmentSettings.newInstance().inStreamingMode().useBlinkPlanner().build();
        StreamExecutionEnvironment streamEnv = StreamExecutionEnvironment.getExecutionEnvironment();
        streamEnv.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);
        StreamTableEnvironment tableEnv = StreamTableEnvironment.create(streamEnv, settings);
        streamEnv.setParallelism(1);

        //3、注册Kafka数据源
        Properties browseProperties = new Properties();
        browseProperties.put("bootstrap.servers",kafkaBootstrapServers);
        browseProperties.put("group.id",browseTopicGroupID);
        DataStream<UserBrowseLog> browseStream=streamEnv
                .addSource(new FlinkKafkaConsumer010<>(browseTopic, new SimpleStringSchema(), browseProperties))
                .process(new BrowseKafkaProcessFunction())
                .assignTimestampsAndWatermarks(new BrowseBoundedOutOfOrdernessTimestampExtractor(Time.seconds(5)));

        // 增加一个额外的字段rowtime为事件时间属性
        tableEnv.registerDataStream("source_kafka",browseStream,"userID,eventTime,eventTimeTimestamp,eventType,productID,productPrice,rowtime.rowtime");

        //4、注册自定义函数
        tableEnv.registerFunction("UDTFOneColumnToMultiColumn",new UDTFOneColumnToMultiColumn());

        //5、运行SQL
        String sql = ""
                + "select "
                + "	userID,eventTime,eventTimeTimestamp,eventType,productID,productPrice,rowtime,date1,time1 "
                + "from source_kafka ,"
                + "lateral table(UDTFOneColumnToMultiColumn(eventTime)) as T(date1,time1)";

        Table table = tableEnv.sqlQuery(sql);
        tableEnv.toAppendStream(table,Row.class).print();

        //6、开始执行
        tableEnv.execute(Test.class.getSimpleName());


    }


    /**
     * 自定义UDTF
     * 将一列变成两列。
     * 如:2019-12-01 10:02:06 转换成date1(2019-12-01)和time1(10:02:06)两列。
     */
    public static class UDTFOneColumnToMultiColumn extends TableFunction<Row>{
        public void eval(String value) {
            String[] valueSplits = value.split(" ");
            
            //一行，两列
            Row row = new Row(2);
            row.setField(0,valueSplits[0]);
            row.setField(1,valueSplits[1]);
            collect(row);
        }
        @Override
        public TypeInformation<Row> getResultType() {
            return new RowTypeInfo(Types.STRING,Types.STRING);
        }
    }

    /**
     * 解析Kafka数据
     */
    static class BrowseKafkaProcessFunction extends ProcessFunction<String, UserBrowseLog> {
        @Override
        public void processElement(String value, Context ctx, Collector<UserBrowseLog> out) throws Exception {
            try {

                UserBrowseLog log = JSON.parseObject(value, UserBrowseLog.class);

                // 增加一个long类型的时间戳
                // 指定eventTime为yyyy-MM-dd HH:mm:ss格式的北京时间
                DateTimeFormatter format = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
                OffsetDateTime eventTime = LocalDateTime.parse(log.getEventTime(), format).atOffset(ZoneOffset.of("+08:00"));
                // 转换成毫秒时间戳
                long eventTimeTimestamp = eventTime.toInstant().toEpochMilli();
                log.setEventTimeTimestamp(eventTimeTimestamp);

                out.collect(log);
            }catch (Exception ex){
                log.error("解析Kafka数据异常...",ex);
            }
        }
    }

    /**
     * 提取时间戳生成水印
     */
    static class BrowseBoundedOutOfOrdernessTimestampExtractor extends BoundedOutOfOrdernessTimestampExtractor<UserBrowseLog> {

        BrowseBoundedOutOfOrdernessTimestampExtractor(Time maxOutOfOrderness) {
            super(maxOutOfOrderness);
        }

        @Override
        public long extractTimestamp(UserBrowseLog element) {
            return element.getEventTimeTimestamp();
        }
    }

}
```

### 结果

```bash
// 最后两列是用UDTF从第二列中解析出来
user_5,2019-12-01 10:02:06,1575165726000,browse,product_5,20,2019-12-01T02:02:06,2019-12-01,10:02:06
user_5,2019-12-01 10:02:06,1575165726000,browse,product_5,20,2019-12-01T02:02:06,2019-12-01,10:02:06
user_5,2019-12-01 10:02:06,1575165726000,browse,product_5,20,2019-12-01T02:02:06,2019-12-01,10:02:06
user_5,2019-12-01 10:02:00,1575165720000,browse,product_5,20,2019-12-01T02:02:00,2019-12-01,10:02:00
user_4,2019-12-01 10:02:10,1575165730000,browse,product_5,20,2019-12-01T02:02:10,2019-12-01,10:02:10
user_4,2019-12-01 10:02:12,1575165732000,browse,product_5,20,2019-12-01T02:02:12,2019-12-01,10:02:12
user_4,2019-12-01 10:02:15,1575165735000,browse,product_5,20,2019-12-01T02:02:15,2019-12-01,10:02:15
user_4,2019-12-01 10:02:02,1575165722000,browse,product_5,20,2019-12-01T02:02:02,2019-12-01,10:02:02
user_5,2019-12-01 10:02:06,1575165726000,browse,product_5,20,2019-12-01T02:02:06,2019-12-01,10:02:06
user_5,2019-12-01 10:02:06,1575165726000,browse,product_5,20,2019-12-01T02:02:06,2019-12-01,10:02:06
user_4,2019-12-01 10:02:16,1575165736000,browse,product_5,20,2019-12-01T02:02:16,2019-12-01,10:02:16
```