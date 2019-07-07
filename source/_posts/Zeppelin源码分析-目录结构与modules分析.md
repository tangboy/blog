---
title: Zeppelin源码分析---目录结构与modules分析
date: 2019-05-27 16:40:50
tags:
    - zeppelin
    - spark
    - java
---


[Zeppelin](https://zeppelin.apache.org)，于2016-5-18日从Apache孵化器项目毕业成为Apache顶级项目，采用Java（主要）+Scala+R+PythonR+Bash+JS混合开发，采用maven作为build工具。涉及的主要技术stack如下：

1. **前台** : AngularJS、Node.JS、WebSocket、Grunt、Bower、Highlight.js、BootStrap3js
2. **后台** : Jetty（embedding）、Thrift、Shiro（权限）、Apache common-exec、Jersey REST API

zeppelin本质上是一个web应用程序，它以独立的jvm进程的方式来启动Interpreter（解释器），交互式（repl）执行各种语言的代码片段，并将结果以html代码片段的形式返回给前端UI。

<!--more-->

## distribution结构分析

编译完成之后，会在zeppelin-distribution/target/目录下生成如下结构的分发包：

{%asset_img 1.png %}

### bin

bin目录存储了zeppelin的启停控制脚本：

{%asset_img 2.png%}

各个脚本作用如下：

|脚本|作用|
|:--:|:---:|
|zeppelin-daemon.sh|提供以daemon形式启停org.apache.zeppelin.server.ZeppelinServer服务，并调用common.sh和function.sh设置env和classpath|
|zeppelin.sh|以foreground的形式启动ZeppelinServer|
|Interpreter.sh|采用单独进程启动org.apache.zeppelin.interpreter.remote.RemoteInterpreterServer服务，该脚本不会被其他脚本直接调用，实际是通过apache common-exec来调用的|
|common.sh|设置zeppelin运行时需要的env和classpath，如果${ZEPPELIN_HOME}/conf/目录中存在zeppelin-env.sh，则会调用该脚本|
|function.sh|一些公共基础函数|

### conf

{%asset_img 3.png%}

|配置文件|作用|
|:--:|:--:|
|shiro.ini|供apache shiro框架使用的权限控制文件|
|zeppelin-env.sh|供${ZEPPELIN_HOME}/bin/common.sh脚本调用，设置诸如：SPARK_HOME、HADOOP_CONF_DIR等zeppelin与外围环境集成的环境变量|
|zeppelin-site.xml.template|zeppelin的配置模板，被注释掉的property是zeppelin的默认配置，可以rename成zeppelin-site.xml然后根据需要override|

### interpreter

{%asset_img 4.png%}

每个针对具体的某种语言实现的Interpreter，存放的是编译好的jar包,以及该interpreter的依赖，其中spark子目录的结构如下：

{%asset_img 5.png%}

需要注意的是：如果定义了SPARK_HOME环境变量，该目录下的zeppelin-spark*.jar脚本将通过${SPARK_HOME}/bin/spark-submit到spark集群上去执行。

详情参见${ZEPPELIN_HOME}/bin/interpreter.sh脚本。

### notebook

{%asset_img 6.png%}

该目录是默认的notebook的持久化存储目录，zeppelin默认的notebook持久化实现类是org.apache.zeppelin.notebook.repo.VFSNotebookRepo，该实现会以zeppelin.notebook.dir配置参数指定的路径来存储notebook（默认是json格式）

```xml
<property>
  <name>zeppelin.notebook.dir</name>
  <value>notebook</value>
  <description>path or URI for notebook persist</description>
</property>
```

由于该value指定的uri不带scheme，默认会在${ZEPPELIN_HOME}目录下创建notebook子目录用于存储各个notebook，以notebook的id为子目录名字。


### 源码结构分析

###  module组成

zeppelin的maven项目共有26个module，其中7个框架相关的module如下：

|Module|作用|开发语言|
|:---:|:---:|:---:|
|zeppelin-server|项目的主入口，通过内嵌Jetty的方式提供Web UI和REST服务，并且提供了基本的权限验证服务|java|
|zeppelin-zengine|实现Notebook的持久化、检索实现interpreter的自动加载，以及maven式的依赖自动加载|java|
|zeppelin-interpreter|为了支持多语言Notebook，抽象出了每种语言都要事先的Interpreter接口，包括：显示、调度、依赖以及和zeppelin-engine之间的Thirft通信协议|java|
|zeppelin-web|AngluarJS开发的web页面|Javascript（主要是AngularJS、Node.JS以及使用websocket）|
|zeppelin-display|实现向前台Angular元素绑定后台数据|scala|
|zeppelin-spark-dependencies|无具体功能，maven的最佳实践，将多个module都要依赖的公共类单独抽离出来，供其他module依赖。目前zeppelin-zinterpreter和zeppelin-spark这2个module依赖它||
|zeppelin-distribution|为了将整个项目打包成发布版，而设置的module，打包格式见src/assembly/distribution.xml||

zeppelin的框架部分代码主要是以上几个module，余下19个module全部是为了支持各种语言的interpreter实现的module，如下：

|Module|作用|开发语言|
|zeppelin-spark|实现Spark、PySpark、SparkR、SparkSql等Interpreter|java+scala|
|zeppelin-zinterpreter|实现R Interpreter (PS:需要在${ZEPPELIN_HOME}/pom.xml中增加<module>r</module>)|R+java+scala|
|zeppelin-hive|实现HiveQL Interpreter，需要hive client|java|
|zeppelin-hbase|实现HBase Interpreter，需要hbase client，实际使用ruby脚本${HBASE_HOME}/bin/hirb.rb，来执行UI传入的脚本|java|
|zeppelin-shell|实现shell interpreter|java|
|zeppelin-file|使用WebHDFS实现hdfs fs命令|java|
|zeppelin-postgresql|实现带自动补全功能的psql interpreter，使用jline实现自动补全|java|
|zeppelin-jdbc|实现通用的jdbc sql Interpreter|java|
|zeppelin-markdown|实现MarkDown Interpreter|java|
|zeppelin-phoenix|实现phoenixInterpreter|java|
|zeppelin-elasticsearch|实现ES interpreter|java|
|zeppelin-canssandra|实现canssandra Interpreter|java+scala|
|zeppelin-kylin|实现支持kylin OLAP引擎的interpreter|java|
|zeppelin-angular|实现支持angularJS引擎|java|
|zeppelin-alluxio|实现alluxio Interpreter，原tacyon|java|
|zeppelin-lens|实现lens interpreter|java|
|zeppelin-ignite|实现ignite Interpreter|java|
|zeppelin-flink|实现flink interpreter|java|
|zeppelin-tajo|实现tajo Interpreter|java|


### module间依赖关系

Module之间的依赖关系如下：

{%asset_img 7.png%}


