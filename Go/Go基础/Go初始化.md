## 空值

Go 里每种类型都有属于自己的空值，这一点与 Java 不同（Java 的空值只有 null，其他类成员变量是一种默认值）。

int 的空值是 0

string 空值是 ”” 而不是 nil

slice 空值是长度为 0 的 slice 而不是 nil，

map 空值是 nil，

error 空值是 nil，

struct 空值是一个“所有成员都是空值”的空 struct 而不是 nil，



因此 slice 定义后可以直接使用（比如调用 `append` 函数），而直接往未初始化 map 里写会出错，但 go 的 map 有一点比较特殊，对未初始化的 map 进行读操作不会报错，他的表现与 empty map 相同（Go 特意这么设计，不知道为啥）。

https://golangtc.com/t/562c39f8b09ecc56ca000198