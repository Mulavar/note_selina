## 1. GC算法

常用的GC算法有：

- 引用计数法：无法处理循环运用；
- 标记清除（mark-sweep）：会有stw；
- 标记压缩：在标记清除的基础上将剩余存活的对象放到一起
- 分代收集：将空间划分为新旧空间，每次将旧空间里存活的对象移到新空间里。



## [2. GC收集器](https://crowhawk.github.io/2017/08/15/jvm_3/)

![gc收集器](../../Resources/gc收集器.png)

基于JDK1.7 Update 14 之后的HotSpot虚拟机包含这7种GC收集器：

### 2.1 新生代：

- Serial：**进行垃圾收集时，必须暂停其他所有的工作线程，直至Serial收集器收集结束为止（“Stop The World”）。单线程收集器，采用复制算法。**
- ParNew：Serial收集器的多线程版本。
- Parallel Scanvenge：一个并行的**多线程复制**算法的GC收集器，目标是**达到一个可控制的吞吐量**。

### 2.2 老生代：

- CMS（Concurrent Mark Sweep）：一种以**获取最短回收停顿时间**为目标的收集器。分为以下四个步骤：

    - 初始标记：标记一下GC Roots能直接关联到的对象，需要STW（运行很快）；
    - 并发标记：进行**GC Roots Tracing**的过程，在整个过程中耗时最长；
    - 重新标记：对并发标记未被标记到的新对象进行标记，需要STW；
    - 并发清除；

    缺点：

    - 并发清除阶段产生的新垃圾无法及时清除；
    - 有很多内存碎片；

- Serial Old（MSC, Mark-Summary-Compaction）：Serial在老年代的收集器，使用**“标记-压缩”（Mark-Compact）**算法。作为CMS收集器的后备预案，在并发收集发生**Concurrent Mode Failure**时使用。

- Parallel Old：Parallel Scavenge收集器的老年代版本，使用**多线程**和**“标记-压缩”**算法，适合**注重吞吐量**以及**CPU资源敏感**的场合。

### 2.3 G1收集器

HotSpot开发团队赋予它的使命是（在比较长期的）未来可以替换掉JDK 1.5中发布的CMS收集器。G1从整体来看是基于**“标记-整理”**算法实现的收集器，从局部（两个Region之间）上来看是基于**“复制”**算法实现的。



### 2.4 停顿时间和吞吐量

**停顿时间越短就越适合需要与用户交互的程序**，良好的响应速度能提升用户体验。

**高吞吐量**则可以**高效率地利用CPU时间**，尽快完成程序的运算任务，主要适合**在后台运算而不需要太多交互的任务**。



## 3. GC Root

垃圾回收器回收的目标主要是那些 `GC Roots` 无法到达的对象，`Gc Roots` 主要有以下几类：

- `Class` - 由**系统类加载器 ( system class loader ) 加载的对象**，这些类是不能够被回收的，他们可以以静态字段的方式保存持有其它对象。我们需要注意的一点就是，通过用户自定义的类加载器加载的类，除非相应的 `java.lang.Class` 实例以其它的某种（或多种）方式成为 `roots`，否则它们并不是 `roots`。
- `Thread` - 活着的线程
- `Stack Local` - `Java`方法的 `local` 变量或参数
- `JNI Local` - `JNI` 方法的 `local` 变量或参数
- `JNI Global` - 全局 `JNI` 引用
- `Monitor Used` - 用于同步的监控对象
- `Held by JVM` - **用于JVM特殊目的由GC保留的对象，但实际上这个与JVM的实现是有关的**。可能已知的一些类型是：系统类加载器、一些JVM知道的重要的异常类、一些用于处理异常的预分配对象以及一些自定义的类加载器等。然而，JVM并没有为这些对象提供其它的信息，因此需要去确定哪些是属于"JVM持有"的了。



# 参考资料

[Java-Parallel GC介绍](https://cloud.tencent.com/developer/article/1764615)

[如何查看GC日志](https://www.cnblogs.com/klvchen/articles/11841337.html)

[GC日志解读，这次别再说看不懂GC日志了](https://juejin.cn/post/7029130033268555807)

