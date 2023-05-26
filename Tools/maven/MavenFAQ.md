# 1 Maven命令格式

Maven的每个生命周期阶段绑定了一个或多个插件目标，每个插件包含多个插件目标可以分别绑定到多个阶段中。

- maven \<plugin-prefix>:\<goal>

- maven \<phase>



# 2 Maven常用命令

## 2.1 打包

mvn package

### 2.1.1 跳过测试

mvn package -Dmaven.tesk.skip=true 或 mvn package -DskipTests=true，两者的区别是，前者会跳过测试源代码的编译，而后者只跳过运行。



## 2.2 安装本地jar文件到本地仓库

```bash
mvn install:install-file -Dfile=d:\setup\dubbo-2.8.4.jar -DgroupId=com.alibaba -DartifactId=dubbo -Dversion=2.8.4 -Dpackaging=jar
```





## 2.3 引用本地jar包（jar包放到项目目录下）

```bash
<dependency>
    <groupId>cn</groupId>
    <artifactId>javasdk</artifactId>
    <version>4.2.7</version>
    <scope>system</scope>
    # 项目根目录
    <systemPath>${project.basedir}/src/test/resource/hyperchainsdk-4.2.7-jar-with-dependencies.jar</systemPath>
</dependency>
```



## 2.4 删除 lastUpdated 文件

```shell
find ~/.m2/repository -name "*.lastUpdated" -exec grep -q "Could not transfer" {} \; -print -exec rm {} \;
```



# 3 模块

## 3.1 聚合

假设我们想一次构建两个模块，两个模块是同一个项目下不同子功能的实现，就可以利用聚合这个特性，而不是到每个模块目录下分别使用 mvn 命令。

聚合模块的 packing 属性必须为 pom。



## 3.2 继承

如果有两个子模块有许多相同的配置，可以将这个配置抽取出来形成一个父模块，同样父模块的 packing 属性也必须为 pom。

**一般继承和聚合经常用在同一个模块上，即该模块既是聚合模块也是集成模块**。



# 4 常见问题

## *.lastUpdated 文件

maven从私服下载依赖包时如果失败会形成 *.lastUpdated文件

## -N -U -D -P 参数区别

-N,--non-recursive Do not recurse into sub-projects

意思是，不递归到子项目(子模块)。

-U,--update-snapshots Forces a check for missing releases
and updated snapshots on remote repositories

意思是：强制刷新本地仓库不存在release版和所有的snapshots版本。

-D 表示设置 Properties属性，

mvn -DpropertyName=propertyValue clean package

如果 propertyName 不存在于 pom.xml 文件中，它将被设置。如果 propertyName 已经存在 pom.xml 文件中，其值将被作为参数传递的值覆盖。要设置多个变量，请使用多个空格分隔符加-D

-P 代表 Profiles 配置文件的属性，也就是说在 \<profiles> 指定的 \<id> 中，可以通过-P进行传递或者赋值。
