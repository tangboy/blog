---
title: Log Compacted Topics in Apache Kafka
date: 2020-03-10 15:19:14
tags:
    - Kafka
    - Log Compacted Topics
---


当我开始阅读Kafka官方文档时，尽管看起来Log Compacted Topics是一个简单的概念，但是我并不是很清楚Kafka内部是如何在文件系统中保存他们的状态。通过详细了解这个feature相关的文档，形成这边文档。

在这篇文档中，我会首先描述介绍kafka的log compacted Topics，然后我会介绍Topics是一个简单的概念，但是我并不是很清楚Kafka内部是如何在文件系统中保存他们的状态。

<!--more-->

## 前提

我假设你已经对应Kafka的基本概念例如，broker、topic、partition、consumer以及producer了解了。而且你本地已经安装了kafka以及zookeeper，且会通过命令行操作kafka

## Log Compacted Topics是什么？

Kafka官方文档的说法为
>> Log compaction is a mechanism to give finer-grained per-record retention, rather than the coarser-grained time-based retention. The idea is to selectively remove records where we have a more recent update with the same primary key. This way the log is guaranteed to have at least the last state for each key.

我们简化上面的说法，其实就是，当在一个partition log中，相同的key有新的记录的时候，Kafka会自动将该key老的记录删除。考虑一下例子，下面一个partition log是一个log compacted topic 名字为latest-product-price:

{%asset_img 1.jpeg%}

如你所见，最开始key p3有两个记录。但是由于这是log compacted topic，因此Kafka会用后台线程(下文会详细介绍)移除老的记录。 现在假设，我们有一个producer并会向这个partition发送新的记录，这个producer分布发送了key为p6、p5、p5三个记录，如下图：

{%asset_img 2.jpeg%}

同样，Kafka Broker内部的后台线程会自动将key为p5和p6的老的记录删除。compacted log是有两部分组成： tail和head. Kafka会保证所有在tail部分的记录的key是唯一的，因为这些数据是在清理线程处理之后的结果，而head部分可能会有多个值。

现在我们需要学习如何通过命令行工具kafka-topics创建一个log compacted topic

## Create a Log Compacted Topic

创建一个compacted topic(我会介绍详细配置)

```bash
kafka-topics --create --zookeeper zookeeper:2181 --topic latest-product-price --replication-factor 1 --partitions 1 --config "cleanup.policy=compact" --config "delete.retention.ms=100"  --config "segment.ms=100" --config "min.cleanable.dirty.ratio=0.01"
```

产生一些记录

```bash
kafka-console-producer --broker-list localhost:9092 --topic latest-product-price --property parse.key=true --property key.separator=:
>p3:10$
>p5:7$
>p3:11$
>p6:25$
>p6:12$
>p5:14$
>p5:17$
```

我们注意到，上面的命令key和value的分隔符采用的是:, 现在起一个consumer命令来消费这个topic

```bash
kafka-console-consumer --bootstrap-server localhost:9092 --topic latest-product-price --property  print.key=true --property key.separator=: --from-beginning
p3:11$
p6:12$
p5:14$
p5:17$
```


我们可以看到，一些重复的key已经被删除了。但是p5:14$并没有被删除，这个我在介绍清理进程的时候会介绍。现在我们需要先看一下kafka内部是如何存储消息的。

## Segments

Partition Log事实上是一种抽象概念，目的是方便我们按照顺序消费一个partition中的消息，而不用担心Kafka内部是如何管理和存储的。
但是，在实际上，Kafka Broker会将partition log按照需要分割成不同的segments。 Segments都是存储在文件系统中的文件(在data 目录下，并在各个partition下)，并且他们的名字都是以.log结尾。下图是一个partition log被分成三个segments

{%asset_img 3.jpeg%}

如上图可见，一个partition log 存储了7个日志分别处于三个独立的segment file。 一个segment的第一个offset被称为这个segment的base offset。 而segment文件的名称一直都是和base offset相同。

一个partition中的最后一个segment被称为active segment。 只有active segment能够接收新产生的消息。我们将会看到Kafka是如何通过active segment来完成一个compacted log的清理进程的。

回到我们的例子，我们可以通过以下命令产看我们topic partition的segments(假如Kafka的数据存储在`/var/lib/kafka/data`)：

```bash
ls /var/lib/kafka/data/latest-product-price-0/
00000000000000000000.index 00000000000000000006.log
00000000000000000000.log 00000000000000000006.snapshot
00000000000000000000.timeindex 00000000000000000006.timeindex
00000000000000000005.snapshot leader-epoch-checkpoint
00000000000000000006.index
```

其中`00000000000000000000.log`和`00000000000000000006.log`都是这个partition的segment，而`00000000000000000006.log`是这个partition的active segment

什么时候Kafka会创建一个新的segment？ 一个控制选项是在创建topic的时候通过设置`segment.bytes`(默认为1GB)来完成。当你的segment size大于这个配置值，Kafka会自动创建一个新的segment。另外一个方式是通过设置`segment.ms`， 这个选项是通过检查active segment的时间是否老于`segment.ms`。 如果是，那么kafka会自动创建一个新的segment。在我们上述的命令中，设置`segment.ms=100`，这样可以保证每100ms会创建一个新的segment。

比较有意思的点在于，当你设置`segment.ms=100`的时候，你可能想要比较小的segment。当在清理进程结束的时候，Kafka broker会将非活跃的segment合并，并生成一个大的segment。

如果你想更加深入的了解segment以及Kafka相关的内部存储支持，可以参考[How Kafka's Storage Internal Work](https://thehoard.blog/how-kafkas-storage-internals-work-3a29b02e026) 和[A Practical Introduction to Kafka Storage Internals](https://medium.com/@durgaswaroop/a-practical-introduction-to-kafka-storage-internals-d5b544f6925f)


## Cleaning Process

在启动的时候，Kafka Broker会创建一定数量的cleaner threads， 负责清理compacted log(而这个线程的数量是通过`log.cleaner.threads`配置来设置)。这些清理线程，通常都是会找出broker中的最需要清理(filthiest)的日志，并尝试清理掉。对于每个日志，它会计算这个日志的dirty ratio:
```
dirty ratio = the number of bytes in the head / total number of bytes in the log(tail + head)
```
清理线程会选择dirty ratio最高的日志，这个日志被称为`filthiest log`。如果这个日志的dirty ratio大于`min.cleanable.dirty.ratio`， 那么清理线程就会开始清理该日志。否则，清理线程会等待一段时间(通过`log.cleaner.backoff.ms`配置)。

 
当找到`filthiest log`之后，我们想进一步找到这个日志能够被清理的部分。我们注意到，日志有些部分不能被清理且不会被扫描

- 所有active segment中的记录。这也是为什么我们看到p:14$这个重复key记录存在在我们的consumer中
- 如果你设置了`min.compaction.lag.ms`并且这个值大于0，那么任何segment中的一个记录的时间如果比距离现在的时间小于该值，那么也不会被清理。这些segments也不会被扫描

现在我们知道哪些记录会被compact，就是从日志中的第一个记录到第一个不能够被清理的记录。这篇文章为了简单起见，我们假设在head中的所有记录也是可以被清理的。

我们注意到日中的tail部分只会有唯一key的记录，因为在清理进程的时候，重复的key都已经被删除。因此只有可能在head部分会存在key不唯一的情况。为了能够快速找到这些重复的记录，Kafka为head部分的记录创建了一个map。回到我们的例子，如下

{%asset_img 4.jpeg%}

如上图所示，Kafka创建了一个offset map，用于记录head 部分每个key的值以及对应的最新的offset。如果我们在head部分有重复的key，Kafka只会用最新的offset。如上图，key为p6的记录等等offset为5，而p5的最新offset为7.现在清理线程可以检查日志中的每个记录，删除那些key在offset map中，且offset小于offset map相同key的值的记录。

在清理进程compacted log中，不光重复记录会被删除，kafka会将值为null的记录也一并删除。这些记录被称作**tombstone**。 你可以通过配置`delete.retention.ms`来延迟删除他们。通过设置该配置，Kafka会检查包含这个记录的segment修改时间，如果修改时间距离当前时间小于该配置值，那么记录会被保留。


现在，记录已经被清理。在清理进程结束之后，我们有了一个新的tail和head。在清理过程中扫码的最后的记录，也就是新tail的最后记录。

Kafka会在数据存储的根目录下的名为`cleaner-offset-checkpoint`的文件中记录新head的开始offset。这个文件用于下一次清理过程。我们可以查看这个文件如下

```bash
cat /var/lib/kafka/data/cleaner-offset-checkpoint
0
1
latest-product-price 0 6
```

如上所见，总共有三行。第一行是这个文件的版本(应该是为了前向兼容),第二行值1表示这行之后有多少行，而最后一行宝航了compacted log topic的名字，partition的number，以及这个partition的head offset


## Conclusion

这篇文章介绍了什么是log compacted topic，以及他们是如何存储和Kafka如何周期性清理的。在文章最后，我先说明，log compaction主要的场景是为了那些只想保留最新记录的cache场景。假设你想在你的应用启动的时候构建一个cache，你可以考虑利用kafka的compacted log topic。

