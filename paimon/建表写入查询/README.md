# 1. 建表

```sql
CREATE CATALOG paimon_catalog WITH (
    'type' = 'paimon',
    'warehouse' = 'hdfs:///paimon/test'
);

use catalog paimon_catalog;

-- create a word count table
CREATE TABLE word_count (
    word STRING PRIMARY KEY NOT ENFORCED,
    cnt BIGINT
);

-- create a word data generator table
CREATE TEMPORARY TABLE word_table (
    word STRING
) WITH (
    'connector' = 'datagen',
    'fields.word.length' = '1'
);


-- paimon requires checkpoint interval in streaming mode
SET 'execution.checkpointing.interval' = '10 s';

-- write streaming data to dynamic table
INSERT INTO word_count SELECT word, COUNT(*) FROM word_table GROUP BY word;





-- use tableau result mode
SET 'sql-client.execution.result-mode' = 'tableau';

-- switch to batch mode
RESET 'execution.checkpointing.interval';
SET 'execution.runtime-mode' = 'batch';

-- olap query the table
SELECT * FROM word_count;
```

# 2. 启动分层服务

```shell
./bin/flink run ./lib/fluss-flink-tiering-0.8-SNAPSHOT.jar \
    --fluss.bootstrap.servers bigdata01:9123 \
    --datalake.format paimon \
    --datalake.paimon.metastore hive \
    --datalake.paimon.uri thrift://bigdata01:9083 \
    --datalake.paimon.warehouse hdfs://bigdata01:9000/paimon/
```

