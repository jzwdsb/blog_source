---
title: golang 中的 defer, panic 以及 recover
category: Golang
date: 2019-01-15 17:00
---

# defer

## 语义

`defer` 关键字表示该语句的函数调用发生在当前的函数体结束时，具体将一个函数调用压入到一个 list 中。
`defer` 通常用语简化函数返回时的清理动作。

示例，以下代码将读取源文件并且拷贝到另一个文件。

```golang
func CopyFile(dstName, srcName string) (written int64, err error) {
    src, err := os.Open(srcName)
    if err != nil {
        return
    }

    dst, err := os.Create(dstName)
    if err != nil {
        return
    }

    written, err = io.Copy(dst, src)
    dst.Close()
    src.Close()
    return
```

事实上这段代码有潜在的 bug, 当 `os.Create()` 调用失败时，这个函数会返回但是 `SouceFile` 没有关闭。可以使用另外的手段使得这段代码是安全的，但是使用 `defer` 语句是最安全和最方便的。

```golang
func CopyFile(dstName, srcName string) (written int64, err error) {
    src, err := os.Open(srcName)
    if err != nil {
        return
    }
    defer src.Close()

    dst, err := os.Create(dstName)
    if err != nil {
        return
    }
    defer dst.Close()

    return io.Copy(dst, src)
```

## defer 注意事项

### 参数的传值

一个 defer 的 function call 的参数的确定是在 defer 语句中确定的。
示例

```golang
func a() {
    defer fmt.Println(i)
    i++
    return
}
```

以上这段代码会打印 0, `fmt.Println` 的参数在 defer 时就已确定

### 多个 defer 的执行顺序

当一个函数体内有多个 `defer` 语句时，顺序为 LIFO(Last In First Out), 即声明顺序的逆序。

```golang
func b() {
    for i := 0; i < 4; i++ {
        defer fmt.Print(i)
    }
}
// prints 3 2 1 0
```

### defer 函数调用可能会影响函数返回值

当一个函数返回时，如果此时调用的 defer function call 中改变了返回值，那么该函数的返回值则会受到影响。

```golang
func c() (i int) {
    defer func() {i++}()
    return 1
}
```

上面这段代码返回值为 2, 其执行过程如下

- defer statment，function call push into list
- assign i to 1
- defer function calls, i = i + 1, i == 2
- function returns

由此可知 golang 中函数返回分为三步

- 返回值赋值
- defer 函数调用
- 函数返回

这个特性通常用于更改函数返回时的错误值。

# panic

`panic` 是内置函数，可以终止正常的控制流从而开始 `panicking`, 类似于 C++, Python 中的异常。
如果没有进行处理会导致当前整个 `goroutine` 的崩溃，进而导致整个程序的崩溃。
`panic` 可以显式调用，也可以由运行时错误产生，例如越界错误。

# recover

`recover` 是 go 的内置函数，可以重获 `panicking gorountine` 的控制权。`recover` 只有在 `defered function` 中有用。
在正常的执行流中，`recover` 会返回 `nil` 并且无任何副作用，如果当前的 `gorountine` 正在 `panicking`, 那么 `recover` 的调用会捕获 `panic` 的调用值，并且恢复正常的执行。