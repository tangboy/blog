---
title: Spark CommitCoordinator 保证数据一致性(转)
date: 2018-12-11 22:52:15
tags:
    - spark
    - commit coordinator
    - 一致性
    - 大数据
---

## 概述

Spark 输出数据到 HDFS 时，需要解决如下问题：

1. 由于多个 Task 同时写数据到 HDFS，如何保证要么所有 Task 写的所有文件要么同时对外可见，要么同时对外不可见，即保证数据一致性
2. 同一 Task 可能因为 Speculation 而存在两个完全相同的 Task 实例写相同的数据到 HDFS中，如何保证只有一个 commit 成功
3. 对于大 Job（如具有几万甚至几十万 Task），如何高效管理所有文件

## commit 原理

本文通过 Local mode 执行如下 Spark 程序详解 commit 原理

```scala
sparkContext.textFile("/jason/input.zstd")
  .map(_.split(","))
  .saveAsTextFile("/jason/test/tmp")

```

在详述 commit 原理前，需要说明几个述语

- Task，即某个 Application 的某个 Job 内的某个 Stage 的一个 Task
- TaskAttempt，Task 每次执行都视为一个 TaskAttempt。对于同一个 Task，可能同时存在多个 TaskAttemp
- Application Attempt，即 Application 的一次执行

在本文中，会使用如下缩写

- ${output.dir.root} 即输出目录根路径
- ${appAttempt} 即 Application Attempt ID，为整型，从 0 开始
- ${taskAttemp} 即 Task Attetmp ID，为整型，从 0 开始

<!--more-->

### 检查 Job 输出目录

在启动 Job 之前，Driver 首先通过 FileOutputFormat 的 checkOutputSpecs 方法检查输出目录是否已经存在。若已存在，则直接抛出 FileAlreadyExistsException

{% asset_img check_output_path.png %}

### Driver执行setupJob

Job 开始前，由 Driver（本例使用 local mode，因此由 main 线程执行）调用 FileOuputCommitter.setupJob 创建 Application Attempt 目录，即 `output.dir.root/temporary/{appAttempt}`

{% asset_img setup_job.png %}

### Task执行setupTask

由各 Task 执行 FileOutputCommitter.setupTask 方法（本例使用 local mode，因此由 task 线程执行）。该方法不做任何事情，因为 Task 临时目录由 Task 按需创建。

{% asset_img setup_task.png %}

### 按需创建 Task 目录

本例中，Task 写数据需要通过 TextOutputFormat 的 getRecordWriter 方法创建 LineRecordWriter。而创建前需要通过 FileOutputFormat.getTaskOutputPath设置 Task 输出路径，即 `${output.dir.root}/_temporary/${appAttempt}/_temporary/${taskAttempt}/${fileName}`。该 Task Attempt 所有数据均写在该目录下的文件内

{% asset_img create_task_output_file.png %}

### 检查是否需要 commit

Task 执行数据写完后，通过 FileOutputCommitter.needsTaskCommit 方法检查是否需要 commit 它写在 `${output.dir.root}/_temporary/${appAttempt}/_temporary/${taskAttempt}` 下的数据。

检查依据是 `${output.dir.root}/_temporary/${appAttempt}/_temporary/${taskAttempt}` 目录是否存在

{% asset_img need_commit_task.png %}

如果需要 commit，并且开启了 Output commit coordination，还需要通过 RPC 由 Driver 侧的 OutputCommitCoordinator 判断该 Task Attempt 是否可以 commit

{% asset_img need_commit_task_detail.png %}

之所以需要由 Driver 侧的 CommitCoordinator 判断是否可以 commit，是因为可能由于 speculation 或者其它原因（如之前的 TaskAttemp 未被 Kill 成功）存在同一 Task 的多个 Attemp 同时写数据且都申请 commit 的情况。

### CommitCoordinator

当申请 commitTask 的 TaskAttempt 为失败的 Attempt，则直接拒绝

若该 TaskAttempt 成功，并且 CommitCoordinator 未允许过该 Task 的其它 Attempt 的 commit 请求，则允许该 TaskAttempt 的 commit 请求

若 CommitCoordinator 之前已允许过该 TaskAttempt 的 commit 请求，则继续同意该 TaskAttempt 的 commit 请求，即 CommitCoordinator 对该申请的处理是幂等的。

若该 TaskAttempt 成功，且 CommitCoordinator 之前已允许该 Task 的其它 Attempt 的 commit 请求，则直接拒绝当前 TaskAttempt 的 commit 请求

{% asset_img coordinator_handle_request.png %}

OutputCommitCoordinator 为了实现上述功能，为每个 ActiveStage 维护一个如下 StageState

```scala
private case class StageState(numPartitions: Int) {
  val authorizedCommitters = Array.fill[TaskAttemptNumber](numPartitions)(NO_AUTHORIZED_COMMITTER)
  val failures = mutable.Map[PartitionId, mutable.Set[TaskAttemptNumber]]()
  }
```

该数据结构中，保存了每个 Task 被允许 commit 的 TaskAttempt。默认值均为 NO_AUTHORIZED_COMMITTER

同时，保存了每个 Task 的所有失败的 Attempt

### commitTask

当 TaskAttempt 被允许 commit 后，Task (本例由于使用 local model，因此由 task 线程执行)会通过如下方式 commitTask。

当 `mapreduce.fileoutputcommitter.algorithm.version` 的值为 1 (默认值)时，Task 将 taskAttemptPath 即 `${output.dir.root}/_temporary/${appAttempt}/_temporary/${taskAttempt}` 重命令为 committedTaskPath 即 `${output.dir.root}/_temporary/${appAttempt}/${taskAttempt}`

{% asset_img commit_task_v1.png %}

若 `mapreduce.fileoutputcommitter.algorithm.version` 的值为 2，直接将taskAttemptPath 即 `${output.dir.root}/_temporary/${appAttempt}/_temporary/${taskAttempt}` 内的所有文件移动到 outputPath 即 `${output.dir.root}/`

{% asset_img commit_task_v2.png %}

### commitJob

当所有 Task 都执行成功后，由 Driver （本例由于使用 local model，故由 main 线程执行）执行 `FileOutputCommitter.commitJob`

若 `mapreduce.fileoutputcommitter.algorithm.version` 的值为 1，则由 Driver 单线程遍历所有 committedTaskPath 即 `${output.dir.root}/_temporary/${appAttempt}/${taskAttempt}`，并将其下所有文件移动到 finalOutput 即 `${output.dir.root}` 下

{% asset_img commit_job_v1.png %}

若 `mapreduce.fileoutputcommitter.algorithm.version` 的值为 2，则无须移动任何文件。因为所有 Task 的输出文件已在 commitTask 内被移动到 finalOutput 即 `${output.dir.root}` 内

{% asset_img commit_job_v2.png %}

所有 commit 过的 Task 输出文件移动到 finalOutput 即 `${output.dir.root}` 后，Driver 通过 cleanupJob 删除 `${output.dir.root}/_temporary/` 下所有内容

{% asset_img cleanup_job.png %}

### recoverTask

上文所述的 commitTask 与 commitJob 机制，保证了一次 Application Attemp 中不同 Task 的不同 Attemp 在 commit 时的数据一致性

而当整个 Application retry 时，在之前的 Application Attemp 中已经成功 commit 的 Task 无须重新执行，其数据可直接恢复

恢复 Task 时，先获取上一次的 Application Attempt，以及对应的 committedTaskPath，即 `${output.dir.root}/_temporary/${preAppAttempt}/${taskAttempt}`

若 `mapreduce.fileoutputcommitter.algorithm.version` 的值为 1，并且 preCommittedTaskPath 存在（说明在之前的 Application Attempt 中该 Task 已被 commit 过），则直接将 preCommittedTaskPath 重命名为 committedTaskPath

若 `mapreduce.fileoutputcommitter.algorithm.version` 的值为 2，无须恢复任何数据，因为在之前 Application Attempt 中 commit 过的 Task 的数据已经在 commitTask 中被移动到 ${output.dir.root} 中

{% asset_img recover_task.png %}

### abortTask

中止 Task 时，由 Task 调用 `FileOutputCommitter.abortTask` 方法删除 `${output.dir.root}/_temporary/${appAttempt}/_temporary/${taskAttempt}`

{% asset_img abort_task.png %}

### abortJob

中止 Job 由 Driver 调用 `FileOutputCommitter.abortJob` 方法完成。该方法通过 `FileOutputCommitter.cleanupJob` 方法删除 `${output.dir.root}/_temporary`


## 总结

### V1 vs. V2 committer 过程

V1 committer（即 mapreduce.fileoutputcommitter.algorithm.version 的值为 1），commit 过程如下

- Task 线程将 TaskAttempt 数据写入 `${output.dir.root}/_temporary/${appAttempt}/_temporary/${taskAttempt}`
- commitTask 由 Task 线程将 `${output.dir.root}/_temporary/${appAttempt}/_temporary/${taskAttempt}` 移动到 `${output.dir.root}/_temporary/${appAttempt}/${taskAttempt}`
- commitJob 由 Driver 单线程依次将所有 `${output.dir.root}/_temporary/${appAttempt}/${taskAttempt}` 移动到 `${output.dir.root}`，然后创建 _SUCCESS 标记文件
- recoverTask 由 Task 线程将 `${output.dir.root}/_temporary/${preAppAttempt}/${preTaskAttempt}` 移动到 `${output.dir.root}/_temporary/${appAttempt}/${taskAttempt}`


V2 committer（即 mapreduce.fileoutputcommitter.algorithm.version 的值为 2），commit 过程如下

- Task 线程将 TaskAttempt 数据写入 `${output.dir.root}/_temporary/${appAttempt}/_temporary/${taskAttempt}`
- commitTask 由 Task 线程将 `${output.dir.root}/_temporary/${appAttempt}/_temporary/${taskAttempt}` 移动到 `${output.dir.root}`
- commitJob 创建 _SUCCESS 标记文件
- recoverTask 无需任何操作


### V1 vs. V2 committer 性能对比

V1 在 Job 执行结束后，在 Driver 端通过 commitJob 方法，单线程串行将所有 Task 的输出文件移动到输出根目录。移动以文件为单位，当 Task 个数较多（大 Job，或者小文件引起的大量小 Task），Name Node RPC 较慢时，该过程耗时较久。在实践中，可能因此发生所有 Task 均执行结束，但 Job 不结束的问题。甚至 commitJob 耗时比 所有 Task 执行时间还要长

而 V2 在 Task 结束后，由 Task 在 commitTask 方法内，将自己的数据文件移动到输出根目录。一方面，Task 结束时即移动文件，不需等待 Job 结束才移动文件，即文件移动更早发起，也更早结束。另一方面，不同 Task 间并行移动文件，极大缩短了整个 Job 内所有 Task 的文件移动耗时

### V1 vs. V2 committer 一致性对比

V1 只有 Job 结束，才会将数据文件移动到输出根目录，才会对外可见。在此之前，所有文件均在 `${output.dir.root}/_temporary/${appAttempt}` 及其子文件内，对外不可见。

当 commitJob 过程耗时较短时，其失败的可能性较小，可认为 V1 的 commit 过程是两阶段提交，要么所有 Task 都 commit 成功，要么都失败。

而由于上文提到的问题， commitJob 过程可能耗时较久，如果在此过程中，Driver 失败，则可能发生部分 Task 数据被移动到 ${output.dir.root} 对外可见，部分 Task 的数据未及时移动，对外不可见的问题。此时发生了数据不一致性的问题

V2 当 Task 结束时，立即将数据移动到 ${output.dir.root}，立即对外可见。如果 Application 执行过程中失败了，已 commit 的 Task 数据仍然对外可见，而失败的 Task 数据或未被 commit 的 Task 数据对外不可见。也即 V2 更易发生数据一致性问题


## 使用建议

1. 如果使用分布式文件系统，例如HDFS，建议用V1，因为mv操作比较便宜，快速
2. 如果采用对象存储系统，例如S3，COS，建议用V1，因为mv操作比较贵，非常耗时