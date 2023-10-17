# 问题



从官方 RFC 的设计中,我们能找到以下几个问题的答案

1. CKP 相关
   1. 为啥 Hudi 要依赖 CKP
   2. 为啥一般只有做了 CKP,才能查到数据
   3. 为啥没做 CKP,我也能查到数据
   4. 使用 CKP 触发 instant 有什么弊端
   5. 如果没有 ckp,instant 会有哪些状态
   
1. 状态是怎么设计的
2. 为啥启动之后会有 BucketAssigner 算子
3. 为啥启动之后,有的是 StreamWrite,有的是 BucketWrite

[官方 RFC-13 链接](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=141724520)

[官方 RFC-24 链接](https://cwiki.apache.org/confluence/display/HUDI/RFC-24%3A+Hoodie+Flink+Writer+Proposal)


# 背景

![image2020-10-8_22-3-56.png](https://cwiki.apache.org/confluence/download/attachments/141724520/image2020-10-8_22-3-56.png?version=1&modificationDate=1602165838000&api=v2)

通过看官方 RFC 可以知道,一开始设计的 Flink 集成(RFC-13)有四个瓶颈

- `InstantGeneratorOperator` (https://github.com/apache/hudi/pull/2430)
  - 由于一批只有一个 CKP,所以这个生成就只能单线程,这就导致成了高吞吐的瓶颈 并且网络IO 的压力也会增加
  - The InstantGeneratorOperator is parallelism 1, which is a limit for high-throughput consumption; because all the split inputs drain to a single thread, the network IO would gains pressure too
- The WriteProcessOperator handles inputs by partition, that means, within each partition write process, the BUCKETs are written one by one, the FILE IO is limit to adapter to high-throughput inputs(https://github.com/apache/hudi/pull/2506)
  - 写入算子,是分区级别,写入只能一个桶一个桶来,文件 IO 就成了瓶颈
- Currently we buffer the data by checkpoints, which is too hard to be robust for production, the checkpoint function is blocking and should not have IO operations. https://github.com/apache/hudi/pull/2553
- The FlinkHoodieIndex is only valid for a per-job scope, it does not work for existing bootstrap data or for different Flink jobs. https://github.com/apache/hudi/pull/2581


# Coordinator



# StreamWriteOperatorCoordinator

**This coordinator starts a new instant when a new checkpoint starts. It commits the instant when all the** operator tasks write the buffer successfully for a round of checkpoint.*

写入算子协调器,作用:

1. 初始化instant,并且每当开启一个新的 ckp,创建一个新的 instant
2. 当一个ckp 周期内的所有算子的缓存之后,提交这个 instant
3. 提交完成之后,compaction,syncHive.最后回调 ckp,告诉这个ckp 完成


## Instant 提前生成

如果一个ckp周期内没有数据流入,就重用这个 instant,重用的代码,这就会导致一个问题,产险生产就遇到了.很长一段时间没有流入,数据就跨天了

`org.apache.hudi.sink.StreamWriteOperatorCoordinator#sendCommitAckEvents`

这个方法上面说的比较清楚

产险生产就遇到这个问题,数据 5 个小时没有流入(22:00-3:00) ,后面数据流入之后,就写到了前一天的 Instant, 查询第二天的数据,就会导致一部分数据丢失

# 为啥 Hudi 要依赖 CKP

coodinator



