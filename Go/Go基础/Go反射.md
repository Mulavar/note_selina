Go 中 interface{} 类型的变量包括 **(type, value)** 两部分。

type：包括 **static type** 和 **concrete type**。

- static type：编码阶段看到的类型
- concrete type：runtime 系统看到的类型



`reflect` 包主要有两个方法：`TypeOf()` 和 `ValueOf()`，分别获得对象的 **concreteType** 和 变量值。

由于编译期间用的是 static type，所以假设一个方法声明需要类型为 float64 的参数，我们不能传一个 static type是 interface{} ，concreate type 是 float64 的参数，这时候需要进行强转。

强转有两种场景：

1. 知道需要强转的类型
2. 不知道强转的类型（可利用反射获取）



Go 的运行时反射功能由 reflect 包实现，

[参考文献](https://juejin.im/post/5a75a4fb5188257a82110544)



go的interface 被两个元素 value 和 type 所表示。

只有在 value 和 type 同时为 nil 的时候，判断 `interface == nil` 才会为 true。



```go
package main

import (
    "errors"
    "fmt"
)

type Err struct {
    err string
}

func (e *Err) Error() string {
    return e.err
}

func returnErr() *Err {
    return nil
}

func main() {
    var err error

    err = returnErr()
    fmt.Println(err, err != nil)
}
```

会打印 ` true`，因为err虽然value为nil，但是type为 *Err。

因此，尽量不要将该结果赋给一个接口变量。



[参考资料](https://deepzz.com/post/why-nil-error-not-equal-nil.html)

