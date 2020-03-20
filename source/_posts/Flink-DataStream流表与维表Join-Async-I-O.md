---
title: Flink DataStream流表与维表Join(Async I/O)
date: 2020-02-24 22:18:10
tags:
    - Flink
    - Async I/O
---

在Flink 流处理过程中，经常需要和外部系统进行交互，如维度补全，用维度表补全事实表中的字段。默认情况下，在MapFunction中，单个并行只能用同步方式去交互: 将请求发送到外部存储，IO阻塞，等待请求返回，然后继续发送下一个请求。这种同步交互的方式往往在网络等待上就耗费了大量时间。为了提高处理效率，可以增加MapFunction的并行度，但增加并行度就意味着更多的资源，并不是一种非常好的解决方式。Flink 在1.2中引入了Async I/O，在异步模式下,将IO操作异步化，单个并行可以连续发送多个请求，哪个请求先返回就先处理，从而在连续的请求间不需要阻塞式等待，大大提高了流处理效率。

{%asset_img 1.jpeg%}

注意:

1. 使用Async I/O，需要外部存储有支持异步请求的客户端。
2. 使用Async I/O，继承`RichAsyncFunction`(接口`AsyncFunction<IN, OUT>`的抽象类)，重写或实现`open`(建立连接)、`close`(关闭连接)、`asyncInvoke`(异步调用)3个方法即可。如下，自定义实现的`ElasticsearchAsyncFunction`类，用于从ES中获取维度数据。
3. 使用Async I/O, 最好结合缓存一起使用，可减少请求外部存储的次数，提高效率。
4. Async I/O 提供了`Timeout`参数来控制请求最长等待时间。默认，异步I/O请求超时时，会引发异常并重启或停止作业。 如果要处理超时，可以重写`AsyncFunction#timeout`方法。
5. Async I/O 提供了`Capacity`参数控制请求并发数，一旦`Capacity`被耗尽，会触发反压机制来抑制上游数据的摄入。
6. Async I/O 输出提供乱序和顺序两种模式。
    1. 乱序, 用`AsyncDataStream.unorderedWait(...)` API，每个并行的输出顺序和输入顺序可能不一致。
    2. 顺序, 用`AsyncDataStream.orderedWait(...)` API，每个并行的输出顺序和输入顺序一致。为保证顺序，需要在输出的Buffer中排序，该方式效率会低一些。
    
<!--more-->

## 用Async I/O 实现流表与维表Join

### 需求背景
实时补全流表中的维度字段。这里，在流表中补全用户的年龄。

### 数据源

1. 流表: 用户行为日志。某个用户在某个时刻点击或浏览了某个商品。自己造的测试数据，数据样例如下:

```json
{"userID": "user_1", "eventTime": "2016-06-06 07:03:42", "eventType": "browse", "productID": 2}
```

2. 维表: 用户基础信息。自己造的测试数据，数据存储在ES上，数据样例如下:

```json
GET dim_user/dim_user/user

{
  "_index": "dim_user",
  "_type": "dim_user",
  "_id": "user_1",
  "_version": 1,
  "found": true,
  "_source": {
    "age": 22
  }
}
```

### 实现逻辑

```java
package com.bigdata.flink;

import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.JSONObject;
import org.apache.flink.api.common.functions.FilterFunction;
import org.apache.flink.api.common.functions.MapFunction;
import org.apache.flink.api.common.serialization.SimpleStringSchema;
import org.apache.flink.api.common.typeinfo.TypeHint;
import org.apache.flink.api.java.tuple.Tuple4;
import org.apache.flink.api.java.tuple.Tuple5;
import org.apache.flink.api.java.utils.ParameterTool;
import org.apache.flink.configuration.*;
import org.apache.flink.streaming.api.datastream.AsyncDataStream;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.connectors.kafka.FlinkKafkaConsumer010;

import java.util.Properties;
import java.util.concurrent.TimeUnit;

/**
 * Author: Wang Pei
 * Summary:
 *     用Async I/O实现流表与维表Join
 */
public class FlinkAsyncIO {
    public static void main(String[] args) throws Exception{

        /**解析命令行参数*/
        ParameterTool parameterTool = ParameterTool.fromArgs(args);
        String kafkaBootstrapServers = parameterTool.get("kafka.bootstrap.servers");
        String kafkaGroupID = parameterTool.get("kafka.group.id");
        String kafkaAutoOffsetReset= parameterTool.get("kafka.auto.offset.reset");
        String kafkaTopic = parameterTool.get("kafka.topic");
        int kafkaParallelism =parameterTool.getInt("kafka.parallelism");

        String esHost= parameterTool.get("es.host");
        Integer esPort= parameterTool.getInt("es.port");
        String esUser = parameterTool.get("es.user");
        String esPassword = parameterTool.get("es.password");
        String esIndex = parameterTool.get("es.index");
        String esType = parameterTool.get("es.type");


        /**Flink DataStream 运行环境*/
        Configuration config = new Configuration();
        config.setInteger(RestOptions.PORT,8081);
        config.setBoolean(ConfigConstants.LOCAL_START_WEBSERVER, true);
        StreamExecutionEnvironment env = StreamExecutionEnvironment.createLocalEnvironmentWithWebUI(config);

        /**添加数据源*/
        Properties kafkaProperties = new Properties();
        kafkaProperties.put("bootstrap.servers",kafkaBootstrapServers);
        kafkaProperties.put("group.id",kafkaGroupID);
        kafkaProperties.put("auto.offset.reset",kafkaAutoOffsetReset);
        FlinkKafkaConsumer010<String> kafkaConsumer = new FlinkKafkaConsumer010<>(kafkaTopic, new SimpleStringSchema(), kafkaProperties);
        kafkaConsumer.setCommitOffsetsOnCheckpoints(true);
        SingleOutputStreamOperator<String> source = env.addSource(kafkaConsumer).name("KafkaSource").setParallelism(kafkaParallelism);

        //数据转换
        SingleOutputStreamOperator<Tuple4<String, String, String, Integer>> sourceMap = source.map((MapFunction<String, Tuple4<String, String, String, Integer>>) value -> {
            Tuple4<String, String, String, Integer> output = new Tuple4<>();
            try {
                JSONObject obj = JSON.parseObject(value);
                output.f0 = obj.getString("userID");
                output.f1 = obj.getString("eventTime");
                output.f2 = obj.getString("eventType");
                output.f3 = obj.getInteger("productID");
            } catch (Exception e) {
                e.printStackTrace();
            }
            return output;
        }).returns(new TypeHint<Tuple4<String, String, String, Integer>>(){}).name("Map: ExtractTransform");

        //过滤掉异常数据
        SingleOutputStreamOperator<Tuple4<String, String, String, Integer>> sourceFilter = sourceMap.filter((FilterFunction<Tuple4<String, String, String, Integer>>) value -> value.f3 != null).name("Filter: FilterExceptionData");

        //Timeout: 超时时间 默认异步I/O请求超时时，会引发异常并重启或停止作业。 如果要处理超时，可以重写AsyncFunction#timeout方法。
        //Capacity: 并发请求数量
        /**Async IO实现流表与维表Join*/
        SingleOutputStreamOperator<Tuple5<String, String, String, Integer, Integer>> result = AsyncDataStream.orderedWait(sourceFilter, new ElasticsearchAsyncFunction(esHost,esPort,esUser,esPassword,esIndex,esType), 500, TimeUnit.MILLISECONDS, 10).name("Join: JoinWithDim");

        /**结果输出*/
        result.print().name("PrintToConsole");

        env.execute();


    }
}
```

### ElasticsearchAsyncFunction

```java
package com.bigdata.flink;

import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.JSONObject;
import com.google.common.cache.Cache;
import com.google.common.cache.CacheBuilder;
import com.google.common.cache.RemovalListener;
import com.google.common.cache.RemovalNotification;
import org.apache.flink.api.java.tuple.Tuple4;
import org.apache.flink.api.java.tuple.Tuple5;
import org.apache.flink.configuration.Configuration;
import org.apache.flink.streaming.api.functions.async.ResultFuture;
import org.apache.flink.streaming.api.functions.async.RichAsyncFunction;
import org.apache.http.HttpHost;
import org.apache.http.auth.AuthScope;
import org.apache.http.auth.UsernamePasswordCredentials;
import org.apache.http.client.CredentialsProvider;
import org.apache.http.impl.client.BasicCredentialsProvider;
import org.elasticsearch.action.ActionListener;
import org.elasticsearch.action.search.SearchRequest;
import org.elasticsearch.action.search.SearchResponse;
import org.elasticsearch.client.RestClient;
import org.elasticsearch.client.RestHighLevelClient;
import org.elasticsearch.index.query.QueryBuilders;
import org.elasticsearch.search.SearchHit;
import org.elasticsearch.search.builder.SearchSourceBuilder;

import java.util.Collections;
import java.util.concurrent.TimeUnit;

/**
 * Author: Wang Pei
 * Summary:
 *      自定义ElasticsearchAsyncFunction，实现从ES中查询维度数据
 */
public class ElasticsearchAsyncFunction extends RichAsyncFunction<Tuple4<String, String, String, Integer>, Tuple5<String, String, String, Integer,Integer>> {


    private String host;

    private Integer port;

    private String user;

    private String password;

    private String index;

    private String type;

    public ElasticsearchAsyncFunction(String host, Integer port, String user, String password, String index, String type) {
        this.host = host;
        this.port = port;
        this.user = user;
        this.password = password;
        this.index = index;
        this.type = type;
    }

    private RestHighLevelClient restHighLevelClient;

    private Cache<String,Integer> cache;

    /**
     * 和ES建立连接
     * @param parameters
     */
    @Override
    public void open(Configuration parameters){

        //ES Client
        CredentialsProvider credentialsProvider = new BasicCredentialsProvider();
        credentialsProvider.setCredentials(AuthScope.ANY, new UsernamePasswordCredentials(user, password));
        restHighLevelClient = new RestHighLevelClient(
                RestClient
                        .builder(new HttpHost(host, port))
                        .setHttpClientConfigCallback(httpAsyncClientBuilder -> httpAsyncClientBuilder.setDefaultCredentialsProvider(credentialsProvider)));

        //初始化缓存
        cache=CacheBuilder.newBuilder().maximumSize(2).expireAfterAccess(5, TimeUnit.MINUTES).build();
    }

    /**
     * 关闭连接
     * @throws Exception
     */
    @Override
    public void close() throws Exception {
        restHighLevelClient.close();
    }


    /**
     * 异步调用
     * @param input
     * @param resultFuture
     */
    @Override
    public void asyncInvoke(Tuple4<String, String, String, Integer> input, ResultFuture<Tuple5<String, String, String, Integer, Integer>> resultFuture) {

        // 1、先从缓存中取
        Integer cachedValue = cache.getIfPresent(input.f0);
        if(cachedValue !=null){
            System.out.println("从缓存中获取到维度数据: key="+input.f0+",value="+cachedValue);
            resultFuture.complete(Collections.singleton(new Tuple5<>(input.f0,input.f1,input.f2,input.f3,cachedValue)));

        // 2、缓存中没有,则从外部存储获取
        }else {
            searchFromES(input,resultFuture);
        }
    }

    /**
     * 当缓存中没有数据时，从外部存储ES中获取
     * @param input
     * @param resultFuture
     */
    private void searchFromES(Tuple4<String, String, String, Integer> input, ResultFuture<Tuple5<String, String, String, Integer, Integer>> resultFuture){

        // 1、构造输出对象
        Tuple5<String, String, String, Integer, Integer> output = new Tuple5<>();
        output.f0=input.f0;
        output.f1=input.f1;
        output.f2=input.f2;
        output.f3=input.f3;

        // 2、待查询的Key
        String dimKey = input.f0;

        // 3、构造Ids Query
        SearchRequest searchRequest = new SearchRequest();
        searchRequest.indices(index);
        searchRequest.types(type);
        searchRequest.source(SearchSourceBuilder.searchSource().query(QueryBuilders.idsQuery().addIds(dimKey)));

        // 4、用异步客户端查询数据
        restHighLevelClient.searchAsync(searchRequest, new ActionListener<SearchResponse>() {

            //成功响应时处理
            @Override
            public void onResponse(SearchResponse searchResponse) {
                SearchHit[] searchHits = searchResponse.getHits().getHits();
                if(searchHits.length >0 ){
                    JSONObject obj = JSON.parseObject(searchHits[0].getSourceAsString());
                    Integer dimValue=obj.getInteger("age");
                    output.f4=dimValue;
                    cache.put(dimKey,dimValue);
                    System.out.println("将维度数据放入缓存: key="+dimKey+",value="+dimValue);
                }

                resultFuture.complete(Collections.singleton(output));
            }

            //响应失败时处理
            @Override
            public void onFailure(Exception e) {
                output.f4=null;
                resultFuture.complete(Collections.singleton(output));
            }
        });

    }

    //超时时处理
    @Override
    public void timeout(Tuple4<String, String, String, Integer> input, ResultFuture<Tuple5<String, String, String, Integer, Integer>> resultFuture) {
        searchFromES(input,resultFuture);
    }
}
```