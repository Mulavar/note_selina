Calcite 在解析 SQL 文本时会将所有类型的 token 统一解析为 SqlDataTypeSpec：

```java
public class SqlDataTypeSpec extends SqlNode {
  private final SqlTypeNameSpec typeNameSpec;
  private final @Nullable TimeZone timeZone;
  private final @Nullable Boolean nullable;
}
```

其中 SqlTypeNameSpec typeNameSpec 存储了具体的类型名字信息，用以区分不同的类型：

```java
public abstract class SqlTypeNameSpec {
  private final SqlIdentifier typeName;
  private final SqlParserPos pos;
}
```

SqlTypeNameSpec 是一个抽象类，主要有以下几种实现：

![AST类型系统](https://tva1.sinaimg.cn/large/008i3skNly1gyovum3sl0j31200amjse.jpg)

SqlBasicTypeNameSpec

指代基本的 SQL 类型名，如 BOOLEAN、INTEGER、INT、TINYINT 等等。

sqlTypeName：其类型 SqlTypeName 是一个枚举，记录了所有内置 SQL 基本类型的信息。

precision：某些类型会有精度信息，即字段长度如 DECIMAL、VARCHAR 等。

scale：范围，小数位数。

如 DECIMAL(2,1) 表示这个数长度 2 位，有 1 位是小数。



SqlAlientSystemTypeNameSpec

某个类型的别名，一般用于兼容一些外置的 SQL 引擎，比如 UNSIGNED 在 MYSQL 中其实是 INTEGER 的别名。

