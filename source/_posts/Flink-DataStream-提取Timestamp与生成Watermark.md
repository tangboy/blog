---
title: Flink DataStream 提取Timestamp与生成Watermark
date: 2020-02-25 15:00:58
tags:
    - Flink
    - Timestamp
    - Watermark
---

为了基于事件时间来处理每个元素，Flink需要知道每个元素(即事件)的事件时间(Timestamp)。为了衡量事件时间的处理进度，需要指定水印(Watermark)。

本文总结Flink DataStream中提取Timestamp与生成Watermark的两种方式。

## 在Source Function中直接指定Timestamp和生成Watermark

在源端(即SourceFunction)中直接指定Timestamp和生成Watermark。

```java
package com.bigdata.flink.dataStreamExtractTimestampAndGenerateWatermark;

import org.apache.flink.api.java.tuple.Tuple4;
import org.apache.flink.streaming.api.TimeCharacteristic;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.source.SourceFunction;
import org.apache.flink.streaming.api.watermark.Watermark;

import java.util.Random;

/**
 * Summary:
 *    在Source Function中直接指定Timestamp和生成Watermark
 */
public class ExtractTimestampAndGenerateWatermark {
    public static void main(String[] args) throws Exception{

        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        // 设定时间特征为EventTime
        env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);

        // 在源端(即SourceFunction)中直接指定Timestamp和生成Watermark
        DataStreamSource<Tuple4<String, Long, String, Integer>> source = env.addSource(new ExampleSourceFunction());

        env.execute();
    }

    public static class ExampleSourceFunction implements SourceFunction<Tuple4<String,Long,String,Integer>>{

        private volatile boolean isRunning = true;
        private static int maxOutOfOrderness = 10 * 1000;

        private static final String[] userIDSample={"user_1","user_2","user_3"};
        private static final String[] eventTypeSample={"click","browse"};
        private static final int[] productIDSample={1,2,3,4,5};

        @Override
        public void run(SourceContext<Tuple4<String,Long,String,Integer>> ctx) throws Exception {
            while (isRunning){

                // 构造测试数据
                String userID=userIDSample[(new Random()).nextInt(userIDSample.length)];
                long eventTime = System.currentTimeMillis();
                String eventType=eventTypeSample[(new Random()).nextInt(eventTypeSample.length)];
                int productID=productIDSample[(new Random()).nextInt(productIDSample.length)];

                Tuple4<String, Long, String, Integer> record = Tuple4.of(userID, eventTime, eventType, productID);

                // 发出一条数据以及数据对应的Timestamp
                ctx.collectWithTimestamp(record,eventTime);

                // 发出一条Watermark
                ctx.emitWatermark(new Watermark(eventTime - maxOutOfOrderness));

                Thread.sleep(1000);
            }
        }

        @Override
        public void cancel() {
            isRunning = false;
        }
    }
}
```

<!--more-->

## 用Flink自带的TimestampAssigner指定Timestamp和生成Watermark

### 升序时间戳分配器 AscendingTimestampExtractor

```java
package com.bigdata.flink.dataStreamExtractTimestampAndGenerateWatermark;

import org.apache.flink.streaming.api.TimeCharacteristic;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.timestamps.AscendingTimestampExtractor;

/**
 * Summary:
 *    AscendingTimestampExtractor: 适用于时间戳递增的情况
 */
public class AscendingTimestampExtractorUse {
    public static void main(String[] args) throws Exception{

        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        // 设定时间特征为EventTime
        env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);

        // 添加Source
        DataStreamSource<String> source = env.socketTextStream("localhost", 8088);

        // 提取Timestamp与生成Watermark
        source.assignTimestampsAndWatermarks(new AscendingTimestampExtractor<String>(){
            @Override
            public long extractAscendingTimestamp(String element) {
                return Long.valueOf(element.split(",")[1]);
            }
        });

        env.execute();
    }
}
```

### 固定延迟的时间戳分配器 BoundedOutOfOrdernessTimestampExtractor

```java
package com.bigdata.flink.dataStreamExtractTimestampAndGenerateWatermark;

import org.apache.flink.streaming.api.TimeCharacteristic;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.timestamps.BoundedOutOfOrdernessTimestampExtractor;
import org.apache.flink.streaming.api.windowing.time.Time;

/**
 * Summary:
 *    BoundedOutOfOrdernessTimestampExtractor: 适用于乱序但最大延迟已知的情况
 */
public class BoundedOutOfOrdernessUse {
    public static void main(String[] args) throws Exception{

        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        // 设定时间特征为EventTime
        env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);

        // 添加Source
        DataStreamSource<String> source = env.socketTextStream("localhost", 8088);

        // 提取Timestamp与生成Watermark
        // 这里设定固定最大延迟为30秒
        int maxOutOfOrderness = 30;
        source.assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor<String>(Time.seconds(maxOutOfOrderness)) {
            @Override
            public long extractTimestamp(String element) {
                return Long.valueOf(element.split(",")[1]);
            }
        });

        env.execute();
    }
}
```

### 有条件的是水印生成器 AssignerWithPunctuatedWatermarks

```java
package com.bigdata.flink.dataStreamExtractTimestampAndGenerateWatermark;

import org.apache.flink.streaming.api.TimeCharacteristic;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.AssignerWithPunctuatedWatermarks;
import org.apache.flink.streaming.api.watermark.Watermark;

import javax.annotation.Nullable;

/**
 * Summary:
 *      提取时间戳，并基于数据中某个字段的特征判断是否生成水印
 */
public class AssignerWithPunctuatedWatermarksUse {
    public static void main(String[] args) {

        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        // 设定时间特征为EventTime
        env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);

        // 添加Source
        DataStreamSource<String> source = env.socketTextStream("localhost", 8088);

        // 提取Timestamp与生成Watermark
        // 这里，检查每条数据，当数据里某个字段的状态为ending即发射水印
        source.assignTimestampsAndWatermarks(new AssignerWithPunctuatedWatermarks<String>() {

            @Override
            public long extractTimestamp(String element, long previousElementTimestamp) {
                return Long.valueOf(element.split(",")[1]);
            }

            @Nullable
            @Override
            public Watermark checkAndGetNextWatermark(String lastElement, long extractedTimestamp) {
                if((lastElement.split(",")[3]).equals("ending")){
                    return new Watermark(extractedTimestamp);
                }else {
                    return null;
                }
            }
        });
        
    }
}
```

## 提取Timestamp与生成Watermark一般步骤

1. 设置时间特性为Event Time: `StreamExecutionEnvironment#setStreamTimeCharacteristic(TimeCharacteristic.EventTime)`
2. 在Source后Window前用DataStream#`assignTimestampsAndWatermarks`方法(`AssignerWithPeriodicWatermarks`或`AssignerWithPunctuatedWatermarks`)提取时间戳并生成水印
3. 写`extractTimestamp`方法提取Timestamp，重写`getCurrentWatermark`方法或`checkAndGetNextWatermark`方法生成水印