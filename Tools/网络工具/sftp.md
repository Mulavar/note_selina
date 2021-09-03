## 1. 介绍

SFTP 是一种安全文件传输协议，全称为 **Secure FileTransferProtocol**，相比于 FTP 更加安全。

## 2. 连接

SFTP 基于 SSH 协议，默认的身份认证方法也与 SSH 协议保持一致，使用

`sftp user_name@remote_server_address[:path]`

如果远程服务器自定义了连接的端口，可以使用 `-P` 参数：

`sftp -P remote_port user_name@remote_server_address[:path]`

## 3. 常用操作

上传文件

put local_file [remote_path]

下载文件

get remote_file [local_path

执行本地 shell 命令

![command]



[SFTP 命令详解](https://zhuanlan.zhihu.com/p/51749905)

