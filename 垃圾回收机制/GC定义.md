GC 是 Garbage Collection 的简称，即垃圾回收。什么是垃圾？程序中不再用到的数据就是垃圾，反映到底层就是一块内存空间，所以 GC 要做的事有两件：

-   找到不用的内存空间；
-   回收这块内存空间供之后使用。



GC 算法可以分为最基本的三类：

-   标记清除算法：1960 年由 John McCarthy 提出；
-   引用计数法：1960 年由 George E. Collins 提出；
-   复制算法：1963 年由 Marvin L. Minsky 提出。



评价标准

-   吞吐量；单位时间内的处理能力；
-   最大暂停时间；因 GC 导致应用程序暂停的最大时间；
-   堆使用效率：如复制算法对堆的使用率一般都不到 100%；
-   访问的局部性

