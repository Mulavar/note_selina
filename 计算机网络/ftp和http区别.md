## 协议介绍

HTTP是Hyper Text Transfer Protocol，超文本传输协议；

FTP是File Transfer Protocol,文件传输协议；



## 两者区别

1. FTP协议比HTTP复杂一些，因为FTP协议需要建立两个TCP连接：
   - 控制连接：端口21
   - 数据连接：端口20



2. FTP服务器必须在整个会话期间保留用户的状态（state）信息，而HTTP是无状态的。



3. FTP的**控制连接是持久连接，数据连接是非持久连接**；而HTTP**既可以使用非持久连接，也可以使用持久连接**。 



## 参考资料

[ftp和http的区别](https://blog.csdn.net/only_musm/article/details/78983364)

