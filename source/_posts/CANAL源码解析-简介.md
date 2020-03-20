---
title: CANAL源码解析-简介
date: 2020-02-06 11:28:32
tags:
    - canal
---

canal是阿里巴巴开源的mysql数据库binlog的增量订阅&消费组件。项目github地址为：https://github.com/alibaba/canal。

# 源码模块划分

canal是基于maven构建的，总共分成了14个模块，如下所示：

{%asset_img 1.png %}

<!--more-->

模块虽多，但是每个模块的代码都很少。各个模块的作用如下所示：

- **common模块**：主要是提供了一些公共的工具类和接口。
- **client模块**：canal的客户端。核心接口为CanalConnector
- **example模块**：提供client模块使用案例。
- **protocol模块**：client和server模块之间的通信协议
- **deployer模块**：部署模块。通过该模块提供的CanalLauncher来启动canal server
- **server模块**：canal服务器端。核心接口为CanalServer
- **instance模块**：一个server有多个instance。每个instance都会模拟成一个mysql实例的slave。instance模块有四个核心组成部分：- - **parser模块、sink模块、store模块，meta模块**。核心接口为CanalInstance
- **parser模块**：数据源接入，模拟slave协议和master进行交互，协议解析。parser模块依赖于dbsync、driver模块。
- **driver模块和dbsync模块**：从这两个模块的artifactId(canal.parse.driver、canal.parse.dbsync)，就可以看出来，这两个模块实际上是parser模块的组件。事实上parser 是通过driver模块与mysql建立连接，从而获取到binlog。由于原始的binlog都是二进制流，需要解析成对应的binlog事件，这些binlog事件对象都定义在dbsync模块中，dbsync 模块来自于淘宝的tddl。
- **sink模块**：parser和store链接器，进行数据过滤，加工，分发的工作。核心接口为CanalEventSink
- **store模块**：数据存储。核心接口为CanalEventStore
- **meta模块**：增量订阅&消费信息管理器，核心接口为CanalMetaManager，主要用于记录canal消费到的mysql binlog的位置，

下面再通过一张图来说明各个模块之间的依赖关系：
{%asset_img 2.png%}

通过deployer模块，启动一个canal-server，一个cannal-server内部包含多个instance，每个instance都会伪装成一个mysql实例的slave。client与server之间的通信协议由protocol模块定义。client在订阅binlog信息时，需要传递一个destination参数，server会根据这个destination确定由哪一个instance为其提供服务。