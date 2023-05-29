# 背景

Upsert 是hudi 最重要的特性之一，根据主键查找，数据存在就更新，不存在就插入



# 流程

Upsert 操作主要分为下面几个步骤

1. 初始化 Hudi表（initTable）
2. 表语法校验  table.validateUpsertSchema();
3. 写前操作(目前只做了 设置操作类型为 upsert)
4. 创建 writeHandle
5. 触发upsert操作（最重要）
6. 更新index指标 updateIndexMetrics
7. 写完成之后操作 postWrite



```java
// org.apache.hudi.client.HoodieFlinkWriteClient#upsert

@Override
public List<WriteStatus> upsert(List<HoodieRecord<T>> records, String instantTime) {
    HoodieTable<T, List<HoodieRecord<T>>, List<HoodieKey>, List<WriteStatus>> table =
      initTable(WriteOperationType.UPSERT, Option.ofNullable(instantTime));
  table.validateUpsertSchema();
  preWrite(instantTime, WriteOperationType.UPSERT, table.getMetaClient());
  HoodieWriteMetadata<List<WriteStatus>> result;
  try (AutoCloseableWriteHandle closeableHandle = new AutoCloseableWriteHandle(records, instantTime, table)) {
    result = ((HoodieFlinkTable<T>) table).upsert(context, closeableHandle.getWriteHandle(), instantTime, records);
  }
  if (result.getIndexLookupDuration().isPresent()) {
    metrics.updateIndexMetrics(LOOKUP_STR, result.getIndexLookupDuration().get().toMillis());
  }
  return postWrite(result, instantTime, table);
}
```

# upsert 动作

```java
// org.apache.hudi.table.HoodieFlinkMergeOnReadTable#upsert
// org.apache.hudi.table.action.commit.delta.FlinkUpsertDeltaCommitActionExecutor#execute
// org.apache.hudi.table.action.commit.FlinkWriteHelper#write
// org.apache.hudi.table.action.commit.BaseFlinkCommitActionExecutor#execute

	@Override
  public HoodieWriteMetadata<List<WriteStatus>> execute(List<HoodieRecord<T>> inputRecords) {
    HoodieWriteMetadata<List<WriteStatus>> result = new HoodieWriteMetadata<>();

    List<WriteStatus> writeStatuses = new LinkedList<>();
    final HoodieRecord<?> record = inputRecords.get(0);
    // 1. 通过第一条找到 存储的信息，比如分区地址，fileId bucketType等 
    // 为啥只找第一条就够了，下面是 GPT 的回答
    /**
    在这段 Apache Hudi 源码中，execute() 方法是处理 upsert（插入和更新）操作的一部分。输入参数 List<HoodieRecord<T>> inputRecords 包含要进行 upsert 操作的记录列表。这段代码中的关键是理解这个列表中的记录已经被分配到相同的文件和 bucket。

		在 Hudi 的 upsert 操作中，数据首先通过 UpsertPartitioner 进行分区。UpsertPartitioner 将具有相同文件 ID 和分区路径的记录组合在一起。这意味着，当处理一个记录列表时，所有记录都已被分配到同一个文件和分区，因此它们具有相同的文件 ID 和分区路径。

		这就是为什么在这段代码中，只需要获取列表中的第一条记录（inputRecords.get(0)），就可以得到分区路径、文件 ID 和 bucket 类型信息。因为所有这些记录都被分配到相同的文件和分区，所以它们具有相同的这些属性。
    */
    final String partitionPath = record.getPartitionPath();
    final String fileId = record.getCurrentLocation().getFileId();
    // 为啥 InstanTime 是 I
    // 这里的 "I" 是一个特殊的即时时间值，表示记录是新插入的记录，而不是已经存在的记录。当新记录被插入时，它的即时时间被设置为 "I"
    final BucketType bucketType = record.getCurrentLocation().getInstantTime().equals("I")
        ? BucketType.INSERT
        : BucketType.UPDATE;
    handleUpsertPartition(
        instantTime,
        partitionPath,
        fileId,
        bucketType,
        inputRecords.iterator())
        .forEachRemaining(writeStatuses::addAll);
    setUpWriteMetadata(writeStatuses, result);
    return result;
  }
```

```java
// 根据上面bucketType，决定是调用 insert 还是 update

protected Iterator<List<WriteStatus>> handleUpsertPartition(
    String instantTime,
    String partitionPath,
    String fileIdHint,
    BucketType bucketType,
    Iterator recordItr) {
  try {
    if (this.writeHandle instanceof HoodieCreateHandle) {
      // During one checkpoint interval, an insert record could also be updated,
      // for example, for an operation sequence of a record:
      //    I, U,   | U, U
      // - batch1 - | - batch2 -
      // the first batch(batch1) operation triggers an INSERT bucket,
      // the second batch batch2 tries to reuse the same bucket
      // and append instead of UPDATE.
      return handleInsert(fileIdHint, recordItr);
    } else if (this.writeHandle instanceof HoodieMergeHandle) {
      return handleUpdate(partitionPath, fileIdHint, recordItr);
    } else {
      switch (bucketType) {
        case INSERT:
          return handleInsert(fileIdHint, recordItr);
        case UPDATE:
          return handleUpdate(partitionPath, fileIdHint, recordItr);
        default:
          throw new AssertionError();
      }
    }
  } catch (Throwable t) {
    String msg = "Error upsetting bucketType " + bucketType + " for partition :" + partitionPath;
    LOG.error(msg, t);
    throw new HoodieUpsertException(msg, t);
  }
}

```

## Insert



## Update

```java
// org.apache.hudi.table.action.commit.delta.BaseFlinkDeltaCommitActionExecutor#handleUpdate
@Override
public Iterator<List<WriteStatus>> handleUpdate(String partitionPath, String fileId, Iterator<HoodieRecord<T>> recordItr) {
  FlinkAppendHandle appendHandle = (FlinkAppendHandle) writeHandle;
  appendHandle.doAppend();
  List<WriteStatus> writeStatuses = appendHandle.close();
  return Collections.singletonList(writeStatuses).iterator();
}


// 
public void doAppend() {
  while (recordItr.hasNext()) {
    HoodieRecord record = recordItr.next();
    // 初始化一些必要信息，只根据第一个记录初始化一次
    init(record);
    // 数据刷入磁盘，如果有必要
    flushToDiskIfRequired(record, false);
    writeToBuffer(record);
  }
  appendDataAndDeleteBlocks(header, true);
  estimatedNumberOfBytesWritten += averageRecordSize * numberOfRecords;
}



  /**
   * Checks if the number of records have reached the set threshold and then flushes the records to disk.
   */
  private void flushToDiskIfRequired(HoodieRecord record, boolean appendDeleteBlocks) {
    if (numberOfRecords >= (int) (maxBlockSize / averageRecordSize)
        || numberOfRecords % NUMBER_OF_RECORDS_TO_ESTIMATE_RECORD_SIZE == 0) {
      averageRecordSize = (long) (averageRecordSize * 0.8 + sizeEstimator.sizeEstimate(record) * 0.2);
    }

    // 如果达到了阈值，将数据写入磁盘
    if (numberOfRecords >= (int) (maxBlockSize / averageRecordSize)) {
      // Recompute averageRecordSize before writing a new block and update existing value with
      // avg of new and old
      LOG.info("Flush log block to disk, the current avgRecordSize => " + averageRecordSize);
      // Delete blocks will be appended after appending all the data blocks.
      appendDataAndDeleteBlocks(header, appendDeleteBlocks);
      estimatedNumberOfBytesWritten += averageRecordSize * numberOfRecords;
      numberOfRecords = 0;
    }
  }
```





```java
// org.apache.hudi.io.HoodieAppendHandle#appendDataAndDeleteBlocks
protected void appendDataAndDeleteBlocks(Map<HeaderMetadataType, String> header, boolean appendDeleteBlocks) {
  try {
    header.put(HoodieLogBlock.HeaderMetadataType.INSTANT_TIME, instantTime);
    header.put(HoodieLogBlock.HeaderMetadataType.SCHEMA, writeSchemaWithMetaFields.toString());
    List<HoodieLogBlock> blocks = new ArrayList<>(2);
    // 如果有新数据，就新建一个 DataBlock
    if (recordList.size() > 0) {
      String keyField = config.populateMetaFields()
          ? HoodieRecord.RECORD_KEY_METADATA_FIELD
          : hoodieTable.getMetaClient().getTableConfig().getRecordKeyFieldProp();

      blocks.add(getBlock(config, pickLogDataBlockFormat(), recordList, header, keyField));
    }

    if (appendDeleteBlocks && recordsToDelete.size() > 0) {
      // 如果要删除数据并且有数据，就新建一个DeleteBlock
      blocks.add(new HoodieDeleteBlock(recordsToDelete.toArray(new DeleteRecord[0]), header));
    }

    if (blocks.size() > 0) {
      AppendResult appendResult = writer.appendBlocks(blocks);
      processAppendResult(appendResult, recordList);
      recordList.clear();
      if (appendDeleteBlocks) {
        recordsToDelete.clear();
      }
    }
  } catch (Exception e) {
    throw new HoodieAppendException("Failed while appending records to " + writer.getLogFile().getPath(), e);
  }
}
```

## AppendBlock

```java
@Override
public AppendResult appendBlocks(List<HoodieLogBlock> blocks) throws IOException, InterruptedException {
  // Find current version
  HoodieLogFormat.LogFormatVersion currentLogFormatVersion =
      new HoodieLogFormatVersion(HoodieLogFormat.CURRENT_VERSION)
  FSDataOutputStream originalOutputStream = getOutputStream();
  long startPos = originalOutputStream.getPos();
  long sizeWritten = 0;
  // HUDI-2655. here we wrap originalOutputStream to ensure huge blocks can be correctly written
  FSDataOutputStream outputStream = new FSDataOutputStream(originalOutputStream, new FileSystem.Statistics(fs.getScheme()), startPos);
  for (HoodieLogBlock block: blocks) {
  	XXXX
  }
  // Flush all blocks to disk
  flush();

  AppendResult result = new AppendResult(logFile, startPos, sizeWritten);
  // roll over if size is past the threshold
  // 如果出错了，或者文件大小超过了阈值，就要滚动创建一个新文件
  rolloverIfNeeded();
  return result;
}
```

## RollOver日志文件滚动创建

 当写入的文件大小超过了阈值，就要新建一个文件

计算下一个文件名的方法在这个类里面

```java
public HoodieLogFile rollOver(FileSystem fs, String logWriteToken) throws IOException {
  String fileId = getFileId();
  String baseCommitTime = getBaseCommitTime();
  Path path = getPath();
  String extension = "." + FSUtils.getFileExtensionFromLog(path);
  int newVersion = FSUtils.computeNextLogVersion(fs, path.getParent(), fileId, extension, baseCommitTime);
  return new HoodieLogFile(new Path(path.getParent(),
      FSUtils.makeLogFileName(fileId, extension, baseCommitTime, newVersion, logWriteToken)));
}
```

### 文件命名规则

//hdfs:xxx/user/aaaa/table/.0000000001-xxxx-xxxxx-xx-xxx-x-xx_20340407125707590.log.1_10-30-33

按照文件名下划线划分，第一个`_` 前的，.0000000001-xxxx-xxxxx-xx-xxx-x-xx 是`fileId`

第二个`20340407125707590`是`BaseCommitTime`时间戳

第三个 `1_10-30-33`是 rolloverLogWriteToken，这个是文件写入时自动生成的，用于保证文件写到一定大小，然后写入一个新的文件，规则如下:



在Hudi中，`rolloverLogWriteToken` 是用于标识Hudi增量日志文件版本的一种特殊的后缀格式。它由三个部分组成，分别是文件版本号、分区ID和分区版本号，格式如下：

```css
[version]-[partitionId]-[partitionVersion]
```

其中，`version` 是日志文件的版本号，从 0 开始递增；`partitionId` 是数据分区的唯一标识符，通常是分区的路径或哈希值等；`partitionVersion` 是分区的版本号，用于标识分区的数据版本，从 0 开始递增。

`rolloverLogWriteToken` 后缀的计算是在Hudi的增量日志文件滚动（`rollover`）时自动生成的。当一个日志文件达到一定大小或记录数时，Hudi会自动滚动日志文件，将当前的数据块和删除块写入到一个新的日志文件中。在这个过程中，Hudi会生成一个新的 `rolloverLogWriteToken` 后缀，并将其追加到新的日志文件名中。

具体地，`rolloverLogWriteToken` 后缀的生成过程通常由日志文件写入器（`HoodieLogAppender`）完成，它会根据当前的时间戳、分区ID和分区版本号等信息生成一个新的后缀，然后将其追加到日志文件名中。生成的后缀会被写入到日志文件头部的元数据信息中，以便后续的数据导入和处理操作。在Hudi中，`rolloverLogWriteToken` 后缀是一个非常重要的概念，它用于标识Hudi增量日志文件的版本，以便实现数据的版本控制和管理。





# 