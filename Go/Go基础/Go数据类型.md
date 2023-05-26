[toc]



## 字符串

字符串底层对应的是字节数组，有只读属性，复制的时候只复制地址和长度。



## 数组

是一种值类型，可修改，但本身赋值和函数传参都是采用整块数组复制的方式。

与 C 语言不同，Go 的数组变量表示整个数组，而不是隐式表示数组第一个元素的地址，所以如上所说，数组的复制是要整块复制。且**数组长度是数组类型的组成部分，因此指向不同长度数组的数组指针类型也不同**。



## 切片

底层结构和字符串类似，但解除了只读限制。复制过程因为每个切片都含有底层数据的指针，所以赋值和传参也是复制指针和长度。

slice 源码结构如下：

```go
// runtime/slice.go
type slice struct {
    array unsafe.Pointer // 元素指针
    len   int // 长度 
    cap   int // 容量
}
```

- 指针，指向底层数组；
- 长度，表示切片可用元素的个数，也就是说使用下标对 slice 的元素进行访问时，下标不能超过 slice 的长度；
- 容量，底层数组的元素个数，容量 >= 长度。在底层数组不进行扩容的情况下，容量也是 slice 可以扩张的最大限度。

### 创建

slice 的创建方式一共有以下几种：

#### 直接声明

```go
var slice []int
```

这是一个 nil slice，和 nil 比较结果为 true，元素指针、长度和容量都是 0。

#### make

```go
slice := make([]int ,10)
// 或
slice := make([]int, 10, 20)
```

make 函数创建 slice 时需要指定三个参数：

-   切片类型
-   长度；
-   容量：可以不传，默认和长度一致。

#### 截取

基于已有的数组或切片截取生成新的切片，又称为 reslice，新 slice 和 老 slice 共用底层数组，任意一方更改数据都会影响到另一方。

```go
slice := []int{1, 2, 3, 4}
newSlice := slice[0:2] // {1, 2}
```



## nil slice 和 empty slice

empty slice 的创建示例如下：

```go
slice := []int{}
slice := make([]int, 0)
```

nil slice 和 empty slice 的长度和容量都是 0，但两者的元素指针不一样，nil slice 的元素指针也是空，而 empty slice 会指向固定的内存地址：0xc42003bda0。

**两者都可以直接 append**，因为会调用 mallocgc 向 go 的内存管理器申请到一块内存，赋给新的 slice。