

## 捕获线程异常的方式

### 1. 定义线程异常捕获的处理器（实现UncaughtExceptionHandler接口的类实例对象）

```java
Thread t = new Thread(new MyThread());
t.setUncaughtExceptionHandler(new MyUnchecckedExceptionhandler());
```



### 2.  设置默认的全局线程异常捕获器

```java
Thread.setDefaultUncaughtExceptionHandler(new MyUnchecckedExceptionhandler());
```



### 3. 使用Executors创建线程时，在ThreadFactory中设置

只能捕捉`execute()`的异常，无法捕捉`submit()`的异常，因为`submit()`的异常会被封装到Future中。



### 4. 使用FutureTask来捕获异常

在FutureTask的源码里有这么一段

```java
// 方法run()
try {
	result = c.call();
	ran = true;
} catch (Throwable ex) {
	result = null;
	ran = false;
	setException(ex);
}

// 方法setException()
outcome = t;
```

而在调用`FutureTask.get()`方法时，会先判断任务的执行状态，如果是异常则抛出异常，因此可以对`FutureTask.get()`使用`try...catch...`捕捉。

基于这一点可以延伸出对2的补充，对`submit()`的返回结果调用`get()`方法，并使用`try...catch...` 捕捉。



### 5. 重写ThreadPoolExecutor的afterExecute方法

例子如下：

```java
ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(10, 10, 0L, TimeUnit.MILLISECONDS,
        new LinkedBlockingQueue<>()) {
    @Override
    protected void afterExecute(Runnable r, Throwable t) {
        // 处理execute的情况
        if (r instanceof Thread) {
            if (t != null) {
                //处理捕获的异常
            }
        } 
        // 处理submit的情况
        else if (r instanceof FutureTask) {
            FutureTask futureTask = (FutureTask) r;
            try {
                futureTask.get();
            } catch (InterruptedException e) {
                e.printStackTrace();
            } catch (ExecutionException e) {
                //处理捕获的异常
            }
        }

    }
};
```



## 总结

1. 线程的异常**最好在线程内直接捕获处理**，因为每个线程都是一个独立的调度执行单位，从设计角度上来说应该自己处理自己的异常问题。
2. 方法5是对基于整个线程池的粗粒度的异常处理，如果想对有返回值和无返回值的任务更细粒度划分处理则可以分别使用方法3和方法4.



## 参考资料

[1.Java捕获线程异常的几种方式](https://blog.csdn.net/pange1991/article/details/82115437)

