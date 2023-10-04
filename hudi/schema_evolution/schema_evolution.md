# SQL 方式

1. 创建 MOR 表

 ```sql
 CREATE TABLE hudi_trips_mor_new (
 rowId STRING,
 partitionId STRING,
 preComb LONG,
 name STRING,
 versionId STRING,
 intToLong LONG
 ) USING hudi
 OPTIONS (
 'hoodie.table.name' = 'hudi_trips_mor_new',
 'hoodie.datasource.write.recordkey.field' = 'rowId',
 'hoodie.datasource.write.partitionpath.field' = 'partitionId',
 'hoodie.datasource.write.precombine.field' = 'preComb',
 'hoodie.datasource.write.table.type' = 'MERGE_ON_READ',
 'path' = 'file:///tmp/hudi_trips_mor_new'
 );
 
 
 INSERT INTO hudi_trips_mor_new
 SELECT rowId, partitionId, preComb, name, versionId, CAST(intToLong AS LONG), null as newField
 FROM hudi_trips_cow_new;
 
 
 ALTER TABLE hudi_trips_mor_new ADD COLUMN newField2 decimal;
 ```

2. 插入数据

   ```sql
   INSERT INTO hudi_trips_mor_new
   VALUES ('row_1', 'part_0', 0, 'bob', 'v_0', 0),
          ('row_2', 'part_0', 0, 'john', 'v_0', 0),
          ('row_3', 'part_0', 0, 'tom', 'v_0', 0);
   
   
   -- 查询数据
   SELECT rowId, partitionId, preComb, name, versionId, intToLong
   FROM hudi_trips_mor_new;
   
   ```

   

3. Schema 变更

   ```sql
   
   ALTER TABLE hudi_trips_mor_new
   ADD COLUMN newField STRING;
   
   ALTER TABLE hudi_trips_mor_new
   ALTER COLUMN intToLong TYPE decimal;
   
   ```

   

4. 插入新数据

   ```sql
   INSERT INTO hudi_trips_mor_new
   VALUES ('row_2', 'part_0', 5, 'john', 'v_3', 3, 'newField_1'),
          ('row_5', 'part_0', 5, 'maroon', 'v_2', 2, 'newField_1'),
          ('row_9', 'part_0', 5, 'michael', 'v_2', 2, 'newField_1');
   ```

   

5. 查询数据及表结构

   ```sql
   SELECT rowId, partitionId, preComb, name, versionId, intToLong, newField
   FROM hudi_trips_mor_new;
   
   desc hudi_trips_mor_new;
   
   ```

   

# API方式



```scala
//  1. 首先，我们需要导入所需的库和模块，这些库和模块可以帮助我们在 Spark 中处理数据和配置 Hudi。以下是导入语句：
import org.apache.hudi.QuickstartUtils._   
import scala.collection.JavaConversions._   
import org.apache.spark.sql.SaveMode._   
import org.apache.hudi.DataSourceReadOptions._   
import org.apache.hudi.DataSourceWriteOptions._   
import org.apache.hudi.config.HoodieWriteConfig._   
import org.apache.spark.sql.types._   
import org.apache.spark.sql.Row   



// 2. 然后，我们需要定义一些变量，如表名、基本路径和模式。我们的模式定义了数据集的结构，包括字段名称、数据类型和是否允许空值：
val tableName = "hudi_trips_cow"  
val basePath = "file:///tmp/hudi_trips_cow"  
val schema = StructType( Array(  
    StructField("rowId", StringType,true),  
    StructField("partitionId", StringType,true),  
    StructField("preComb", LongType,true),  
    StructField("name", StringType,true),  
    StructField("versionId", StringType,true),  
    StructField("intToLong", IntegerType,true)  
))


// 3. 创建一些数据并将其写入 Hudi 表：
val data1 = Seq(Row("row_1", "part_0", 0L, "bob", "v_0", 0),  
            Row("row_2", "part_0", 0L, "john", "v_0", 0),  
            Row("row_3", "part_0", 0L, "tom", "v_0", 0))  
var dfFromData1 = spark.createDataFrame(data1, schema)  
dfFromData1.write.format("hudi").  
    options(getQuickstartWriteConfigs).  
    option(PRECOMBINE_FIELD_OPT_KEY.key, "preComb").  
    option(RECORDKEY_FIELD_OPT_KEY.key, "rowId").  
    option(PARTITIONPATH_FIELD_OPT_KEY.key, "partitionId").  
    option("hoodie.index.type","SIMPLE").  
    option(TABLE_NAME.key, tableName).  
    mode(Overwrite).  
    save(basePath) 

// 4. 读取并查看 Hudi 表：
var tripsSnapshotDF1 = spark.read.format("hudi").load(basePath)  
tripsSnapshotDF1.createOrReplaceTempView("hudi_trips_snapshot")  
spark.sql("desc hudi_trips_snapshot").show()  
spark.sql("select rowId, partitionId, preComb, name, versionId, intToLong from hudi_trips_snapshot").show()  


// 5. 接下来，我们将定义一个新的模式，其中包含一个新的字符串字段和一个从整数类型更改为长整数类型的字段。我们将使用这个新模式来创建一些新的数据，并将其追加到我们的 Hudi 表中：
val newSchema = StructType( Array(  
    StructField("rowId", StringType,true),  
    StructField("partitionId", StringType,true),  
    StructField("preComb", LongType,true),  
    StructField("name", StringType,true),  
    StructField("versionId", StringType,true),  
    StructField("intToLong", LongType,true),  
    StructField("newField", StringType,true)  
))  
val data2 = Seq(Row("row_2", "part_0", 5L, "john", "v_3", 3L, "newField_1"),  
            Row("row_5", "part_0", 5L, "maroon", "v_2", 2L, "newField_1"),  
            Row("row_9", "part_0", 5L, "michael", "v_2", 2L, "newField_1"))  
var dfFromData2 = spark.createDataFrame(data2, newSchema)  
dfFromData2.write.format("hudi").  
    options(getQuickstartWriteConfigs).  
    option(PRECOMBINE_FIELD_OPT_KEY.key, "preComb").  
    option(RECORDKEY_FIELD_OPT_KEY.key, "rowId").  
    option(PARTITIONPATH_FIELD_OPT_KEY.key, "partitionId").  
    option("hoodie.index.type","SIMPLE").  
    option(TABLE_NAME.key, tableName).  
    mode(Append).  
    save(basePath) 

// 6. 最后，我们再次读取并查看 Hudi 表，以查看模式如何变化：
var tripsSnapshotDF2 = spark.read.format("hudi").load(basePath + "/*/*")  
tripsSnapshotDF2.createOrReplaceTempView("hudi_trips_snapshot")  
spark.sql("desc hudi_trips_snapshot").show()  
spark.sql("select rowId, partitionId, preComb, name, versionId, intToLong, newField from hudi_trips_snapshot").show()  


```



上面第三步执行报错

```shell
<console>:56: error: value key is not a member of String
           option(PRECOMBINE_FIELD_OPT_KEY.key, "preComb").
                                           ^
<console>:57: error: value key is not a member of String
           option(RECORDKEY_FIELD_OPT_KEY.key, "rowId").
                                          ^
<console>:58: error: value key is not a member of String
           option(PARTITIONPATH_FIELD_OPT_KEY.key, "partitionId").
                                              ^
```



```scala

// 第三步改为
dfFromData1.write.format("hudi").
  options(getQuickstartWriteConfigs).
  option("hoodie.datasource.write.precombine.field", "preComb").
  option("hoodie.datasource.write.recordkey.field", "rowId").
  option("hoodie.datasource.write.partitionpath.field", "partitionId").
  option("hoodie.table.name", tableName).
  mode(Overwrite).
  save(basePath)

// 第五步改为
var dfFromData2 = spark.createDataFrame(data2, newSchema)  
dfFromData2.write.format("hudi").  
    options(getQuickstartWriteConfigs).
  	option("hoodie.datasource.write.precombine.field", "preComb").
  	option("hoodie.datasource.write.recordkey.field", "rowId").
  	option("hoodie.datasource.write.partitionpath.field", "partitionId").
 		option("hoodie.table.name", tableName).
    option("hoodie.index.type","SIMPLE").  
    mode(Append).  
    save(basePath) 
```



# ERROR

如果不设置  hive  参数,会报如下错误

```xml
  <property>
    <name>hive.metastore.disallow.incompatible.col.type.changes</name>
    <value>false</value>
  </property>
```



```yaml
Caused by: org.apache.hadoop.hive.ql.metadata.HiveException: Unable to alter table. The following columns have types incompatible with the existing columns in their respective positions :
col
        at org.apache.hadoop.hive.ql.metadata.Hive.alterTable(Hive.java:634)
        at org.apache.hadoop.hive.ql.metadata.Hive.alterTable(Hive.java:612)
        at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
        at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
        at java.lang.reflect.Method.invoke(Method.java:498)
        at org.apache.spark.sql.hive.client.Shim_v2_1.alterTable(HiveShim.scala:1303)
        at org.apache.spark.sql.hive.client.HiveClientImpl.$anonfun$alterTableDataSchema$1(HiveClientImpl.scala:605)
        at scala.runtime.java8.JFunction0$mcV$sp.apply(JFunction0$mcV$sp.java:23)
        at org.apache.spark.sql.hive.client.HiveClientImpl.$anonfun$withHiveState$1(HiveClientImpl.scala:305)
        at org.apache.spark.sql.hive.client.HiveClientImpl.liftedTree1$1(HiveClientImpl.scala:236)
        at org.apache.spark.sql.hive.client.HiveClientImpl.retryLocked(HiveClientImpl.scala:235)
        at org.apache.spark.sql.hive.client.HiveClientImpl.withHiveState(HiveClientImpl.scala:285)
        at org.apache.spark.sql.hive.client.HiveClientImpl.alterTableDataSchema(HiveClientImpl.scala:586)
        at org.apache.spark.sql.hive.HiveExternalCatalog.$anonfun$alterTableDataSchema$1(HiveExternalCatalog.scala:693)
        at scala.runtime.java8.JFunction0$mcV$sp.apply(JFunction0$mcV$sp.java:23)
        at org.apache.spark.sql.hive.HiveExternalCatalog.withClient(HiveExternalCatalog.scala:102)
        ... 62 more
Caused by: InvalidOperationException(message:The following columns have types incompatible with the existing columns in their respective positions :
col)
        at org.apache.hadoop.hive.metastore.api.ThriftHiveMetastore$alter_table_with_environment_context_result$alter_table_with_environment_context_resultStandardScheme.read(ThriftHiveMetastore.java:59744)
        at org.apache.hadoop.hive.metastore.api.ThriftHiveMetastore$alter_table_with_environment_context_result$alter_table_with_environment_context_resultStandardScheme.read(ThriftHiveMetastore.java:59730)
        at org.apache.hadoop.hive.metastore.api.ThriftHiveMetastore$alter_table_with_environment_context_result.read(ThriftHiveMetastore.java:59672)
        at org.apache.thrift.TServiceClient.receiveBase(TServiceClient.java:88)
        at org.apache.hadoop.hive.metastore.api.ThriftHiveMetastore$Client.recv_alter_table_with_environment_context(ThriftHiveMetastore.java:1693)
        at org.apache.hadoop.hive.metastore.api.ThriftHiveMetastore$Client.alter_table_with_environment_context(ThriftHiveMetastore.java:1677)
        at org.apache.hadoop.hive.metastore.HiveMetaStoreClient.alter_table_with_environmentContext(HiveMetaStoreClient.java:373)
        at org.apache.hadoop.hive.ql.metadata.SessionHiveMetaStoreClient.alter_table_with_environmentContext(SessionHiveMetaStoreClient.java:322)
        at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
        at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
        at java.lang.reflect.Method.invoke(Method.java:498)
        at org.apache.hadoop.hive.metastore.RetryingMetaStoreClient.invoke(RetryingMetaStoreClient.java:173)
        at com.sun.proxy.$Proxy21.alter_table_with_environmentContext(Unknown Source)
        at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
        at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
        at java.lang.reflect.Method.invoke(Method.java:498)
        at org.apache.hadoop.hive.metastore.HiveMetaStoreClient$SynchronizedHandler.invoke(HiveMetaStoreClient.java:2327)
        at com.sun.proxy.$Proxy21.alter_table_with_environmentContext(Unknown Source)
        at org.apache.hadoop.hive.ql.metadata.Hive.alterTable(Hive.java:630)
        ... 78 more
```

