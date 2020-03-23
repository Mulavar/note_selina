## Violate关键字

violate关键字主要有三个作用：

- 原子性
- 可见性
- 有序性



### 原子性

在32位平台下，java对64位的数据类型赋值其实不是原子性操作，如long类型，它的实际操作是分为两步，对低32位赋值和对高32位赋值，则会发生错乱。

violate则能保证对基本类型数据的读、写都是原子操作（不包括**自增这种非原子操作**）。



### 可见性

由于每个线程都有自己的缓存寄存器，每个线程读取值会优先从自己的缓存中读取而不是中心内存。那如果在多线程环境下如何保证缓存的一致性是个难题。

violate关键字能保证每次值的写入对其他线程是可见的，即每次有写操作发生后，别的线程就会把自己的缓存设为无效，下次读取值会从内存中读取。

由此可见，自增操作为什么不能保证原子性。

对于`a++`

假设线程1读取了 a 的值后，线程2将 a 的值做了自增修改，此时由于线程1已经读取了 a  的值，所以他不会再去内存中读取，这样就会发生错误。



### 有序性

编译器对无数据依赖的指令会进行重排序优化。

violate关键字可以禁止重排序，在**访问violate修饰的变量语句**之前的语句不能重排序到**访问violate修饰的变量语句**之后，同理，之后的语句也不能重排序到之前。

```java
// x、y为非volatile变量
// flag为volatile变量
// 语句1、2之间可以重排序，语句3、4之间可以重排序，
// 但语句1、2和3、4不能重排序，即1、2必须在flag之前，3、4必须在flag之后

x = 1;        //语句1
y = 2;        //语句2
flag = true;  //语句3
x = 3;         //语句4
y = 4;       //语句5
```

**对一个volatile变量的写操作及这个写操作之前的所有操作Happens-Before对这个变量的读操作及这个读操作之后的所有操作。**



### 实现机制

如果把加入volatile关键字的代码和未加入volatile关键字的代码都生成汇编代码，会发现加入volatile关键字的代码会多出一个lock前缀指令。

lock前缀指令实际相当于一个内存屏障，内存屏障提供了以下功能：

1. 重排序时不能把后面的指令重排序到内存屏障之前的位置 
2.  使得本CPU的Cache写入内存
3. 写入动作也会引起别的CPU或者别的内核无效化其Cache，相当于让新写入的值对别的线程可见。

[资料链接](https://juejin.im/post/5a2b53b7f265da432a7b821c)



## 拓展

### happens-before原则

JMM（Java内存模型）具备一些内置的**有序性，**即不需要通过任何手段就可以保证的有序性，通常称为**happens-before**原则。

happens-before原则有一个误区，严格来说，**如果操作A happens-before B，则A的结果对B可见，而非A一定先于B执行。**

> <JSR-133：Java Memory Model and Thread Specification>定义了如下happens-before规则：
>
> 1. **程序顺序规则**： 一个线程中的每个操作，happens-before于该线程中的任意后续操作
> 2. **监视器锁规则**：对一个线程的解锁，happens-before于随后对这个线程的加锁
> 3. **volatile变量规则**： 对一个volatile域的写，happens-before于后续对这个volatile域的读
> 4. **传递性**：如果A happens-before B ,且 B happens-before C, 那么 A happens-before C
> 5. **start()规则**： 如果线程A执行操作`ThreadB_start()`(启动线程B) ,  那么A线程的`ThreadB_start()`happens-before 于B中的任意操作
> 6. **join()原则**： 如果A执行`ThreadB.join()`并且成功返回，那么线程B中的任意操作happens-before于线程A从`ThreadB.join()`操作成功返回。
> 7. **interrupt()原则**： 对线程`interrupt()`方法的调用先行发生于被中断线程代码检测到中断事件的发生，可以通过`Thread.interrupted()`方法检测是否有中断发生
> 8. **finalize()原则**：一个对象的初始化完成先行发生于它的`finalize()`方法的开始

