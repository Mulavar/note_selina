Watermark 定义：当看到这个 watermark 时，不再处理比这个 watermark 更早的数据。

首先，watermark 是一个特殊的记录（StreamElement），标注了时间戳，表示小于等于这个时间戳的数据已经都到了。它也像 StreamElement 一样在普通算子之间传播，主要用于触发窗口计算或一些需要依靠事件时间进行处理的计算（如 Temporal Join）。

Flink SQL 中，用于生成 watermark 的算子叫 WatermarkAssignerOperator，根据 source 的数据生成 watermark 并下发，这里会采用 broadcast 广播下发给所有连接的下游 subtask。

# 注册定时器

关于下游算子，需要 watermark 推进的逻辑一般会先封装到 timer service 中，即放到一个队列中：

```java
// InternalTimerServiceImpl
public void registerEventTimeTimer(N namespace, long time) {
    eventTimeTimersQueue.add(
            new TimerHeapInternalTimer<>(time, (K) keyContext.getCurrentKey(), namespace));
}
```



# 触发定时任务

当下游算子收到 watermark 时，会先尝试推进 watermark，在这个过程中，会从 eventTimeTimersQueue 拿出所有小于等于该 watermark 时间的任务：

```java
// AbstractStreamOperator
public void processWatermark(Watermark mark) throws Exception {
    if (timeServiceManager != null) {
        timeServiceManager.advanceWatermark(mark);
    }
    output.emitWatermark(mark);
}

// InternalTimeServiceManagerIml
public void advanceWatermark(Watermark watermark) throws Exception {
    for (InternalTimerServiceImpl<?, ?> service : timerServices.values()) {
        service.advanceWatermark(watermark.getTimestamp());
    }
}

// InternalTimerServiceImpl
public void advanceWatermark(long time) throws Exception {
    currentWatermark = time;

    InternalTimer<K, N> timer;

    while ((timer = eventTimeTimersQueue.peek()) != null && timer.getTimestamp() <= time) {
        keyContext.setCurrentKey(timer.getKey());
        eventTimeTimersQueue.poll();
        triggerTarget.onEventTime(timer);
    }
}
```

triggerTarget 是一个 Triggerable，其接口定义如下：

```java
public interface Triggerable<K, N> {

    /** Invoked when an event-time timer fires. */
    void onEventTime(InternalTimer<K, N> timer) throws Exception;

    /** Invoked when a processing-time timer fires. */
    void onProcessingTime(InternalTimer<K, N> timer) throws Exception;
}
```

一些与事件时间或分区相关的算子都实现了该接口，以 KeyedProcessOperator 为例：

```java
public void onEventTime(InternalTimer<K, VoidNamespace> timer) throws Exception {
    collector.setAbsoluteTimestamp(timer.getTimestamp());
    invokeUserFunction(TimeDomain.EVENT_TIME, timer);
}

private void invokeUserFunction(TimeDomain timeDomain, InternalTimer<K, VoidNamespace> timer)
        throws Exception {
    onTimerContext.timeDomain = timeDomain;
    onTimerContext.timer = timer;
    userFunction.onTimer(timer.getTimestamp(), onTimerContext, collector);
    onTimerContext.timeDomain = null;
    onTimerContext.timer = null;
}
```

最终调用 userFunction（KeyedProcessFunction）的 onTimer 方法。



# Watermark 流转

数据链路传输：

StreamTask#runMailboxLoop

​	MailboxLoop#runMailboxLoop

​		StreamTask#processInput

​			StreamOneInputProcessor#processInput

​				AbstractStreamTaskNetworkInput#emitNext

​					AbstractStreamTaskNetworkInput#processElement

```java
 private void processElement(StreamElement recordOrMark, DataOutput<T> output) throws Exception {
    if (recordOrMark.isRecord()) {
        output.emitRecord(recordOrMark.asRecord());
    } else if (recordOrMark.isWatermark()) {
        statusWatermarkValve.inputWatermark(
                recordOrMark.asWatermark(), flattenedChannelIndices.get(lastChannel), output);
    } else if (recordOrMark.isLatencyMarker()) {
        output.emitLatencyMarker(recordOrMark.asLatencyMarker());
    } else if (recordOrMark.isWatermarkStatus()) {
        statusWatermarkValve.inputWatermarkStatus(
                recordOrMark.asWatermarkStatus(),
                flattenedChannelIndices.get(lastChannel),
                output);
    } else {
        throw new UnsupportedOperationException("Unknown type of StreamElement");
    }
}
```

如果是 watermark，则进入第二个分支。

实际任务中，大部分 subtask 都和上游多个 subtask 连接，每个上游 subtask 都可能发送 watermark，当前算子一般选择上游所有 channel 中最小的 watermark 作为标准，来确定不再处理数据的边界时间，但有的 subtask 可能不发 watermark（比如这个 subtask 从属于一个 kafka source，对应分区无数据），那么会导致该算子的 watermark 无法推进，从而一直无法触发新的数据计算。

为了避免这个问题，Flink 提出了 watermark status 的概念，将 watermark 的状态分为两类：ACTIVE 和 IDLE，对上游每个 channel 都会维护一个 watermark status。

有了这两个状态后，当前算子每次接收到 watermark 信息，会先判断该 watermark 来自的 channel 是否处于 ACTIVE 状态，以及当前算子的 watermark 是否处于 ACTIVE 状态，如果是，则更新对应 channel 的 watermark 值（选较大的），并标识该 channel 已对齐，之后调用 findAndOutputNewMinWatermarkAcrossAlignedChannels 从所有以对齐的 channel 中找到最小的 watermark。如果 该 watermark 比上次处理的 watermark 更大，则调用 OneInputStreamTask#emitWatermark 进行对 watermark 的处理。

```java
// OneInputStreamTask
public void emitWatermark(Watermark watermark) throws Exception {
    watermarkGauge.setCurrentWatermark(watermark.getTimestamp());
    operator.processWatermark(watermark);
}
```





# watermark 对齐和 watermark status 的区别

两者很接近，但 watermark 在 watermark status 为 ACTIVE 的基础上被更新过才会标识为已对齐。



# Watermark Status

针对上游可能有些分区无数据，导致 watermark 一直不往前推动，数据不被处理，Flink 提出了一个 watermark status 的概念，将 watermark 的状态分为两类：ACTIVE 和 IDLE。

用户可以设置一个超时时间（**table.exec.source.idle-timeout**），这个超时时间表示当数据源多久没收到数据后，就会更新 watermark status 为 IDLE，并下发一条 WatermarkStatus，这也就对应 Watermark 流转的 第三个分支：`statusWatermarkValve.inputWatermarkStatus`。



https://blog.csdn.net/adddimin/article/details/124588789



https://blog.csdn.net/qq_36329933/article/details/110520280