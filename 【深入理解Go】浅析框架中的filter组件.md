## 从一个简单的server说起
我们看这样一段代码。首先我们开启了一个http服务，通过访问127.0.0.1:8080，hello函数处理请求，输出hello字符串。实现如下：
```go
func hello(wr http.ResponseWriter, r *http.Request) {
    wr.Write([]byte("hello"))
}

func main() {
    http.HandleFunc("/", hello)
    if err := http.ListenAndServe(":8080", nil); err != nil {
        panic(err)
    }
}
```
接下来有一个需求，需要打印这个请求执行的时间，这个也是我们业务中比较常见的场景。我们可能会这样实现：
```go
func hello(wr http.ResponseWriter, r *http.Request) {
    // 增加计算执行时间逻辑
    start := time.Now()
    wr.Write([]byte("hello"))
    timeElapsed := time.Since(start)
    fmt.Println(timeElapsed)
}

func main() {
    http.HandleFunc("/", hello)
    if err := http.ListenAndServe(":8080", nil); err != nil {
        panic(err)
    }
}
```
但是这样实现仍然有一定问题。假设我们有一万个请求路径定义、所以有一万个handler和它对应，我们在这一万个handler如果都要加上请求执行时间的计算

## middleware（filter）代码复用