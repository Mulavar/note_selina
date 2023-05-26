启动步骤

1.  启动 Flink 集群：./bin/start-cluster.sh
2.  启动 SQL Client：
    -   ./bin/sql-client.sh embedded
    -   SQL 客户端将从 ./conf/sqll-client-defaults.yaml 中读取配置
3.  执行 DML

可视化模式

CLI 提供三种展示模式

表格模式 (table mode)

变更日志模式 (changelog mode)：

Tableau 模式 (tableau mode)：接近传统数据库的展示模式

通过 SET execution.result-mode=table/changelog/tableau 设置