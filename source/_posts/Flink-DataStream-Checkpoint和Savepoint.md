---
title: Flink DataStream Checkpoint和Savepoint
date: 2020-02-25 14:26:25
tags:
    - Flink
    - Savepoint
    - Checkpoint
---

针对不同场景，Flink提供了Checkpoint和Savepoint两种容错机制。

本文总结Checkpoint和Savepoint的使用。

## Checkpoint

Checkpoint存储状态数据，由Flink自己定期触发和清除，轻量快速，主要应用于作业生命周期内的故障恢复。

### Checkpoint配置

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

CheckpointConfig checkpointConfig = env.getCheckpointConfig();

# 1、开启Checkpoint
# 默认情况下，不开启Checkpoint
# 设置Checkpoint间隔(单位毫秒)大于0，即开启Checkpoint
# 如果State比较大，建议增大该值
checkpointConfig.setCheckpointInterval(10L * 1000);

# 2、设置Checkpoint状态管理器
# 默认MemoryStateBackend 支持MemoryStateBackend、FsStateBackend、RocksDBStateBackend三种
# MemoryStateBackend: 基于内存的状态管理器，状态存储在JVM堆内存中。一般不应用于生产。
# FsStateBackend: 基于文件系统的状态管理器，文件系统可以是本地文件系统，或者是HDFS分布式文件系统。
# RocksDBStateBackend: 基于RocksDB的状态管理器，需要引入相关依赖才可使用。
# true: 是否异步
env.setStateBackend((StateBackend) new FsStateBackend("CheckpointDir", true));

# 3、设置Checkpoint语义
# EXACTLY_ONCE: 准确一次，结果不丢不重
# AT_LEAST_ONCE: 至少一次，结果可能会重复
# 注意: 如果要实现端到端的准确一次性语义(End-To-End EXACTLY_ONCE)，除了这里设置EXACTLY_ONCE语义外，也需要Source和Sink支持EXACTLY_ONCE
checkpointConfig.setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);

# 4、任务取消后保留Checkpoint目录
checkpointConfig.enableExternalizedCheckpoints(CheckpointConfig.ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION);

# 5、设置要保留的Checkpoint数量
# 在conf/flink-conf.yaml中设置
# 默认是1，只保留最新的一份Checkpoint
# 如果需要从历史某个时刻恢复，这个参数很有用，可以根据Checkpoint间隔，设置成多个
state.checkpoints.num-retained=20

# 6、设置Checkpoint超时时间
# Checkpoint超时时间,默认10分钟。当Checkpoint执行时间超过该值，Flink会丢弃此次Checkpoint并标记为失败，可从Flink WebUI Checkpoints上看到
checkpointConfig.setCheckpointTimeout(long checkpointTimeout);

# 7、设置Checkpoint之间的最小间隔
# 两次Checkpoint之间的最小间隔，默认是0，单位毫秒。State太大，Checkpoint时间太长，而间隔又很短，则会导致大量Checkpoint任务积压，占用大量计算资源，进而影响任务性能
checkpointConfig.setMinPauseBetweenCheckpoints(30000);

# 8、设置同一时间点最多进行Checkpoint的数量，默认是1个
checkpointConfig.setMaxConcurrentCheckpoints(1);
```

<!--more-->
### Checkpoint目录结构

```java
# 以FsStateBackend为例,查看Checkpoint的目录结构

hdfs dfs -get /data/flink/checkpoint/c6b137944167dd0db822a3b961c25b64

tree c6b137944167dd0db822a3b961c25b64/
c6b137944167dd0db822a3b961c25b64/
├── chk-30
│   ├── 1849dc63-05ee-4990-845d-2da9416de805
│   ├── 1e78cdef-fff9-4e19-af0e-c0cc6fc687d3
│   ├── b47f5fd5-fcc1-48d7-9634-f0aa91f03662
│   ├── c7933c2d-fcdb-48a3-bcd2-6d7200ec3c7d
│   ├── c86bd78e-9f9e-4dcc-8111-5b86679f9c9c
│   ├── f3617a4e-c084-4632-8517-1a1d5b09ae08
│   ├── fe39aebb-5c52-49f5-905a-32a8601fd8f5
│   └── _metadata
├── chk-31
│   ├── 2c142723-5758-4286-85ab-72cd7788391d
│   ├── 45e7c99a-63db-4373-9f9b-7f072e3267ac
│   ├── 8cf1b3be-5f40-41a7-acba-f0dbe882f27f
│   ├── b1f8264c-b101-4f19-b22d-6c809a7de439
│   ├── be82e45d-832a-4e46-abb4-d67077fbabd8
│   ├── cdc6e132-794c-44bd-a79d-2bfe72dbfadb
│   ├── da0b4742-d272-45f2-a86a-fdf3aec86e08
│   └── _metadata
├── shared
└── taskowned

# taskowned: TaskManagers拥有的状态
# shared: 共享的状态
# chk-30/chk-31: 存储checkpoint的元数据和数据
# 如: chk-30中的文件
#   chk-30/_metadata: checkpoint的元数据文件,保存着checkpoint数据文件的路径
#   chk-30/1849dc63-05ee-4990-845d-2da9416de805 ...: checkpoint的数据文件，主要存储各种状态，如KafkaConsumer(存储着各分区Offset状态)、如KafkaProducer(存储着事务状态)
```

### 从Checkpoint恢复

任务取消，任务失败，或需要从历史某个时刻重跑数据，即可用fromSavepoint从某个Checkpoint中恢复状态。

#### 以新的YarnApplication从Checkpoint恢复

```bash
applicationName=bigdata_flink

/data/software/flink-1.8.0/bin/flink run \
--jobmanager yarn-cluster \
--yarnname ${applicationName} \
--yarnqueue default \
--yarnjobManagerMemory 1024 \
--yarntaskManagerMemory 1024 \
--yarncontainer 2  \
--yarnslots 2 \
--parallelism 2 \
--fromSavepoint hdfs://bigdata-cluster:8020/data/flink/checkpoint/c6b137944167dd0db822a3b961c25b64/chk-252 \
--class com.bigdata.flink.ReadWriteKafka \
bigData-1.0-SNAPSHOT.jar \
--applicationProperties application.properties
```

#### 在老的YarnApplication中从Checkpoint恢复

```bash
/data/software/flink-1.8.0/bin/flink run \
--yarnapplicationId application_1559561472125_0021 \
--fromSavepoint hdfs://bigdata-cluster:8020/data/flink/checkpoint/c6b137944167dd0db822a3b961c25b64/chk-252 \
# 当不能回滚到某个状态时(如删除了某个Operator),允许忽略状态启动
--allowNonRestoredState \
--class com.bigdata.flink.ReadWriteKafka \
bigData-1.0-SNAPSHOT.jar \
--applicationProperties application.properties
```

## Savepoint

Savepoint是基于Checkpoint机制实现的一种特殊的Checkpoint，可将作业的状态全量快照至外部存储，需手动触发，且不会自动清除，主要应用于作业不同版本间、Flink不同版本间任务恢复。

### 配置Operator ID

默认，Operator ID通过JobGraph和Hash Operator的特定属性值来生成。对JobGraph的更改(如,交换Operator位置、修改并行度、修改代码逻辑等)可能会导致产生新的Operator ID。不利于作业从保存的状态恢复，这时可以手动设置uid来唯一标识Operator ID。

```java
DataStream<String> source = env
                .addSource(kafkaConsumer)
                .name("KafkaSource")
                # 设置Operator ID
                .uid("source-id");
```

### 触发Savepoint

#### 手动触发Savepoint

对于以Yarn Per Job模式运行的Flink作业，可用如下命令触发Savepoint

```bash
命令: bin/flink savepoint --yarnapplicationId yarnAppId flinkJobId savepointTargetDirectory

举例:
bin/flink savepoint --yarnapplicationId application_1559561472125_0020 d904f6adf8a3e94d70465fc717a3f30b hdfs://bigdata-cluster:8020/data/flink/savepoint/
```

#### 取消任务并触发Savepoint

对于以Yarn Per Job模式运行的Flink作业，可用如下命令取消并触发Savepoint

```bash
命令: bin/flink cancel --yarnapplicationId yarnAppId --withSavepoint savepointTargetDirectory flinkJobId

举例:
bin/flink cancel --yarnapplicationId application_1559561472125_0020 --withSavepoint hdfs://bigdata-cluster:8020/data/flink/savepoint/ d904f6adf8a3e94d70465fc717a3f30b
```

### Savepoint目录结构

```bash
hdfs dfs -get hdfs://bigdata-cluster:8020/data/flink/savepoint/savepoint-f793f8-3d4e669257fa/ ./

tree savepoint-f793f8-3d4e669257fa/
savepoint-f793f8-3d4e669257fa/
├── 03ea1c3a-9243-45ba-8613-43911e34e007
├── 380c016c-b0b1-4a92-a3cd-36e7503b0f48
├── 77241d6d-5c34-49b1-97dc-79edeeb217b9
├── 83aa014c-dff8-490e-b132-227fb89c84eb
├── 9696f63c-3d88-4066-9b13-d479dc2ff26e
├── d02828fc-1591-4241-9cda-adba649d1cef
├── da749c45-db28-4a66-8876-67b84c3240e9
└── _metadata

# savepoint-f793f8-3d4e669257fa/: savepoint目录,目录名=savepoint-flinkJobShordId-savepointId
# _metadata: savepoint metadata(元数据),里边以绝对路径保存了各state的位置。当用MemoryStateBackend，里边也同时保存了state数据。
# 03ea1c3a-9243-45ba-8613-43911e34e007 ... : savepoint state
```

### 从Savepoint恢复

#### 以新的YarnApplication从Savepoint恢复

```bash
applicationName=bigdata_flink

/data/software/flink-1.8.0/bin/flink run \
--jobmanager yarn-cluster \
--yarnname ${applicationName} \
--yarnqueue default \
--yarnjobManagerMemory 1024 \
--yarntaskManagerMemory 1024 \
--yarncontainer 2  \
--yarnslots 2 \
--parallelism 2 \
--fromSavepoint hdfs://bigdata-cluster:8020/data/flink/savepoint/savepoint-d904f6-d06500edc454 \
--class com.bigdata.flink.ReadWriteKafka \
bigData-1.0-SNAPSHOT.jar \
--applicationProperties application.properties
```

#### 在老的YarnApplication中从Savepoint恢复

```bash
/data/software/flink-1.8.0/bin/flink run \
--yarnapplicationId application_1559561472125_0021 \
--fromSavepoint hdfs://bigdata-cluster:8020/data/flink/savepoint/savepoint-f793f8-3d4e669257fa \
# 当不能回滚到某个状态时(如删除了某个Operator),允许忽略状态启动
--allowNonRestoredState \
--class com.bigdata.flink.ReadWriteKafka \
bigData-1.0-SNAPSHOT.jar \
--applicationProperties application.properties
```

#### 删除Savepoint

```bash
命令: /data/software/flink-1.8.0/bin/flink savepoint --dispose savepointDirectory

举例: /data/software/flink-1.8.0/bin/flink savepoint --dispose hdfs://bigdata-cluster:8020/data/flink/savepoint/savepoint-f793f8-3d4e669257fa
```