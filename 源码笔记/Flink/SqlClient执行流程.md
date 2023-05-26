# 1. 简介

SqlClient 是 Flink 提供的一个 SQL 命令行交互工具，在下载 flink 二进制包时，在其 bin 目录下有一个 sql-client.sh，通过启动该脚本就可以进入交互页面。SqlClient 的具体源码实现在 flink-table 模块的 flink-sql-client 子模块下可以找到，其启动函数在 `org/apache/flink/table/client/SqlClient.java` 中，该启动函数在创建好交互环境后，会调用 CliClient 的 open 函数，进入一个死循环：

```java
public void open() {
    isRunning = true;

    // print welcome
    terminal.writer().append(CliStrings.MESSAGE_WELCOME);

    // begin reading loop
    while (isRunning) {
        // make some space to previous command
        terminal.writer().append("\n");
        terminal.flush();

        final String line;
        try {
            // 1. 读取用户输入（以“;”为终结符）
            line = lineReader.readLine(prompt, null, (MaskingCallback) null, null);
        } catch (UserInterruptException e) {
            // user cancelled line with Ctrl+C
            continue;
        } catch (EndOfFileException | IOError e) {
            // user cancelled application with Ctrl+D or kill
            break;
        } catch (Throwable t) {
            throw new SqlClientException("Could not read from command line.", t);
        }
        if (line == null) {
            continue;
        }
        // 2. 调用 parseCommand 解析用户输入，获取相应的命令
        final Optional<SqlCommandCall> cmdCall = parseCommand(line);
        // 3. 调用 callCommand 执行命令
        cmdCall.ifPresent(this::callCommand);
    }
}
```

# 2. 解析命令—parseCommand

## 2.1 SqlCommandCall

 SqlCommandCall 是 SqlCommandParser 的内部类，定义如下：

```java
public static class SqlCommandCall {
    public final SqlCommand command;
    public final String[] operands;
}
```

其中 SqlCommand 是一个枚举类，这个枚举类声明了每种 SQL 操作类型，除此之外还有一个作用：用于做 SQL 命令的正则匹配。SqlCommand 为每种操作类型都定义了两个字段：

-   String pattern：用于匹配 Sql 的正则表达式；
-   Function<String[], Optional\<String>[]> operandConverter：函数式接口，将上面使用正则匹配到的结果进一步解析转换，转换的结果最终会存到 operands 里。

如上所述，operands 用于存储 SQL 命令的解析结果。

## 2.2 解析入口

```java
public static SqlCommandCall parse(Parser sqlParser, String stmt) {
    // normalize
    stmt = stmt.trim();
    // remove ';' at the end
    if (stmt.endsWith(";")) {
        stmt = stmt.substring(0, stmt.length() - 1).trim();
    }

    // parse statement via regex matching first
    Optional<SqlCommandCall> callOpt = parseByRegexMatching(stmt);
    if (callOpt.isPresent()) {
        return callOpt.get();
    } else {
        return parseBySqlParser(sqlParser, stmt);
    }
}
```

Flink 首先尝试调用 parseByRegexMatching 对 Sql 进行正则匹配，若失败了再调用 parseBySqlParser 做 SQL（底层使用了 Calcite 框架）解析。笔者初读时不太理解为什么会有两套方案做解析，后面听说是最初都是用正则进行匹配，但由于 Flink SQL 要解析成逻辑执行计划本身就使用了 Calcite 作为 Parser，为了设计上的统一，就增加了 SQL Parser 的方式。但目前 Flink 仍有一部分与数据操作无关的命令保留了正则匹配，如 QUIT、EXIT、HELP、SET 等命令。

>    在 [Flink-17893] 之前，SqlClient 的解析顺序与现在相反，会先使用 Sql Parser 尝试解析，解析失败后再使用正则。

## 2.3 parseByRegexMatching

parseByRegexMatching 的具体流程如下：

1.  遍历 SqlCommand 枚举类的每个成员，如果指定了 pattern，则尝试进行正则匹配；
2.  若匹配成功，则调用 operandConverter 进行解析转换；
3.  否则直接返回 Optional.empty()。

## 2.4 parseBySqlParser

```java
private static SqlCommandCall parseBySqlParser(Parser sqlParser, String stmt) {
    List<Operation> operations;
    try {
        operations = sqlParser.parse(stmt);
    } catch (Throwable e) {
        throw new SqlExecutionException("Invalidate SQL statement.", e);
    }
    if (operations.size() != 1) {
        throw new SqlExecutionException("Only single statement is supported now.");
    }

    final SqlCommand cmd;
    String[] operands = new String[] {stmt};
    Operation operation = operations.get(0);
    if (operation instanceof CatalogSinkModifyOperation) {
        boolean overwrite = ((CatalogSinkModifyOperation) operation).isOverwrite();
        cmd = overwrite ? SqlCommand.INSERT_OVERWRITE : SqlCommand.INSERT_INTO;
    } else if (operation instanceof CreateTableOperation) {
        cmd = SqlCommand.CREATE_TABLE;
    } 
    // 省略其余的分支情况
    return new SqlCommandCall(cmd, operands);
}
```

首先调用 Parser 解析 SQL 文本，这部分底层使用了 Calcite 框架进行词法、语法解析，将 SQL 文本转成了 SqlNode，再通过 SqlToOperationConverter 转成了 Operation。

之后则是根据 Operation 的具体类型转成相应的 SqlCommand，并从 Operation 中抽取具体的 operands 信息，封装成 SqlCommandCall 返回。

这部分逻辑可以简要概括为这四步转换：String -> SqlNode -> Opreration -> SqlCommand。

# 3. 执行命令—callCommand

callCommand 的本质就是检查 SqlCommandCall 里的 command 类型，并进入不同的分支执行逻辑。

```java
private void callCommand(SqlCommandCall cmdCall) {
    switch (cmdCall.command) {
        case QUIT:
            callQuit();
            break;
        // ......
        case SELECT:
            callSelect(cmdCall);
            break;
        case INSERT_INTO:
        case INSERT_OVERWRITE:
            callInsert(cmdCall);
            break;
        // ......
        default:
            throw new SqlClientException("Unsupported command: " + cmdCall.command);
    }
}
```

从 callSelect 触发，层层探究，可以得到调用链：CliClient.callSelect => LocalExecutor.executeQuery => LocalExecutor.executeQueryInternal，查看 executeQueryInternal 的源码如下：

```java
private <C> ResultDescriptor executeQueryInternal(
        String sessionId, ExecutionContext<C> context, String query) {
    // create table
    final Table table = createTable(context, context.getTableEnvironment(), query);
    // TODO refactor this after Table#execute support all kinds of changes
    // initialize result
    final DynamicResult<C> result =
            resultStore.createResult(
                    context.getEnvironment(),
                    removeTimeAttributes(table.getSchema()),
                    context.getExecutionConfig());
    final String jobName = sessionId + ": " + query;
    final String tableName = String.format("_tmp_table_%s", Math.abs(query.hashCode()));
    final Pipeline pipeline;
    try {
        // writing to a sink requires an optimization step that might reference UDFs during code
        // compilation
        context.wrapClassLoader(
                () -> {
                    ((TableEnvironmentInternal) context.getTableEnvironment())
                            .registerTableSinkInternal(tableName, result.getTableSink());
                    table.insertInto(tableName);
                });
        pipeline = context.createPipeline(jobName);
    } catch (Throwable t) {
        // the result needs to be closed as long as
        // it not stored in the result store
        result.close();
        // catch everything such that the query does not crash the executor
        throw new SqlExecutionException("Invalid SQL query.", t);
    } finally {
        // Remove the temporal table object.
        context.wrapClassLoader(
                () -> {
                    context.getTableEnvironment().dropTemporaryTable(tableName);
                });
    }

    // create a copy so that we can change settings without affecting the original config
    Configuration configuration = new Configuration(context.getFlinkConfig());
    // for queries we wait for the job result, so run in attached mode
    configuration.set(DeploymentOptions.ATTACHED, true);
    // shut down the cluster if the shell is closed
    configuration.set(DeploymentOptions.SHUTDOWN_IF_ATTACHED, true);

    // create execution
    final ProgramDeployer deployer =
            new ProgramDeployer(configuration, jobName, pipeline, context.getClassLoader());

    JobClient jobClient;
    // wrap in classloader because CodeGenOperatorFactory#getStreamOperatorClass
    // requires to access UDF in deployer.deploy().
    jobClient =
            context.wrapClassLoader(
                    () -> {
                        try {
                            // blocking deployment
                            return deployer.deploy().get();
                        } catch (Exception e) {
                            throw new SqlExecutionException("Error while submitting job.", e);
                        }
                    });

    String jobId = jobClient.getJobID().toString();
    // store the result under the JobID
    resultStore.storeResult(jobId, result);

    // start result retrieval
    result.startRetrieval(jobClient);

    return new ResultDescriptor(
            jobId,
            removeTimeAttributes(table.getSchema()),
            result.isMaterialized(),
            context.getEnvironment().getExecution().isTableauMode());
}
```

1.   创建 Table 用于存储结果 Schema；
2.   创建 Pipeline，这一步也是最核心的，会将 SQL 转为具体的 StreamGraph；
3.   创建 JobClient 并部署该作业；
4.   启动一个线程 ResultRetrievalThread 不断获取新的结果，并更新本地队列；
5.   返回结果标识符。

进一步查看创建 Pipeline 的代码 `pipeline = context.createPipeline(jobName)`：

```java
public Pipeline getPipeline(String jobName) {
    return execEnv.createPipeline(translateAndClearBuffer(), tableConfig, jobName);
}
```

translateAndClearBuffer 主要是负责将缓存在 TableEnvironmentImpl 的 List\<ModifyOperation> 转换成 List\<Transformation>，这部分细节此节暂且不提，而 createPipeline 则是将这些 Transformation 拼成一个 StreamGraph：

```java
public Pipeline createPipeline(
        List<Transformation<?>> transformations, TableConfig tableConfig, String jobName) {
    StreamGraph streamGraph =
            ExecutorUtils.generateStreamGraph(getExecutionEnvironment(), transformations);
    streamGraph.setJobName(getNonEmptyJobName(jobName));
    return streamGraph;
}
```

查看 ExecutorUtils.generateStreamGraph 如下，它创建了一个 StreamGraphGenerator，并调用其 generate 方法，从这里开始就和 DataStream 当中提到的 StreamGraph 生成的逻辑完全一致了，也就串联起来了 SQL 和 Transformation。

```java
public static StreamGraph generateStreamGraph(
        StreamExecutionEnvironment execEnv, List<Transformation<?>> transformations) {
    if (transformations.size() <= 0) {
        throw new IllegalStateException(
                "No operators defined in streaming topology. Cannot generate StreamGraph.");
    }
    StreamGraphGenerator generator =
            new StreamGraphGenerator(
                            transformations, execEnv.getConfig(), execEnv.getCheckpointConfig())
                    .setStateBackend(execEnv.getStateBackend())
                    .setChaining(execEnv.isChainingEnabled())
                    .setUserArtifacts(execEnv.getCachedFiles())
                    .setTimeCharacteristic(execEnv.getStreamTimeCharacteristic())
                    .setDefaultBufferTimeout(execEnv.getBufferTimeout());
    return generator.generate();
}
```
