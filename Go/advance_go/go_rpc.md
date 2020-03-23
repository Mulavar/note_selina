go标准库提供的RPC框架默认使用Go语言特有的Gob编码，因此想要实现跨语言交互需要一些扩展。

同样，go语言内在的HTTP协议上的RPC服务也使用了Gob编码。





### grpc和protobuf

protoc是 Protobuf 编译器，protoc-gen-go 是被 protoc 使用生成 go 语言可用的源文件的插件。

