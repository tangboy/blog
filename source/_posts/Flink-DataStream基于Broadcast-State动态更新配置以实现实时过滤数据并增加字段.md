---
title: Flink DataStream基于Broadcast State动态更新配置以实现实时过滤数据并增加字段
date: 2020-02-25 11:31:56
tags:
    - Flink
    - Broadcast State
---

Flink从1.5.0开始支持广播状态(`Broadcast State`)。广播状态可以用来解决如下问题:
一条流需要根据规则或配置处理数据，而规则或配置又是随时变化的。此时，就可将规则或配置作为广播流广播出去，并以`Broadcast State`的形式存储在下游Task中。下游Task根据`Broadcast State`中的规则或配置来处理常规流中的数据。

场景举例:

1. 动态更新计算规则: 如事件流需要根据最新的规则进行计算，则可将规则作为广播状态广播到下游Task中。
2. 实时增加额外字段: 如事件流需要实时增加用户的基础信息，则可将用户的基础信息作为广播状态广播到下游Task中。

注意:

1. `Broadcast State`是Map类型，即K-V类型。
2. `Broadcast State`只有在广播的一侧,即在`BroadcastProcessFunction`或`KeyedBroadcastProcessFunction`的`processBroadcastElement`方法中可以修改。在非广播的一侧，即在`BroadcastProcessFunction`或`KeyedBroadcastProcessFunction`的`processElement`方法中只读。
3. `Broadcast State`中元素的顺序，在各Task中可能不同。基于顺序的处理，需要注意。
4. `Broadcast State`在`Checkpoint`时，每个Task都会`Checkpoint`广播状态。
5. `Broadcast State`在运行时保存在内存中，目前还不能保存在RocksDB State Backend中。

<!--more-->

## 基于Broadcast State 动态更新配置以实现实时过滤数据并增加字段

### 需求背景

假设有这样的一个需求，需要实时过滤出配置中的用户，并在事件流中补全这批用户的基础信息。

1. 事件: 在Kafka中，自己造的数据，格式如下。

```json
# 某个用户在某个时刻浏览或点击了某个商品
{"userID": "user_3", "eventTime": "2019-08-17 12:19:47", "eventType": "browse", "productID": 1}
{"userID": "user_2", "eventTime": "2019-08-17 12:19:48", "eventType": "click", "productID": 1}
```

配置: 在Mysql中，自己造的数据，表结构如下。

```sql
# 用户的基础信息
+----------+-------------+------+-----+---------+-------+
| Field    | Type        | Null | Key | Default | Extra |
+----------+-------------+------+-----+---------+-------+
| userID   | varchar(20) | NO   | PRI | NULL    |       |
| userName | varchar(10) | YES  |     | NULL    |       |
| userAge  | int(11)     | YES  |     | NULL    |       |
+----------+-------------+------+-----+---------+-------+
```

### 部分依赖

```xml
<!--Kafka连接器-->
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-connector-kafka-0.10_2.11</artifactId>
    <version>1.8.0</version>
</dependency>

<!--Mysql JDBC Driver-->
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>5.1.47</version>
</dependency>
```

### 代码实现

```java
package com.bigdata.flink;

import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.JSONObject;
import lombok.extern.slf4j.Slf4j;
import org.apache.flink.api.common.serialization.SimpleStringSchema;
import org.apache.flink.api.common.state.BroadcastState;
import org.apache.flink.api.common.state.MapStateDescriptor;
import org.apache.flink.api.common.state.ReadOnlyBroadcastState;
import org.apache.flink.api.common.typeinfo.Types;
import org.apache.flink.api.java.tuple.*;
import org.apache.flink.api.java.utils.ParameterTool;
import org.apache.flink.configuration.Configuration;
import org.apache.flink.runtime.state.StateBackend;
import org.apache.flink.runtime.state.filesystem.FsStateBackend;
import org.apache.flink.shaded.guava18.com.google.common.collect.Maps;
import org.apache.flink.streaming.api.CheckpointingMode;
import org.apache.flink.streaming.api.datastream.*;
import org.apache.flink.streaming.api.environment.CheckpointConfig;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.ProcessFunction;
import org.apache.flink.streaming.api.functions.co.BroadcastProcessFunction;
import org.apache.flink.streaming.api.functions.source.RichSourceFunction;
import org.apache.flink.streaming.connectors.kafka.FlinkKafkaConsumer010;
import org.apache.flink.util.Collector;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.util.HashMap;
import java.util.Map;
import java.util.Properties;

/**
 * Author: Wang Pei
 * Summary:
 *   基于Broadcast State 动态更新配置以实现实时过滤数据并增加字段
 */
@Slf4j
public class TestBroadcastState {
    public static void main(String[] args) throws Exception{

        //1、解析命令行参数
        ParameterTool fromArgs = ParameterTool.fromArgs(args);
        ParameterTool parameterTool = ParameterTool.fromPropertiesFile(fromArgs.getRequired("applicationProperties"));

        //checkpoint配置
        String checkpointDirectory = parameterTool.getRequired("checkpointDirectory");
        long checkpointSecondInterval = parameterTool.getLong("checkpointSecondInterval");

        //事件流配置
        String fromKafkaBootstrapServers = parameterTool.getRequired("fromKafka.bootstrap.servers");
        String fromKafkaGroupID = parameterTool.getRequired("fromKafka.group.id");
        String fromKafkaTopic = parameterTool.getRequired("fromKafka.topic");

        //配置流配置
        String fromMysqlHost = parameterTool.getRequired("fromMysql.host");
        int fromMysqlPort = parameterTool.getInt("fromMysql.port");
        String fromMysqlDB = parameterTool.getRequired("fromMysql.db");
        String fromMysqlUser = parameterTool.getRequired("fromMysql.user");
        String fromMysqlPasswd = parameterTool.getRequired("fromMysql.passwd");
        int fromMysqlSecondInterval = parameterTool.getInt("fromMysql.secondInterval");

        //2、配置运行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        //设置StateBackend
        env.setStateBackend((StateBackend) new FsStateBackend(checkpointDirectory, true));
        //设置Checkpoint
        CheckpointConfig checkpointConfig = env.getCheckpointConfig();
        checkpointConfig.setCheckpointInterval(checkpointSecondInterval * 1000);
        checkpointConfig.setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);
        checkpointConfig.enableExternalizedCheckpoints(CheckpointConfig.ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION);

        //3、Kafka事件流
        //从Kafka中获取事件数据
        //数据：某个用户在某个时刻浏览或点击了某个商品,如
        //{"userID": "user_3", "eventTime": "2019-08-17 12:19:47", "eventType": "browse", "productID": 1}
        Properties kafkaProperties = new Properties();
        kafkaProperties.put("bootstrap.servers",fromKafkaBootstrapServers);
        kafkaProperties.put("group.id",fromKafkaGroupID);

        FlinkKafkaConsumer010<String> kafkaConsumer = new FlinkKafkaConsumer010<>(fromKafkaTopic, new SimpleStringSchema(), kafkaProperties);
        kafkaConsumer.setStartFromLatest();
        DataStream<String> kafkaSource = env.addSource(kafkaConsumer).name("KafkaSource").uid("source-id-kafka-source");

        SingleOutputStreamOperator<Tuple4<String, String, String, Integer>> eventStream = kafkaSource.process(new ProcessFunction<String, Tuple4<String, String, String, Integer>>() {
            @Override
            public void processElement(String value, Context ctx, Collector<Tuple4<String, String, String, Integer>> out){
                try {
                    JSONObject obj = JSON.parseObject(value);
                    String userID = obj.getString("userID");
                    String eventTime = obj.getString("eventTime");
                    String eventType = obj.getString("eventType");
                    int productID = obj.getIntValue("productID");
                    out.collect(new Tuple4<>(userID, eventTime, eventType, productID));
                }catch (Exception ex){
                    log.warn("异常数据:{}",value,ex);
                }
            }
        });

        //4、Mysql配置流
        //自定义Mysql Source，周期性地从Mysql中获取配置，并广播出去
        //数据: 用户ID,用户姓名，用户年龄
        DataStreamSource<HashMap<String, Tuple2<String, Integer>>> configStream = env.addSource(new MysqlSource(fromMysqlHost, fromMysqlPort, fromMysqlDB, fromMysqlUser, fromMysqlPasswd, fromMysqlSecondInterval));

        /*
          (1) 先建立MapStateDescriptor
          MapStateDescriptor定义了状态的名称、Key和Value的类型。
          这里，MapStateDescriptor中，key是Void类型，value是Map<String, Tuple2<String,Int>>类型。
         */
        MapStateDescriptor<Void, Map<String, Tuple2<String,Integer>>> configDescriptor = new MapStateDescriptor<>("config", Types.VOID, Types.MAP(Types.STRING, Types.TUPLE(Types.STRING, Types.INT)));

        /*
          (2) 将配置流广播，形成BroadcastStream
         */
        BroadcastStream<HashMap<String, Tuple2<String, Integer>>> broadcastConfigStream = configStream.broadcast(configDescriptor);

        //5、事件流和广播的配置流连接，形成BroadcastConnectedStream
        BroadcastConnectedStream<Tuple4<String, String, String, Integer>, HashMap<String, Tuple2<String, Integer>>> connectedStream = eventStream.connect(broadcastConfigStream);

        //6、对BroadcastConnectedStream应用process方法，根据配置(规则)处理事件
        SingleOutputStreamOperator<Tuple6<String, String, String, Integer, String, Integer>> resultStream = connectedStream.process(new CustomBroadcastProcessFunction());

        //7、输出结果
        resultStream.print();

        //8、生成JobGraph，并开始执行
        env.execute();

    }

    /**
     * 自定义BroadcastProcessFunction
     * 当事件流中的用户ID在配置中出现时，才对该事件处理, 并在事件中补全用户的基础信息
     * Tuple4<String, String, String, Integer>: 第一个流(事件流)的数据类型
     * HashMap<String, Tuple2<String, Integer>>: 第二个流(配置流)的数据类型
     * Tuple6<String, String, String, Integer,String, Integer>: 返回的数据类型
     */
    static class CustomBroadcastProcessFunction extends BroadcastProcessFunction<Tuple4<String, String, String, Integer>, HashMap<String, Tuple2<String, Integer>>, Tuple6<String, String, String, Integer, String, Integer>>{

        /**定义MapStateDescriptor*/
        MapStateDescriptor<Void, Map<String, Tuple2<String,Integer>>> configDescriptor = new MapStateDescriptor<>("config", Types.VOID, Types.MAP(Types.STRING, Types.TUPLE(Types.STRING, Types.INT)));

        /**
         * 读取状态，并基于状态，处理事件流中的数据
         * 在这里，从上下文中获取状态，基于获取的状态，对事件流中的数据进行处理
         * @param value 事件流中的数据
         * @param ctx 上下文
         * @param out 输出零条或多条数据
         * @throws Exception
         */
        @Override
        public void processElement(Tuple4<String, String, String, Integer> value, ReadOnlyContext ctx, Collector<Tuple6<String, String, String, Integer, String, Integer>> out) throws Exception {

            //事件流中的用户ID
            String userID = value.f0;

            //获取状态
            ReadOnlyBroadcastState<Void, Map<String, Tuple2<String, Integer>>> broadcastState = ctx.getBroadcastState(configDescriptor);
            Map<String, Tuple2<String, Integer>> broadcastStateUserInfo = broadcastState.get(null);

            //配置中有此用户，则在该事件中添加用户的userName、userAge字段。
            //配置中没有此用户，则丢弃
            Tuple2<String, Integer> userInfo = broadcastStateUserInfo.get(userID);
            if(userInfo!=null){
                out.collect(new Tuple6<>(value.f0,value.f1,value.f2,value.f3,userInfo.f0,userInfo.f1));
            }

        }

        /**
         * 处理广播流中的每一条数据，并更新状态
         * @param value 广播流中的数据
         * @param ctx 上下文
         * @param out 输出零条或多条数据
         * @throws Exception
         */
        @Override
        public void processBroadcastElement(HashMap<String, Tuple2<String, Integer>> value, Context ctx, Collector<Tuple6<String, String, String, Integer, String, Integer>> out) throws Exception {

            //获取状态
            BroadcastState<Void, Map<String, Tuple2<String, Integer>>> broadcastState = ctx.getBroadcastState(configDescriptor);

            //清空状态
            broadcastState.clear();

            //更新状态
            broadcastState.put(null,value);

        }
    }



    /**
     * 自定义Mysql Source，每隔 secondInterval 秒从Mysql中获取一次配置
     */
    static class MysqlSource extends RichSourceFunction<HashMap<String, Tuple2<String, Integer>>> {

        private String host;
        private Integer port;
        private String db;
        private String user;
        private String passwd;
        private Integer secondInterval;

        private volatile boolean isRunning = true;

        private Connection connection;
        private PreparedStatement preparedStatement;

        MysqlSource(String host, Integer port, String db, String user, String passwd,Integer secondInterval) {
            this.host = host;
            this.port = port;
            this.db = db;
            this.user = user;
            this.passwd = passwd;
            this.secondInterval = secondInterval;
        }

        /**
         * 开始时, 在open()方法中建立连接
         * @param parameters
         * @throws Exception
         */
        @Override
        public void open(Configuration parameters) throws Exception {
            super.open(parameters);
            Class.forName("com.mysql.jdbc.Driver");
            connection= DriverManager.getConnection("jdbc:mysql://"+host+":"+port+"/"+db+"?useUnicode=true&characterEncoding=UTF-8", user, passwd);
            String sql="select userID,userName,userAge from user_info";
            preparedStatement=connection.prepareStatement(sql);
        }

        /**
         * 执行完，调用close()方法关系连接，释放资源
         * @throws Exception
         */
        @Override
        public void close() throws Exception {
            super.close();

            if(connection!=null){
                connection.close();
            }

            if(preparedStatement !=null){
                preparedStatement.close();
            }
        }

        /**
         * 调用run()方法获取数据
         * @param ctx
         */
        @Override
        public void run(SourceContext<HashMap<String, Tuple2<String, Integer>>> ctx) {
            try {
                while (isRunning){
                    HashMap<String, Tuple2<String, Integer>> output = new HashMap<>();
                    ResultSet resultSet = preparedStatement.executeQuery();
                    while (resultSet.next()){
                        String userID = resultSet.getString("userID");
                        String userName = resultSet.getString("userName");
                        int userAge = resultSet.getInt("userAge");
                        output.put(userID,new Tuple2<>(userName,userAge));
                    }
                    ctx.collect(output);
                    //每隔多少秒执行一次查询
                    Thread.sleep(1000*secondInterval);
                }
            }catch (Exception ex){
                log.error("从Mysql获取配置异常...",ex);
            }


        }

        /**
         * 取消时，会调用此方法
         */
        @Override
        public void cancel() {
            isRunning = false;
        }
    }
}
```

### 运行结果

```json
//修改配置前
(user_5,2019-08-18 08:17:19,browse,3,user_name5,25)
(user_3,2019-08-18 08:17:19,click,1,user_name3,23)
(user_2,2019-08-18 08:17:20,click,3,user_name2,22)
(user_5,2019-08-18 08:17:20,browse,1,user_name5,25)

//Flink 应用不停，更新配置(用户基础信息)，这里更新user_2、user_5的年龄
//可以看到，动态修改的配置已生效
(user_5,2019-08-18 08:19:51,click,3,user_name5,15)
(user_3,2019-08-18 08:19:52,browse,1,user_name3,23)
(user_4,2019-08-18 08:19:52,browse,1,user_name4,24)
(user_2,2019-08-18 08:19:53,click,3,user_name2,12)
(user_3,2019-08-18 08:19:53,click,3,user_name3,23)

//Flink 应用不停，删除配置(删除部分用户)，这里只保留user_2
//可以看到，动态修改的配置已生效
(user_2,2019-08-18 08:23:14,click,1,user_name2,12)
(user_2,2019-08-18 08:23:16,click,1,user_name2,12)
(user_2,2019-08-18 08:23:20,browse,2,user_name2,12)
```