# Flink-Hudi 使用指南

## 1. 介绍
- ### 1.1. 什么是 Flink
- ### 1.2. 什么是 Hudi
- ### 1.3. Flink 与 Hudi 的结合的价值

## 2. QuickStart
- ### 2.1. Hudi connector 编译打包
- ### 2.2. Flink 写入与查询

## 3. 集群搭建
- ### 3.1. Hadoop 集群搭建
- ### 3.2. Flink 提交到 Yarn集群
- ### 3.3. Hive 环境搭建

## 4. Apache Hudi 基础概念
- ### 3.1. 表类型(COW/MOR)
- ### 3.2. 索引(Index)
- ### 3.3. 文件管理与存储布局(File Layout)
- ### 3.4. 时间概念：提交时间、分区时间(Timeline)
- ### 3.5. 写类型(Insert/Upsert/BulkInsert)
- ### 3.6. Payload

## 5. Apache Hudi 主要特性
- ### 5.1. Hudi 索引策略
- ### 5.2. 增量查询与读优化
- ### 5.3. MOR 表与 Compaction
- ### 5.4. 时间旅行查询(TimeTravel)
- ### 5.5. 数据清理 (Cleaning)
- ### 5.6. 原子性写操作
- ### 5.7. Clustering
- ### 5.8. Hive Sync

## 6. Hudi 的数据分区与桶策略
- ### 6.1. 分区方法介绍
- ### 6.2. 分区键与桶键的选择策略
- ### 6.3. 优化数据布局

## 7. 官方RFC与源码分析
- ### 7.1. 写入流程分析
- ### 7.2. Hudi与Flink整合(RFC-24)
- ### 7.3. 索引 
  - #### 7.3.1. 桶索引 BucketIndex(RFC-29)
  - #### 7.3.2. 动态桶索引 ConsistentHashIndex(RFC-42)
  - #### 7.3.3. 布隆索引 BloomIndex
  - #### 7.3.4. 状态索引 FlinkStateIndex(RFC-24)
- ### 7.4. Schema Evolution(RFC-33)

## 8. Hudi 与大数据生态的整合
- ### 8.1. 与 Hive 的整合
- ### 8.2. 与 Spark 的整合
- ### 8.3. 与 Presto 的整合

## 9. 性能优化与调优
- ### 9.1. 生产配置与操作步骤
- ### 9.2. 写操作的性能提升
- ### 9.3. 查询性能的调优
- ### 9.4. Hudi 集群资源管理策略

## 10. 生产问题分析
- ### 10.1. 当天数据查询不到
- ### 10.2. 主键脏数据包含 ':' 导致任务重启
- ### 10.3. 非分区表 InsertOverWrite 导致整张表数据被删除

## 11. 其他数据湖组件
- ### 11.1. Iceberg
- ### 11.2. Paimon

