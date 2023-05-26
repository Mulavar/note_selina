SmoothBursty

平滑创建令牌



-   storedPermits：当前存储的令牌数；
-   maxPermits：最多能保存的令牌数，maxBurstSeconds / stableIntervalMicros；
-   stableIntervalMicros：生成令牌的间隔时间；
-   maxBurstSeconds：最多保存多少秒的令牌数；
-   nextFreeTicketMicros：下一个请求被同意处理的时间。



SmoothBursty.doSetRate(double, long)

```java
final void doSetRate(double permitsPerSecond, long nowMicros) {
  resync(nowMicros);
  double stableIntervalMicros = SECONDS.toMicros(1L) / permitsPerSecond;
  this.stableIntervalMicros = stableIntervalMicros;
  doSetRate(permitsPerSecond, stableIntervalMicros);
}
```

resync 设置 nextFreeTicketMicros 和 storedPermits；

doSetRate(permitsPerSecond, stableIntervalMicros) 更新 maxPermits，根据旧的 maxPermits 和新的 maxPermits 值的比例对 storedPermits 进行调整。



 acquire

```java
public double acquire(int permits) {
  long microsToWait = reserve(permits);
  stopwatch.sleepMicrosUninterruptibly(microsToWait);
  return 1.0 * microsToWait / SECONDS.toMicros(1L);
}
```

计算需要等待的时间 microsToWait，并让当前线程 sleep 响应时间。



reserve - reserveAndGetWaitLength - reserveEarliestAvailable

```java
final long reserveEarliestAvailable(int requiredPermits, long nowMicros) {
  resync(nowMicros);
  long returnValue = nextFreeTicketMicros;
  double storedPermitsToSpend = min(requiredPermits, this.storedPermits);
  double freshPermits = requiredPermits - storedPermitsToSpend;
  long waitMicros =
      storedPermitsToWaitTime(this.storedPermits, storedPermitsToSpend)
          + (long) (freshPermits * stableIntervalMicros);

  this.nextFreeTicketMicros = LongMath.saturatedAdd(nextFreeTicketMicros, waitMicros);
  this.storedPermits -= storedPermitsToSpend;
  return returnValue;
}
```



1.  在 resync 里，如果当前时间 nowMicros 大于 nextFreeTicketMicros，则更新 nextFreeTicketMicros 和当前可用的令牌数；
2.  根据需要的令牌数和现有的令牌数计算出等待令牌生成的时间 freshPermits * stableIntervalMicros（storedPermitsToWaitTime 在 SmoothBursty 中返回值为 0）；
3.  更新能够接受下次请求的时间 nextFreeTicketMicros 和保留的令牌数 storedPermits。

这个实现和标准的令牌桶有点不一样，Guava 提供了预消费的支持。什么是预消费呢？比如我创建了一个令牌生成速度为 1个/s SmoothBursty 限流器，在刚创建的时候，SmoothBursty 中没有令牌，此时我们调用 acquire(1) 尝试获取令牌。按照正常的算法