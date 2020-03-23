## Mybatis项目构建

 MyBatis 框架的详细步骤：

- 导入 MyBatis 框架的 jar 包；
- 创建核心配置文件 sqlMapConfig.xml ；
- 创建映射文件 Mapper.xml；

### 导入MyBatis的jar包

```
<!-- https://mvnrepository.com/artifact/org.mybatis/mybatis -->
<dependency>
    <groupId>org.mybatis</groupId>
    <artifactId>mybatis</artifactId>
    <version>3.4.6</version>
</dependency>
<!-- https://mvnrepository.com/artifact/mysql/mysql-connector-java -->
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>6.0.6</version>
</dependency>
```

### 创建核心配置文件

如下一个最简单的配置例子：

```
<configuration>
	<!-- 可指定properties文件用于动态传参 -->
    <properties resource="test.properties">
    	<property name="" value=""/>
    </properties>
    <settings>
        <setting name="cacheEnabled" value="true"/>
        <setting name="lazyLoadingEnabled" value="false"/>
        <!--<setting name="logImpl" value="STDOUT_LOGGING"/> &lt;!&ndash; 打印日志信息 &ndash;&gt;-->
    </settings>

<!--    <typeAliases>-->
<!--        <typeAlias type="com.luo.dao.UserDao" alias="User"/>-->
<!--    </typeAliases>-->


    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"/> <!--事务管理类型-->
            <dataSource type="POOLED">
            	<!-- 设置占位符，动态传参 -->
                <property name="username" value="${username}"/>
                <property name="password" value="${password}"/>
                <property name="driver" value="com.mysql.jdbc.Driver"/>
                <property name="url" value="jdbc:mysql://localhost:3306/WEB13"/>
            </dataSource>
        </environment>
    </environments>

    <mappers>
    	<!-- 加载映射文件 mapper -->
        <mapper resource="config/user.xml"/>
    </mappers>

</configuration>
```

### 创建映射文件

```
<mapper namespace="User">
    <!-- 
    	1. 根据id查询得到一个user对象(id属性值必须是唯一的);
    	2. 使用parameterType属性指明查询时使用的参数类型;
    	3. resultType属性指明查询返回的结果集类型;
    	4. resultType="cn.web.entity.User"就表示将查询结果封装成一个User类的对象返回;
    -->
    <select id="getUser" parameterType="String"
            resultType="User">
        select * from ${table} where id=#{id}
    </select>

    <insert id="insertUser" parameterType="String">
        insert into ${table}(username, password, email) values(#{username}, #{password}, #{email})
    </insert>

    <update id="create_table" parameterType="map">
        ${sql}
    </update>

    <delete id="deleteUser" parameterType="String">
        delete from ${table} where username = #{username};
    </delete>
</mapper>
```



## 生命周期

#### SqlSessionFactoryBuilder

这个类可以被实例化、使用和丢弃，一旦创建了 SqlSessionFactory，就不再需要它了。 因此 SqlSessionFactoryBuilder 实例的最佳作用域是**方法作用域（也就是局部方法变量）**。 你可以重用 SqlSessionFactoryBuilder 来创建多个 SqlSessionFactory 实例，但是最好还是不要让其一直存在，以保证所有的 XML 解析资源可以被释放给更重要的事情。

#### SqlSessionFactory

SqlSessionFactory 一旦被创建就应该在应用的运行期间一直存在，没有任何理由丢弃它或重新创建另一个实例。 使用 SqlSessionFactory 的最佳实践是在应用运行期间不要重复创建多次，多次重建 SqlSessionFactory 被视为一种代码“坏味道（bad smell）”。因此 SqlSessionFactory 的最佳作用域是**应用作用域。 有很多方法可以做到，最简单的就是使用单例模式或者静态单例模式。**

#### SqlSession

每个线程都应该有它自己的 SqlSession 实例。SqlSession 的实例不是线程安全的，因此是不能被共享的，所以它的最佳的作用域是**请求或方法作用域**。 绝对不能将 SqlSession 实例的引用放在一个类的静态域，甚至一个类的实例变量也不行。 如果你现在正在使用一种 Web 框架，要考虑 SqlSession 放在一个和 HTTP 请求对象相似的作用域中。 换句话说，每次收到的 HTTP 请求，就可以打开一个 SqlSession，返回一个响应，就关闭它。 这个关闭操作是很重要的，你应该把这个关闭操作放到 finally 块中以确保每次都能执行关闭。 下面的示例就是一个确保 SqlSession 关闭的标准模式：

```java
SqlSession session = sqlSessionFactory.openSession();
try {
  // 你的应用逻辑代码
} finally {
  session.close();
}
```



## XML配置

- 在 properties 元素体内指定的属性首先被读取。
- 然后根据 properties 元素中的 resource 属性读取类路径下属性文件或根据 url 属性指定的路径读取属性文件，并覆盖已读取的同名属性。
- 最后读取作为方法参数传递的属性，并覆盖已读取的同名属性。

## XML映射文件





## 常见问题

### $和#的区别

1. \#将传入的数据都当成一个字符串，会对自动传入的数据加一个双引号。

    $将传入的数据直接显示生成在sql中，**一般用于传入数据库对象，例如传入表名**。

2. \#方式能够很大程度防止sql注入。

    $方式无法防止Sql注入。