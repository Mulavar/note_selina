### 核心接口

具体的处理逻辑通过`ServeHTTP`函数实现，用户只需要与`ResponseWriter`,  `*Request`交互即可。

go的http库使用接口 `Handler` 包装了该方法。

```go
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```



### 注册Handler

`http.HandleFunc("/", helloHandlerF)`：这里的helloHandlerF是一个 `func(ResponseWriter, *Request)` 函数，内部被封装为 `Handler` 接口。

`http.Handle("/", &helloHandler{})`：这里的helloHandler是一个实现了 `Handler` 接口的结构体。




### ServeMux

- URL 分为两种：
    - 末尾是 `/`：表示一个子树，后面可以跟其他子路径；
    -  末尾不是 `/`，表示一个叶子，固定的路径；
- **以`/` 结尾的 URL 可以匹配它的任何子路径**，比如 `/images` 会匹配 `/images/cute-cat.jpg`
- 它采用最长匹配原则，如果有多个匹配，一定采用匹配路径最长的那个进行处理
- 如果没有找到任何匹配项，会返回 404 错误
- `ServeMux` 也会识别和处理 `.` 和 `..`，正确转换成对应的 URL 地址

`ServeMux` 实现的核心路由方法为 `Handle()` 和 `Handler()`。

#### Handle()

注册路由，绑定 **url** 和对应的 `Handler`

#### Handler()

根据路由寻找对应的 `Handler` 处理



### Server

`http.ListenAndServe()`  内部其实是调用了 `Server` 结构体的 `ListenAndServe()`。

```go
func ListenAndServe(addr string, handler Handler) error {
    server := &Server{Addr: addr, Handler: handler}
    return server.ListenAndServe()
}
```

通过层层调用，最后调用到 `Server.Handler.ServeHTTP` 方法。





### Request

封装好的客户端请求



### ResponseWriter

ResponseWriter 是一个接口，定义了三个方法：

- Header()：返回一个 Header 对象，可以通过它的 Set() 方法设置头部，注意最终返回的头部信息可能和你写进去的不完全相同，因为后续处理还可能修改头部的值（比如设置 **Content-Length、Content-type** 等操作）
- Write()： 写 response 的主体部分，比如 html  或者 json 的内容就是放到这里的
- WriteHeader()：设置 status code，如果没有调用这个函数，默认设置为 http.StatusOK， 就是 200 状态码





[参考链接](https://juejin.im/post/58cffa535c497d0057cfcdfe)