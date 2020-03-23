Maven的每个生命周期阶段绑定了一个或多个插件目标，每个插件包含多个插件目标可以分别绑定到多个阶段中。

`maven <plugin>:<goal>`

`maven <phase>`



### 常用命令

#### 打包（跳过测试）

`mvn package -Dmaven.test.skip=true`



#### 安装本地jar文件到本地仓库

```
mvn install:install-file -Dfile=d:\setup\dubbo-2.8.4.jar -DgroupId=com.alibaba -DartifactId=dubbo -Dversion=2.8.4 -Dpackaging=jar
```





#### 引用本地jar包（jar包放到项目目录下）

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



maven从私服下载依赖包时如果失败会形成 *.lastUpdated文件



### 模块

#### 聚合

假设我们想一次构建两个模块，两个模块是同一个项目下不同子功能的实现，就可以利用聚合这个特性，而不是到每个模块目录下分别使用 mvn 命令。

聚合模块的 packing 属性必须为 pom。



#### 继承

如果有两个子模块有许多相同的配置，可以将这个配置抽取出来形成一个父模块，同样父模块的 packing 属性也必须为 pom。



一般继承和聚合经常用在同一个模块上，即该模块既是聚合模块也是集成模块。

