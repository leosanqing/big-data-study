# 为什么要读 RFC

RFC是官方设计某些功能或者特性时的思路和讨论, 短短的文章凝结了所有的精华部分,阅读这个就能明白为什么当初是这样设计的,设计的时候考虑了什么因素

而且看这个也能知道后期对这个功能进行了什么优化

真正做到,知其然,知其所以然, 知其所以不然

# 读哪些

## Flink 

相关的RFC 建议读取的有这几个

- 13 [ Integrate Hudi with Flink](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=141724520) : Hudi 第一版整合 Flink
- 24 [Hoodie Flink Writer Proposal](https://cwiki.apache.org/confluence/display/HUDI/RFC-24%3A+Hoodie+Flink+Writer+Proposal)  基于第一版的四个瓶颈,优化与 Flink 的整合
- 29 [ Hash Index](https://cwiki.apache.org/confluence/display/HUDI/RFC+-+29%3A+Hash+Index) 新的索引-桶索引,也是 Flink 目前唯二生产能用的索引.另外一个在 RFC-24 的时候引入的 Flink State
- 42 [ Consistent Hashing Index]: 一致性 Hash 索引 针对上面 Hash 索引桶数无法动态缩扩设计的

## Spark 

spark 相关的建议重点读这几个

- 8 Metadata based Record Index



## Presto

- 44 Hudi Connector for Presto