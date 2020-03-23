## goproxy

需要翻墙的包要设置 `export GOPROXY=https://goproxy.io`

其他包需要关掉代理



## go module中用到的关键字

- require：引用哪些包
- replace：替换一些包的下载和引用路径
- exclude：不下载和引用哪些包



## go module常用命令

`go mod init packagename ` 创建一个空的`go.mod` (只有第一行 `module github.com/smallnest/rpcx`)。

 `go get ./... ` 查找依赖，并记录在`go.mod`文件中(可以指定 `-tags`,这样可以把tags的依赖都查找到)

`go mod tidy` 可以用来为`go.mod`增加丢失的依赖，删除不需要的依赖

`go mod download` 下载包依赖

`go clean --modcache` 清除缓存命令





[go module使用](https://colobu.com/2018/08/27/learn-go-module/)





