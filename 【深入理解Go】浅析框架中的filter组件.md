filter（也称middleware）是我们平时业务中用的非常广泛的框架组件，很多web框架、微服务框架都有集成。通常在一些请求的前后，我们会把比较通用的逻辑都会放到filter组件来实现。如打请求日志、耗时、权限、接口限流等通用逻辑。那么接下来我会和你一起实现一个filter组件，同时让你了解到，它是如何从0到1搭建起来的，具体在演进过程中遇到了哪些问题，是如何解决的。

## 从一个简单的server说起
我们看这样一段代码。首先我们在服务端开启了一个http server，配置了/这个路由，hello函数处理这个路由的请求，并往body中写入hello字符串响应给客户端。我们通过访问127.0.0.1:8080就可以看到响应结果。具体的实现如下：
```go
// 模拟业务代码
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

### 打印请求耗时v1.0
接下来有一个需求，需要打印这个请求执行的时间，这个也是我们业务中比较常见的场景。我们可能会这样实现，在hello这个handler方法中加入时间计算逻辑，主函数不变：
```go
// 模拟业务代码
func hello(wr http.ResponseWriter, r *http.Request) {
    // 增加计算执行时间逻辑
    start := time.Now()
    wr.Write([]byte("hello"))
    timeElapsed := time.Since(start)
    // 打印请求耗时
    fmt.Println(timeElapsed)
}

func main() {
    http.HandleFunc("/", hello)
    if err := http.ListenAndServe(":8080", nil); err != nil {
        panic(err)
    }
}
```
但是这样实现仍然有一定问题。假设我们有一万个请求路径定义、所以有一万个handler和它对应，我们在这一万个handler中，如果都要加上请求执行时间的计算，那必然代价是相当大的。

为了提升代码复用率，我们使用filter组件来解决此类问题。大多数web框架或微服务框架都提供了这个组件，在有些框架中也叫做middleware。

## filter登场
filter的基本思路，是把功能性（业务代码）与非功能性（非业务代码）分离，保证对业务代码无侵入，同时提高代码复用性。在讲解2.0的需求实现之前，我们先回顾一下1.0中比较重要的函数调用http.HandleFunc("/", hello)

这个函数会接收一个路由规则pattern，以及这个路由对应的处理函数handler。我们一般的业务逻辑都会写在handler里面，在这里就是hello函数。我们接下来看一下http.HandleFunc()函数的详细定义：
```go
func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) 
```
这里要注意一下，标准库中又把func(ResponseWriter, \*Request)这个func重新定义成一个类型别名HandlerFunc：
```go
type HandlerFunc func(ResponseWriter, *Request)
```
所以我们一开始用的http.HandleFunc()函数定义，可以直接简化成这样：
```go
func HandleFunc(pattern string, handler HandlerFunc) 
```
我们只要把「HandlerFunc类型」与「HandleFunc函数」区分开就可以一目了然了。因为hello这个用户函数也符合HandlerFunc这个类型的定义，所以自然可以直接传给http.HandlerFunc函数。而HandlerFunc类型其实是Handler接口的一个实现，Handler接口的实现如下，它只有ServeHTTP这一个方法：
```go
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```
HandlerFunc就是标准库中提供的默认的Handler接口实现，所以它要实现ServeHTTP方法。它在ServeHTTP中只做了一件事，那就是调用用户传入的handler，执行具体的业务逻辑，在我们这里就是执行hello()，打印字符串，整个请求响应流程结束
```go
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
	f(w, r)
}
```

### 打印请求耗时v2.0
所以我们能想到的比较容易的办法，就是把传入的用户业务函数hello在外面包一层，而非在hello里面去加打印时间的代码。我们可以单独定义一个timeFilter函数，他接收一个参数f，也是http.HandlerFunc类型，然后在我们传入的f前后加上的time.Now、time.Since代码。

这里注意，timeFilter最终返回值也是一个http.HandlerFunc函数类型，因为毕竟最终还是要传给http.HandleFunc函数的，所以filter必须也要返回这个类型，这样就可以实现最终业务代码与非业务代码分离的同时，实现打印请求时间。详细实现如下：
```go
// 打印请求时间filter，和具体的业务逻辑hello解耦
func timeFilter(f http.HandlerFunc) http.HandlerFunc {
    return func(wr http.ResponseWriter, r *http.Request) {
        start := time.Now()
        // 这里就是上面我们看过HandlerFun类型中ServeHTTP的默认实现，会直接调用f()执行业务逻辑，这里就是我们的hello，最终会打印出字符串
        f.ServeHTTP(wr, r)
        timeElapsed := time.Since(start)
        // 打印请求耗时
        fmt.Println(timeElapsed)
    }
}

func hello(wr http.ResponseWriter, r *http.Request) {
    wr.Write([]byte("hello\n"))
}

func main() {
    // 在hello的外面包上一层timeFilter
    http.HandleFunc("/", timeFilter(hello))
    if err := http.ListenAndServe(":8080", nil); err != nil {
        panic(err)
    }
}
```
然而这样还是有两个问题：
 - 如果有十万个路由，那我要在这十万个路由上，每个都去加上相同的包裹代码吗？
 - 如果有十万个filter，那我们要包裹十万层吗，代码可读性会非常差

目前的实现很可能造成以下后果：
```go
http.HandleFunc("/123", filter3(filter2(filter1(hello))))
http.HandleFunc("/456", filter3(filter2(filter1(hello))))
http.HandleFunc("/789", filter3(filter2(filter1(hello))))
http.HandleFunc("/135", filter3(filter2(filter1(hello))))
...
```
那么如何更优雅的去管理filter与路由之间的关系，能够让filter3(filter2(filter1(hello)))只写一次就能作用到所有路由上呢？

### 打印请求耗时v3.0
我们可以想到，我们先把filter的定义抽出来单独定义为Filter类型，然后可以定义一个结构体Frame，里面的filters字段用来专门管理所有的filter。这里可以从main函数看起。我们添加了timeFilter、路由、最终开启服务，大体上和1.0版本的流程是一样的：
```go
// Filter类型定义
type Filter func(f http.HandlerFunc) http.HandlerFunc

type Frame struct {
    // 存储所有注册的过滤器
    filters []Filter
}

// AddFilter 注册filter
func  (r *Frame) AddFilter(filter Filter) {
    r.filters = append(r.filters, filter)
}

// AddRoute 注册路由，并把handler按filter添加顺序包起来。这里采用了递归实现比较好理解，后面会讲迭代实现
func (r *Frame) AddRoute(pattern string, f http.HandlerFunc) {
    r.process(pattern, f, len(r.filters) - 1)
}

func (r *Frame) process(pattern string, f http.HandlerFunc, index int) {
    if index == -1 {
        http.HandleFunc(pattern, f)
        return
    }
    fWrap := r.filters[index](f)
    index--
    r.process(pattern, fWrap, index)
}

// Start 框架启动
func (r *Frame) Start() {
    if err := http.ListenAndServe(":8080", nil); err != nil {
        panic(err)
    }
}

func main() {
    r := &Frame{}
    r.AddFilter(timeFilter)
    r.AddFilter(logFilter)
    r.AddRoute("/", hello)
    r.Start()
}
```
r.AddRoute之前都很好理解，初始化主结构，并把我们定义好的filter放到主结构中的切片统一管理。接下来AddRoute这里是核心逻辑，接下来我们详细讲解一下

#### AddRoute
r.AddRoute("/", hello) 其实和v1.0里的 http.HandleFunc("/", hello) 其实一摸一样，只不过内部增加了filter的逻辑。在r.AddRoute内部会调用process函数，我将参数全部替换成具体的值：
```go
r.process("/", hello, 1)
```
那么在process内部，首先index不等于-1，往下执行到
```go
fWrap := r.filters[index](f)
```
他的含义就是，取出第index个filter，当前是r.filters[1]，r.filters[1]就是我们的logFilter，logFilter接收一个f（这里就是hello），logFilter里的f.ServerHTTP可以直接看成执行f()，即hello，相当于直接用hello里的逻辑替换掉了logFilter里的f.ServerHTTP这一行，在下图里用箭头表示。最后将logFilter的返回值赋值给fWrap，将包裹后的fWrap继续往下递归，index--：
![](https://baiyan-1300428464.cos.ap-beijing.myqcloud.com/article/2021/9/26/1632587977825.png)
同理，接下来的递归参数为：
```go
r.process("/", hello, 0)
```
这里就轮到r.filters[0]了，即timeFilter，过程同上：
![](https://baiyan-1300428464.cos.ap-beijing.myqcloud.com/article/2021/9/26/1632588193825.png)
最后一轮递归，index = -1，即所有filter都处理完了，我们就可以最终和v1.0一样，调用http.HandleFunc(pattern, f)将最终我们层层包裹后的f，最终注册上去，整个流程结束：
![](https://baiyan-1300428464.cos.ap-beijing.myqcloud.com/article/2021/9/26/1632588286464.png)

AddRoute的递归版本相对容易理解，我也同样用迭代实现了一个版本。每次循环会在本层filter将f包裹后重新赋值给f，这样就可以将之前包裹后的f沿用到下一轮迭代，基于上一轮的f继续包裹剩余的filter。在gin框架中就用了迭代这种方式来实现：
```go
// AddRouteIter AddRoute的迭代实现
func (r *Frame) AddRouteIter(pattern string, f http.HandlerFunc) {
    filtersLen := len(r.filters)
    for i := filtersLen; i >= 0; i-- {
        f = r.filters[i](f)
    }
    http.HandleFunc(pattern, f)
}
```
这种filter的实现也叫做洋葱模式，最里层是我们的业务逻辑helllo，然后外面是logFilter、在外面是timeFilter，很像这个洋葱，相信到这里你已经可以体会到了：
![](https://baiyan-1300428464.cos.ap-beijing.myqcloud.com/article/2021/9/26/1632588906978.png)

## 小结
我们从最开始1.0版本业务逻辑和非业务逻辑耦合严重，到2.0版本引入filter但实现仍不优雅，到3.0版本解决2.0版本的遗留问题，最终实现了一个简易的filter管理框架