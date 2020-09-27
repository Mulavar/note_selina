## gradle-wrapper

Gradle 的构建脚本通过 Groovy 编写，Groovy是用于Java虚拟机的一种敏捷的动态语言。



Gradle Wrapper是gradle的一层包装。由于gradle迭代非常迅速，如果每个项目都直接去引用，更改版本会非常麻烦，手动配置每一个项目对应的gradle也很不明智，所以gradle引入了gradle-wrapper，**通过读取配置文件中gradle的版本，为每个项目自动下载和配置gradle**。



gradle-wrapper根据项目中的 **gradle/wrapper/gradle-wrapper.properties** 去下载。

gradle-wrapper.properties内容如下所示：

```java
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists
distributionUrl=http
```



distributionUrl：要下载的gradle地址及版本，**gradle-wrapper**会去`/<user_path>/.gradle/wrapper/dists`目录下查找，如果没有对应版本的gradle就下载，所以如果我们卡顿下载不下来，可以先去找国内的镜像网站下载下来，再放入相应的路径中。

gradle有三种版本：

- gradle-xx-all.zip：完整版gradle，包含了各种二进制文件、源代码、离线文档。
- gradle-xx-bin.zip：只包含了二进制文件，没有文档和源代码。
- gradle-xx-src.zip：只包含了源码，不能用来编译工程。



具体顺序：

1. 下载gradle对应版本
2. 根据下载的gradle在`/<user_path>/.gradle/wrapper/dists`创建对应目录**gradle-xx-bin**，在**gradle-xx-bin**中创建名字为<distributionUrl_md5_hash>的目录，将下载的gradle放到里面
3. 解压



由于gradle命令像gradle一样需要迭代，所以wrapper对gradle命令行也做了封装，使用同一个gradlew命令，就会自动去执行具体版本对应的gradle命令。



## Gradle配置文件

gradle配置的目录结构跟maven项目非常类似，只是原来的pom.xml替换成了：

- settings.gradle：设置父子模块依赖一类的配置，用于模块交互；
- build.gradle：用于项目构建。



```gradle
plugins {
    id 'java' // 表示这是java语言项目
}

group 'org.example'
version '1.0'

sourceCompatibility = 1.8

repositories {
    mavenLocal()
    mavenCentral()
}

dependencies {
    testCompile group: 'junit', name: 'junit', version: '4.12'
}

// 设置打包名字和版本号
jar {
    baseName = "data-tructure"
    version = "1.0"
}
```