docker是一种linux容器的封装，并提供了简单易用的接口供用户使用。

linux容器是相当于在进程外面套了一个容器，将容器和操作系统进行了隔离，可以理解为一个简单的虚拟环境，但不像虚拟机占用大量资源。

## 使用docker的原因

- 占用资源少
- 体积小
- 接口简单易懂



## docker概念

- image：想要使用的服务及应用程序的打包镜像。
- container：运行image生成的容器实例，一个image可对应多个container。
- 仓库



## 常用命令：

### image

```
docker imgaes

docker image ls

docker image rm <image_name>

# docker image pull <repository_name>/<image_name>
docker image pull library/hello-world
```



### container

```
docker container run [-it] [-p <local_port>:<docker_port>] <container_name> <command_name>
# example
docker container run hello-world
docker container run -it -p 8081:8081 opensuse bash

docker container ls
docker container ls --all
docker container rm <container_id>

# 与run的区别，run是每次新建一个容器，start是重复使用容器
docker container start <container_id>

# docker cp
docker container cp <path/to/file> <container_id>:<path/to/file>
docker container cp <container_id>:<path/to/file> <path/to/file> 
```



### 拉取镜像

```
docker pull [选项] [Docker Registry 地址[:端口号]/]仓库名[:标签]
```



### 停止所有容器

```
docker stop $(docker ps -q)
```



### 删除所有容器

```
docker rm $(docker ps -aq)
```

