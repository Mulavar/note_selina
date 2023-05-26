# 语法

https://nightlies.apache.org/flink/flink-docs-master/docs/dev/table/sql/queries/deduplication/

```
SELECT [column_list]
FROM (
    SELECT [column_list],
         ROW_NUMBER() OVER ([PARTITIONBY col1[, col2...]]
    ORDER BY time_attr [asc|desc])AS rownum
    FROM table_name
)
WHERE rownum= 1
```

注意点：

-   ORDER BY 后的字段必须是时间属性（process time/row time）



# Minibatch 开关

开启 MiniBatch 时使用 KeyedMapBundleOperator，否则使用 KeyedProcessOperator。



# 状态使用

|                  | Event Time                          | Process Time                                     |
| ---------------- | ----------------------------------- | ------------------------------------------------ |
| 开启 minibatch   | RowTimeMiniBatchDeduplicateFunction | ProcTimeMiniBatchDeduplicateKeepLastRowFunction  |
|                  |                                     | ProcTimeMiniBatchDeduplicateKeepFirstRowFunction |
| 没开启 minibatch | RowTimeDeduplicateFunction          | ProcTimeDeduplicateKeepLastRowFunction           |
|                  |                                     | ProcTimeDeduplicateKeepFirstRowFunction          |

在 Event Time 场景下，每条数据到来后必须对比其附带的事件时间和该算子已存储的事件时间进行对比，因此只需一个函数统一做逻辑处理。

而处理时间是由当前处理数据的算子赋予，因此可以直接简化为两种场景：保留第一条和保留最后一条：

-   First Row：存储第一条过来的数据，并丢弃后面来的所有数据即可，只能处理上游是 Append-only 的数据，如果是 Process Time 场景也只会产生 Insert 数据；
-   Last Row：每次到来数据需要根据时间属性留下最新的一条，如果当前的数据是最新的，则下发回撤老数据。

所有数据的最终处理逻辑最终会落到 DeduplicateFunctionHelper，因此我们可以通过阅读 DeduplicateFunctionHelper 的源码查看不同场景的处理情况。



# DeduplicateFunctionHelper

去重函数的处理都最终调用这个工具类的方法

## Process Time&Last Row

Process Time 根据能否处理回撤消息分为两种：

-   processLastRowOnProcTime：仅支持处理 INSERT 消息；
-   processLastRowOnChangelog：除 INSERT 外可处理回撤等消息。

### Last Row on Proctime

```java
static void processLastRowOnProcTime(
            RowData currentRow,
            boolean generateUpdateBefore,
            boolean generateInsert,
            ValueState<RowData> state,
            Collector<RowData> out)
            throws Exception {

  checkInsertOnly(currentRow);
  if (generateUpdateBefore || generateInsert) {
    // use state to keep the previous row content if we need to generate UPDATE_BEFORE
    // or use to distinguish the first row, if we need to generate INSERT
    RowData preRow = state.value();
    state.update(currentRow);
    if (preRow == null) {
      // the first row, send INSERT message
      currentRow.setRowKind(RowKind.INSERT);
      out.collect(currentRow);
    } else {
      if (generateUpdateBefore) {
        preRow.setRowKind(RowKind.UPDATE_BEFORE);
        out.collect(preRow);
      }
      currentRow.setRowKind(RowKind.UPDATE_AFTER);
      out.collect(currentRow);
    }
  } else {
    // always send UPDATE_AFTER if INSERT is not needed
    currentRow.setRowKind(RowKind.UPDATE_AFTER);
    out.collect(currentRow);
  }
}
```

1.  查看该消息是否是 INSERT 格式（只接受 INSERT 格式消息）；
2.  检查该节点发送的消息类型（generateUpdateBefore 的值由 changelogmode 和下游需要的消息类型共同决定是否需要下发回撤，generateInsert 由 table.exec.deduplicate.insert-update-after-sensitive-enabled 配置确定）是否支持 UA 和 I；
3.  如果不支持发回撤且不支持发 INSERT 则直接发送 UA 消息到下游；
4.  如果支持，检查当前这条数据是否是第一条数据（根据状态查询，状态里会保存上一条到来的数据），并更新状态为当前数据；
5.  如果是第一条数据，则附上 +I 标识发送消息到下游；
6.  如果不是第一条数据，检查该节点是否支持发送 UB 消息；
7.  如果需要，则附上 -UB 标识发送回撤上一条数据；
8.  发送 +UA 消息。

### Last Row on Changelog

```java
static void processLastRowOnChangelog(
        RowData currentRow,
        boolean generateUpdateBefore,
        ValueState<RowData> state,
        Collector<RowData> out,
        boolean isStateTtlEnabled,
        RecordEqualiser equaliser)
        throws Exception {
    RowData preRow = state.value();
    RowKind currentKind = currentRow.getRowKind();
    if (currentKind == RowKind.INSERT || currentKind == RowKind.UPDATE_AFTER) {
        if (preRow == null) {
            // the first row, send INSERT message
            currentRow.setRowKind(RowKind.INSERT);
            out.collect(currentRow);
        } else {
            if (!isStateTtlEnabled && equaliser.equals(preRow, currentRow)) {
                // currentRow is the same as preRow and state cleaning is not enabled.
                // We do not emit retraction and update message.
                // If state cleaning is enabled, we have to emit messages to prevent too early
                // state eviction of downstream operators.
                return;
            } else {
                if (generateUpdateBefore) {
                    preRow.setRowKind(RowKind.UPDATE_BEFORE);
                    out.collect(preRow);
                }
                currentRow.setRowKind(RowKind.UPDATE_AFTER);
                out.collect(currentRow);
            }
        }
        // normalize row kind
        currentRow.setRowKind(RowKind.INSERT);
        // save to state
        state.update(currentRow);
    } else {
        // DELETE or UPDATER_BEFORE
        if (preRow != null) {
            // always set to DELETE because this row has been removed
            // even the input is UPDATE_BEFORE, there may no UPDATE_AFTER after it.
            preRow.setRowKind(RowKind.DELETE);
            // output the preRow instead of currentRow,
            // because preRow always contains the full content.
            // currentRow may only contain key parts (e.g. Kafka tombstone records).
            out.collect(preRow);
            // clear state as the row has been removed
            state.clear();
        }
        // nothing to do if removing a non-existed row
    }
}
```

changelog 场景下，因为需要处理回撤信息，去重逻辑相对复杂一点：

1.   查看当前数据是 accumulate（INSERT 或 UPDATE）还是回撤消息；
2.   如果是回撤消息，且状态不为空，则将状态的数据附上 -D 标识下发回撤，注意：这里回撤用的数据必须是状态里的，因为当前的数据可能只包含 key 部分；
3.   如果不是回撤数据，则查看状态数据是否为空（即当前数据是否是第一条数据），如果是，则附上 +I 标识下发；
4.   如果没有启用 TTL 且当前数据和状态里的数据一致，则不下发（减少大量重复数据场景下下发的数据量）；
5.   如果支持发送 UB 消息（generateUpdateBefore 为 true），则附上 -UB 标识下发回撤；
6.   为当前消息附上 +UA 标识下发；
7.   更新状态为当前消息。



## Process Time&First Row

```
static void processFirstRowOnProcTime(
            RowData currentRow, ValueState<Boolean> state, Collector<RowData> out)
            throws Exception {

  checkInsertOnly(currentRow);
  // ignore record if it is not first row
  if (state.value() != null) {
    return;
  }
  state.update(true);
  // emit the first row which is INSERT message
  out.collect(currentRow);
}
```

1.  查看该消息是否是 INSERT 格式（只接受 INSERT 格式消息）；
2.  检查该条数据是否是第一条到达的数据（通过状态查询是否来过数据）；
3.  如果是，则直接忽略不下发，否则附上 +I 标识下发该条数据。



## Event Time

由于数据到来可能存在乱序，最早到的数据不一定是 Event Time 最老的，最后到的数据也不一定是 Event Time 最新的，因此 Event Time 的去重场景不需要像 Process Time 那样针对 First Row 和 Last Row 分别实现一套逻辑，只需在检查当前数据是否需要下发时采取不同的策略即可。

Event Time 去重场景实现可参考 RowTimeDeduplicateFunction：

```java
public static void deduplicateOnRowTime(
        ValueState<RowData> state,
        RowData currentRow,
        Collector<RowData> out,
        boolean generateUpdateBefore,
        boolean generateInsert,
        int rowtimeIndex,
        boolean keepLastRow)
        throws Exception {
    checkInsertOnly(currentRow);
    RowData preRow = state.value();

    if (isDuplicate(preRow, currentRow, rowtimeIndex, keepLastRow)) {
        updateDeduplicateResult(generateUpdateBefore, generateInsert, preRow, currentRow, out);
        state.update(currentRow);
    }
}
```

1.   首先检查消息是否是 INSERT；
2.   调用 isDuplicate 判断该消息是否应该下发，而 Event Time 去重的逻辑精髓就在于此；
3.   下发消息并更新状态。

### 判断消息优先级

```
static boolean isDuplicate(
            RowData preRow, RowData currentRow, int rowtimeIndex, boolean keepLastRow) {
  if (keepLastRow) {
    return preRow == null
      || getRowtime(preRow, rowtimeIndex) <= getRowtime(currentRow, rowtimeIndex);
  } else {
    return preRow == null
      || getRowtime(currentRow, rowtimeIndex) < getRowtime(preRow, rowtimeIndex);
  }
}
```

1.  如果是保存最新一条数据（Last Row），则比较当前数据的事件时间是否大于等于先前数据的事件时间；
2.  如果是保存最早一条数据（First Row），则比较当前数据的事件时间是否小于先前数据的事件时间。

**注：这里为什么第一个判断用大于等于，第二个判断用小于？因为第一种情况的语义是最新一条数据，因此两条数据事件时间一样，取后来的数据，而第二种情况的语义是最早一条数据，两条数据事件时间一样时取先来的数据。**

### 更新下发结果

```
static void updateDeduplicateResult(
            boolean generateUpdateBefore,
            boolean generateInsert,
            RowData preRow,
            RowData currentRow,
            Collector<RowData> out) {

  if (generateUpdateBefore || generateInsert) {
    if (preRow == null) {
      // the first row, send INSERT message
      currentRow.setRowKind(RowKind.INSERT);
      out.collect(currentRow);
    } else {
      if (generateUpdateBefore) {
        final RowKind preRowKind = preRow.getRowKind();
        preRow.setRowKind(RowKind.UPDATE_BEFORE);
        out.collect(preRow);
        preRow.setRowKind(preRowKind);
      }
      currentRow.setRowKind(RowKind.UPDATE_AFTER);
      out.collect(currentRow);
    }
  } else {
    currentRow.setRowKind(RowKind.UPDATE_AFTER);
    out.collect(currentRow);
  }
}
```

这段逻辑与 Process Time&Last Row 的逻辑非常相似，可直接参考上述的代码讲解。