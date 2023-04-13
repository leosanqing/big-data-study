# 1. 一致性Hash 索引



Consistent Hashing Index(https://hudi.apache.org/releases/release-0.13.0#consistent-hashing-index)



## 背景

### 之前 BucketIndex限制

1. bucket Index 静态
2. 扩缩节点成本很高，需要全部重新rehashing

![image-20230404112358683](./img/image-20230404112358683.png)



## 相关 RFC

https://github.com/apache/hudi/blob/master/rfc/rfc-42/rfc-42.md

[JIRA ISSUE ](https://issues.apache.org/jira/browse/HUDI-3000)

[一致性Hash算法](https://developer.huawei.com/consumer/cn/forum/topic/0203810951415790238)



一致性Hash算法，会将index认为是一个环，他会将metadata的Node，放到这个环上，然后计算Hash的时候，会顺时针找他下一个节点。实现方式TreeMap，所以能看到他查找的时间是LogN，扩充时间是O(1)



We introduce the Consistent Hashing Index as yet another indexing option for your writes with Hudi. This is an enhancement to [Bucket Index](https://hudi.apache.org/releases/older-releases#bucket-index) which is added in the 0.11.0 release. With Bucket Index, buckets/file groups per partition are statically allocated, whereas with Consistent Hashing Index, buckets can grow dynamically and so users don't need to sweat about data skews. Buckets will expand and shrink depending on the load factor for each partition. You can find the [RFC](https://github.com/apache/hudi/blob/master/rfc/rfc-42/rfc-42.md) for the design of this feature.



## 参数

```properties
hoodie.index.type=bucket
hoodie.index.bucket.engine=CONSISTENT_HASHING
hoodie.bucket.index.max.num.buckets=128
hoodie.bucket.index.min.num.buckets=32
hoodie.bucket.index.num.buckets=4
## do split if the bucket size reach 1.5 * max_file_size
hoodie.bucket.index.split.threshold=1.5
## do merge if the bucket size smaller than 0.2 * max_file_size
hoodie.bucket.index.merge.threshold=0.1 
```

To enforce shrinking or scaling up of buckets, you need to enable clustering using the following configs

要强制缩小或扩展存储桶，您需要使用以下配置启用集群

```properties
## check resize for every 4 commit
hoodie.clustering.inline=true
hoodie.clustering.inline.max.commits=4
hoodie.clustering.plan.strategy.class=org.apache.hudi.client.clustering.plan.strategy.SparkConsistentBucketClusteringPlanStrategy
hoodie.clustering.execution.strategy.class=org.apache.hudi.client.clustering.run.strategy.SparkConsistentBucketClusteringExecutionStrategy
## for supporting concurrent write & resizing
hoodie.clustering.updates.strategy=org.apache.hudi.client.clustering.update.strategy.SparkConsistentBucketDuplicateUpdateStrategy
```



## 限制

- 此索引仅支持使用MOR表的Spark引擎。(This index is supported only for Spark engine using a MOR table.)
- 它不能在启用元数据表的情况下工作。(It does not work with metadata table enabled.)
- 要放大或缩小存储桶，用户必须使用上述配置（以一定的节奏）手动触发集群，但他们不能同时运行 Compation。(To scale up or shrink the buckets, users have to manually trigger clustering using above configs (at some cadence), but they cannot have compaction concurrently running.)
- 因此，如果您的常规写入管道(regular write pipeline) 启用了Compation，请遵循以下建议：您可以选择每12小时触发一次伸缩。在这种情况下，每12小时一次，您可能需要禁用Compation，停止写入管道并启用 clustering。您应该格外小心，不要同时运行两者，因为这可能会导致冲突和管道失败。clustering 完成后，您可以恢复常规写入管道(regular write pipeline)，该管道将启用Compation。



## 性能

![image-20230404135312627](./img/image-20230404135312627.png)

## 代码实现原理

![image-20230404135842632](./img/image-20230404135842632.png)



### 当成Clustring子任务

使用clustering服务的功能，把Split/Merge 功能当成 Clustering 的子任务进行，所以才有之前的限制，必须开启clustring的功能![image-20230410144842425](/Users/liurongtong/Library/Application Support/typora-user-images/image-20230410144842425.png)



### 解决并发写

拆分的时候，如果并发写入会怎么样。如果采用 BucketIndex的话，就可以避免并发写入的问题，原因如下：

![image-20230410144942644](./img/image-20230410144942644.png)

并发写的时候方案有两种，双写和设置虚拟文件，现在好像只采用了双写的方案

![image-20230410145107517](./img/image-20230410145107517.png)



### 双写方案



```java
/**
 * Update strategy for (consistent hashing) bucket index
 * If updates to file groups that are under clustering are identified, then generate
 * two same records for each update, routing to both old and new file groups
 */
public class SparkConsistentBucketDuplicateUpdateStrategy<T extends HoodieRecordPayload<T>> extends UpdateStrategy<T, HoodieData<HoodieRecord<T>>> {

  private static final Logger LOG = LogManager.getLogger(SparkConsistentBucketDuplicateUpdateStrategy.class);

  public SparkConsistentBucketDuplicateUpdateStrategy(HoodieEngineContext engineContext, HoodieTable table, Set<HoodieFileGroupId> fileGroupsInPendingClustering) {
    super(engineContext, table, fileGroupsInPendingClustering);
  }
}
```

### Merge/Split

merge有两种方案，按照索引从中间拆分，按照实际数据量拆分。这里目前只实现了最简单的，从索引中间拆分。因为这样最终也会实现平衡

![image-20230410145759337](./img/image-20230410145759337.png)



# 2. 缺省字段补全

[JIRA](https://issues.apache.org/jira/browse/HUDI-4781)

- [[HUDI-4781](https://issues.apache.org/jira/browse/HUDI-4781)] - Allow omit metadata fields for hive sync





# 3. MetaServer(EXPERIMENTAL)

## 参考资料

[相关RFC](https://cwiki.apache.org/confluence/display/HUDI/RFC-36%3A+HUDI+Metastore+Server)



![image-20230413093506658](./img/image-20230413093506658.png)

## 背景

1. 缺乏统一的管理视图
2. 使用 Flie Listing 操作是一个开销非常昂贵的
3. Hive MetaStore 中缺少 Timeline属性，不满足 Hudi的要求



## Service

### Table

 is for table related request. To client, it exposes API about database and table CRUD.

### Partition

is for partition related requests. To client, it exposes API about CRUD:

- support multiple ways of reading, like checking the partition's existence, getting partition info, getting partitions statisfys a specific condition(partition pruning).
- creating or updating API cannot be invoked directly, only a new commit completion can trigger it.
- dropping a partition not only deletes the partition and files at metadata level, but also triggers a clean action to do the physical clean that deletes the data on the file system.

### Timeline

 is for timeline related request. The basic functions are the same as the `HoodieActiveTimeline`.

 To client, it exposes API that

- the instant CRUD
- get the metadata of an instant

### Snapshot

适用于文件级相关请求。旧方法是，写入客户端通过文件列表获取路径下的所有文件，然后通过将文件组织为文件组中的文件切片来构建文件系统视图。之后，获取每个文件组中最新的文件切片进行读写。快照服务只是给客户端最后一步的结果。

is for file level related request. The old way is that a write client gets all files under the path by file listing, and then builds the file system view by organizing the files as file slices in file groups. After that, get the latest file slice in each file group to read and write. Snapshot service just gives the client the result of the last step.

 To client, it exposes API that

- get the latest snapshot of a partition without multiple file version
- get the incremental files after a specified timestamp, for incremental reading
- get all files of a partition, the same as the files under the filesystem, which is used to clean
- update the snapshot when a new commit completes, it cannot be invoked by client directly.



## Storage

根据存储的元数据类型，存储分为四个部分：table、partition、timeline、snapshot。表由数据库、表、模式的元数据组成。分区只是分区元数据。时间线包含即时和每个即时的元数据。快照有文件状态。

Table consists of the metadata of database, table, schema. 

Partition is just partition metadata. 

Timeline contains instant and the metadata of each instant.

 Snapshot has file statuses.



可更换存储引擎，目前为Mysql



## 4. 
