---
title: Flink Table & SQL 时态表Temporal Table
date: 2020-02-19 23:10:10
tags:
    - Flink
    - Table API
    - SQL
    - Temporal Table
---

举个栗子，假设你在Mysql中有两张表: browse_event、product_history_info。

- browse_event: 事件表，某个用户在某个时刻浏览了某个商品，以及商品的价值。如下:
```sql
SELECT * FROM browse_event;
    
+--------+---------------------+-----------+-----------+--------------+
| userID | eventTime           | eventType | productID | productPrice |
+--------+---------------------+-----------+-----------+--------------+
| user_1 | 2016-01-01 00:00:00 | browse    | product_5 |           20 |
| user_1 | 2016-01-01 00:00:01 | browse    | product_5 |           20 |
| user_1 | 2016-01-01 00:00:02 | browse    | product_5 |           20 |
| user_1 | 2016-01-01 00:00:03 | browse    | product_5 |           20 |
| user_1 | 2016-01-01 00:00:04 | browse    | product_5 |           20 |
| user_1 | 2016-01-01 00:00:05 | browse    | product_5 |           20 |
| user_1 | 2016-01-01 00:00:06 | browse    | product_5 |           20 |
| user_2 | 2016-01-01 00:00:01 | browse    | product_3 |           20 |
| user_2 | 2016-01-01 00:00:02 | browse    | product_3 |           20 |
| user_2 | 2016-01-01 00:00:05 | browse    | product_3 |           20 |
| user_2 | 2016-01-01 00:00:06 | browse    | product_3 |           20 |
+--------+---------------------+-----------+-----------+--------------+
```

- product_history_info:商品基础信息表，记录了商品历史以来的基础信息。如下:
```sql
SELECT * FROM product_history_info;
+-----------+-------------+-----------------+---------------------+
| productID | productName | productCategory | updatedAt           |
+-----------+-------------+-----------------+---------------------+
| product_5 | name50      | category50      | 2016-01-01 00:00:00 |
| product_5 | name52      | category52      | 2016-01-01 00:00:02 |
| product_5 | name55      | category55      | 2016-01-01 00:00:05 |
| product_3 | name32      | category32      | 2016-01-01 00:00:02 |
| product_3 | name35      | category35      | 2016-01-01 00:00:05 |
+-----------+-------------+-----------------+---------------------+
```

此刻，你想获取事件发生时，对应的最新的商品基础信息。可能需要借助以下SQL实现:

```sql
SELECT l.userID,
       l.eventTime,
       l.eventType,
       l.productID,
       l.productPrice,
       r.productID,
       r.productName,
       r.productCategory,
       r.updatedAt
FROM
    browse_event AS l,
    product_history_info AS r
WHERE r.productID = l.productID
 AND r.updatedAt = (
    SELECT max(updatedAt)
    FROM product_history_info AS r2
    WHERE r2.productID = l.productID
      AND r2.updatedAt <= l.eventTime
)

// 结果
+--------+---------------------+-----------+-----------+--------------+-----------+-------------+-----------------+---------------------+
| userID | eventTime           | eventType | productID | productPrice | productID | productName | productCategory | updatedAt           |
+--------+---------------------+-----------+-----------+--------------+-----------+-------------+-----------------+---------------------+
| user_1 | 2016-01-01 00:00:00 | browse    | product_5 |           20 | product_5 | name50      | category50      | 2016-01-01 00:00:00 |
| user_1 | 2016-01-01 00:00:01 | browse    | product_5 |           20 | product_5 | name50      | category50      | 2016-01-01 00:00:00 |
| user_1 | 2016-01-01 00:00:02 | browse    | product_5 |           20 | product_5 | name52      | category52      | 2016-01-01 00:00:02 |
| user_1 | 2016-01-01 00:00:03 | browse    | product_5 |           20 | product_5 | name52      | category52      | 2016-01-01 00:00:02 |
| user_1 | 2016-01-01 00:00:04 | browse    | product_5 |           20 | product_5 | name52      | category52      | 2016-01-01 00:00:02 |
| user_1 | 2016-01-01 00:00:05 | browse    | product_5 |           20 | product_5 | name55      | category55      | 2016-01-01 00:00:05 |
| user_1 | 2016-01-01 00:00:06 | browse    | product_5 |           20 | product_5 | name55      | category55      | 2016-01-01 00:00:05 |
| user_2 | 2016-01-01 00:00:02 | browse    | product_3 |           20 | product_3 | name32      | category32      | 2016-01-01 00:00:02 |
| user_2 | 2016-01-01 00:00:05 | browse    | product_3 |           20 | product_3 | name35      | category35      | 2016-01-01 00:00:05 |
| user_2 | 2016-01-01 00:00:06 | browse    | product_3 |           20 | product_3 | name35      | category35      | 2016-01-01 00:00:05 |
+--------+---------------------+-----------+-----------+--------------+-----------+-------------+-----------------+---------------------+
```

在Flink中，为了处理类似场景，从1.7开始，提出了时态表(即Temporal Table)的概念。Temporal Table可以简化和加速此类查询，并减少对状态的使用。Temporal Table是将一个Append-Only表(如上product_history_info)中追加的行，根据设置的主键和时间(如上productID、updatedAt)，解释成Chanlog，并在特定时间提供数据的版本。

以下，在Flink中，实现如上逻辑，并总结在使用Flink Temporal Table时得注意点。
<!--more-->

## 测试数据

自己造的测试数据，browse log和product history info,如下:

```json
// browse log
{"userID": "user_1", "eventTime": "2016-01-01 00:00:00", "eventType": "browse", "productID": "product_5", "productPrice": 20}
{"userID": "user_1", "eventTime": "2016-01-01 00:00:01", "eventType": "browse", "productID": "product_5", "productPrice": 20}
{"userID": "user_1", "eventTime": "2016-01-01 00:00:02", "eventType": "browse", "productID": "product_5", "productPrice": 20}
{"userID": "user_1", "eventTime": "2016-01-01 00:00:03", "eventType": "browse", "productID": "product_5", "productPrice": 20}
{"userID": "user_1", "eventTime": "2016-01-01 00:00:04", "eventType": "browse", "productID": "product_5", "productPrice": 20}
{"userID": "user_1", "eventTime": "2016-01-01 00:00:05", "eventType": "browse", "productID": "product_5", "productPrice": 20}
{"userID": "user_1", "eventTime": "2016-01-01 00:00:06", "eventType": "browse", "productID": "product_5", "productPrice": 20}
{"userID": "user_2", "eventTime": "2016-01-01 00:00:01", "eventType": "browse", "productID": "product_3", "productPrice": 20}
{"userID": "user_2", "eventTime": "2016-01-01 00:00:02", "eventType": "browse", "productID": "product_3", "productPrice": 20}
{"userID": "user_2", "eventTime": "2016-01-01 00:00:05", "eventType": "browse", "productID": "product_3", "productPrice": 20}
{"userID": "user_2", "eventTime": "2016-01-01 00:00:06", "eventType": "browse", "productID": "product_3", "productPrice": 20}

// product history info
{"productID":"product_5","productName":"name50","productCategory":"category50","updatedAt":"2016-01-01 00:00:00"}
{"productID":"product_5","productName":"name52","productCategory":"category52","updatedAt":"2016-01-01 00:00:02"}
{"productID":"product_5","productName":"name55","productCategory":"category55","updatedAt":"2016-01-01 00:00:05"}
{"productID":"product_3","productName":"name32","productCategory":"category32","updatedAt":"2016-01-01 00:00:02"}
{"productID":"product_3","productName":"name35","productCategory":"category35","updatedAt":"2016-01-01 00:00:05"}

```

## 示例

```java
package com.bigdata.flink.tableSqlTemporalTable;

import com.alibaba.fastjson.JSON;
import com.bigdata.flink.beans.table.ProductInfo;
import com.bigdata.flink.beans.table.UserBrowseLog;
import lombok.extern.slf4j.Slf4j;
import org.apache.flink.api.common.serialization.SimpleStringSchema;
import org.apache.flink.api.java.utils.ParameterTool;
import org.apache.flink.configuration.Configuration;
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
import org.apache.flink.table.functions.TemporalTableFunction;
import org.apache.flink.types.Row;
import org.apache.flink.util.Collector;

import java.time.LocalDateTime;
import java.time.OffsetDateTime;
import java.time.ZoneOffset;
import java.time.format.DateTimeFormatter;
import java.util.Properties;


/**
 * Summary:
 *  时态表(Temporal Table)
 */
@Slf4j
public class Test {
    public static void main(String[] args) throws Exception{

        args=new String[]{"--application","flink/src/main/java/com/bigdata/flink/tableSqlTemporalTable/application.properties"};

        //1、解析命令行参数
        ParameterTool fromArgs = ParameterTool.fromArgs(args);
        ParameterTool parameterTool = ParameterTool.fromPropertiesFile(fromArgs.getRequired("application"));

        //browse log
        String kafkaBootstrapServers = parameterTool.getRequired("kafkaBootstrapServers");
        String browseTopic = parameterTool.getRequired("browseTopic");
        String browseTopicGroupID = parameterTool.getRequired("browseTopicGroupID");

        //product history info
        String productInfoTopic = parameterTool.getRequired("productHistoryInfoTopic");
        String productInfoGroupID = parameterTool.getRequired("productHistoryInfoGroupID");

        //2、设置运行环境
        StreamExecutionEnvironment streamEnv = StreamExecutionEnvironment.createLocalEnvironmentWithWebUI(new Configuration());
        streamEnv.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);
        EnvironmentSettings settings = EnvironmentSettings.newInstance().inStreamingMode().useBlinkPlanner().build();
        StreamTableEnvironment tableEnv = StreamTableEnvironment.create(streamEnv, settings);
        streamEnv.setParallelism(1);

        //3、注册Kafka数据源
        //注意: 为了在北京时间和时间戳之间有直观的认识，这里的UserBrowseLog中增加了一个字段eventTimeTimestamp作为eventTime的时间戳
        Properties browseProperties = new Properties();
        browseProperties.put("bootstrap.servers",kafkaBootstrapServers);
        browseProperties.put("group.id",browseTopicGroupID);
        DataStream<UserBrowseLog> browseStream=streamEnv
                .addSource(new FlinkKafkaConsumer010<>(browseTopic, new SimpleStringSchema(), browseProperties))
                .process(new BrowseKafkaProcessFunction())
                .assignTimestampsAndWatermarks(new BrowseTimestampExtractor(Time.seconds(0)));

        tableEnv.registerDataStream("browse",browseStream,"userID,eventTime,eventTimeTimestamp,eventType,productID,productPrice,browseRowtime.rowtime");
        //tableEnv.toAppendStream(tableEnv.scan("browse"),Row.class).print();

        //4、注册时态表(Temporal Table)
        //注意: 为了在北京时间和时间戳之间有直观的认识，这里的ProductInfo中增加了一个字段updatedAtTimestamp作为updatedAt的时间戳
        Properties productInfoProperties = new Properties();
        productInfoProperties.put("bootstrap.servers",kafkaBootstrapServers);
        productInfoProperties.put("group.id",productInfoGroupID);
        DataStream<ProductInfo> productInfoStream=streamEnv
                .addSource(new FlinkKafkaConsumer010<>(productInfoTopic, new SimpleStringSchema(), productInfoProperties))
                .process(new ProductInfoProcessFunction())
                .assignTimestampsAndWatermarks(new ProductInfoTimestampExtractor(Time.seconds(0)));

        tableEnv.registerDataStream("productInfo",productInfoStream, "productID,productName,productCategory,updatedAt,updatedAtTimestamp,productInfoRowtime.rowtime");
        //设置Temporal Table的时间属性和主键
        TemporalTableFunction productInfo = tableEnv.scan("productInfo").createTemporalTableFunction("productInfoRowtime", "productID");
        //注册TableFunction
        tableEnv.registerFunction("productInfoFunc",productInfo);
        //tableEnv.toAppendStream(tableEnv.scan("productInfo"),Row.class).print();

        //5、运行SQL
        String sql = ""
                + "SELECT "
                + "browse.userID, "
                + "browse.eventTime, "
                + "browse.eventTimeTimestamp, "
                + "browse.eventType, "
                + "browse.productID, "
                + "browse.productPrice, "
                + "productInfo.productID, "
                + "productInfo.productName, "
                + "productInfo.productCategory, "
                + "productInfo.updatedAt, "
                + "productInfo.updatedAtTimestamp "
                + "FROM "
                + " browse, "
                + " LATERAL TABLE (productInfoFunc(browse.browseRowtime)) as productInfo "
                + "WHERE "
                + " browse.productID=productInfo.productID";

        Table table = tableEnv.sqlQuery(sql);
        tableEnv.toAppendStream(table,Row.class).print();

        //6、开始执行
        tableEnv.execute(Test.class.getSimpleName());


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
    static class BrowseTimestampExtractor extends BoundedOutOfOrdernessTimestampExtractor<UserBrowseLog> {

        BrowseTimestampExtractor(Time maxOutOfOrderness) {
            super(maxOutOfOrderness);
        }

        @Override
        public long extractTimestamp(UserBrowseLog element) {
            return element.getEventTimeTimestamp();
        }
    }





    /**
     * 解析Kafka数据
     */
    static class ProductInfoProcessFunction extends ProcessFunction<String, ProductInfo> {
        @Override
        public void processElement(String value, Context ctx, Collector<ProductInfo> out) throws Exception {
            try {

                ProductInfo log = JSON.parseObject(value, ProductInfo.class);

                // 增加一个long类型的时间戳
                // 指定eventTime为yyyy-MM-dd HH:mm:ss格式的北京时间
                DateTimeFormatter format = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
                OffsetDateTime eventTime = LocalDateTime.parse(log.getUpdatedAt(), format).atOffset(ZoneOffset.of("+08:00"));
                // 转换成毫秒时间戳
                long eventTimeTimestamp = eventTime.toInstant().toEpochMilli();
                log.setUpdatedAtTimestamp(eventTimeTimestamp);

                out.collect(log);
            }catch (Exception ex){
                log.error("解析Kafka数据异常...",ex);
            }
        }
    }

    /**
     * 提取时间戳生成水印
     */
    static class ProductInfoTimestampExtractor extends BoundedOutOfOrdernessTimestampExtractor<ProductInfo> {

        ProductInfoTimestampExtractor(Time maxOutOfOrderness) {
            super(maxOutOfOrderness);
        }

        @Override
        public long extractTimestamp(ProductInfo element) {
            return element.getUpdatedAtTimestamp();
        }
    }

}
```

## 结果

在对应Kafka Topic中发送如上测试数据后，得到结果。

```
// 可以看到，获取到了，事件发生时，对应的历史最新的商品基础信息
user_1,2016-01-01 00:00:01,1451577601000,browse,product_5,20,product_5,name50,category50,2016-01-01 00:00:00,1451577600000
user_1,2016-01-01 00:00:04,1451577604000,browse,product_5,20,product_5,name52,category52,2016-01-01 00:00:02,1451577602000
user_1,2016-01-01 00:00:02,1451577602000,browse,product_5,20,product_5,name52,category52,2016-01-01 00:00:02,1451577602000
user_1,2016-01-01 00:00:05,1451577605000,browse,product_5,20,product_5,name55,category55,2016-01-01 00:00:05,1451577605000
user_1,2016-01-01 00:00:00,1451577600000,browse,product_5,20,product_5,name50,category50,2016-01-01 00:00:00,1451577600000
user_1,2016-01-01 00:00:03,1451577603000,browse,product_5,20,product_5,name52,category52,2016-01-01 00:00:02,1451577602000
user_2,2016-01-01 00:00:02,1451577602000,browse,product_3,20,product_3,name32,category32,2016-01-01 00:00:02,1451577602000
user_2,2016-01-01 00:00:05,1451577605000,browse,product_3,20,product_3,name35,category35,2016-01-01 00:00:05,1451577605000
```

## 总结

在使用时态表(Temporal Table)时，要注意以下问题。

1. Temporal Table可提供历史某个时间点上的数据。
2. Temporal Table根据时间来跟踪版本。
3. Temporal Table需要提供时间属性和主键。
4. emporal Table一般和关键词`LATERAL TABLE`结合使用。
5. Temporal Table在基于ProcessingTime时间属性处理时，每个主键只保存最新版本的数据。
6. Temporal Table在基于EventTime时间属性处理时，每个主键保存从上个Watermark到当前系统时间的所有版本。
7. 侧Append-Only表Join右侧Temporal Table，本质上还是左表驱动Join，即从左表拿到Key，根据Key和时间(可能是历史时间)去右侧Temporal Table表中查询。
8. Temporal Table Join目前只支持Inner Join & Left Join。
9. Temporal Table Join时，右侧Temporal Table表返回最新一个版本的数据。举个栗子，左侧事件时间如是2016-01-01 00:00:01秒，Join时，只会从右侧Temporal Table中选取<=2016-01-01 00:00:01的最新版本的数据。
