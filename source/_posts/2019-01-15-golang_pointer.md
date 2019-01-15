---
title: golang 中的指针
category: golang
date: 2019-01-15 10:00
---

golang 中的指针使用和声明方法与 C/C++ 大致相同。
示例

```golang
var p *int
i := 42
p = &i
*p = 43
*p += 1
```

与 C/C++ 不同的是 golang 中的指针没有指针运算。

当指针指向一个结构体时，对其成员的引用无需显式写出 `*p`

```golang
type Point struct {
    x int
    y int
}

func main() {
    v := Point{1, 2}
    var p *Point
    p = &v
    p.x = 2
    p.y = 3
}
```
