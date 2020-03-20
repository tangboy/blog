---
title: Kafka Connect简介与部署
date: 2020-03-10 18:06:35
tags:
    - Kafka Connect
---

## 什么是Kafka Connect

Kafka 0.9+增加了一个新的特性**Kafka Connect,**可以更方便的创建和管理数据流管道。它为Kafka和其它系统创建规模可扩展的、可信赖的流数据提供了一个简单的模型，通过**connectors**可以将大数据从其它系统导入到Kafka中，也可以从Kafka中导出到其它系统。Kafka Connect可以将完整的数据库注入到Kafka的Topic中，或者将服务器的系统监控指标注入到Kafka，然后像正常的Kafka流处理机制一样进行数据流处理。而导出工作则是将数据从Kafka Topic中导出到其它数据存储系统、查询系统或者离线分析系统等，比如数据库、Elastic Search、Apache Ignite等。

Kafka Connect特性包括：

- Kafka connector通用框架,提供统一的集成API
- 同时支持分布式模式和单机模式
- REST 接口，用来查看和管理Kafka connectors
- 自动化的offset管理，开发人员不必担心错误处理的影响
- 分布式、可扩展
- 流/批处理集成

KafkaCnnect有两个核心概念：Source和Sink。 Source负责导入数据到Kafka，Sink负责从Kafka导出数据，它们都被称为Connector。

{%asset_img Kafka_and_the_age_of_streaming_data_integration_image_sources_sinks.png%}

<!--more-->

## Kafka connect概念

Kafka connect的几个重要的概念包括：connectors、tasks、workers和converters。

- **Connectors**: 通过管理任务来协调数据流的高级抽象
- **Tasks**: 数据写入kafka和数据从kafka读出的实现
- **Workers**: 运行connectors和tasks的进程
- **Converters**: kafka connect和其他存储系统直接发送或者接受数据之间转换数据
- **Transforms**: 用在connect消费或者产生的记录上的简单转换逻辑
- **Dead Letter Queue**: Connect如何处理connector错误

### Connectors

在kafka connect中，connector决定了数据应该从哪里复制过来以及数据应该写入到哪里去，一个connector实例是一个需要负责在kafka和其他系统之间复制数据的逻辑作业，connector plugin是jar文件，实现了kafka定义的一些接口来完成特定的任务。

目前业界已经提供了很多[connector](https://www.confluent.io/hub/)，优先可以使用现有的connector来解决自己的问题，当然，你也可以从头写一个新的connector plugin。可以按照如下流程来开发自己的connector plugin。

开发文档可以参考
1. [developer guide](https://docs.confluent.io/current/connect/devguide.html#connect-devguide)
2. [kafka documentation](https://kafka.apache.org/documentation/#connect)

{%asset_img connector-model-simple.png%}

### Tasks

task是kafka connect数据模型的主角，每一个connector都会协调一系列的task去执行任务，connector可以把一项工作分割成许多的task，然后再把task分发到各个worker中去执行（分布式模式下），task不自己保存自己的状态信息，而是交给特定的kafka 主题去保存（config.storage.topic 和status.storage.topic）。

{%asset_img data-model-simple.png%}

#### Task Rebalancing

在分布式模式下有一个概念叫做任务再平衡（Task Rebalancing），当一个connector第一次提交到集群时，所有的worker都会做一个task rebalancing从而保证每一个worker都运行了差不多数量的工作，而不是所有的工作压力都集中在某个worker进程中，而当某个进程挂了之后也会执行task rebalance。

当一个task fail，但是是由于是一个异常case，那么task rebalancing并不会被触发，这个时候框架并不会自动重启task，需要通过[rest api](https://docs.confluent.io/current/connect/managing/monitoring.html#connect-managing-rest-examples)来重启
{%asset_img task-failover.png%}

### Workers

connectors和tasks都是逻辑工作单位，必须安排在进程中执行，而在kafka connect中，这些进程就是workers，分别有两种worker：standalone和distributed。这里不对standalone进行介绍，具体的可以查看官方文档。我个人觉得distributed worker很棒，因为它提供了可扩展性以及自动容错的功能，你可以使用一个group.ip来启动很多worker进程，在有效的worker进程中它们会自动的去协调执行connector和task，如果你新加了一个worker或者挂了一个worker，其他的worker会检测到然后在重新分配connector和task。

{%asset_img worker-model-basics.png%}

### Converters

converter会把bytes数据转换成kafka connect内部的格式，也可以把kafka connect内部存储格式的数据转变成bytes。

converter对connector来说是解耦的，所以其他的connector都可以重用，例如，使用了avro converter，那么jdbc connector可以写avro格式的数据到kafka，当然，hdfs connector也可以从kafka中读出avro格式的数据。

{%asset_img converter-basics.png%}

### Transforms

connector可以配置一些transformations，用来对于单独一条message做一些轻量级或者简单的逻辑修改。这对于小的数据调整和事件路由都很方便，当然多种transformations可以在connector配置中链式串起来。当然，对于复杂的转换和多条message的处理逻辑还是建议采用[KSQL](https://docs.confluent.io/current/ksql/docs/index.html)或者Kafka Streams。

一个transform就是一个简单的函数，接受一个record作为输入，并输出修改之后的结果。Kafka Connect提供了很多非常有用的transform。当然你可以自己实现一个基于自己的逻辑的Transformation，然后将实现之后的transform打包作为一个Kafka Connect 插件，并在任何connector上使用。

常见的Transform参考[Kafka Connect Transformations](https://docs.confluent.io/current/connect/transforms/index.html#connect-transforms-supported)

### Dead Letter Queue

一个无效的记录发生的原因有很多。一个原因就是在记录到达sink的时候，是采用的JSON序列号，但是sink配置的期待格式是Avro。当一个无效记录不能够被sink connector处理的时候，错误就会基于connector的配置`errors.tolerance`来处理。 这个配置有两个有效的参数，`none(默认值)`或者`all`。 下表给出Connector处于不同状态下，错误是否会被Connect处理

|**Connector State**|**Description**|**Errors Handled by Connect**|
|:-----------------:|:-------------:|:----------------------------:|
|Starting|Can't connect to the datastore at connector startup|No|
|Polling(source connector)|Can't read records from the source database|No|
|Converting data|Can't read from or write to a Kafka topic, or deserialize or serialize JSON,Avro, etc.|Yes|
|Transforming data|Can't apply a single message transform(SMT)|Yes|
|Putting(sink connector)|Can't write records to the target dataset|No|


需要注意的是，Dead letter queues 只适用于sink connector

当`errors.tolerance`设置为`none`， 一个无效的记录会导致connector task立即失败，并且connector会进入到failed state。 为了解决这个问题，你需要去查看Kafka connect Worker的自制，找出导致错误的原因，修改并重启connector

当`errors.tolerance`设置为`all`， 所有无效或者错误的记录都会被忽略，并且connect会正常继续处理。没有任何错误信息会被写到Connect Worker 日志。 所以需要采用其他手段来监控错误的情况，例如采用internal metrics或者记录源和处理之后的记录的条数

#### Creating a Dead Letter Queue Topic

可以在sink配置中，采用以下方式来创建一个dead letter queue

```
errors.tolerance = all
errors.deadletterqueue.topic.name = <dead-letter-topic-name>
```

以下为一个sink配置示例

```json
 {
  "name": "gcs-sink-01",
  "config": {
    "connector.class": "io.confluent.connect.gcs.GcsSinkConnector",
    "tasks.max": "1",
    "topics": "gcs_topic",
    "gcs.bucket.name": "<my-gcs-bucket>",
    "gcs.part.size": "5242880",
    "flush.size": "3",
    "storage.class": "io.confluent.connect.gcs.storage.GcsStorage",
    "format.class": "io.confluent.connect.gcs.format.avro.AvroFormat",
    "partitioner.class": "io.confluent.connect.storage.partitioner.DefaultPartitioner",
    "value.converter": "io.confluent.connect.avro.AvroConverter",
    "value.converter.schema.registry.url": "http://localhost:8081",
    "schema.compatibility": "NONE",
    "confluent.topic.bootstrap.servers": "localhost:9092",
    "confluent.topic.replication.factor": "1",
    "errors.tolerance": "all",
    "errors.deadletterqueue.topic.name": "dlq-gcs-sink-01",
  }
}
```

即使dead letter topic中记录了失败的record，但是也不会显示出为什么失败，可以增加如下配置将失败信息也放到记录的header中

```
errors.deadletterqueue.context.headers.enable = true
```

当这个配置设置为`true`之后，Record Headers会被加入到dead letter queue中。你可以使用kafkacat Utility 来查看header并找出失败原因。

需要注意： 为了避免和原有的record header冲突，dead letter queue的header key是以`_connect.errors`开头

以下是更新之后的配置

```json
 {
  "name": "gcs-sink-01",
  "config": {
    "connector.class": "io.confluent.connect.gcs.GcsSinkConnector",
    "tasks.max": "1",
    "topics": "gcs_topic",
    "gcs.bucket.name": "<my-gcs-bucket>",
    "gcs.part.size": "5242880",
    "flush.size": "3",
    "storage.class": "io.confluent.connect.gcs.storage.GcsStorage",
    "format.class": "io.confluent.connect.gcs.format.avro.AvroFormat",
    "partitioner.class": "io.confluent.connect.storage.partitioner.DefaultPartitioner",
    "value.converter": "io.confluent.connect.avro.AvroConverter",
    "value.converter.schema.registry.url": "http://localhost:8081",
    "schema.compatibility": "NONE",
    "confluent.topic.bootstrap.servers": "localhost:9092",
    "confluent.topic.replication.factor": "1",
    "errors.tolerance": "all",
    "errors.deadletterqueue.topic.name": "dlq-gcs-sink-01",
    "errors.deadletterqueue.context.headers.enable":true
  }
}
```

#### Using a Dead Letter Queue with Security

当kafka开启了security， 那么相应的dead letter queue也要增加配置，如下

```json
admin.ssl.endpoint.identification.algorithm=https
admin.sasl.mechanism=PLAIN
admin.security.protocol=SASL_SSL
admin.request.timeout.ms=20000
admin.retry.backoff.ms=500
admin.sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required \
  username="<user>" \
  password="<secret>";
```



## Kafka connect的启动

Kafka connect的工作模式分为两种，分别是standalone模式和distributed模式。

在独立模式种，所有的work都在一个独立的进程种完成，如果用于生产环境，建议使用分布式模式，都在真的就有点浪费kafka connect提供的容错功能了。

standalone启动的命令很简单，如下：

```bash
bin/connect-standalone.shconfig/connect-standalone.properties connector1.properties[connector2.properties ...]
```

一次可以启动多个connector，只需要在参数中加上connector的配置文件路径即可。

启动distributed模式命令如下：
  

```bash
bin/connect-distributed.shconfig/connect-distributed.properties
```

在connect-distributed.properties的配置文件中，其实并没有配置了你的connector的信息，因为在distributed模式下，启动不需要传递connector的参数，而是通过REST API来对kafka connect进行管理，包括启动、暂停、重启、恢复和查看状态的操作，具体介绍详见下文。

在启动kafkaconnect的distributed模式之前，首先需要创建三个主题，这三个主题的配置分别对应connect-distributed.properties文件中config.storage.topic(default connect-configs)、offset.storage.topic (default connect-offsets) 、status.storage.topic (default connect-status)的配置，那么它们分别有啥用处呢？

- config.storage.topic：用以保存connector和task的配置信息，需要注意的是这个主题的分区数只能是1，而且是有多副本的。（推荐partition 1，replica 3）
- offset.storage.topic:用以保存offset信息。（推荐partition50，replica 3）
- status.storage.topic:用以保存connetor的状态信息。（推荐partition10，replica 3）

```bash
# config.storage.topic=connect-configs
$ bin/kafka-topics --create --zookeeper localhost:2181 --topicconnect-configs --replication-factor 3 --partitions 1
 
# offset.storage.topic=connect-offsets
$ bin/kafka-topics --create --zookeeper localhost:2181 --topicconnect-offsets --replication-factor 3 --partitions 50
 
# status.storage.topic=connect-status
$ bin/kafka-topics --create --zookeeper localhost:2181 --topicconnect-status --replication-factor 3 --partitions 10
```

具体配置可以参考[Kafka官方文档](http://kafka.apache.org/documentation/#connect)

## 通过rest api管理connector

因为kafka connect的意图是以服务的方式去运行，所以它提供了REST API去管理connectors，默认的端口是8083，你也可以在启动kafka connect之前在配置文件中添加rest.port配置。

- **GET /connectors** – 返回所有正在运行的connector名
- **POST /connectors** – 新建一个connector; 请求体必须是json格式并且需要包含name字段和config字段，name是connector的名字，config是json格式，必须包含你的connector的配置信息。
- **GET /connectors/{name}** – 获取指定connetor的信息
- **GET /connectors/{name}/config** – 获取指定connector的配置信息
- **PUT /connectors/{name}/config** – 更新指定connector的配置信息
- **GET /connectors/{name}/status** – 获取指定connector的状态，包括它是否在运行、停止、或者失败，如果发生错误，还会列出错误的具体信息。
- **GET /connectors/{name}/tasks** – 获取指定connector正在运行的task。
- **GET /connectors/{name}/tasks/{taskid}/status** – 获取指定connector的task的状态信息
- **PUT /connectors/{name}/pause** – 暂停connector和它的task，停止数据处理知道它被恢复。
- **PUT /connectors/{name}/resume** – 恢复一个被暂停的connector
- **POST /connectors/{name}/restart** – 重启一个connector，尤其是在一个connector运行失败的情况下比较常用
- **POST /connectors/{name}/tasks/{taskId}/restart** – 重启一个task，一般是因为它运行失败才这样做。
- **DELETE /connectors/{name}** – 删除一个connector，停止它的所有task并删除配置。