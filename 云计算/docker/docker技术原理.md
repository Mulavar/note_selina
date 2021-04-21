容器其实一种沙盒技术，通过对进程限定边界，使得进程之间不互相干扰。



Docker 与 PaaS 项目一样，本质是使用 Cgoups 和 Namespace 机制来为每一个应用创建一个独立的沙箱环境，然后在沙箱中启动这些应用，但 Docker 最大的创新点在于 **打包镜像** ：打包执行依赖的文件系统和应用的二进制文件。通过这个创新，Docker 可以保证本地和云端环境的高度一致。因此，Docker的技术原理主要可以分为三部分：

- Cgoups：资源隔离和分组；
- Namespace：修改进程视图，隔离文件系统、进程和网络；
- rootfs



## Cgroups

Cgroup 全称为 Control Groups，是一种物理资源（CPU、内存等资源）隔离机制。

在



## Namespace

Linux 中有许多 namespace 用来隔离内核资源，使用 namespace 可以让进程各自拥有独立的部分全局资源，每个进程的运行只会影响自己 namespace 的进程，而不会影响到其他 namesapce。

Linux 支持的 namespace 有：

```
名称        宏定义             隔离内容
Cgroup      CLONE_NEWCGROUP   Cgroup root directory (since Linux 4.6)
IPC         CLONE_NEWIPC      System V IPC, POSIX message queues (since Linux 2.6.19)
Network     CLONE_NEWNET      Network devices, stacks, ports, etc. (since Linux 2.6.24)
Mount       CLONE_NEWNS       Mount points (since Linux 2.4.19)
PID         CLONE_NEWPID      Process IDs (since Linux 2.6.24)
User        CLONE_NEWUSER     User and group IDs (started in Linux 2.6.23 and completed in Linux 3.8)
UTS         CLONE_NEWUTS      Hostname and NIS domain name (since Linux 2.6.19)
```



## rootfs



**docker为什么轻量？**

docker只打包了文件系统，不包括操作系统内核。



[Linux Namespace 简介](https://www.cnblogs.com/sparkdev/p/9365405.html)

[Namepsace 概述](https://www.cnblogs.com/sparkdev/p/9365405.html)

[浅谈Linux Cgroups机制](https://zhuanlan.zhihu.com/p/81668069)

[Linux 资源管理之 cgroups 简介](https://tech.meituan.com/2015/03/31/cgroups.html)