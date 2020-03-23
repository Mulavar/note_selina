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

