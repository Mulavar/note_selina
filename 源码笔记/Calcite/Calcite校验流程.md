# 1. 核心结构与概念

Calcite 提供的 Validator 流程极为复杂，但概括下来主要做了这么一件事，**对每个 SqlNode 结合元数据校验其语义是否正确**，这些语义包括：

-   验证表名是否存在；
-   select 的列在对应表中是否存在，且该匹配到的列名是否唯一，比如 join 多表，两个表有相同名字的字段，如果此时 select 的列不指定表名就会报错；
-   如果是 insert，需要插入列和数据源进行校验，如列数、类型、权限等；
-   ……

![class](https://tva1.sinaimg.cn/large/008i3skNly1gw8rg61yufj31580jk413.jpg)

Calcite 提供的 validator 和前面提到的 Catalog 关系紧密，Calcite 定义了一个 CatalogReader 用于在校验过程中访问元数据 (Table schema)，并对元数据做了运行时的一些封装，最核心的两部分是 SqlValidatorNamespace 和 SqlValidatorScope。

-   SqlValidatorNamespace：描述了 SQL 查询返回的关系，一个 SQL 查询可以拆分为多个部分，查询的列组合，表名等等，当中每个部分都有一个对应的 SqlValidatorNamespace。
-   SqlValidatorScope：可以认为是校验流程中每个 SqlNode 的工作上下文，当校验表达式时，通过 SqlValidatorScope 的 resolve 方法进行解析，如果成功的话会返回对应的 SqlValidatorNamespace 描述结果类型。

在此基础上，Calcite 提供了 SqlValidator 接口，该接口提供了所有与校验相关的核心逻辑，并提供了内置的默认实现类 SqlValidatorImpl 定义如下：

```java
public class SqlValidatorImpl implements SqlValidatorWithHints {
	// ...
  
  final SqlValidatorCatalogReader catalogReader;
  
  /**
   * Maps {@link SqlNode query node} objects to the {@link SqlValidatorScope}
   * scope created from them.
   */
  protected final Map<SqlNode, SqlValidatorScope> scopes =
      new IdentityHashMap<>();

  /**
   * Maps a {@link SqlSelect} node to the scope used by its WHERE and HAVING
   * clauses.
   */
  private final Map<SqlSelect, SqlValidatorScope> whereScopes =
      new IdentityHashMap<>();

  /**
   * Maps a {@link SqlSelect} node to the scope used by its GROUP BY clause.
   */
  private final Map<SqlSelect, SqlValidatorScope> groupByScopes =
      new IdentityHashMap<>();

  /**
   * Maps a {@link SqlSelect} node to the scope used by its SELECT and HAVING
   * clauses.
   */
  private final Map<SqlSelect, SqlValidatorScope> selectScopes =
      new IdentityHashMap<>();

  /**
   * Maps a {@link SqlSelect} node to the scope used by its ORDER BY clause.
   */
  private final Map<SqlSelect, SqlValidatorScope> orderScopes =
      new IdentityHashMap<>();

  /**
   * Maps a {@link SqlSelect} node that is the argument to a CURSOR
   * constructor to the scope of the result of that select node
   */
  private final Map<SqlSelect, SqlValidatorScope> cursorScopes =
      new IdentityHashMap<>();

  /**
   * The name-resolution scope of a LATERAL TABLE clause.
   */
  private TableScope tableScope = null;

  /**
   * Maps a {@link SqlNode node} to the
   * {@link SqlValidatorNamespace namespace} which describes what columns they
   * contain.
   */
  protected final Map<SqlNode, SqlValidatorNamespace> namespaces =
      new IdentityHashMap<>();
  
  // ...
}
```

可以看到 SqlValidatorImpl 当中有许多 scopes 映射 (SqlNode -> SqlValidatorScope) 和 namespaces (SqlNode -> SqlValidatorNamespace)，校验其实就是在一个个 SqlValidatorScope 中校验 SqlValidatorNamespace 的过程，另外 SqlValidatorImpl 有一个成员 catalogReader，也就是上面说到的 SqlValidatorCatalogReader，为 SqlValidatorImpl 提供了访问元数据的入口。（**注：为了简便，后续本文均以 scope 指代 SqlValidatorScope，使用 namespace 指代 SqlValidatorNamespace**）。

# 2. 入口函数

在第 3 章对 SqlNode 的介绍中，我们认识到 SqlNode 是一个嵌套的树结构，因此很自然的我们可以想到用处理树数据结构的一些思路或算法来处理整个 SqlNode 树，Calcite 正是基于这个思路，通过递归的方式遍历到每一个 SqlNode，并对每一个 SqlNode 都校验元数据。

校验流程极为繁琐，为了能够聚焦最核心的逻辑，我们下面代码块中的 SQL 为例**，**该 SQL 语法简单但同样是一个完整的 ETL 流程，且该 SQL 覆盖了 INSERT 和 SELECT 这两个最核心常用的 DML 语句**。**

```sql
INSERT INTO sink_table SELECT id FROM source_table WHERE id > -1
```

SQL 校验的整体入口是 SqlValidatorImpl 的 validate(SqlNode topNode) 方法。

```java
public SqlNode validate(SqlNode topNode) {
  SqlValidatorScope scope = new EmptyScope(this);
  scope = new CatalogScope(scope, ImmutableList.of("CATALOG"));
  final SqlNode topNode2 = validateScopedExpression(topNode, scope);
  final RelDataType type = getValidatedNodeType(topNode2);
  Util.discard(type);
  return topNode2;
}
```

首先，SqlValidatorImpl 会创建 CatalogScope 作为最外层的工作上下文，用于后续校验，这个  scope 也是后面一些 namespace 的 parentScope，创建 scope 完毕后，Calcite 校验进入到 validateScopedExpression。（**EmptyScope 的存在是为了更方便处理判空问题，并提供了一些核心的解析逻辑，它就类似一个 root scope**）

```java
private SqlNode validateScopedExpression(
    SqlNode topNode,
    SqlValidatorScope scope) {
	// 1. 规范 SqlNode
  SqlNode outermostNode = performUnconditionalRewrites(topNode, false);

	// ...

	// 2. 注册 namespace 和 scope 信息
  if (outermostNode.isA(SqlKind.TOP_LEVEL)) {
    registerQuery(scope, null, outermostNode, outermostNode, null, false);
  }

	// 3. 进行校验
  outermostNode.validate(this, scope);
  
	// ...
  return outermostNode;
}
```

第一步的 performUnconditionalRewrites 是为了**对我们的 SqlNode 进行规范，以简化后续校验处理**，规范的内容包括不限于以下几点：

-   如果一个 SELECT 子句带有 ORDER BY 关键字，SQL Parser 会将整个 SELECT 子句解析为 SqlOrderBy，在这一步会将 SqlOrderBy 转成 SqlSelect；
-   给 SqlDelete 和 SqlUpdate 设置 sourceSelect（这是一个 SqlSelect），后续对这两类进行校验的时候就会对它们的 sourceSelect 进行校验（即 validateSelect）；
-   ……

**第二步的 registerQuery，会创建该 SqlNode 对应的 namespace 和 scope，以及将 namespace 注入到对应的 scope 中**，以本例 SQL 进行调试可得到信息如下：

![idea-debug](https://tva1.sinaimg.cn/large/008i3skNly1gw8riky5syj31hc0od106.jpg)

1.  第一部分是 scopes 映射，这里包含了 SQL 文本每部分对应的名称解析空间，一般以 SELECT 为一个完整的空间，像 where、groupBy 等子句的 scope 也是其所属的 SELECT 子句的 SelectScope；
2.  第二部分是关于 namespace 注入到 scope，SelectScope 继承了 ListScope，有一个成员变量 children，这里面会存 SELECT 的数据源对应的 namespace，如该例存储的是 source_table 对应的 IdentifierNamespace；
3.  最后一部分是 namespaces 映射，INSERT、SELECT 或某个具体表和视图，都会有相对应的 namespace，用来表示他们执行结果的数据关系。

第三步校验调用 SqlNode 的 validate 方法，以前面的例子会走 SqlInsert 的 validate，然后调用 SqlValidatorImpl 的 validateInsert。

```java
public void validateInsert(SqlInsert insert) {
	// 1. 校验 namespace
  final SqlValidatorNamespace targetNamespace = getNamespace(insert);
  validateNamespace(targetNamespace, unknownType);

	// ...
	// 计算/校验 insert 插入列，insert 语句插入有两种形式
	// `insert sink_table values(...)：不指定插入列，默认为全部列
	// `insert sink_table(idx) ：指定插入列
	final RelDataType targetRowType = createTargetRowType(table, insert.getTargetColumnList(), false);

	// 2. 校验 source
	
  final SqlNode source = insert.getSource();
  if (source instanceof SqlSelect) {
    final SqlSelect sqlSelect = (SqlSelect) source;
    validateSelect(sqlSelect, targetRowType);
  } else {
    final SqlValidatorScope scope = scopes.get(source);
    validateQuery(source, scope, targetRowType);
  }

	// ...
	// 3. 校验 source 和 sink 是否兼容
  checkFieldCount(insert.getTargetTable(), table, source,
      logicalSourceRowType, logicalTargetRowType);

  checkTypeAssignment(logicalSourceRowType, logicalTargetRowType, insert);

  checkConstraint(table, source, logicalTargetRowType);

  validateAccess(insert.getTargetTable(), table, SqlAccessEnum.INSERT);
}
```

validateInsert 的核心逻辑有三块：

1.  校验 namespace；
2.  校验 SqlInsert 的 source，在该例里 source 是一个 SqlSelect，因此会走到 validateSelect；
3.  校验 source（数据来源） 和 target（目标表）是否兼容



# 3. 校验 namespace

Calcite 在校验 namespace 的元数据时采用了**模板方法**设计模式，**在 AbstractNamespace 的 validate 方法中定义了主校验流程，真正的校验逻辑（validateImpl）则交给每个具体的 namespace 各自实现。**

**AbstarctNamespace 中有一个成员变量 rowType，校验 namespace 其实就是解析得到 rowType 的值赋给对应的 namespace。**

![validate](https://tva1.sinaimg.cn/large/008i3skNly1gw8rcd9t8wj30db0c774i.jpg)

## 3.1 SqlValidatorImpl.validateNamespace

校验 namespace 的入口是 validateNamespace，做的是如下代码显示，即校验 namespace ，并建立 SqlNode → RelDataType 的映射关系放到 nodeToTypeMap 中。。

```java
protected void validateNamespace(final SqlValidatorNamespace namespace,
      RelDataType targetRowType) {
  // 1. 模板方法校验 namespace
  namespace.validate(targetRowType);
  if (namespace.getNode() != null) {
	// 2. 建立 SqlNode -> RelDataType 的映射关系
    setValidatedNodeType(namespace.getNode(), namespace.getType());
  }
}
```

## 3.2 AbstractNamespace.validate

namespace 的校验主流程由 validate 方法定义

```java
public final void validate(RelDataType targetRowType) {
  switch (status) {
  // 1. 第一次进入该方法时，status 都是 UNVALIDATED
  case UNVALIDATED:
    try {
	  // 2. 标记 status 为正在处理，避免重复处理
      status = SqlValidatorImpl.Status.IN_PROGRESS;
      Preconditions.checkArgument(rowType == null,
          "Namespace.rowType must be null before validate has been called");
	  // 3. 调用各自实现的 validateImpl
      RelDataType type = validateImpl(targetRowType);
      Preconditions.checkArgument(type != null,
          "validateImpl() returned null");
	  // 4. 记录解析得到的结果类型
      setType(type);
    } finally {
	  // 5. 标记 status 已完成
      status = SqlValidatorImpl.Status.VALID;
    }
    break;
  case IN_PROGRESS:
    throw new AssertionError("Cycle detected during type-checking");
  case VALID:
    break;
  default:
    throw Util.unexpected(status);
  }
}
```

除去 status 的标记更新，validate 实际的步骤也是两步：

1.  调用各自 namespace 实现的 validateImpl 方法获取对应的 type（RelDataType）；
2.  将解析得到的 type 赋值给 namespace 的 rowType。

以上述例子为例，SqlInsert 会首先开始校验其对应的 InsertNamespace，InsertNamespace 没有实现自己的 validateImpl 方法，但它继承了 IdentifierNamespace，会直接调用 IdentifierNamespace 的 validateImpl。**因此 InsertNamespace 的校验即是解析 target table（这是一个 Identifier） 的 rowType。**

## 3.3 validateImpl（IdentifierNamespace）

IdentifierNamespace 有个成员 resolvedNamespace（也是一个 SqlValidatorNamespace），该 IdentifierNamespace 对应的 SqlNode 指向一个表时，resolvedNamespace 就是一个 TableNamespace，存有真正的类型信息。

```java
public RelDataType validateImpl(RelDataType targetRowType) {
  // 1. 解析该 identifier 对应的 namespace，通常为 TableNamespace
	resolvedNamespace = Objects.requireNonNull(resolveImpl(id));

	// ...

	// 2. 获取 rowType，第一次执行时需要计算
	RelDataType rowType = resolvedNamespace.getRowType();
	
	// ...

	return rowType;
}
```

IdentifierNamespace 会先调用 resolveImpl，拿到对应的 TableNamespace，再调用 resolvedNamespace.getRowType() 得到 rowType。

### 3.3.1 IdentifierNamespace.resolveImpl

在 resolveImpl 中，IdentifierNamespace 会创建一个 SqlValidatorScope.ResolvedImpl 用于存放解析得到的 TableNamespace。

```java
private SqlValidatorNamespace resolveImpl(SqlIdentifier id) {
	// ...

	parentScope.resolveTable(names, nameMatcher,
	    SqlValidatorScope.Path.EMPTY, resolved);

	// ...

	return resolve.namespace;
}
```

这里的 parentScope 其实就是最开始创建的 CatalogScope，几乎是一个空实现，经过层层调用最后会调到 EmptyScope 的 resolve_ 方法。

### 3.3.2 EmptyScope.resolve_

在这里会调用 parentScope 中的 CalciteSchema（parentScope 中存有 SqlValidatorImpl，SqlValidatorImpl 中有 CalciteCatalogReader，CalciteCatalogReader 中有 CalciteSchema）的 getTable 拿到 TableEntryImpl 对象（定义如下），

```java
public static class TableEntryImpl extends TableEntry {
  private final Table table;

  // ...

  public Table getTable() {
    return table;
  }
}
```

整个调用链如图所示：

![invokerchain](https://tva1.sinaimg.cn/large/008i3skNly1gw8re5ia73j30wh0eut9m.jpg)

拿到 TableEntryImpl 对象，通过 TableEntryImpl.getTable() 就可以拿到我们注册进去的 Table 数据。

```
public static class TableEntryImpl extends TableEntry {
  private final Table table;

  // ...

  public Table getTable() {
    return table;
  }
}
```

在拿到 Table 后，会尝试将其转为一个 SqlValidatorTable（实际类型为 RelOptTableImpl），并注册到 TableNamespace 中。

最重要的是第3步，这里走了 Table 接口的 getRowType() 方法获取 rowType，如果我们有自定义的表实现了 Table 接口，也可以通过重写 getRowType()  自定义返回的行类型。拿到 rowType 后，将 rowType 赋给 table2，并基于 table2 创建了 TableNamespace。

直到此时 resolveImpl 执行完毕，我们得拿到了 TableNamespace，它保存有一个 table 变量，和继承自 AbstractNamespace 的类型为 RelDataType 的 rowType。rowType 记录这个 namespace 对应的行数据类型，而此时 TableNamespace 刚创建，rowType 为 null。

**需要注意的是，此时 rowType 仅赋值给了 TableNamespace 中的 table，TableNamespace 保存的 rowType 仍是 null（这可能是一个优化点，可以减少后续额外对 TableNamespace 做校验的步骤，但也可能是考虑到 TableNamespace 刚创建，status 仍为 UNVALIDATED）**

## 3.4 计算得到 rowType

因此当 IdentifierNamesapce 的 validateNamespace 函数进行到第二步：RelDataType rowType = resolvedNamespace.getRowType();，由于 rowType 为空，又会触发 TableNamespace 的校验。

```java
public RelDataType getRowType() {
  if (rowType == null) {
    validator.validateNamespace(this, validator.unknownType);
    Preconditions.checkArgument(rowType != null, "validate must set rowType");
  }
  return rowType;
}
```

校验 TableNamespace 结束后回到 AbstractNamespace.validate （此时该 AbstractNamespace 的真实类型为 IdentifierNamespace），我们拿到了 IdentifierNamespace 对应的行类型信息（即 resolvedNamespace 的 rowType），通过 setType 注入到 IdentifierNamespace 中，此时就完成了 IdentifierNamespace 的校验。

### 3.4.1 validateImpl（TableNamespace）

TableNamespace 的 validateImpl 实现如下：

```
protected RelDataType validateImpl(RelDataType targetRowType) {
  if (extendedFields.isEmpty()) {
    return table.getRowType();
  }
  final RelDataTypeFactory.Builder builder =
      validator.getTypeFactory().builder();
  builder.addAll(table.getRowType().getFieldList());
  builder.addAll(extendedFields);
  return builder.build();
}
```

前文提到，经过 EmptyScope.resolve_ 方法，TableNamespace 中绑定的 table 已经注入了 rowType 信息，因此这里可以直接获取到并返回。

回到 AbstractNamespace.validate （此时该 AbstractNamespace 的真实类型为 TableNamespace），我们拿到了 TableNamespace 对应的行类型信息（即它的 TableNamespace 的 rowType），通过 setType 注入到 TableNamespace 中，完成TableNamespace 的校验。

## 3.5 总结

下图是 IdentifierNamespace 校验的完整流程。

![zongjie](https://tva1.sinaimg.cn/large/008i3skNly1gw8req302vj31a30f240c.jpg)

# 4. 校验 source

校验 source 就是一个校验 select 的过程，延续上面的例子，校验的 SqlNode（实际类型为SqlSelect）对应的语句为 select id as idx from source_table where id > -1，SqlSelect 有自己绑定的 SelectNamespace，当然也有校验 namespace 的部分（给绑定的  SelectNamespace 注入 rowType 属性）。

校验 select 的入口函数为 validateSelect，下图是整个 validateSelect 的工作流程。

![work](https://tva1.sinaimg.cn/large/008i3skNly1gw8rezfvirj31hc0ph0ut.jpg)

可以看到 validateSelect 对 select 语句的每个组成部分都做了 validate，当中**最重要的是 validateSelectList，这个方法校验 select 的列是否存在（能从数据源表中检索到），以及为这部分查出来的列建立对应的类型信息（RelDataType），也即 SelectNamespace 的 rowType**。

举个例子，针对 `SELECT id FROM source_table WHERE id > -1`，validateSelectList 则需要校验 id 这个列是否在 source_table 中存在。



# 5. 校验 source 和 target 是否兼容

这部分校验具体检查四块内容：

1.  checkFieldCount：检查插入数据列的个数是否合法，检查非空；
2.  checkTypeAssignment：检查来源列和插入表对应列的类型是否兼容；
3.  checkConstraint：约束检查；
4.  validateAccess：权限校验，默认情况下都能通过。



# 6. 参考资料

1.  Apache Calcite教程-validate校验：https://blog.csdn.net/QXC1281/article/details/90343560