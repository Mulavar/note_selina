Source 链路

```java
SingleOutputStreamOperator<RowData> source = execEnv.addSource(monitoringFunction, getSourceOperatorName("split_monitor"))
    .uid(Pipelines.opUID("split_monitor", conf))
    .setParallelism(1)
    .keyBy(MergeOnReadInputSplit::getFileId)
    .transform("split_reader", typeInfo, factory)
    .uid(Pipelines.opUID("split_reader", conf))
    .setParallelism(conf.getInteger(FlinkOptions.READ_TASKS));
```

StreamReadMonitoringFunction

-> keyby file id

​	-> split reader

可以认为每个 FileGroup 对应一个 Split。



# HoodieLogFileReader

读取 block，可以研究下 block 的详细信息：org.apache.hudi.common.table.log.HoodieLogFileReader#readBlock



# StreamReadMonitoringFunction

流程可以分为三步：

1.   根据配置筛选要读的 instant 范围；
2.   根据 instants 从 .hoodie 中读取对应的元数据，构建对应的 MergeOnReadInputSplit，每个 Split 对应一个 FileGroup（TODO）。



## 筛选 Instant 范围

1.   获取所有的 completed + compaction (inflight + requested) instants, 如果配置了 skipCompaction 或 skipClustering，则过滤掉对应的 instants；
2.   筛选时间范围内的 instants：
     1.   如果 issuedInstant 不为空，拿到所有比 issuedInstant 更新的 instant，(issuedInstant, latestInstant]；
     2.   如果没指定 read.start-commit 和 read.end-commit, 读最新的 instant，[latestInstant, latestInstant]；
     3.   如果指定 read.start-commit 或 read.end-commit，则给指定区间内的 instants。



## 读取元数据构建 Split

首先根据 instants 读取对应的元数据（HoodieCommitMetadata），这里会尝试从 .hoodie 和 .archive 中分别读取，如果 .archive 中也有 instant 范围内的数据，则会打 warn 日志提示要读的数据范围比较大，然后将这两个元数据合并。

根据这些元数据获取 instant 范围内有改动的分区，

根据这些元数据获取 instant 范围内有改动的文件（FileStatus）

skipBaseFiles 为 true：获取所有小于 instantToUse 的 LogFiles；

skipBaseFiles 为 false：获取最新一个 FileSlice，如果这个 FileSlice 的 BaseFile 尚未生成，那么和前一个 FileSlice 合并（如果前一个的也没生成呢？），返回前一个 FileSlice 的 BaseFile 以及这两个 FileSlice 的所有 LogFile。



获取所有有变动的文件路径（可能有多个 file id 相同的）；MergeOnReadInputSplit：每个 MergeOnReadInputSplit 都对应一个 File Slice（不一定完整）





# MergeOnReadInputFormat

Iterator



四种情况（有优先级）：

1.   没有 LogFile 存在：
     1.   如果有 instantRange => BaseFileOnlyFilteringIterator：内部封装了个 BaseFile 的 iterator，在此之上增加了 instantRange 的检测；    TODO 复习下设计模式
2.   没有 BaseFile 存在：=> BaseFile 的 iterator；
3.   检查 hoodie.datasource.merge.type 配置的值（默认为 payload_combine），如果等于 skip_merge => SkipMergeIterator 里面分别包装了 BaseFile 和 LogFile 的 iterator，先读 BaseFile 再读 LogFile；
4.   检查 hoodie.datasource.merge.type 配置的值，如果等于 payload_combine => MergeIterator



## BaseFileOnlyFilteringIterator

查看 hasNext 和 next 方法：

```java
public boolean hasNext() {
  while (this.nested.hasNext()) {
    currentRecord = this.nested.next();
    boolean isInRange = instantRange.isInRange(currentRecord.getString(commitTimePos).toString());
    if (isInRange) {
      return true;
    }
  }
  return false;
}

public RowData next() {
  // can promote: no need to project with null instant range
  return projection.project(currentRecord);
}
```

BaseFileOnlyFilteringIterator 里面包装了一个读取 BaseFile 数据的 iterator，在此之上做了两件事：

1.   查看该数据是否在 instantRange 内，如果不在直接过滤；
2.   投影剪枝，只获取想要的字段数据。



## LogFileOnlyIterator

该迭代器没有做任何进一步操作，只是调用内部包装的迭代器的 hasNext 和 next，该 iterator 的设计是用于只有 LogFile 的场景的一层封装。



getUnMergedLogFileIterator

getLogFileIterator



## SkipMergeIterator

分别包含两个 iterator：getBaseFileIterator 和 getLogFileIterator。

先读 base'file，basefile 读完后标注 readLogs 标志为 true，后续读 logfiles。

读 basefile 时不过滤任何数据，                 读 logfiles 过不过滤 instant？



## MergeIterator

也包含两个 iterator：basefile 和 logfiles

读 basefile 时会检查是否在 instant 范围内，如果在检查 logfiles 里是否包含这个 key（有的话说明该记录被更新过），这条记录放到 keyToSkip 中标注，将这条记录和 logfiles 中的记录合并并下发。

后面扫 logfiles 的时候就会跳过 keyToSkip 里已存的数据