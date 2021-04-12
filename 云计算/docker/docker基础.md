## 1. Docker简介

docker是一种linux容器的封装，并提供了简单易用的接口供用户使用。

linux容器是相当于在进程外面套了一个容器，将容器和操作系统进行了隔离，可以理解为一个简单的虚拟环境，但不像虚拟机占用大量资源。

- image（镜像）：想要使用的服务及应用程序的打包镜像；
- container（容器）：运行image生成的容器实例，一个image可对应多个container；
- registry（仓库）：分为公有和私有，用来保存用户构建的镜像。



## 2. 使用docker的原因

- 占用资源少
- 体积小
- 接口简单易懂



## 3. 常用命令

### 3.1 镜像

#### 3.1.1 列出镜像

```bash
docker imgaes

docker image ls
```

查看镜像摘要的命令为：`docker image ls --digests`。



#### 3.1.2 删除镜像

```bash
docker image rm xxx 
# xxx可以是
# 	repository:tag
# 	镜像短ID
# 	镜像完整ID
# 	镜像摘要
```

> **Untagged和Deleted**
>
> 一个镜像可以有多个标签，但只有一个ID。当删除镜像时，首先要取消其连接的标签信息，即 untagged 信息。如果此时仍有其他标签连接该镜像，则 delete 行为不会发生。

#### 3.1.3 拉取镜像

```bash
docker pull [选项] [Docker Registry 地址[:端口号]/]仓库名[:标签]
# 其中 Docker Registry 默认是官方仓库，标签默认为latest
# example 
# docker image pull library/hello-world
```



### 3.2 容器

#### 3.2.1 列出容器

```bash
docker container ls
docker container ls -a
docker container ls --all

docker ps -a # 列出所有容器
docker ps -q # 列出所有正在运行的容器的编号
```



#### 3.2.2 运行容器

```bash
docker container run [-it] [--rm] [-p <local_port>:<docker_port>] <container_name> <command_name>
# -i: 交互式操作
# -t: 分配终端
# --rm: 退出时删除该容器
# -p: 映射容器端口和主机端口
##################################
# example
# docker container run hello-world
# docker container run -it -p 8081:8081 opensuse bash

# run是每次新建一个容器，start是重复使用容器
docker container start <container_id>
```

**启动容器的时候为容器命名: **增加 `--name` 参数。



#### 3.2.3 停止容器

```bash
docker container stop <container_id>
```

**停止所有容器: **`docker stop $(docker ps -q)`



#### 3.2.4 删除容器

```bash
docker container rm <container_id>
```

**删除所有容器: **`docker rm $(docker ps -aq)`



#### 3.2.5 容器和宿主机文件交互

```bash
docker container cp <path/to/file> <container_id>:<path/to/file>
docker container cp <container_id>:<path/to/file> <path/to/file> 
```

