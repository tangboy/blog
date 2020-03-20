---
title: >-
  Flink Table & SQL
  AppendStreamTableSink、RetractStreamTableSink、UpsertStreamTableSink
date: 2020-02-25 16:22:29
tags:
  - Flink
  - AppendStreamTableSink
  - RetractStreamTableSink
  - UpsertStreamTableSink
---

Flink Table & SQL `StreamTableSink`有三类接口: `AppendStreamTableSink`、`UpsertStreamTableSink`、`RetractStreamTableSink`。

- AppendStreamTableSink: 可将动态表转换为Append流。适用于动态表只有Insert的场景。
- RetractStreamTableSink: 可将动态表转换为Retract流。适用于动态表有Insert、Delete、Update的场景。
- UpsertStreamTableSink: 可将动态表转换为Upsert流。适用于动态表有Insert、Delete、Update的场景。

注意:

- RetractStreamTableSink中: Insert被编码成一条Add消息; Delete被编码成一条Retract消息;Update被编码成两条消息(先是一条Retract消息，再是一条Add消息)，即先删除再增加。
- UpsertStreamTableSink: Insert和Update均被编码成一条消息(Upsert消息); Delete被编码成一条Delete消息。
- UpsertStreamTableSink和RetractStreamTableSink最大的不同在于Update编码成一条消息，效率上比RetractStreamTableSink高。
- 上述说的编码指的是动态表转换为DataStream时，表的增删改如何体现到DataStream上。

本文主要想总结在Update场景下，RetractStreamTableSink和UpsertStreamTableSink的消息编码。

## 测试数据

```json
// 自己造的测试数据
// 某个用户在某个时刻浏览了某个商品，以及商品的价值
// eventTime: 北京时间，方便测试。如下，乱序数据:
{"userID": "user_1", "eventTime": "2016-01-01 10:02:00", "eventType": "browse", "productID": "product_5", "productPrice": 20}
{"userID": "user_1", "eventTime": "2016-01-01 10:02:02", "eventType": "browse", "productID": "product_5", "productPrice": 20}
{"userID": "user_1", "eventTime": "2016-01-01 10:02:06", "eventType": "browse", "productID": "product_5", "productPrice": 20}
{"userID": "user_2", "eventTime": "2016-01-01 10:02:10", "eventType": "browse", "productID": "product_5", "productPrice": 20}
{"userID": "user_2", "eventTime": "2016-01-01 10:02:06", "eventType": "browse", "productID": "product_5", "productPrice": 20}
{"userID": "user_3", "eventTime": "2016-01-01 10:02:06", "eventType": "browse", "productID": "product_5", "productPrice": 20}
{"userID": "user_3", "eventTime": "2016-01-01 10:02:12", "eventType": "browse", "productID": "product_5", "productPrice": 20}
{"userID": "user_4", "eventTime": "2016-01-01 10:02:06", "eventType": "browse", "productID": "product_5", "productPrice": 20}
{"userID": "user_5", "eventTime": "2016-01-01 10:02:06", "eventType": "browse", "productID": "product_5", "productPrice": 20}
{"userID": "user_1", "eventTime": "2016-01-01 10:02:15", "eventType": "browse", "productID": "product_5", "productPrice": 20}
{"userID": "user_1", "eventTime": "2016-01-01 10:02:16", "eventType": "browse", "productID": "product_5", "productPrice": 20}
```

<!--more-->
## AppendStreamTableSink

### 示例

```java
package com.bigdata.flink.tableSqlAppendUpsertRetract.append;

import com.alibaba.fastjson.JSON;
import com.bigdata.flink.beans.table.UserBrowseLog;
import lombok.extern.slf4j.Slf4j;
import org.apache.flink.api.common.serialization.SimpleStringSchema;
import org.apache.flink.api.common.typeinfo.TypeInformation;
import org.apache.flink.api.java.utils.ParameterTool;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.datastream.DataStreamSink;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.ProcessFunction;
import org.apache.flink.streaming.api.functions.sink.RichSinkFunction;
import org.apache.flink.streaming.connectors.kafka.FlinkKafkaConsumer010;
import org.apache.flink.table.api.DataTypes;
import org.apache.flink.table.api.EnvironmentSettings;
import org.apache.flink.table.api.TableSchema;
import org.apache.flink.table.api.java.StreamTableEnvironment;
import org.apache.flink.table.sinks.AppendStreamTableSink;
import org.apache.flink.table.sinks.TableSink;
import org.apache.flink.table.types.DataType;
import org.apache.flink.types.Row;
import org.apache.flink.util.Collector;
import java.time.LocalDateTime;
import java.time.OffsetDateTime;
import java.time.ZoneOffset;
import java.time.format.DateTimeFormatter;
import java.util.Properties;

/**
 * Summary:
 *     自定义 AppendStreamTableSink 查看从只有Insert的Table到DataStream的消息编码
 *
 */
@Slf4j
public class Test {
    public static void main(String[] args) throws Exception{

        args=new String[]{"--application","flink/src/main/java/com/bigdata/flink/tableSqlAppendUpsertRetract/application.properties"};

        //1、解析命令行参数
        ParameterTool fromArgs = ParameterTool.fromArgs(args);
        ParameterTool parameterTool = ParameterTool.fromPropertiesFile(fromArgs.getRequired("application"));

        String kafkaBootstrapServers = parameterTool.getRequired("kafkaBootstrapServers");
        String browseTopic = parameterTool.getRequired("browseTopic");
        String browseTopicGroupID = parameterTool.getRequired("browseTopicGroupID");

        //2、设置运行环境
        EnvironmentSettings settings = EnvironmentSettings.newInstance().inStreamingMode().useBlinkPlanner().build();
        StreamExecutionEnvironment streamEnv = StreamExecutionEnvironment.getExecutionEnvironment();
        StreamTableEnvironment tableEnv = StreamTableEnvironment.create(streamEnv, settings);
        streamEnv.setParallelism(1);

        //3、注册Kafka数据源
        Properties browseProperties = new Properties();
        browseProperties.put("bootstrap.servers",kafkaBootstrapServers);
        browseProperties.put("group.id",browseTopicGroupID);
        DataStream<UserBrowseLog> browseStream=streamEnv
                .addSource(new FlinkKafkaConsumer010<>(browseTopic, new SimpleStringSchema(), browseProperties))
                .process(new BrowseKafkaProcessFunction());
        tableEnv.registerDataStream("source_kafka_browse_log",browseStream,"userID,eventTime,eventType,productID,productPrice,eventTimeTimestamp");


        //4、注册AppendStreamTableSink
        String[] sinkFieldNames={"userID","eventTime","eventType","productID","productPrice","eventTimeTimestamp"};
        DataType[] sinkFieldTypes={DataTypes.STRING(),DataTypes.STRING(),DataTypes.STRING(),DataTypes.STRING(),DataTypes.INT(),DataTypes.BIGINT()};
        TableSink<Row> myAppendStreamTableSink = new MyAppendStreamTableSink(sinkFieldNames,sinkFieldTypes);
        tableEnv.registerTableSink("sink_stdout",myAppendStreamTableSink);


        //5、连续查询
        //将userID为user_1的记录输出到外部存储
        String sql="insert into sink_stdout select userID,eventTime,eventType,productID,productPrice,eventTimeTimestamp from source_kafka_browse_log where userID='user_1'";
        tableEnv.sqlUpdate(sql);

        //6、开始执行
        tableEnv.execute(Test.class.getSimpleName());

    }


    /**
     * 解析Kafka数据
     * 将Kafka JSON String 解析成JavaBean: UserBrowseLog
     * UserBrowseLog(String userID, String eventTime, String eventType, String productID, int productPrice, long eventTimeTimestamp)
     */
    private static class BrowseKafkaProcessFunction extends ProcessFunction<String, UserBrowseLog> {
        @Override
        public void processElement(String value, Context ctx, Collector<UserBrowseLog> out) throws Exception {
            try {

                UserBrowseLog log = JSON.parseObject(value, UserBrowseLog.class);

                // 增加一个long类型的时间戳
                // 指定eventTime为yyyy-MM-dd HH:mm:ss格式的北京时间
                java.time.format.DateTimeFormatter format = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
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
     * 自定义 AppendStreamTableSink
     * AppendStreamTableSink 适用于表只有Insert的场景
     */
    private static class MyAppendStreamTableSink implements AppendStreamTableSink<Row> {

        private TableSchema tableSchema;

        public MyAppendStreamTableSink(String[] fieldNames, DataType[] fieldTypes) {
            this.tableSchema = TableSchema.builder().fields(fieldNames,fieldTypes).build();
        }

        @Override
        public TableSchema getTableSchema() {
            return tableSchema;
        }

        @Override
        public DataType getConsumedDataType() {
            return tableSchema.toRowDataType();
        }

        // 已过时
        @Override
        public TableSink<Row> configure(String[] fieldNames, TypeInformation<?>[] fieldTypes) {
            return null;
        }

        // 已过时
        @Override
        public void emitDataStream(DataStream<Row> dataStream) {}

        @Override
        public DataStreamSink<Row> consumeDataStream(DataStream<Row> dataStream) {
            return dataStream.addSink(new SinkFunction());
        }

        private static class SinkFunction extends RichSinkFunction<Row> {
            public SinkFunction() {
            }

            @Override
            public void invoke(Row value, Context context) throws Exception {
                System.out.println(value);
            }
        }
    }

}
```

### 结果

```bash
// Table只有Insert操作，不需要特殊编码
user_1,2016-01-01 10:02:06,browse,product_5,20,1451613726000
user_1,2016-01-01 10:02:00,browse,product_5,20,1451613720000
user_1,2016-01-01 10:02:15,browse,product_5,20,1451613735000
user_1,2016-01-01 10:02:02,browse,product_5,20,1451613722000
user_1,2016-01-01 10:02:16,browse,product_5,20,1451613736000
```

## RetractStreamTableSink

### 示例

```java
package com.bigdata.flink.tableSqlAppendUpsertRetract.retract;

import com.alibaba.fastjson.JSON;
import com.bigdata.flink.beans.table.UserBrowseLog;
import lombok.extern.slf4j.Slf4j;
import org.apache.flink.api.common.serialization.SimpleStringSchema;
import org.apache.flink.api.common.typeinfo.TypeInformation;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.api.java.typeutils.RowTypeInfo;
import org.apache.flink.api.java.utils.ParameterTool;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.datastream.DataStreamSink;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.ProcessFunction;
import org.apache.flink.streaming.api.functions.sink.RichSinkFunction;
import org.apache.flink.streaming.connectors.kafka.FlinkKafkaConsumer010;
import org.apache.flink.table.api.DataTypes;
import org.apache.flink.table.api.EnvironmentSettings;
import org.apache.flink.table.api.TableSchema;
import org.apache.flink.table.api.java.StreamTableEnvironment;
import org.apache.flink.table.sinks.RetractStreamTableSink;
import org.apache.flink.table.sinks.TableSink;
import org.apache.flink.table.types.DataType;
import org.apache.flink.types.Row;
import org.apache.flink.util.Collector;
import java.time.LocalDateTime;
import java.time.OffsetDateTime;
import java.time.ZoneOffset;
import java.time.format.DateTimeFormatter;
import java.util.Properties;

/**
 * Summary:
 *    自定义RetractStreamTableSink 查看从有更新的Table到DataStream的消息编码
 */
@Slf4j
public class Test {
    public static void main(String[] args) throws Exception{

        args=new String[]{"--application","flink/src/main/java/com/bigdata/flink/tableSqlAppendUpsertRetract/application.properties"};

        //1、解析命令行参数
        ParameterTool fromArgs = ParameterTool.fromArgs(args);
        ParameterTool parameterTool = ParameterTool.fromPropertiesFile(fromArgs.getRequired("application"));

        String kafkaBootstrapServers = parameterTool.getRequired("kafkaBootstrapServers");
        String browseTopic = parameterTool.getRequired("browseTopic");
        String browseTopicGroupID = parameterTool.getRequired("browseTopicGroupID");

        //2、设置运行环境
        EnvironmentSettings settings = EnvironmentSettings.newInstance().inStreamingMode().useBlinkPlanner().build();
        StreamExecutionEnvironment streamEnv = StreamExecutionEnvironment.getExecutionEnvironment();
        StreamTableEnvironment tableEnv = StreamTableEnvironment.create(streamEnv, settings);
        streamEnv.setParallelism(1);

        //3、注册Kafka数据源
        Properties browseProperties = new Properties();
        browseProperties.put("bootstrap.servers",kafkaBootstrapServers);
        browseProperties.put("group.id",browseTopicGroupID);
        DataStream<UserBrowseLog> browseStream=streamEnv
                .addSource(new FlinkKafkaConsumer010<>(browseTopic, new SimpleStringSchema(), browseProperties))
                .process(new BrowseKafkaProcessFunction());
        tableEnv.registerDataStream("source_kafka_browse_log",browseStream,"userID,eventTime,eventType,productID,productPrice,eventTimeTimestamp");


        //4、注册RetractStreamTableSink
        String[] sinkFieldNames={"userID","browseNumber"};
        DataType[] sinkFieldTypes={DataTypes.STRING(),DataTypes.BIGINT()};
        RetractStreamTableSink<Row> myRetractStreamTableSink = new MyRetractStreamTableSink(sinkFieldNames,sinkFieldTypes);
        tableEnv.registerTableSink("sink_stdout",myRetractStreamTableSink);


        //5、连续查询
        //统计每个Uid的浏览次数
        String sql="insert into sink_stdout select userID,count(1) as browseNumber from source_kafka_browse_log where userID in ('user_1','user_2') group by userID ";
        tableEnv.sqlUpdate(sql);


        //6、开始执行
        tableEnv.execute(Test.class.getSimpleName());


    }


    /**
     * 解析Kafka数据
     * 将Kafka JSON String 解析成JavaBean: UserBrowseLog
     * UserBrowseLog(String userID, String eventTime, String eventType, String productID, int productPrice, long eventTimeTimestamp)
     */
    private static class BrowseKafkaProcessFunction extends ProcessFunction<String, UserBrowseLog> {
        @Override
        public void processElement(String value, Context ctx, Collector<UserBrowseLog> out) throws Exception {
            try {

                UserBrowseLog log = JSON.parseObject(value, UserBrowseLog.class);

                // 增加一个long类型的时间戳
                // 指定eventTime为yyyy-MM-dd HH:mm:ss格式的北京时间
                java.time.format.DateTimeFormatter format = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
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
     * 自定义RetractStreamTableSink
     *
     * Table在内部被转换成具有Add(增加)和Retract(撤消/删除)的消息流，最终交由DataStream的SinkFunction处理。
     * DataStream里的数据格式是Tuple2类型,如Tuple2<Boolean, Row>。
     * Boolean是Add(增加)或Retract(删除)的flag(标识)。Row是真正的数据类型。
     * Table中的Insert被编码成一条Add消息。如Tuple2<True, Row>。
     * Table中的Update被编码成两条消息。一条删除消息Tuple2<False, Row>，一条增加消息Tuple2<True, Row>。
     */
    private static class MyRetractStreamTableSink implements RetractStreamTableSink<Row> {

        private TableSchema tableSchema;

        public MyRetractStreamTableSink(String[] fieldNames, DataType[] fieldTypes) {
            this.tableSchema = TableSchema.builder().fields(fieldNames,fieldTypes).build();
        }

        @Override
        public TableSchema getTableSchema() {
            return tableSchema;
        }

        // 已过时
        @Override
        public TableSink<Tuple2<Boolean, Row>> configure(String[] fieldNames, TypeInformation<?>[] fieldTypes) {
            return null;
        }

        // 已过时
        @Override
        public void emitDataStream(DataStream<Tuple2<Boolean, Row>> dataStream) {}

        // 最终会转换成DataStream处理
        @Override
        public DataStreamSink<Tuple2<Boolean, Row>> consumeDataStream(DataStream<Tuple2<Boolean, Row>> dataStream) {
            return dataStream.addSink(new SinkFunction());
        }

        @Override
        public TypeInformation<Row> getRecordType() {
            return new RowTypeInfo(tableSchema.getFieldTypes(),tableSchema.getFieldNames());
        }

        private static class SinkFunction extends RichSinkFunction<Tuple2<Boolean, Row>> {
            public SinkFunction() {
            }

            @Override
            public void invoke(Tuple2<Boolean, Row> value, Context context) throws Exception {
                Boolean flag = value.f0;
                if(flag){
                    System.out.println("增加... "+value);
                }else {
                    System.out.println("删除... "+value);
                }
            }
        }
    }
}
```

### 结果

```bash
增加... (true,user_1,1)
//user_1更新时被编译成两条消息
//先是一条删除的消息
删除... (false,user_1,1)
//再是一条增加的消息
增加... (true,user_1,2)
//同理user_2
增加... (true,user_2,1)
删除... (false,user_1,2)
增加... (true,user_1,3)
删除... (false,user_1,3)
增加... (true,user_1,4)
删除... (false,user_2,1)
增加... (true,user_2,2)
删除... (false,user_1,4)
增加... (true,user_1,5)
```

## UpsertStreamTableSink

### 示例

```java
package com.bigdata.flink.tableSqlAppendUpsertRetract.upsert;

import com.alibaba.fastjson.JSON;
import com.bigdata.flink.beans.table.UserBrowseLog;
import lombok.extern.slf4j.Slf4j;
import org.apache.flink.api.common.serialization.SimpleStringSchema;
import org.apache.flink.api.common.typeinfo.TypeInformation;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.api.java.typeutils.RowTypeInfo;
import org.apache.flink.api.java.utils.ParameterTool;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.datastream.DataStreamSink;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.ProcessFunction;
import org.apache.flink.streaming.api.functions.sink.RichSinkFunction;
import org.apache.flink.streaming.connectors.kafka.FlinkKafkaConsumer010;
import org.apache.flink.table.api.DataTypes;
import org.apache.flink.table.api.EnvironmentSettings;
import org.apache.flink.table.api.TableSchema;
import org.apache.flink.table.api.java.StreamTableEnvironment;
import org.apache.flink.table.sinks.TableSink;
import org.apache.flink.table.sinks.UpsertStreamTableSink;
import org.apache.flink.table.types.DataType;
import org.apache.flink.types.Row;
import org.apache.flink.util.Collector;
import org.joda.time.DateTime;
import org.joda.time.format.DateTimeFormat;

import java.time.LocalDateTime;
import java.time.OffsetDateTime;
import java.time.ZoneOffset;
import java.time.format.DateTimeFormatter;
import java.util.Properties;

/**
 * Summary:
 *    自定义UpsertStreamTableSink 查看从有Insert/Update的Table到DataStream的消息编码
 */
@Slf4j
public class Test {
    public static void main(String[] args) throws Exception{

        args=new String[]{"--application","flink/src/main/java/com/bigdata/flink/tableSqlAppendUpsertRetract/application.properties"};

        //1、解析命令行参数
        ParameterTool fromArgs = ParameterTool.fromArgs(args);
        ParameterTool parameterTool = ParameterTool.fromPropertiesFile(fromArgs.getRequired("application"));

        String kafkaBootstrapServers = parameterTool.getRequired("kafkaBootstrapServers");
        String browseTopic = parameterTool.getRequired("browseTopic");
        String browseTopicGroupID = parameterTool.getRequired("browseTopicGroupID");

        //2、设置运行环境
        EnvironmentSettings settings = EnvironmentSettings.newInstance().inStreamingMode().useBlinkPlanner().build();
        StreamExecutionEnvironment streamEnv = StreamExecutionEnvironment.getExecutionEnvironment();
        StreamTableEnvironment tableEnv = StreamTableEnvironment.create(streamEnv, settings);
        streamEnv.setParallelism(1);

        //3、注册Kafka数据源
        Properties browseProperties = new Properties();
        browseProperties.put("bootstrap.servers",kafkaBootstrapServers);
        browseProperties.put("group.id",browseTopicGroupID);
        DataStream<UserBrowseLog> browseStream=streamEnv
                .addSource(new FlinkKafkaConsumer010<>(browseTopic, new SimpleStringSchema(), browseProperties))
                .process(new BrowseKafkaProcessFunction());
        tableEnv.registerDataStream("source_kafka_browse_log",browseStream,"userID,eventTime,eventType,productID,productPrice,eventTimeTimestamp");

        //4、注册UpsertStreamTableSink
        String[] sinkFieldNames={"userID","browseNumber"};
        DataType[] sinkFieldTypes={DataTypes.STRING(),DataTypes.BIGINT()};
        UpsertStreamTableSink<Row> myRetractStreamTableSink = new MyUpsertStreamTableSink(sinkFieldNames,sinkFieldTypes);
        tableEnv.registerTableSink("sink_stdout",myRetractStreamTableSink);

        //5、连续查询
        //统计每个Uid的浏览次数
        String sql="insert into sink_stdout select userID,count(1) as browseNumber from source_kafka_browse_log where userID in ('user_1','user_2') group by userID ";
        tableEnv.sqlUpdate(sql);

        //6、开始执行
        tableEnv.execute(Test.class.getSimpleName());
    }


    /**
     * 解析Kafka数据
     * 将Kafka JSON String 解析成JavaBean: UserBrowseLog
     * UserBrowseLog(String userID, String eventTime, String eventType, String productID, int productPrice, long eventTimeTimestamp)
     */
    private static class BrowseKafkaProcessFunction extends ProcessFunction<String, UserBrowseLog> {
        @Override
        public void processElement(String value, Context ctx, Collector<UserBrowseLog> out) throws Exception {
            try {

                UserBrowseLog log = JSON.parseObject(value, UserBrowseLog.class);

                // 增加一个long类型的时间戳
                // 指定eventTime为yyyy-MM-dd HH:mm:ss格式的北京时间
                java.time.format.DateTimeFormatter format = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
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
     * 自定义UpsertStreamTableSink
     * Table在内部被转换成具有Add(增加)和Retract(撤消/删除)的消息流，最终交由DataStream的SinkFunction处理。
     * Boolean是Add(增加)或Retract(删除)的flag(标识)。Row是真正的数据类型。
     * Table中的Insert被编码成一条Add消息。如Tuple2<True, Row>。
     * Table中的Update被编码成一条Add消息。如Tuple2<True, Row>。
     * 在SortLimit(即order by ... limit ...)的场景下，被编码成两条消息。一条删除消息Tuple2<False, Row>，一条增加消息Tuple2<True, Row>。
     */
    private static class MyUpsertStreamTableSink implements UpsertStreamTableSink<Row> {

        private TableSchema tableSchema;

        public MyUpsertStreamTableSink(String[] fieldNames, DataType[] fieldTypes) {
            this.tableSchema = TableSchema.builder().fields(fieldNames,fieldTypes).build();
        }

        @Override
        public TableSchema getTableSchema() {
            return tableSchema;
        }


        // 设置Unique Key
        // 如上SQL中有GroupBy，则这里的唯一键会自动被推导为GroupBy的字段
        @Override
        public void setKeyFields(String[] keys) {}

        // 是否只有Insert
        // 如上SQL场景，需要Update，则这里被推导为isAppendOnly=false
        @Override
        public void setIsAppendOnly(Boolean isAppendOnly) {}

        @Override
        public TypeInformation<Row> getRecordType() {
            return new RowTypeInfo(tableSchema.getFieldTypes(),tableSchema.getFieldNames());
        }

        // 已过时
        @Override
        public void emitDataStream(DataStream<Tuple2<Boolean, Row>> dataStream) {}

        // 最终会转换成DataStream处理
        @Override
        public DataStreamSink<Tuple2<Boolean, Row>> consumeDataStream(DataStream<Tuple2<Boolean, Row>> dataStream) {
            return dataStream.addSink(new SinkFunction());
        }

        @Override
        public TableSink<Tuple2<Boolean, Row>> configure(String[] fieldNames, TypeInformation<?>[] fieldTypes) {
            return null;
        }

        private static class SinkFunction extends RichSinkFunction<Tuple2<Boolean, Row>> {
            public SinkFunction() {
            }

            @Override
            public void invoke(Tuple2<Boolean, Row> value, Context context) throws Exception {
                Boolean flag = value.f0;
                if(flag){
                    System.out.println("增加... "+value);
                }else {
                    System.out.println("删除... "+value);
                }
            }
        }
    }
}
```

### 结果

```bash
增加... (true,user_1,1)
//user_1更新时被编译成一条消息
增加... (true,user_1,2)
//user_1更新时被编译成一条消息
增加... (true,user_1,3)
增加... (true,user_1,4)
//同理user_2更新
增加... (true,user_2,1)
增加... (true,user_2,2)
增加... (true,user_1,5)
```