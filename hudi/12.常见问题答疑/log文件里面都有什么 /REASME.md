# 写入数据
```sql

CREATE TABLE mor_1009_001_achived(
  uuid VARCHAR(20),
  name VARCHAR(10),
  age INT,
  ts TIMESTAMP(3),
  `partition` VARCHAR(20)
)
PARTITIONED BY (`partition`)
WITH (
  'connector' = 'hudi',
  'path' = 'hdfs://bigdata01:9000/hudi_test/mor_1009_001_achived',
  'table.type' = 'MERGE_ON_READ',  -- If MERGE_ON_READ, hive query will not have output until the parquet file is generated
  
  'hive_sync.enable' = 'true',     -- . To enable hive synchronization
  'hive_sync.mode' = 'hms',        -- Required. Setting hive sync mode to hms, default hms
  'hive_sync.db' = 'hudi_test',
  'hive_sync.table' = 'mor_1009_001_achived',
  'hive_sync.metastore.uris' = 'thrift://bigdata01:9083' -- Required. The port need set on hive-site.xml
);

-- 重复三次,一共插入三批数据,记得修改 name 版本
INSERT INTO mor_1009_001_achived VALUES
                                     ('id1','Danny',23,TIMESTAMP '1970-01-01 00:00:01','par1'),
                                     ('id2','Stephen',33,TIMESTAMP '1970-01-01 00:00:02','par1'),
                                     ('id3','Julian',53,TIMESTAMP '1970-01-01 00:00:03','par2'),
                                     ('id4','Fabian',31,TIMESTAMP '1970-01-01 00:00:04','par2'),
                                     ('id5','Sophia',18,TIMESTAMP '1970-01-01 00:00:05','par3'),
                                     ('id6','Emma',20,TIMESTAMP '1970-01-01 00:00:06','par3'),
                                     ('id7','Bob',44,TIMESTAMP '1970-01-01 00:00:07','par4'),
                                     ('id8','Han',56,TIMESTAMP '1970-01-01 00:00:08','par4');


```
# 代码
```xml
<dependencies>
    <!-- Hudi dependencies -->
    <dependency>
        <groupId>org.apache.hudi</groupId>
        <artifactId>hudi-common</artifactId>
        <version>0.13.1</version> <!-- 使用最新的Hudi版本 -->
    </dependency>
    <!-- Avro dependencies -->
    <dependency>
        <groupId>org.apache.avro</groupId>
        <artifactId>avro</artifactId>
        <version>1.10.0</version> <!-- 使用与Hudi兼容的Avro版本 -->
    </dependency>
    <!-- Hadoop dependencies -->
    <dependency>
        <groupId>org.apache.hadoop</groupId>
        <artifactId>hadoop-common</artifactId>
        <version>3.2.2</version>
    </dependency>
    <dependency>
        <groupId>org.apache.hadoop</groupId>
        <artifactId>hadoop-hdfs</artifactId>
        <version>3.2.2</version>
    </dependency>
</dependencies>

```
```java

import com.google.common.collect.Lists;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.apache.hudi.common.model.HoodieLogFile;
import org.apache.hudi.common.model.HoodieRecord;
import org.apache.hudi.common.table.log.HoodieLogFileReader;
import org.apache.hudi.common.table.log.LogReaderUtils;
import org.apache.hudi.common.table.log.block.HoodieAvroDataBlock;
import org.apache.hudi.common.table.log.block.HoodieLogBlock;
import org.apache.hudi.common.table.log.block.HoodieLogBlock.HoodieLogBlockType;
import org.apache.hudi.common.util.ClosableIterator;
import org.apache.hudi.org.apache.avro.Schema;

import java.io.IOException;
import java.util.List;

import static org.apache.hudi.common.model.HoodieRecord.HoodieRecordType.AVRO;

public class HudiLogReader {

    public static void main(String[] args) throws IOException {
        String hdfsUri = "hdfs://bigdata01:9000"; // 根据你的设置来修改
        String logFilePath = "/hudi_test/mor_1009_001_achived/par1/.28736613-e40b-4c6e-91d6-704b13b375c7_20231008155156008.log.1_0-2-0"; // 你的Hudi log文件的HDFS路径

        Configuration conf = new Configuration();
        conf.set("fs.defaultFS", hdfsUri);
        FileSystem fs = FileSystem.get(conf);


        List<HoodieLogFile> logFiles = Lists.newArrayList(new HoodieLogFile(new Path(logFilePath)));

        String basePath = "/hudi_test/mor_1009_001_achived";
        Schema schema = LogReaderUtils.readLatestSchemaFromLogFiles(basePath, logFiles, conf);
        try (HoodieLogFileReader reader = new HoodieLogFileReader(fs, new HoodieLogFile(new Path(logFilePath)), schema, 1024,  false)) {
            while (reader.hasNext()) {
                HoodieLogBlock block = reader.next();
                if (block.getBlockType() == HoodieLogBlockType.AVRO_DATA_BLOCK) {
                    HoodieAvroDataBlock avroDataBlock = (HoodieAvroDataBlock) block;
                    ClosableIterator<HoodieRecord<Object>> recordIterator = avroDataBlock.getRecordIterator(AVRO);
                    while (recordIterator.hasNext()) {
                        HoodieRecord<Object> next = recordIterator.next();
                        // String recordKey = next.getRecordKey();
                        Object data = next.getData();
                        System.out.println(data);

                        System.out.println(next);
                    }
                }
            }
        }
    }

}

```

# 结果
```shell
{"_hoodie_commit_time": "20231008155156008", "_hoodie_commit_seqno": "20231008155156008_0_1", "_hoodie_record_key": "id1", "_hoodie_partition_path": "par1", "_hoodie_file_name": "28736613-e40b-4c6e-91d6-704b13b375c7", "uuid": "id1", "name": "Danny", "age": 23, "ts": 1000, "partition": "par1"}
HoodieRecord{key=null, currentLocation='null', newLocation='null'}
{"_hoodie_commit_time": "20231008155156008", "_hoodie_commit_seqno": "20231008155156008_0_4", "_hoodie_record_key": "id2", "_hoodie_partition_path": "par1", "_hoodie_file_name": "28736613-e40b-4c6e-91d6-704b13b375c7", "uuid": "id2", "name": "Stephen", "age": 33, "ts": 2000, "partition": "par1"}
HoodieRecord{key=null, currentLocation='null', newLocation='null'}
{"_hoodie_commit_time": "20231008155500387", "_hoodie_commit_seqno": "20231008155500387_0_3", "_hoodie_record_key": "id1", "_hoodie_partition_path": "par1", "_hoodie_file_name": "28736613-e40b-4c6e-91d6-704b13b375c7", "uuid": "id1", "name": "Danny1", "age": 23, "ts": 1000, "partition": "par1"}
HoodieRecord{key=null, currentLocation='null', newLocation='null'}
{"_hoodie_commit_time": "20231008155500387", "_hoodie_commit_seqno": "20231008155500387_0_4", "_hoodie_record_key": "id2", "_hoodie_partition_path": "par1", "_hoodie_file_name": "28736613-e40b-4c6e-91d6-704b13b375c7", "uuid": "id2", "name": "Stephen1", "age": 33, "ts": 2000, "partition": "par1"}
HoodieRecord{key=null, currentLocation='null', newLocation='null'}
{"_hoodie_commit_time": "20231008161022270", "_hoodie_commit_seqno": "20231008161022270_0_3", "_hoodie_record_key": "id1", "_hoodie_partition_path": "par1", "_hoodie_file_name": "28736613-e40b-4c6e-91d6-704b13b375c7", "uuid": "id1", "name": "Danny2", "age": 23, "ts": 1000, "partition": "par1"}
HoodieRecord{key=null, currentLocation='null', newLocation='null'}
{"_hoodie_commit_time": "20231008161022270", "_hoodie_commit_seqno": "20231008161022270_0_4", "_hoodie_record_key": "id2", "_hoodie_partition_path": "par1", "_hoodie_file_name": "28736613-e40b-4c6e-91d6-704b13b375c7", "uuid": "id2", "name": "Stephen2", "age": 33, "ts": 2000, "partition": "par1"}
HoodieRecord{key=null, currentLocation='null', newLocation='null'}
```

# 结论
从上面结果可以看出来, 上面那个文件一共做了三次提交, 一共写了三个块, 每个块每次写入两条数据

所以不同批次的 commit 也可能会提交到同一个 log 文件中