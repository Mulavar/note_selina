## 前言

既然要讲session和cookie，首先要明确为什么会有session和cookie这两个概念。

session的引入是因为HTTP是个无状态的协议，所以为了保证用户多次连接有状态保留和传递，服务器就存储了一个名为session的东西，里面包含着一些用户的信息和状态，而后为了识别用户，又引入了cookie机制，由客户端保存cookie，每次访问时客户端发送自己的cookie，这服务器就能够识别用户并读取相应的session信息了。



这里抛出一个问题，我们都说HTTP是一种无状态的协议，而HTTP是基于TCP的应用层协议，TCP本身又是面向连接的协议，那为什么HTTP是个无状态的协议呢？

这里明确一个点，HTTP使用TCP是为了保证HTTP数据传输的可靠性，然而两次数据传输是相互独立的，所以HTTP连接本身是无状态的。

所以：

- TCP的面向连接是**基于网络底层的数据传输；**
- HTTP的无状态是**基于应用层面的沟通交互。**



## 

在第一次访问网站时，服务器会创建一个session保存用户的上下文信息，通过唯一的sessionId进行区分，并返回HTTP响应，响应的信息中包含Set-Cookie头部。该头部包含了sessionId。

当用户第二次访问网站时，服务器读取HTTP请求带的cookie信息，验证成功后读取session信息返回响应。



## 参考资料

[session，cookie，token究竟是什么](https://segmentfault.com/a/1190000017831088)

