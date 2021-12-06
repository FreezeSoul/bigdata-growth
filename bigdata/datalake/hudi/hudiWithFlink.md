* `Configuration`:`$FLINK_HOME/conf/flink-conf.yaml`修改相关配置
* `Writing Data`:Flink支持[Bulk Insert](https://hudi.apache.org/docs/flink-quick-start-guide#bulk-insert), [Index Bootstrap](https://hudi.apache.org/docs/flink-quick-start-guide#index-bootstrap), [Changelog Mode](https://hudi.apache.org/docs/flink-quick-start-guide#changelog-mode), [Insert Mode](https://hudi.apache.org/docs/flink-quick-start-guide#insert-mode) 和 [Offline Compaction](https://hudi.apache.org/docs/flink-quick-start-guide#offline-compaction).
* `Querying Data`:Flink支持[Hive Query](https://hudi.apache.org/docs/flink-quick-start-guide#hive-query), [Presto Query](https://hudi.apache.org/docs/flink-quick-start-guide#presto-query).
* `Optimization`:对于写/读任务，提供了 [Memory Optimization](https://hudi.apache.org/docs/flink-quick-start-guide#memory-optimization)和 [Write Rate Limit](https://hudi.apache.org/docs/flink-quick-start-guide#write-rate-limit).

# 快速开始

## 设置

* 下载1.11.2或1.12.2版本的flink
* 启动Flink集群
  * `flink-conf.yaml`中添加`taskmanager.numberOfTaskSlots: 4`

```shell
bash start-cluster.sh
```

* 启动Flink SQL客户端

```shell
bash sql-client.sh embedded -j ~/Downloads/hudi-flink-bundle_2.11-0.9.0.jar shell
```

## Insert Data

```sql
-- sets up the result mode to tableau to show the results directly in the CLI
set execution.result-mode=tableau;

CREATE TABLE t1(
  uuid VARCHAR(20),
  name VARCHAR(10),
  age INT,
  ts TIMESTAMP(3),
  `partition` VARCHAR(20)
)
PARTITIONED BY (`partition`)
WITH (
  'connector' = 'hudi',
  'path' = 'schema://base-path',
  'table.type' = 'MERGE_ON_READ' -- this creates a MERGE_ON_READ table, by default is COPY_ON_WRITE
);

-- insert data using values
INSERT INTO t1 VALUES
  ('id1','Danny',23,TIMESTAMP '1970-01-01 00:00:01','par1'),
  ('id2','Stephen',33,TIMESTAMP '1970-01-01 00:00:02','par1'),
  ('id3','Julian',53,TIMESTAMP '1970-01-01 00:00:03','par2'),
  ('id4','Fabian',31,TIMESTAMP '1970-01-01 00:00:04','par2'),
  ('id5','Sophia',18,TIMESTAMP '1970-01-01 00:00:05','par3'),
  ('id6','Emma',20,TIMESTAMP '1970-01-01 00:00:06','par3'),
  ('id7','Bob',44,TIMESTAMP '1970-01-01 00:00:07','par4'),
  ('id8','Han',56,TIMESTAMP '1970-01-01 00:00:08','par4');
```

## Update data

```sql
-- 主键相同为修改
insert into t1 values
  ('id1','Danny',27,TIMESTAMP '1970-01-01 00:00:01','par1');
```

## Streaming Query

* 可以通过提交的时间戳去流式消费数据

```sql
CREATE TABLE t1(
  uuid VARCHAR(20),
  name VARCHAR(10),
  age INT,
  ts TIMESTAMP(3),
  `partition` VARCHAR(20)
)
PARTITIONED BY (`partition`)
WITH (
  'connector' = 'hudi',
  'path' = 'oss://vvr-daily/hudi/t1',
  'table.type' = 'MERGE_ON_READ',
  'read.streaming.enabled' = 'true',  -- this option enable the streaming read
  'read.streaming.start-commit' = '20210316134557' -- specifies the start commit instant time
  'read.streaming.check-interval' = '4' -- specifies the check interval for finding new source commits, default 60s.
);

-- Then query the table in stream mode
select * from t1;
```

# Table Option

* 通过Flink SQL WITH配置的表配置

## Memory

| Option Name              | Description                                                  | Default | Remarks                                                      |
| ------------------------ | ------------------------------------------------------------ | ------- | ------------------------------------------------------------ |
| `write.task.max.size`    | 写任务的最大内存(以MB为单位)，当达到阈值时，它刷新最大大小的数据桶以避免OOM。默认1024 mb | `1024D` | 为写缓冲区预留的内存为write.task.max.size - compact .max_memory。当写任务的总缓冲区达到阈值时，将刷新内存中最大的缓冲区 |
| `write.batch.size`       | 为了提高写的效率，Flink写任务会根据写桶将数据缓存到缓冲区中，直到内存达到阈值。当达到阈值时，数据缓冲区将被清除。默认64 mb | `64D`   | 推荐使用默认值                                               |
| `write.log_block.size`   | hudi的日志写入器接收到消息后不会立刻flush数据，写入器以LogBlock为单位将数据刷新到磁盘。在LogBlock达到阈值之前，记录将以序列化字节的形式在写入器中进行缓冲。默认128 mb | `128`   | 推荐使用默认值                                               |
| `write.merge.max_memory` | 如果写入类型是`COPY_ON_WRITE`，Hudi将会合并增量数据和base文件数据。增量数据将会被缓存和溢写磁盘。这个阈值控制可使用的最大堆大小。默认100 mb | `100`   | 推荐使用默认值                                               |
| `compaction.max_memory`  | 与write.merge.max内存相同，但在压缩期间发生。默认100 mb      | `100`   | 如果是在线压缩，则可以在资源足够时打开它，例如设置为1024MB   |

## Parallelism

| Option Name                  | Description                                                  | Default                                                      | Remarks                                                      |
| ---------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `write.tasks`                | 写入器任务的并行度，每个写任务依次向1到N个桶写。默认的4      | `4`                                                          | 增加并行度对小文件的数量没有影响                             |
| `write.bucket_assign.tasks`  | 桶分配操作符的并行性。无默认值，使用Flink parallelism.default | [`parallelism.default`](https://hudi.apache.org/docs/flink-quick-start-guide#parallelism) | 增加并行度也会增加桶的数量，从而增加小文件(小桶)的数量。     |
| `write.index_boostrap.tasks` | index bootstrap的并行度，增加并行度可以提高bootstarp阶段的效率。因此，需要设置更多的检查点容错时间。默认使用Flink并行 | [`parallelism.default`](https://hudi.apache.org/docs/flink-quick-start-guide#parallelism) | 只有当index. bootstrap .enabled为true时才生效                |
| `read.tasks`                 | Default `4`读操作的并行度(批和流)                            | `4`                                                          |                                                              |
| `compaction.tasks`           | 实时compaction的并行度，默认为10                             | `10`                                                         | `Online compaction` 会占用写任务的资源，推荐使用offline compaction`](https://hudi.apache.org/docs/flink-quick-start-guide#offline-compaction) |

## Compaction

* 以下配置近支持实时compaction
* 通过设置`compaction.async.enabled = false`关闭在线压缩，但我们仍然建议对写作业启用`compaction.schedule.enable`。然后，您可以通过脱机压缩来执行压缩计划。

| Option Name                   | Description                                             | Default       | Remarks                                                      |
| ----------------------------- | ------------------------------------------------------- | ------------- | ------------------------------------------------------------ |
| `compaction.schedule.enabled` | 是否定期生成compaction计划                              | `true`        | 即使compaction.async.enabled = false，也建议打开它           |
| `compaction.async.enabled`    | 异步压缩，MOR默认启用                                   | `true`        | 通过关闭此选项来关闭`online compaction`                      |
| `compaction.trigger.strategy` | 触发compaction的策略                                    | `num_commits` | Options are `num_commits`: 当达到N个delta提交时触发压缩; `time_elapsed`: 当距离上次压缩时间> N秒时触发压缩; `num_and_time`: 当满足`NUM_COMMITS`和`TIME_ELAPSED`时，进行rigger压缩;`num_or_time`: 在满足`NUM_COMMITS`或`TIME_ELAPSED`时触发压缩。 |
| `compaction.delta_commits`    | 触发压缩所需的最大delte提交，默认为5次提交              | `5`           | --                                                           |
| `compaction.delta_seconds`    | 触发压缩所需的最大增量秒数，默认为1小时                 | `3600`        | --                                                           |
| `compaction.max_memory`       | `compaction`溢出映射的最大内存(以MB为单位)，默认为100MB | `100`         | 如果您有足够的资源，建议调整到1024MB                         |
| `compaction.target_io`        | 每次压缩的目标IO(读和写)，默认为5GB                     | `5120`        | `offline compaction` 的默认值是500GB                         |

# Memory Optimization

## MOR

* 设置Flink状态后端为`RocksDB`(默认为`in memory`状态后端)
* 如果有足够的内存，`compaction.max_memory`可以设置大于100MB建议调整至1024MB。
* 注意taskManager分配给每个写任务的内存，确保每个写任务都能分配到所需的内存大小`write.task.max.size`。例如，taskManager有4GB内存运行两个streamWriteFunction，所以每个写任务可以分配2GB内存。请保留一些缓冲区，因为taskManager上的网络缓冲区和其他类型的任务(如bucketAssignFunction)也会占用内存。
* 注意compaction的内存变化，`compaction.max_memory`控制在压缩任务读取日志时可以使用每个任务的最大内存。`compaction.tasks`控制压缩任务的并行性。

## COW

* 设置Flink状态后端为`RocksDB`(默认为`in memory`状态后端)
* 增大`write.task.max.size`和`write.merge.max_memory`(默认1024MB和100MB，调整为2014MB和1024MB)
* 注意taskManager分配给每个写任务的内存，确保每个写任务都能分配到所需的内存大小`write.task.max.size`。例如，taskManager有4GB内存运行两个streamWriteFunction，所以每个写任务可以分配2GB内存。请保留一些缓冲区，因为taskManager上的网络缓冲区和其他类型的任务(如bucketAssignFunction)也会占用内存。

# Bulk Insert

* 用于快照数据导入。如果快照数据来自其他数据源，可以使用bulk_insert模式将快照数据快速导入到Hudi中。
* Bulk_insert消除了序列化和数据合并。用户无需重复数据删除，因此需要保证数据的唯一性。
* Bulk_insert在批处理执行模式下效率更高。默认情况下，批处理执行方式根据分区路径对输入记录进行排序，并将这些记录写入Hudi，避免了频繁切换文件句柄导致的写性能下降。有序写入一个分区中不会频繁写换对应的数据分区
* bulk_insert的并行度由write.tasks指定。并行度会影响小文件的数量。从理论上讲，bulk_insert的并行性是bucket的数量(特别是，当每个bucket写到最大文件大小时，它将转到新的文件句柄。最后，文件的数量>= write.bucket_assign.tasks)。

| Option Name                              | Required | Default  | Remarks                                                      |
| ---------------------------------------- | -------- | -------- | ------------------------------------------------------------ |
| `write.operation`                        | `true`   | `upsert` | Setting as `bulk_insert` to open this function               |
| `write.tasks`                            | `false`  | `4`      | The parallelism of `bulk_insert`, `the number of files` >= [`write.bucket_assign.tasks`](https://hudi.apache.org/docs/flink-quick-start-guide#parallelism) |
| `write.bulk_insert.shuffle_by_partition` | `false`  | `true`   | 写入前是否根据分区字段进行shuffle。启用此选项将减少小文件的数量，但可能存在数据倾斜的风险 |
| `write.bulk_insert.sort_by_partition`    | `false`  | `true`   | 写入前是否根据分区字段对数据进行排序。启用此选项将在写任务写多个分区时减少小文件的数量 |
| `write.sort.memory`                      | `false`  | `128`    | Available managed memory of sort operator. default `128` MB  |

# Index Bootstrap

* 用于`snapshot data`+`incremental data`导入的需求。如果`snapshot data`已经通过`bulk insert`插入到Hudi中。通过`Index Bootstrap`功能，用户可以实时插入`incremental data`，保证数据不重复。
* 如果您认为这个过程非常耗时，可以在写入快照数据的同时增加资源以流模式写入，然后减少资源以写入增量数据(或打开速率限制函数)。

| Option Name               | Required | Default | Remarks                                                      |
| ------------------------- | -------- | ------- | ------------------------------------------------------------ |
| `index.bootstrap.enabled` | `true`   | `false` | 开启index.bootstrap.enabled时，Hudi表中的剩余记录将一次性加载到Flink状态 |
| `index.partition.regex`   | `false`  | `*`     | 优化选择。设置正则表达式来过滤分区。默认情况下，所有分区都被加载到flink状态 |

## 使用方式

1. `CREATE TABLE`创建一条与Hudi表对应的语句。注意这个`table.type`必须正确。
2. 设置`index.bootstrao.enabled`为true开启index bootstrap。
3. 设置`execution.checkpointing.tolerable-failed-checkpoints = n`
4. 等待第一次ck成功则index boostrap执行完毕。
5. 等待index boostrap完成，用户可以退出并保存保存点(或直接使用外部化检查点)。
6. 重启job, 设置 `index.bootstrap.enable` 为 `false`.

# Changelog Mode

* Hudi可以保留消息的所有中间变化(I / -U / U / D)，然后通过flink的状态计算消费，从而拥有一个接近实时的数据仓库ETL管道(增量计算)。Hudi MOR表以行的形式存储消息，支持保留所有更改日志(格式级集成)。所有的更新日志记录可以使用Flink流reader

| Option Name         | Required | Default | Remarks                                                      |
| ------------------- | -------- | ------- | ------------------------------------------------------------ |
| `changelog.enabled` | `false`  | `false` | 它在默认情况下是关闭的，为了拥有upsert语义，只有合并的消息被确保保留，中间的更改可以被合并。设置为true以支持使用所有更改 |

* 批处理(快照)读取仍然合并所有中间更改，不管格式是否存储了中间更改日志消息。
* `changelog.enable`设置为true后，更改日志记录的保留只是最好的工作:异步压缩任务将更改日志记录合并到一个记录中，因此，如果流源不及时使用，则压缩后只能读取每个键的合并记录。解决方案是通过调整压缩策略，比如压缩选项:compress .delta_commits和compression .delta_seconds，为读取器保留一些缓冲时间。

# Insert Mode

* 默认情况下，Hudi对插入模式采用小文件策略:MOR将增量记录追加到日志文件中，COW合并 base parquet文件(增量数据集将被重复数据删除)。这种策略会导致性能下降。
* 如果要禁止文件合并行为，可将`write.insert.deduplicate`设置为`false`，则跳过重复数据删除。每次刷新行为直接写入一个now parquet文件(MOR表也直接写入parquet文件)。

| Option Name                | Required | Default | Remarks                                                      |
| -------------------------- | -------- | ------- | ------------------------------------------------------------ |
| `write.insert.deduplicate` | `false`  | `true`  | Insert mode 默认启用重复数据删除功能。关闭此选项后，每次刷新行为直接写入一个now parquet文件 |

# Hive Query

* hive1.x只能同步元数据不能进行Hive查询，需要通过Spark查询Hive

## Hive环境

1. 导入`hudi-hadoop-mr-bundle`到hive，在hive的根目录创建`auxlib`,将`hudi-hadoop-mr-bundle-0.x.x-SNAPSHOT.jar`放入`auxlib`
2. 启动`hive metastore`和`hiveServer2`服务

## 同步模板

* Flink hive sync现在支持两种hive同步模式，hms和jdbc。HMS模式只需要配置metastore uris即可。对于jdbc模式，需要配置jdbc属性和metastore uri。选项模板如下:

```sql
-- hms mode template
CREATE TABLE t1(
  uuid VARCHAR(20),
  name VARCHAR(10),
  age INT,
  ts TIMESTAMP(3),
  `partition` VARCHAR(20)
)
PARTITIONED BY (`partition`)
WITH (
  'connector' = 'hudi',
  'path' = 'oss://vvr-daily/hudi/t1',
  'table.type' = 'COPY_ON_WRITE',  --If MERGE_ON_READ, hive query will not have output until the parquet file is generated
  'hive_sync.enable' = 'true',     -- Required. To enable hive synchronization
  'hive_sync.mode' = 'hms'         -- Required. Setting hive sync mode to hms, default jdbc
  'hive_sync.metastore.uris' = 'thrift://ip:9083' -- Required. The port need set on hive-site.xml
);


-- jdbc mode template
CREATE TABLE t1(
  uuid VARCHAR(20),
  name VARCHAR(10),
  age INT,
  ts TIMESTAMP(3),
  `partition` VARCHAR(20)
)
PARTITIONED BY (`partition`)
WITH (
  'connector' = 'hudi',
  'path' = 'oss://vvr-daily/hudi/t1',
  'table.type' = 'COPY_ON_WRITE',  --If MERGE_ON_READ, hive query will not have output until the parquet file is generated
  'hive_sync.enable' = 'true',     -- Required. To enable hive synchronization
  'hive_sync.mode' = 'hms'         -- Required. Setting hive sync mode to hms, default jdbc
  'hive_sync.metastore.uris' = 'thrift://ip:9083'  -- Required. The port need set on hive-site.xml
  'hive_sync.jdbc_url'='jdbc:hive2://ip:10000',    -- required, hiveServer port
  'hive_sync.table'='t1',                          -- required, hive table name
  'hive_sync.db'='testDB',                         -- required, hive database name
  'hive_sync.username'='root',                     -- required, HMS username
  'hive_sync.password'='your password'             -- required, HMS password
);
```

## 查询

* 使用hive beeline查询需要设置以下参数：

```sql
set hive.input.format = org.apache.hudi.hadoop.hive.HoodieCombineHiveInputFormat;
```

# Offline Compaction

* 默认情况下，MERGE_ON_READ表的压缩是启用的。触发器策略是在完成五次提交后执行压缩。因为压缩会消耗大量内存，并且与写操作处于相同的管道中，所以当数据量很大(> 100000 /秒)时，很容易干扰写操作。此时，使用离线压缩更稳定地执行压缩任务。
* 压缩任务的执行包含俩个部分：调度压缩计划和执行压缩计划。建议调度压缩计划的进程由写任务周期性触发，默认情况下写参数`compact.schedule.enable`为启用状态。
* offline compaction需要提交一个flink任务后台运行

```shell
./bin/flink run -c org.apache.hudi.sink.compact.HoodieFlinkCompactor lib/hudi-flink-bundle_2.11-0.9.0.jar --path hdfs://xxx:9000/table
```

| Option Name               | Required | Default | Remarks                                                      |
| ------------------------- | -------- | ------- | ------------------------------------------------------------ |
| `--path`                  | `frue`   | `--`    | 目标表存储在Hudi上的路径                                     |
| `--compaction-max-memory` | `false`  | `100`   | 压缩期间日志数据的索引映射大小，默认为100 MB。如果您有足够的内存，您可以打开这个参数 |
| `--schedule`              | `false`  | `false` | 是否执行调度压缩计划的操作。当写进程仍在写时，打开此参数有丢失数据的风险。因此，开启该参数时，必须确保当前没有写任务向该表写入数据 |
| `--seq`                   | `false`  | `LIFO`  | 压缩任务执行的顺序。默认情况下从最新的压缩计划执行。`LIFO`:从最新的计划开始执行。`FIFO`:从最古老的计划执行。 |

# Write Rate Limit

* 在现有的数据同步中，`snapshot data`和`incremental data`发送到kafka，然后通过Flink流写入到Hudi。因为直接使用`snapshot data`会导致高吞吐量和严重无序(随机写分区)等问题，这会导致写性能下降和吞吐量故障。此时，可以打开`write.rate.limit`选项，以确保顺利写入。

| Option Name        | Required | Default | Remarks   |
| ------------------ | -------- | ------- | --------- |
| `write.rate.limit` | `false`  | `0`     | 默认 关闭 |

# Flink SQL Writer

| Option Name                                | Required | Default                              | Remarks                                                      |
| ------------------------------------------ | -------- | ------------------------------------ | ------------------------------------------------------------ |
| path                                       | Y        | N/A                                  | 目标hudi表的基本路径。如果路径不存在，则会创建该路径，否则hudi表将被成功初始化 |
| table.type                                 | N        | COPY_ON_WRITE                        | Type of table to write. COPY_ON_WRITE (or) MERGE_ON_READ     |
| write.operation                            | N        | upsert                               | The write operation, that this write should do (insert or upsert is supported) |
| write.precombine.field                     | N        | ts                                   | 实际写入前precombine中使用的字段。当两条记录具有相同的键值时，我们将挑选一个值最大的precombine字段，由Object.compareTo(..)决定。预聚合 |
| write.payload.class                        | N        | OverwriteWithLatestAvroPayload.class | 插入/插入时滚动您自己的合并逻辑                              |
| write.insert.drop.duplicates               | N        | false                                | 插入时是否删除重复项。                                       |
| write.ignore.failed                        | N        | true                                 | Flag to indicate whether to ignore any non exception error (e.g. writestatus error). within a checkpoint batch. By default true (in favor of streaming progressing over data integrity) |
| hoodie.datasource.write.recordkey.field    | N        | uuid                                 | Record key field. Value to be used as the `recordKey` component of `HoodieKey`. Actual value will be obtained by invoking .toString() on the field value. Nested fields can be specified using the dot notation eg: `a.b.c` |
| hoodie.datasource.write.keygenerator.class | N        | SimpleAvroKeyGenerator.class         | Key generator class, that implements will extract the key out of incoming record |
| write.tasks                                | N        | 4                                    | Parallelism of tasks that do actual write, default is 4      |
| write.batch.size.MB                        | N        | 128                                  | Batch buffer size in MB to flush data into the underneath filesystem |

# Key Generation

* Hudi维护hoodie keys(record key + partition path)，以唯一地标识一个特定的记录。key的生成类将从传入的记录中提取这些信息。
* Hudi目前支持如下不同的记录键和分区路径组合：
  * 简单的记录键(只包含一个字段)和简单的分区路径(可选的hive风格分区)
  * 简单的记录键和自定义基于时间戳的分区路径
  * 复杂的记录键(多个字段的组合)和复杂的分区路径
  * 复杂的记录键和基于时间戳的分区路径
  * 非分区表

# Deletes

* hudi支持2种类型删除hudi表数据，通过允许用户指定不同的记录有效负载实现。
  * `Soft Deletes`:保留记录键，并将所有其他字段的值空出来。这可以通过确保表模式中适当的字段为空，并在将这些字段设置为空后简单地插入表来实现。
  * `Hard Deletes`:更强的删除形式是物理地从表中删除记录的任何跟踪。这可以通过3种不同的方式实现。
    * 使用DataSource，设置`OPERATION.key()`为`DELETE_OPERATION_OPT_VAL`.这将会移除这个正在提交的数据集的全部记录。
    * 使用DataSource，设置`HoodieWriteConfig.WRITE_PAYLOAD_CLASS_NAME.key()`为`org.apache.hudi.EmptyHoodieRecordPayload`.这将会移除这个正在提交的数据集的全部记录。
    * 使用DataSource或DeltaStreamer，添加列名为`_hoodie_is_deleted`在 这个数据集。对于所有要删除的记录，该列的值必须设置为true，对于要被推翻的记录，该列的值必须设置为false或为空。

```scala
 deleteDF // dataframe containing just records to be deleted
   .write().format("org.apache.hudi")
   .option(...) // Add HUDI options like record-key, partition-path and others as needed for your setup
   // specify record_key, partition_key, precombine_fieldkey & usual params
   .option(HoodieWriteConfig.WRITE_PAYLOAD_CLASS_NAME.key(), "org.apache.hudi.EmptyHoodieRecordPayload")
```

# Flink Hudi Connector源码分析

* 源码都基于Flink-hudi 0.9.0版本

## StreamWriteFunction

* 用于将数据写入外部系统，这个函数首先会buffer一批HoodieRecord数据，当一批buffer数据上超过` FlinkOptions#WRITE_BATCH_SIZE`大小或者全部的buffer数据超过`FlinkOptions#WRITE_TASK_MAX_SIZE`，或者是Flink开始做ck，则flush。如果一批数据写入成功，则StreamWriteOperatorCoordinator会标识写入成功。
* 这个operator coordinator会校验和提交最后一个instant，当最后一个instant提交成功时会启动一个新的instant。它会开启一个新的instant之前回滚全部inflight instant，hoodie的instant只会在一个ck中。写函数在它ck超时抛出异常时刷新数据buffer，任何检查点失败最终都会触发作业失败。

### 核心属性

```java
 /**
   * Write buffer as buckets for a checkpoint. The key is bucket ID.
   */
  private transient Map<String, DataBucket> buckets;

  /**
   * Config options.
   */
  private final Configuration config;

  /**
   * Id of current subtask.
   */
  private int taskID;

  /**
   * Write Client.
   */
  private transient HoodieFlinkWriteClient writeClient;

/**
* 写入函数
*/
  private transient BiFunction<List<HoodieRecord>, String, List<WriteStatus>> writeFunction;

  /**
   * The REQUESTED instant we write the data.
   */
  private volatile String currentInstant;

  /**
   * Gateway to send operator events to the operator coordinator.
   */
  private transient OperatorEventGateway eventGateway;

  /**
   * Commit action type.
   */
  private transient String actionType;

  /**
   * Total size tracer. 记录大小的tracer
   */
  private transient TotalSizeTracer tracer;

  /**
   * Flag saying whether the write task is waiting for the checkpoint success notification
   * after it finished a checkpoint.
   *
   * <p>The flag is needed because the write task does not block during the waiting time interval,
   * some data buckets still flush out with old instant time. There are two cases that the flush may produce
   * corrupted files if the old instant is committed successfully:
   * 1) the write handle was writing data but interrupted, left a corrupted parquet file;
   * 2) the write handle finished the write but was not closed, left an empty parquet file.
   *
   * <p>To solve, when this flag was set to true, we block the data flushing thus the #processElement method,
   * the flag was reset to false if the task receives the checkpoint success event or the latest inflight instant
   * time changed(the last instant committed successfully).
   */
  private volatile boolean confirming = false;

  /**
   * List state of the write metadata events.
   */
  private transient ListState<WriteMetadataEvent> writeMetadataState;

  /**
   * Write status list for the current checkpoint.
   */
  private List<WriteStatus> writeStatuses;
```

### open

```java
public void open(Configuration parameters) throws IOException {
    this.tracer = new TotalSizeTracer(this.config);
    initBuffer();
    initWriteFunction();
  }
  
  private static class TotalSizeTracer {
    private long bufferSize = 0L;
    private final double maxBufferSize;

    TotalSizeTracer(Configuration conf) {
      long mergeReaderMem = 100; // constant 100MB
      long mergeMapMaxMem = conf.getInteger(FlinkOptions.WRITE_MERGE_MAX_MEMORY);
      // 最大的buffer大小
      this.maxBufferSize = (conf.getDouble(FlinkOptions.WRITE_TASK_MAX_SIZE) - mergeReaderMem - mergeMapMaxMem) * 1024 * 1024;
      final String errMsg = String.format("'%s' should be at least greater than '%s' plus merge reader memory(constant 100MB now)",
          FlinkOptions.WRITE_TASK_MAX_SIZE.key(), FlinkOptions.WRITE_MERGE_MAX_MEMORY.key());
      ValidationUtils.checkState(this.maxBufferSize > 0, errMsg);
    }

    /**
     * Trace the given record size {@code recordSize}.
     *
     * @param recordSize The record size
     * @return true if the buffer size exceeds the maximum buffer size
     */
    boolean trace(long recordSize) {
      // 判断是否大于maxBufferSize
      this.bufferSize += recordSize;
      return this.bufferSize > this.maxBufferSize;
    }

    void countDown(long size) {
      this.bufferSize -= size;
    }

    public void reset() {
      this.bufferSize = 0;
    }
  }

// 初始化bucket
  private void initBuffer() {
    this.buckets = new LinkedHashMap<>();
  }

// 初始writeFunction
 private void initWriteFunction() {
    final String writeOperation = this.config.get(FlinkOptions.OPERATION);
    switch (WriteOperationType.fromValue(writeOperation)) {
      case INSERT:
        this.writeFunction = (records, instantTime) -> this.writeClient.insert(records, instantTime);
        break;
      case UPSERT:
        this.writeFunction = (records, instantTime) -> this.writeClient.upsert(records, instantTime);
        break;
      case INSERT_OVERWRITE:
        this.writeFunction = (records, instantTime) -> this.writeClient.insertOverwrite(records, instantTime);
        break;
      case INSERT_OVERWRITE_TABLE:
        this.writeFunction = (records, instantTime) -> this.writeClient.insertOverwriteTable(records, instantTime);
        break;
      default:
        throw new RuntimeException("Unsupported write operation : " + writeOperation);
    }
  }
```

### 初始化状态

* 创建hudi写入客户端，获取actionType，然后根据是否savepoint确定是否重新提交inflight阶段的instant

```java
  public void initializeState(FunctionInitializationContext context) throws Exception {
    this.taskID = getRuntimeContext().getIndexOfThisSubtask();
    // 创建hudi写入客户端
    this.writeClient = StreamerUtil.createWriteClient(this.config, getRuntimeContext());
    // 读取with配置
    this.actionType = CommitUtils.getCommitActionType(
        WriteOperationType.fromValue(config.getString(FlinkOptions.OPERATION)),
        HoodieTableType.valueOf(config.getString(FlinkOptions.TABLE_TYPE)));

    this.writeStatuses = new ArrayList<>();
    // 写入元数据状态
    this.writeMetadataState = context.getOperatorStateStore().getListState(
        new ListStateDescriptor<>(
            "write-metadata-state",
            TypeInformation.of(WriteMetadataEvent.class)
        ));

    this.currentInstant = this.writeClient.getLastPendingInstant(this.actionType);
    if (context.isRestored()) {
      restoreWriteMetadata();
    } else {
      sendBootstrapEvent();
    }
    // blocks flushing until the coordinator starts a new instant
    this.confirming = true;
  }


//WriteMetadataEvent
public class WriteMetadataEvent implements OperatorEvent {
  private static final long serialVersionUID = 1L;

  public static final String BOOTSTRAP_INSTANT = "";

  private List<WriteStatus> writeStatuses;
  private int taskID;
  // instant时间
  private String instantTime;
  // 是否最后一个批次
  private boolean lastBatch;

  /**
   * Flag saying whether the event comes from the end of input, e.g. the source
   * is bounded, there are two cases in which this flag should be set to true:
   * 1. batch execution mode
   * 2. bounded stream source such as VALUES
   */
  private boolean endInput;

  /**
   * Flag saying whether the event comes from bootstrap of a write function.
   */
  private boolean bootstrap;
}

// 恢复写入元数据
private void restoreWriteMetadata() throws Exception {
    String lastInflight = this.writeClient.getLastPendingInstant(this.actionType);
    boolean eventSent = false;
    for (WriteMetadataEvent event : this.writeMetadataState.get()) {
      if (Objects.equals(lastInflight, event.getInstantTime())) {
        // The checkpoint succeed but the meta does not commit,
        // re-commit the inflight instant
        // 重新提交inflight的instant
        this.eventGateway.sendEventToCoordinator(event);
        LOG.info("Send uncommitted write metadata event to coordinator, task[{}].", taskID);
        eventSent = true;
      }
    }
    if (!eventSent) {
      sendBootstrapEvent();
    }
  }

  private void sendBootstrapEvent() {
    // 发送空的event
    this.eventGateway.sendEventToCoordinator(WriteMetadataEvent.emptyBootstrap(taskID));
    LOG.info("Send bootstrap write metadata event to coordinator, task[{}].", taskID);
  }
```

### snapshotState

```java
  public void snapshotState(FunctionSnapshotContext functionSnapshotContext) throws Exception {
  //基于协调器首先启动检查点的事实，
  //它将检查有效性。
  //等待缓冲区数据刷新，并请求一个新的即时
    flushRemaining(false);
    // 重新加载writeMeta状态
    reloadWriteMetaState();
  }
// endInput标识是否无界流
private void flushRemaining(boolean endInput) {
  	// hasData==(this.buckets.size() > 0 && this.buckets.values().stream().anyMatch(bucket -> bucket.records.size() > 0);)
  // 获取当前instant
    this.currentInstant = instantToWrite(hasData());
    if (this.currentInstant == null) {
      // in case there are empty checkpoints that has no input data
      throw new HoodieException("No inflight instant when flushing data!");
    }
    final List<WriteStatus> writeStatus;
    if (buckets.size() > 0) {
      writeStatus = new ArrayList<>();
      this.buckets.values()
          // The records are partitioned by the bucket ID and each batch sent to
          // the writer belongs to one bucket.
          .forEach(bucket -> {
            List<HoodieRecord> records = bucket.writeBuffer();
            if (records.size() > 0) {
              if (config.getBoolean(FlinkOptions.INSERT_DROP_DUPS)) {
                // 去重
                records = FlinkWriteHelper.newInstance().deduplicateRecords(records, (HoodieIndex) null, -1);
              }
              // 预写 在刷新之前设置:用正确的分区路径和fileID修补第一个记录。
              bucket.preWrite(records);
              // 写入数据
              writeStatus.addAll(writeFunction.apply(records, currentInstant));
              records.clear();
              bucket.reset();
            }
          });
    } else {
      LOG.info("No data to write in subtask [{}] for instant [{}]", taskID, currentInstant);
      writeStatus = Collections.emptyList();
    }
  // 构造WriteMetadataEvent
    final WriteMetadataEvent event = WriteMetadataEvent.builder()
        .taskID(taskID)
        .instantTime(currentInstant)
        .writeStatus(writeStatus)
        .lastBatch(true)
        .endInput(endInput)
        .build();
	// 发送event
    this.eventGateway.sendEventToCoordinator(event);
    this.buckets.clear();
    this.tracer.reset();
    this.writeClient.cleanHandles();
  // 写入状态放入状态
    this.writeStatuses.addAll(writeStatus);
    // blocks flushing until the coordinator starts a new instant
    this.confirming = true;
  }

 /**
   * Reload the write metadata state as the current checkpoint.
   */
  private void reloadWriteMetaState() throws Exception {
    // 清理writeMetadataState
    this.writeMetadataState.clear();
    WriteMetadataEvent event = WriteMetadataEvent.builder()
        .taskID(taskID)
        .instantTime(currentInstant)
        .writeStatus(new ArrayList<>(writeStatuses))
        .bootstrap(true)
        .build();
    this.writeMetadataState.add(event);
    writeStatuses.clear();
  }
```

### 数据buffer逻辑

```java
private void bufferRecord(HoodieRecord<?> value) {
  // 根据record获取bucketId {partition path}_{fileID}.
    final String bucketID = getBucketID(value);
		// 获取对应bucket
    DataBucket bucket = this.buckets.computeIfAbsent(bucketID,
        k -> new DataBucket(this.config.getDouble(FlinkOptions.WRITE_BATCH_SIZE), value));
  // 包装hoodie记录
    final DataItem item = DataItem.fromHoodieRecord(value);
		// 判断是否需要刷新bucket，是否超过WRITE_BATCH_SIZE大小
    boolean flushBucket = bucket.detector.detect(item);
   // 判断是否需要刷新buffer，是否超过maxBufferSize，总buffer大小
    boolean flushBuffer = this.tracer.trace(bucket.detector.lastRecordSize);
    if (flushBucket) {
      if (flushBucket(bucket)) {
        // 清理总buffer大小
        this.tracer.countDown(bucket.detector.totalSize);
        // 重置bucket大小
        bucket.reset();
      }
    } else if (flushBuffer) {
      // find the max size bucket and flush it out，找到最大的bucket然后flush
      List<DataBucket> sortedBuckets = this.buckets.values().stream()
          .sorted((b1, b2) -> Long.compare(b2.detector.totalSize, b1.detector.totalSize))
          .collect(Collectors.toList());
      final DataBucket bucketToFlush = sortedBuckets.get(0);
      if (flushBucket(bucketToFlush)) {
        this.tracer.countDown(bucketToFlush.detector.totalSize);
        bucketToFlush.reset();
      } else {
        LOG.warn("The buffer size hits the threshold {}, but still flush the max size data bucket failed!", this.tracer.maxBufferSize);
      }
    }
  // 一批buffer数据上不超过` FlinkOptions#WRITE_BATCH_SIZE`大小或者全部的buffer数据超过`FlinkOptions#WRITE_TASK_MAX_SIZE`
    bucket.records.add(item);
  }

// 刷新bucket
private boolean flushBucket(DataBucket bucket) {
    String instant = instantToWrite(true);

    if (instant == null) {
      // in case there are empty checkpoints that has no input data
      LOG.info("No inflight instant when flushing data, skip.");
      return false;
    }

    List<HoodieRecord> records = bucket.writeBuffer();
    ValidationUtils.checkState(records.size() > 0, "Data bucket to flush has no buffering records");
    if (config.getBoolean(FlinkOptions.INSERT_DROP_DUPS)) {
      // 去重hoodieRecord
      records = FlinkWriteHelper.newInstance().deduplicateRecords(records, (HoodieIndex) null, -1);
    }
  	// 预写 在刷新之前设置:用正确的分区路径和fileID修补第一个记录。
    bucket.preWrite(records);
    // 写入记录
    final List<WriteStatus> writeStatus = new ArrayList<>(writeFunction.apply(records, instant));
    records.clear();
   // 发送数据
    final WriteMetadataEvent event = WriteMetadataEvent.builder()
        .taskID(taskID)
        .instantTime(instant) // the write instant may shift but the event still use the currentInstant.
        .writeStatus(writeStatus)
        .lastBatch(false)
        .endInput(false)
        .build();

    this.eventGateway.sendEventToCoordinator(event);
    writeStatuses.addAll(writeStatus);
    return true;
  }
```
