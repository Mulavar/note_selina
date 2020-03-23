## Callable

Callable接口定义如下：

```java
public interface Callable<V> {
    V call() throws Exception;
}
```

由于runnable接口无返回类型，所以当我们在多线程编程中需要数据传递时可以考虑使用callable接口。

Callable接口



## Future

Future的作用是用于获取Callable的执行结果或取消任务、查询是否完成等工作，使用get()返回结果（如果没执行完，该方法会阻塞）。

Future接口定义如下：

```java
public interface Future<V> {
    boolean cancel(boolean mayInterruptIfRunning);
    boolean isCancelled();
    boolean isDone();
    V get() throws InterruptedException, ExecutionException;
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```



## FutureTask

是Future和Runnable的一个实现类，里面封装了一个Callable对象，每次run时，调用Callable的call()方法，然后将结果存到变量outcome中。

run()核心逻辑如下：

```java
Callable<V> c = callable;
if (c != null && state == NEW) {
    V result;
    boolean ran;
    try {
        result = c.call();
        ran = true;
    } catch (Throwable ex) {
        result = null;
        ran = false;
        setException(ex);
    }
    if (ran)
        set(result);
}
```





[Callable、Future和FutureTask](https://www.cnblogs.com/dolphin0520/p/3949310.html)

[彻底理解java的future模式](https://www.cnblogs.com/cz123/p/7693064.html)



