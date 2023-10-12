# 前言

Hudi 系列文章在这个这里查看 https://github.com/leosanqing/big-data-study

Hudi 官网的介绍 https://hudi.apache.org/docs/next/write_operations

WriteType 决定了数据怎么写入, 不同的写入类型处理逻辑不同,常用的一般有 Upsert, BulkInsert, Insert,delete



# 问题

1. Upsert 与 Insert 区别
2. Upsert 执行流程



# 支持的操作

先说一个概念 :

“heuristics”（启发式）通常指的是一种快速解决问题的方法或策略，可能不总是导致最佳结果，但在某些情况下可能足够好或接近最佳。在 Hudi 的上下文中，这可能与数据如何分布或写入存储有关，以便在查询时获得更好的性能。简而言之，它是一种优化数据写入和存储的方法。



## UPSERT

Update or Insert. 有相同的 recordKey 就更新,没有就插入

默认操作，通过查找索引，输入记录首先被标记为插入或更新。

记录最终在运行启发式后编写，以确定如何最好地将它们打包到存储中，以优化文件大小等内容。对于数据库更改捕获(CDC)等用例，建议使用此操作，因为输入几乎肯定包含更新。目标表永远不会显示重复项。

## INSERT

这个操作在启发式方法（heuristics）和文件大小方面与`upsert`非常相似，但是完全跳过了索引查找步骤。因此，在诸如日志去重（与下面提到的去重选项结合使用）这样的用例中，它比`upsert`快得多。这也适用于那些表可以容忍重复项，但仅需要Hudi的事务写入、增量抽取和存储管理能力的场景。



## BULK_INSERT

批量插入:

Upsert,Insert插入操作都将输入记录保存在内存中，以加快存储启发式(heuristics)计算速度，因此最初对Hudi表进行初始加载/引导可能很麻烦。批量插入提供了与插入(Insert)相同的语义，同时实现了基于排序的数据写入算法，该算法可以很好地扩展数百TB的初始负载。然而，这只是在超大数据量方面尽力做了最好的工作，而不是像Inserts/upserts那样可以保证文件大小。



## DELETE

Hudi通过允许用户指定不同的记录有效负载实现，支持对存储在Hudi表中的数据实现两种类型的删除。

- 软删除：保留记录键(RecordKey)，并置空所有其他字段的值。这可以通过确保适当的字段在表模式中为空，并在将这些字段设置为空后简单地 Upsert 插入表来实现。
- 硬删除：一种更强的删除形式是从表中物理删除记录的任何痕迹。这可以通过3种不同的方式实现。
  - 使用DataSource，将OPERATION_OPT_KEY设置为DELETE_OPERATION_OPT_VAL。这将删除正在提交的数据集中的所有记录
  - 使用DataSource，将PAYLOAD_CLASS_OPT_KEY设置为“org.apache.hudi.EmptyHoodieRecordPayload”。这将删除正在提交的数据集中的所有记录。
  - 使用DataSource或Hudi Streamer，将名为_hoodie_is_deleted的列添加到DataSet中。对于要删除的所有记录，此列的值必须设置为true，对于要upserts的任何记录，必须设置为false或为空。

## BOOTSTRAP

Hudi 支持将你现有的大表(如 hive 表)变成 Hudi 表,通过这个 bootstrap 操作. 总共有很多中方式实现, 具体可以参考这个文章去操作.[bootstrapping page](https://hudi.apache.org/docs/migration_guide) 

## INSERT_OVERWRITE

Insert_orverwrite 此操作用于重写输入中存在的所有分区。

对于批量 ETL 任务，这种操作可能比 `upsert` 更快，因为这些任务一次性重新计算整个目标分区（而不是增量地更新目标表）。这是因为，我们可以完全绕过 `upsert` 写路径中的索引、预组合和其他重新分区的步骤。如果你正在进行任何回填(backfill)或类似的操作，这会非常方便。

## INSERT_OVERWRITE_TABLE

这个操作可以用于出于任何原因重写整个表。根据配置的清理策略，Hudi 清理器最终会异步清理之前表快照的文件组。这个操作比发出明确的删除指令要快得多。

## DELETE_PARTITION


除了删除单个记录外，Hudi还支持使用此操作批量删除整个分区。可以使用配置 [`hoodie.datasource.write.partitions.to.delete`](https://hudi.apache.org/docs/configurations#hoodiedatasourcewritepartitionstodelete).  来删除特定的分区



# 写入步骤

The following is an inside look on the Hudi write path and the sequence of events that occur during a write.

1. Deduping(去重)
   1. 首先, 输入的数据可能在同一批内存在重复, 我们需要根据 字段 去重或者合并
2. Index Lookup(查询索引)
   1. 下一步,会根据定义的主键查询索引,确定他属于哪一个文件组
3. File Sizing(调整文件大小)
   1. 然后, 基于之前提交写入的文件的平均大小. Hudi 会创建一个计划(plan) 往小文件里面写入足够的数据 直到达到配置的文件大小的最大值,然后关闭文件
4. Partitioning(分区)
   1. 我们现在到了分区阶段, 决定将哪些更新和插入放入哪些文件组，或者是否创建新的文件组。
5. Write I/O(写入)
   1. 线下我们真正执行写入操作, 可能创建一个新的基础文件(base file, 如 parquet) 或者追加到 log 文件中, 或者对现有的 log 文件进行版本修改
6. Update Index
   1. 当写入完成,我们回过头去更新索引
7. Commit
   1. 最终自动提交这些变更. (暴露回调接口[callback notification](https://hudi.apache.org/docs/next/writing_data#commit-notifications) )Finally we commit all of these changes atomically. (A [callback notification](https://hudi.apache.org/docs/next/writing_data#commit-notifications) is exposed)
8. Clean(if needed)
   1. 提交之后, 如有必要,会调用 clean 服务
9. Compaction
   1. 如果是 MOR 表, compaction 可能会在线压缩,或者异步执行
10. Archive
    1. 最后, 我们会运行归档服务, 将过时的 timeline 移到 归档目录.

# 总结

1. Upsert 与 Insert 区别: upsert 和 Insert 流程基本一致, 只是 upsert 会有查询索引的步骤,Insert 不需要这个步骤
2. Upsert 执行流程: 最后三个步骤 是可选项,并不是每次提交都执行
   1. 批内去重
   2. 查询索引
   3. 调整文件大小
   4. 分区
   5. 写入
   6. 更新索引
   7. 提交
   8. clean 清理
   9. compaction压缩
   10. archived 归档

