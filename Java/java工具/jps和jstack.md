## jps

用于查看java的一些进程。



## jstack

jstack是jdk自带的线程堆栈分析工具，使用该命令可以查看或导出 Java 应用程序中线程堆栈信息。

使用方式：`jstack <pid>`

参数说明：

- -l：长列表. 打印关于锁的附加信息,例如属于java.util.concurrent 的 ownable synchronizers列表.
- -F：当’jstack [-l] pid’没有相应的时候强制打印栈信息
- -m：打印java和native c/c++框架的所有栈信息.
- -h | -help：打印帮助信息

