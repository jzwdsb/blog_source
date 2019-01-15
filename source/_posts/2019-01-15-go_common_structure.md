---
title: golang 中常见的内置类型
category: golang
date: 2019-01-19 15:00
mathjax: true
---

golang 中常用的几种内置类型如下

- string
- Slice
- Channel
- map
- Interface {}

## string 

golang 中的 string 为值类型，与 python 的 str 大致相同，赋值后无法再修改 string 中的内容，但可以通过方法函数构造新的 string

## Slice

在 golang 中的数组类型是值类型，传参时会复制整个数组，但是 Slice 是对底层数组的一段内容的引用。

```golang
data := [...] int {1, 2, 3, 4 ,5 ,6 ,7}
sli := data[1:4:5]
```

`data[1:4:5]` 中三个参数分别代表 data 中的 low, high, max
得到的 slice 的即为 `{2, 3, 4}`, 它的 $len = high - low$, $cap = max - low$

## Channel

channel 用来在多个 rountine 之间无阻塞的发送消息

```golang
ch := make(chan int, 10)
v := 10
ch <- v
v = <- ch
close(ch)
```

channel 支持三个操作

- read

  ```golang
  v := <- ch
  ```

- write

  ```golang
  ch <- v
  ```

- close

  ```golang
  close(ch)
  ```

channel 是并发安全的

## map

```golang
// nil map, can't use
var m map[string]int
// init with make
m = make(map[string]int)

m := make(map[string]int)
```

map 实现了键值表，能够实现 $O(1)$ 时间的查询。

map 支持如下几种操作

- insert or upadte

  ```golang
  m[key] = element
  ```

- retrieve an element

  ```golang
  ele = m[key]
  ```

- delete

  ```golang
  delete(m, key)
  ```

- test key

  ```golang
  ele, ok := m[key]
  ```

## Interface

抽象类型，没有具体值，唯一确定的是他包含某种方法

```golang
type geometry interface {
    area() float64
    perim() float64
}
```

在 golang 中实现该接口，只需要在定义类型时实现该接口中的同名方法即可。

```golang
type rect struct {
    width, height float64
}

func (r rect) area() float64 {
    return r.width * r.height
}

func (r rect) perim() float64 {
    return 2 * r.width + 2 * r.height
}

func measure(g geometry) {
    fmt.Println(g)
    fmt.Print(g.area())
    fmt.Print(g.perim())
}
```
