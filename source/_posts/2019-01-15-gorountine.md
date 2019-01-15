---
title: golang 中的 go 与 channel
date: 2019
category: golang
---

golang 与其他语言最大不同在于其对并发的支持，由 gorountine 实现，`go` 关键字声明一个函数放在 gorountie(协程) 中执行，多个 routine 之间通过 channel 实现非阻塞调用。

## http 的 Multi get 的一个简单实现

```golang
type RemoteResult struct {
    Url string
    Result string
}

func RemoteGet(requestUrl string, resultChan chan RemoteResult)  {
    request := httplib.NewBeegoRequest(requestUrl, "GET")
    request.SetTimeout(2 * time.Second, 5 * time.Second)
    //request.String()
    content, err := request.String()
    if err != nil {
        content = "" + err.Error()
    }
    resultChan <- RemoteResult{Url:requestUrl, Result:content}
}
func MultiGet(urls []string) []RemoteResult {
    fmt.Println(time.Now())
    resultChan := make(chan RemoteResult, len(urls))
    defer close(resultChan)
    var result []RemoteResult
    //fmt.Println(result)
    for _, url := range urls {
        go RemoteGet(url, resultChan)
    }
    for i:= 0; i < len(urls); i++ {
        res := <-resultChan
        result = append(result, res)
    }
    fmt.Println(time.Now())
    return result
}

func main() {
    urls := []string{
        "http://127.0.0.1/test.php?i=13",
        "http://127.0.0.1/test.php?i=14",
        "http://127.0.0.1/test.php?i=15",
        "http://127.0.0.1/test.php?i=16",
        "http://127.0.0.1/test.php?i=17",
        "http://127.0.0.1/test.php?i=18",
        "http://127.0.0.1/test.php?i=19",
        "http://127.0.0.1/test.php?i=20"
    }
    content := MultiGet(urls)
    fmt.Println(content)
}
```

## 分析

给出多个 url 列表，使用 goroutine 实现并发获取网页内容。

获取 url content 方法为 `Remote Get`, 在 `MultiGet` 中使用 `go` 关键字调用，每当调用一次就会产生一个新的 goroutine, 代码运行总的时间小于其串行时间，最后的 master 从 `resultchan` 通道中获取数据，最后打印。
