## goproxy

```bash
export GOPROXY=https://goproxy.io,direct
# or
export GOPROXY=https://goproxy.cn,direct
```



## go module中用到的关键字

- require：引用哪些包；
- replace：替换一些包的下载和引用路径，注意 replace 不会嵌套生效，比如本项目对将 A replace 为 A1，A1 当中声明了对 B 的 replace，则 B 的 replace 不会生效；
- exclude：排除下载和引用哪些包。



## go module 常用命令

`go mod init ${module_name}` ：初始化 go module；

`go mod tidy`：整理当前 go module 的依赖，并更新 go.sum；

`go mod download`：下载 go.mod 中声明的包依赖

`go clean --modcache`：清除 go module 缓存





[go module使用](https://colobu.com/2018/08/27/learn-go-module/)





