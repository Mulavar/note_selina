Hadoop 组件：

HDFS：文件系统，包含 NameNode 和 DataNode

YARN：任务资源调度系统，包含 ResourceManager 和 NodeManager



怎么启动本地 Hadoop 环境

https://zhuanlan.zhihu.com/p/33117305



hadoop namenode -format

```bash
WARNING: Use of this script to execute namenode is deprecated.
WARNING: Attempting to execute replacement "hdfs namenode" instead
```



./sbin/start-all.sh

主要就是启动上面提到的组件



常用网页：

http://localhost:9870/ hadoop 管理页面

http://localhost:8088/cluster yarn 管理页面