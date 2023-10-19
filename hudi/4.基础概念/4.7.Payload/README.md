# 前言

Hudi 系列文章在这个这里查看 https://github.com/leosanqing/big-data-study

Hudi 官网的介绍 https://hudi.apache.org/cn/docs/next/metadata/

Payload 是一个非常关键的步骤,决定了数据怎么去重合并等步骤

# 问题

1. Payload 是什么,在哪里起作用
2. Payload 中的关键方法,作用是什么

# 概念

Payload 这个不知道怎么翻译成中文更准确, 字面意思是 负载,只能从他做的事情和功能上来理解这个概念了

我们看官方给的Upsert 的流程

payload 在 Upsert 中起作用的阶段是 

1. Precombining 同一批数据合并的时候
2. COW 表Write 阶段的 Update 流程的一部分 
3. MOR 表 Compaction以及 Read 阶段

![upsert_path.png](./img/upsert_path-0935f9caee7f799b5ba273d3b077c87d.png)

所以 Payload 实际就是告诉 Hudi, 当我Update 的时候遇到相同的 RecordKey 的字段时候,我要做什么

# 参数

## Flink

| Config Name   | Default                                                      | Description                                                  |
| ------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| payload.class | org.apache.hudi.common.model.EventTimeAvroPayload (Optional) | Payload class used. Override this, if you like to roll your own merge logic, when upserting/inserting. This will render any value set for the option in-effective  `Config Param: PAYLOAD_CLASS_NAME` |

# 现有类型

举例说明一下, 现在 我们有一张表,字段是这些,`ts`字段为排序字段

```json
{
  [
    {"name":"id","type":"string"},
    {"name":"ts","type":"long"},
    {"name":"name","type":"string"},
    {"name":"price","type":"string"}
  ]
}
```

当前已写入数据为

```shell
id      ts      name    price
1       2       name_2  price_2
```



准备写入数据为

    id      ts      name    price
    1       1       name_1    price_1

## OverwriteWithLatestAvroPayload

`hoodie.datasource.write.payload.class=org.apache.hudi.common.model.OverwriteWithLatestAvroPayload`

最新的一条覆盖之前的, 这里最新的一条只最后一条接收到的.实际上应该没有人使用这个, 因为网络原因,你并不能保证到达的顺序是你想要的顺序. 可能旧数据覆盖了新数据

最终结果为

    id      ts      name    price
    1       1       name_1    price_1

## DefaultHoodieRecordPayload

`hoodie.datasource.write.payload.class=org.apache.hudi.common.model.DefaultHoodieRecordPayload`

这里就是按照 precombine 字段进行排序, 后面的覆盖之前的

最终结果为

```
id      ts      name    price
1       2       name_2  price_2
```



## EventTimeAvroPayload

`hoodie.datasource.write.payload.class=org.apache.hudi.common.model.EventTimeAvroPayload`

Flink 写入特有的 Payload 策略. 根据 EventTime 进行排序, 新的覆盖旧的

## OverwriteNonDefaultsWithLatestAvroPayload

`hoodie.datasource.write.payload.class=org.apache.hudi.common.model.OverwriteNonDefaultsWithLatestAvroPayload`

这个与上面`OverwriteWithLatestAvroPayload` 非常相似,只是更新字段的时候, 会合并两个字段的值, 新数据不是默认的的字段 才会覆盖

```sql
id      ts      name    price
1       1               price_1


-- 覆盖之后就变成了
id      ts      name    price
1       1      name_2   price_1
```



## PartialUpdateAvroPayload

部分字段更新 `hoodie.datasource.write.payload.class=org.apache.hudi.common.model.PartialUpdateAvroPayload`

```sql
    -- 现有数据
    id      ts      name    price
    1       2       name_1  null
    
    -- 预写入数据
    id      ts      name    price
    1       1       null    price_1
    
    -- 写入之后结果
    id      ts      name    price
    1       2       name_1  price_1
    
    
```

我们可以看到这个和之前的还是有很大的区别,这里只更新部分字段,并且根据 ts 字段排序



但是上面的这个有个问题, 假如我就是要把 name 变成 null, 实际是变不成的,写入数据就有误了

## 自定义 Payload

如果上面的 payload 都不满足需求, 我们可以自定义 Payload, 比如我们 OGG 发送的字段并不是传统的 OGG格式,

```json
-- 传统 OGG

{
  "before": {
    "id": 111,
    "name": "scooter",
    "description": "Big 2-wheel scooter",
    "weight": 5.18
  },
  "after": {
    "id": 111,
    "name": "scooter",
    "description": "Big 2-wheel scooter",
    "weight": 5.15
  },
  "op_type": "U",
  "op_ts": "2020-05-13 15:40:06.000000",
  "current_ts": "2020-05-13 15:40:07.000000",
  "primary_keys": [
    "id"
  ],
  "pos": "00000000000000000000143",
  "table": "PRODUCTS"
}

--- 缺失的 OGG 格式
{
  "after": {
    "id": 111,
    "name": "scooter"
  },
  "op_type": "U",
  "op_ts": "2020-05-13 15:40:06.000000",
  "current_ts": "2020-05-13 15:40:07.000000",
  "primary_keys": [
    "id"
  ],
  "pos": "00000000000000000000143",
  "table": "PRODUCTS"
}

--- 我们可以看到,传统的 OGG 格式包含三部分, befor, after, 其他
但是我们 OGG-json 是缺失的,只有 after,没有 before,并且 after 也只包含部分更新字段,并不是全字段
这种情况下,如果我们想实现更新,会变得非常的麻烦,上述的 Payload 都无法满足需求,这个时候我们就要自定义 Payload 了
```



# 初始化

Flink payload 初始化逻辑在 `org.apache.hudi.table.HoodieTableSink#getSinkRuntimeProvider` 

![image-20231017113332194](./img/image-20231017113332194.png)

![image-20231017145825312](./img/image-20231017145825312.png)

不论是上面的桶索引还是 FlinkState, 走上面两个路径都会调用下面的方法

![image-20231017150205242](./img/image-20231017150205242.png)

![image-20231017150257890](./img/image-20231017150257890.png)

![image-20231017150406442](./img/image-20231017150406442.png)

![image-20231017151249050](./img/image-20231017151249050.png)



不过上面仅仅支持做一些初始化,并没有真正的创建, 真正创建是下面这里调用`org.apache.hudi.sink.utils.PayloadCreation#createPayload`

![image-20231017150948410](./img/image-20231017150948410.png)

![image-20231017152115312](./img/image-20231017152115312.png)

# 自定义 Payload



![image-20231017170418033](./img/image-20231017170418033.png)

Payload 需要关注的主要是上面三个方法

1. preCombine 一批数据相同的时候,合并调用
2. combineAndGetUpdateValue MOR compaction 或者Read 的时候调用,COW 写入的时候调用
3. getOrderingValue 排序规则



我们只需要在规则中改写上面三个方法, 就能获取到旧值,然后做字段合并逻辑就好



更详细的参考 7.2.2 Payload 源码分析这里 



# 总结

1. Payload 是什么,在哪里起作用
   1. Payload 决定了重复数据怎么处理
   2. precombine, COW 表写入,MOR 表读取以及 compaction 阶段 会调用
2. Payload 中的关键方法,作用是什么
   1. preCombine: 同一批数据去重
   2. combineAndGetUpdateValue MOR compaction 或者Read 的时候调用,COW 写入的时候调用
   3. getOrderingValue 排序规则