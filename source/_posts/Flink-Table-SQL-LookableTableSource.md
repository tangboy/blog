---
title: Flink Table & SQL LookableTableSource
date: 2020-02-25 17:56:41
tags:
    - Flink
    - LookableTableSource
---

在DataStream中，要实现流维Join，可以用Function，如`MapFunction`、`FlatMapFunction`、`ProcessFunction`等等; 或通过Async I/O实现。

从Flink 1.9.0开始，提供了`LookableTableSource`，只需将Lookup数据源(如Mysql、HBase表)注册成`LookableTableSource`，即可用SQL的方式，实现流维Join。


注意：

1. `LookableTableSource`只支持Blink Planner。
2. Lookup数据源要注册成`LookableTableSource`，需要实现`LookableTableSource`接口。

## LookableTableSource接口

```java
public interface LookupableTableSource<T> extends TableSource<T> {

	TableFunction<T> getLookupFunction(String[] lookupKeys);

	AsyncTableFunction<T> getAsyncLookupFunction(String[] lookupKeys);

	boolean isAsyncEnabled();
}
```

可看到，`LookupableTableSource`主要有三个方法:`getLookupFunction`、`getAsyncLookupFunction`、`isAsyncEnabled`。

- `getLookupFunction`: 返回一个TableFunction。该Function可和LATERAL TABLE关键字结合使用，根据指定的key同步查找匹配的行。
- `getAsyncLookupFunction`: 返回一个TableFunction。该Function可和`LATERAL TABLE`关键字结合使用，根据指定的key异步(Async I/O的方式)查找匹配的行。
- `isAsyncEnabled`: 如果启用了异步Lookup，则此方法应返回true。当返回true时，必须实现`getAsyncLookupFunction(String[] lookupKeys)`方法。

<!--more-->
## 用LookableTableSource实现Kafka Join HBase/Mysql维度数据

### 示例

```java
package com.bigdata.flink.tableSqlLookableTableSource;

import com.alibaba.fastjson.JSON;
import com.bigdata.flink.beans.table.UserBrowseLog;
import lombok.extern.slf4j.Slf4j;
import org.apache.flink.addons.hbase.HBaseTableSource;
import org.apache.flink.api.common.serialization.SimpleStringSchema;
import org.apache.flink.api.java.io.jdbc.JDBCLookupOptions;
import org.apache.flink.api.java.io.jdbc.JDBCOptions;
import org.apache.flink.api.java.io.jdbc.JDBCTableSource;
import org.apache.flink.api.java.utils.ParameterTool;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.ProcessFunction;
import org.apache.flink.streaming.connectors.kafka.FlinkKafkaConsumer010;
import org.apache.flink.table.api.DataTypes;
import org.apache.flink.table.api.EnvironmentSettings;
import org.apache.flink.table.api.TableSchema;
import org.apache.flink.table.api.java.StreamTableEnvironment;
import org.apache.flink.table.types.DataType;
import org.apache.flink.types.Row;
import org.apache.flink.util.Collector;
import org.apache.hadoop.conf.Configuration;

import java.time.LocalDateTime;
import java.time.OffsetDateTime;
import java.time.ZoneOffset;
import java.time.format.DateTimeFormatter;
import java.util.Properties;

/**
 * Summary:
 *  Lookup Table Source
 */
@Slf4j
public class Test {
    public static void main(String[] args) throws Exception{

        args=new String[]{"--application","flink/src/main/java/com/bigdata/flink/tableSqlLookableTableSource/application.properties"};

        //1、解析命令行参数
        ParameterTool fromArgs = ParameterTool.fromArgs(args);
        ParameterTool parameterTool = ParameterTool.fromPropertiesFile(fromArgs.getRequired("application"));

        String kafkaBootstrapServers = parameterTool.getRequired("kafkaBootstrapServers");
        String browseTopic = parameterTool.getRequired("browseTopic");
        String browseTopicGroupID = parameterTool.getRequired("browseTopicGroupID");

        String hbaseZookeeperQuorum = parameterTool.getRequired("hbaseZookeeperQuorum");
        String hbaseZnode = parameterTool.getRequired("hbaseZnode");
        String hbaseTable = parameterTool.getRequired("hbaseTable");

        String mysqlDBUrl = parameterTool.getRequired("mysqlDBUrl");
        String mysqlUser = parameterTool.getRequired("mysqlUser");
        String mysqlPwd = parameterTool.getRequired("mysqlPwd");
        String mysqlTable = parameterTool.getRequired("mysqlTable");

        //2、设置运行环境
        EnvironmentSettings settings = EnvironmentSettings.newInstance().inStreamingMode().useBlinkPlanner().build();
        StreamExecutionEnvironment streamEnv = StreamExecutionEnvironment.getExecutionEnvironment();
        StreamTableEnvironment tableEnv = StreamTableEnvironment.create(streamEnv, settings);
        streamEnv.setParallelism(1);

        //3、注册Kafka数据源
        //自己造的测试数据,某个用户在某个时刻点击了某个商品，以及商品的价值，如下
        //{"userID": "user_1", "eventTime": "2016-01-01 10:02:00", "eventType": "browse", "productID": "product_1", "productPrice": 20}
        Properties browseProperties = new Properties();
        browseProperties.put("bootstrap.servers",kafkaBootstrapServers);
        browseProperties.put("group.id",browseTopicGroupID);
        DataStream<UserBrowseLog> browseStream=streamEnv
                .addSource(new FlinkKafkaConsumer010<>(browseTopic, new SimpleStringSchema(), browseProperties))
                .process(new BrowseKafkaProcessFunction());
        tableEnv.registerDataStream("kafka",browseStream,"userID,eventTime,eventTimeTimestamp,eventType,productID,productPrice");
        //tableEnv.toAppendStream(tableEnv.scan("kafka"),Row.class).print();

        //4、注册HBase数据源(Lookup Table Source)
        Configuration conf = new Configuration();
        conf.set("hbase.zookeeper.quorum", hbaseZookeeperQuorum);
        conf.set("zookeeper.znode.parent",hbaseZnode);
        HBaseTableSource hBaseTableSource = new HBaseTableSource(conf, hbaseTable);
        hBaseTableSource.setRowKey("uid",String.class);
        hBaseTableSource.addColumn("f1","name",String.class);
        hBaseTableSource.addColumn("f1","age",Integer.class);
        tableEnv.registerTableSource("hbase",hBaseTableSource);
        //注册TableFunction
        tableEnv.registerFunction("hbaseLookup", hBaseTableSource.getLookupFunction(new String[]{"uid"}));

        //5、注册Mysql数据源(Lookup Table Source)
        String[] mysqlFieldNames={"pid","productName","productCategory","updatedAt"};
        DataType[] mysqlFieldTypes={DataTypes.STRING(),DataTypes.STRING(),DataTypes.STRING(),DataTypes.STRING()};
        TableSchema mysqlTableSchema = TableSchema.builder().fields(mysqlFieldNames, mysqlFieldTypes).build();
        JDBCOptions jdbcOptions = JDBCOptions.builder()
                .setDriverName("com.mysql.jdbc.Driver")
                .setDBUrl(mysqlDBUrl)
                .setUsername(mysqlUser)
                .setPassword(mysqlPwd)
                .setTableName(mysqlTable)
                .build();

        JDBCLookupOptions jdbcLookupOptions = JDBCLookupOptions.builder()
                .setCacheExpireMs(10 * 1000) //缓存有效期
                .setCacheMaxSize(10) //最大缓存数据条数
                .setMaxRetryTimes(3) //最大重试次数
                .build();

        JDBCTableSource jdbcTableSource = JDBCTableSource.builder()
                .setOptions(jdbcOptions)
                .setLookupOptions(jdbcLookupOptions)
                .setSchema(mysqlTableSchema)
                .build();
        tableEnv.registerTableSource("mysql",jdbcTableSource);
        //注册TableFunction
        tableEnv.registerFunction("mysqlLookup",jdbcTableSource.getLookupFunction(new String[]{"pid"}));


        //6、查询
        //根据userID, 从HBase表中获取用户基础信息
        //根据productID, 从Mysql表中获取产品基础信息
        String sql = ""
                + "SELECT "
                + "       userID, "
                + "       eventTime, "
                + "       eventType, "
                + "       productID, "
                + "       productPrice, "
                + "       f1.name AS userName, "
                + "       f1.age AS userAge, "
                + "       productName, "
                + "       productCategory "
                + "FROM "
                + "     kafka, "
                + "     LATERAL TABLE(hbaseLookup(userID)), "
                + "     LATERAL TABLE (mysqlLookup(productID))";

        tableEnv.toAppendStream(tableEnv.sqlQuery(sql),Row.class).print();

        //7、开始执行
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

}
```

### 结果

```json
//向Kafka Topic中输入测试数据
{"userID": "user_1", "eventTime": "2016-01-01 10:02:00", "eventType": "browse", "productID": "product_1", "productPrice": 20}
{"userID": "user_1", "eventTime": "2016-01-01 10:02:02", "eventType": "browse", "productID": "product_1", "productPrice": 20}
{"userID": "user_1", "eventTime": "2016-01-01 10:02:06", "eventType": "browse", "productID": "product_2", "productPrice": 20}
{"userID": "user_2", "eventTime": "2016-01-01 10:02:10", "eventType": "browse", "productID": "product_2", "productPrice": 20}
{"userID": "user_2", "eventTime": "2016-01-01 10:02:15", "eventType": "browse", "productID": "product_3", "productPrice": 20}

//得到如下结果
//第一行name10,10 是从HBase获取的数据，productName1,productCategory1 是从Mysql获取的数据
//其他行，以此类推
user_1,2016-01-01 10:02:02,browse,product_1,20,name10,10,productName1,productCategory1
user_2,2016-01-01 10:02:15,browse,product_3,20,name2,20,productName3,productCategory3
user_1,2016-01-01 10:02:06,browse,product_2,20,name10,10,productName2,productCategory2
user_1,2016-01-01 10:02:00,browse,product_1,20,name10,10,productName1,productCategory1
user_2,2016-01-01 10:02:10,browse,product_2,20,name2,20,productName2,productCategory2
```

注意: 默认提供的`HBaseTableSource`、`JDBCTableSource`都只实现了同步查找方法，即`getLookupFunction`方法，如有需要，可自行实现`getAsyncLookupFunction`方法。
