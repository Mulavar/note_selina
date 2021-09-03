[toc]

结合《dubbo-getty 架构剖析》这一节内容，想要理解 dubbo-go 如何使用 getty，只需弄清楚这四个子问题：

1. dubbo-go 如何实现 Writer/Reader 接口（client/server）；
2. dubbo-go 如何实现 EventListener 接口（client/server）；
3. dubbo-go 如何实现 NewSessionCallback 函数（client/server）；
4. dubbo-go 如何接入 getty（client/server）；

# Writer/Reader

dubbo-go 定义了一个 Codec 接口，用于统一做请求/响应的编解码。

```go
type Codec interface{
   EncodeRequest(request *Request) (*bytes.Buffer, error)
   EncodeResponse(response *Response) (*bytes.Buffer, error)
   Decode(data []byte) (DecodeResult, int, error)
}
```

## server

### RpcServerPackageHandler

## client

### RpcClientPackageHandler 方法



# NewSessionCallback

## server

定义在 remoting/getty/getty_server.go 中

```go
func (s *Server) newSession(session getty.Session) error {
    // ...
    
    conf := s.conf
	session.SetName(conf.GettySessionParam.SessionName)
	session.SetMaxMsgLen(conf.GettySessionParam.MaxMsgLen)
    // 核心部分是设置 PkgHandler 和 EventListener
	session.SetPkgHandler(NewRpcServerPackageHandler(s))
	session.SetEventListener(s.rpcHandler)
	session.SetReadTimeout(conf.GettySessionParam.tcpReadTimeout)
	session.SetWriteTimeout(conf.GettySessionParam.tcpWriteTimeout)
	session.SetCronPeriod((int)(conf.heartbeatPeriod.Nanoseconds() / 1e6))
    session.SetWaitTime(conf.GettySessionParam.waitTimeout)
    
	// ...
}

```

