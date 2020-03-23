GitLab CI/CD（后简称 GitLab CI）是一套基于 GitLab 的 CI/CD 系统，通过`.gitlab-ci.yml`配置。



## Pipeline

相当于一次构建任务，一般每次commit或pr会触发一次pipeline。

每个pipeline包含一或多个stages。



## Stages

stage表示我们自定义的一些构建阶段，并决定这些stage的构建运行顺序：

- 所有 stages 会按照顺序运行，即当一个 Stage 完成后，下一个 Stage 才会开始
- 如果其中一个stage失败（且未设置允许失败），后面的都不会执行

每个stage的具体工作由jobs去描述



##  Jobs

job表示每个阶段的具体任务目标，每个job绑定在一个stage，并通过定义script字段说明需要执行的具体命令。

需要注意：

- 一个stage的jobs是并行运行的；
- 每个stage之间的jobs是串行运行的；

image

services



## cache

cache指定需要**在job之间缓存的文件或目录**，只能使用该项目工作空间内的路径。





## Anchors(锚点)

锚点可以用来复制属性或者继承属性





[官方文档](https://docs.gitlab.com/ce/ci/docker)

[常用变量](http://www.ttlsa.com/auto/gitlab-cicd-variables-zh-document/)

