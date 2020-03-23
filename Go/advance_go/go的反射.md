go中interface{}类型的变量包括 **(type, value)** 两部分。

type：包括 **static type** 和 **concrete type**。

- static type：编码阶段看到的类型
- concrete type：runtime 系统看到的类型



`reflect` 包主要有两个方法：`TypeOf()` 和 `ValueOf()`，分别获得对象的 **concreteType** 和 变量值。

由于编译期间用的是static type，所以假设一个方法声明需要类型为 float64 的参数，我们不能传一个 static type是 interface{} ，concreate type 是 float64 的参数，这时候需要进行强转。

强转有两种场景：

1. 知道需要强转的类型
2. 不知道强转的类型（可利用反射获取）



[参考文献](https://juejin.im/post/5a75a4fb5188257a82110544)