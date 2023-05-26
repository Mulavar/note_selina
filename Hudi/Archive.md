相关配置

```java
public static final ConfigProperty<String> MAX_COMMITS_TO_KEEP_PROP = ConfigProperty
    .key("hoodie.keep.max.commits")
    .defaultValue("150")
    .withDocumentation("Archiving service moves older entries from timeline into an archived log after each write, to "
        + " keep the metadata overhead constant, even as the table size grows."
        + "This config controls the maximum number of instants to retain in the active timeline. ");

public static final ConfigProperty<String> MIN_COMMITS_TO_KEEP_PROP = ConfigProperty
    .key("hoodie.keep.min.commits")
    .defaultValue("145")
    .withDocumentation("Similar to " + MAX_COMMITS_TO_KEEP_PROP.key() + ", but controls the minimum number of"
        + "instants to retain in the active timeline.");
        
public static final ConfigProperty<String> COMMITS_ARCHIVAL_BATCH_SIZE_PROP = ConfigProperty
  .key("hoodie.commits.archival.batch")
  .defaultValue(String.valueOf(10))
  .withDocumentation("Archiving of instants is batched in best-effort manner, to pack more instants into a single"
      + " archive log. This config controls such archival batch size.");
```

MAX_COMMITS_TO_KEEP_PROP：ActiveTimeLine 最多保留的 instant 个数

MIN_COMMITS_TO_KEEP_PROP：ActiveTimeLine 最少保留的 instant 个数

COMMITS_ARCHIVAL_BATCH_SIZE_PROP：每次 archive 最多的 instant 个数



入口

checkpoint 的时候提交完 writeStat 后，postCommit 会触发，（那是不是也有手动触发）

HoodieFlinkWriteClient#postCommit

```java
protected void postCommit(HoodieTable<T, List<HoodieRecord<T>>, List<HoodieKey>, List<WriteStatus>> table,
                        HoodieCommitMetadata metadata,
                        String instantTime,
                        Option<Map<String, String>> extraMetadata) {
try {
  // Delete the marker directory for the instant.
  new MarkerFiles(createTable(config, hadoopConf), instantTime)
      .quietDeleteMarkerDir(context, config.getMarkersDeleteParallelism());
  // We cannot have unbounded commit files. Archive commits if we have to archive
  HoodieTimelineArchiveLog archiveLog = new HoodieTimelineArchiveLog(config, table);
  archiveLog.archiveIfRequired(context);
} catch (IOException ioe) {
  throw new HoodieIOException(ioe.getMessage(), ioe);
} finally {
  this.heartbeatClient.stop(instantTime);
}
}
```



删除该次 instant 期间创建的 markfile

对文件归档



HoodieTimelineArchiveLog

```java
public boolean archiveIfRequired(HoodieEngineContext context) throws IOException {
try {
  List<HoodieInstant> instantsToArchive = getInstantsToArchive().collect(Collectors.toList());

  boolean success = true;
  if (!instantsToArchive.isEmpty() && instantsToArchive.size() >= this.config.getCommitArchivalBatchSize()) {
    this.writer = openWriter();
    LOG.info("Archiving instants " + instantsToArchive);
    archive(context, instantsToArchive);
    LOG.info("Deleting archived instants " + instantsToArchive);
    success = deleteArchivedInstants(instantsToArchive);
  } else {
    LOG.info("No Instants to archive");
  }

  return success;
} finally {
  close();
}
}
```

先找出所有可归档的 instant

写到 archive 目录下

删除 .hoodie 下所有对应的 instant，以及 .aux 目录下所有的 compaction.requested





找出可归档的 instant

分两部分：

1.   找 CLEAN instant；
2.   找 COMMIT instant；



CLEAN instant

找所有的 clean 和 rollback 的 instant，无过滤条件

COMMIT instant（包含 COMMIT、DELTA_COMMIT、REPLACE_COMMIT）

找最早的 savepoint，如果存在，只能归档这个 savepoint 之前的 instants，

结合 clean 的流程是怎么样的？是不是得先 archive 了才能 clean 这个 savepoint

思考 clean savepoint archive 三者的关系



找比最早 pending 状态的 compaction instant 更小的 instant

找比最早 inflight 状态的 commit instant 更小的 instant



归档

删除所有对应的 markfile

将每个要归档的 instant 转成一个 record，这个 record 具体实现为 HoodieArchivedMetaEntry，包含四部分信息：

commitTime：instant 对应 time；

actionState：操作状态（感觉应该都是 completed？也可能包含一些没执行完的，比如 CLEAN 刚被调度）

actionType：操作类型

metadata：操作的元数据

最后讲这些 record 组织成 HoodieAvroDataBlock 写入 .hoodie/archived/commits_.archive.1



搞定



